# Changelog

## 1.1.0 — Decay safety net and durable state

Context: this project emulates CloudHub 2.0 autoscaling, which is gated behind the **Titanium**
subscription tier. Below that tier the alternatives are permanent over-provisioning or a licence
upgrade, so the value of this app is entirely in reclaiming replicas that would otherwise be held
at peak allocation around the clock. That makes a stranded scale-up the failure mode that costs
real money, and 1.0.0 had no defence against it.

### Added

- **Decay flow** (`autoscale-decay-flow`, `decay-one-deployment`). A scheduled sweep removes one
  step from any tracked deployment idle beyond `decay.idle.seconds`, repeatedly, until it is back at
  `scale.min.replicas`. It converges on **elapsed time rather than load**, deliberately: without the
  top tier there is no metrics API to poll, and time is a signal always available.

  This closes the gap that a purely edge-triggered design leaves open — if a scale-down alert is
  never configured, is dropped, or the webhook path breaks after a scale-up, replicas previously
  stayed elevated indefinitely. Decay fails safe: a total failure of the alert path now drifts
  deployments down to minimum rather than stranding them at maximum.

  Guarded so it cannot fight a live workload: any scaling action resets the idle clock, and an open
  cooldown suppresses the sweep for that deployment. Per-deployment failures are isolated so one bad
  entry cannot halt the sweep.
- `Managed_Deployments` object store recording which deployments this app has scaled, and when it
  last acted, so decay can act without needing another alert. Entries are removed once a deployment
  is back at the minimum.
- Properties: `decay.enabled`, `decay.interval.seconds`, `decay.start.delay.seconds`,
  `decay.idle.seconds`.
- Five tests covering decay on idle, decay skipped when recently active, decay suppressed by an open
  cooldown, tracking removed at the minimum, and registration on scale-up. Suite is now 20 tests.

### Changed

- **Both object stores are now persistent.** `Cooldown_Store` was in-memory, so a restart silently
  bypassed the debounce. This app has no way to observe load directly and so cannot reconstruct that
  state after a restart — losing it was a correctness problem, not just an inconvenience.

### Fixed

- **A fresh clone could not run the tests.** `src/main/resources/config.properties` is gitignored,
  and MUnit needs those values to exist, so `mvn test` failed on any machine without a local copy —
  including CI. Added a committed `src/test/resources/config.properties` with fake values, which
  also takes classpath precedence during tests and so makes the suite deterministic regardless of
  local configuration.

### Documentation

- README now states plainly that this emulates a Titanium-gated feature, explains the tier
  reasoning, and says when *not* to use it (if you have Titanium, use the native feature).
- Added the decay rationale, tuning guidance, and the API-shape assumptions to known limitations.

---

## 1.0.0 — Rebuild as a working Mule 4 application

The project previously consisted of two loose XML files at the repository root with no build
descriptor. It could not be compiled, packaged, tested or deployed, and both files mixed Mule 3 and
Mule 4 syntax in ways that would not have loaded on either runtime. This release rebuilds it as a
real Mule 4 application against the same intent.

**This is a breaking change to every external contract.** See [Migration](#migration) below.

### Added

- `pom.xml` and `mule-artifact.json` — the project now builds (`mvn clean package`) and produces a
  deployable artifact. Targets Mule 4.11, MUnit 3.7.
- **Shared-secret authentication.** `X-Autoscaler-Secret` is checked before any other work. The
  endpoint was previously unauthenticated, so anyone who could reach it could change replica counts.
- **Cooldown window.** An ObjectStore entry keyed by deployment id, expiring after
  `scale.cooldown.seconds`, prevents a repeatedly firing alert from ratcheting replicas upward.
- **Scale-down.** A metric at or below `scale.down.threshold` removes a replica. Previously replicas
  only ever increased, which defeated the project's own cost-control rationale.
- **Replica bounds.** `scale.min.replicas` / `scale.max.replicas` clamp every decision.
- **Error handling.** Custom `AUTOSCALER:UNAUTHORIZED` and `AUTOSCALER:BAD_REQUEST` error types
  mapped onto 401 and 400; unexpected failures return 500 with a detail message.
- 15 MUnit tests covering both clamps, both scaling directions, cooldown suppression, authentication
  rejection, alert normalization, and four end-to-end paths through the webhook flow.
- Full documentation in `README.md`: endpoint contract, Anypoint setup, flow reference,
  troubleshooting, and known limitations.

### Changed

- Consolidated `autoscaler.xml` and `autoscaler_updated.xml` — which disagreed on endpoint path,
  payload shape, target API, HTTP method, config filename and property names — into a single
  implementation split across `src/main/mule/global-config.xml` and `src/main/mule/autoscaler.xml`.
- Standardized on the **CloudHub 2.0 Application Manager API**
  (`/amc/application-manager/api/v2/…`). The original file called the Runtime Fabric API, which is a
  different deployment target, while the README described the two as the same thing.
- The scaling call is now a `PATCH` carrying only `target.replicas`. The previous implementations
  either hardcoded `cpu`/`memory` or reconstructed the entire deployment body, both of which could
  silently resize an application.
- The access token is now fetched only after the threshold and cooldown checks pass, rather than on
  every inbound alert.
- URL construction uses `uri-params` instead of string interpolation.
- Alert parsing is consolidated into a single `normalize-alert` sub-flow that tolerates the payload
  shapes both previous files assumed, so a contract correction touches one place.

### Fixed

- Replica count is read from `target.replicas`. The original read `payload.replicas`, so its
  `default 1` always applied and the app scaled to exactly 2 regardless of actual size.
- `min` / `max` are called with an array (`min([a, b])`). The previous two-argument form was not
  valid DataWeave.
- The OAuth request body is a DataWeave expression. It was previously literal text with embedded
  `#[…]`, which Mule 4 does not interpolate — the credentials would have been transmitted as the
  literal string `#[(p('client.id'))]`.
- `<error-handler>` elements nested inside `<http:request>` (invalid on any runtime) replaced with a
  flow-level handler.
- Removed Mule 3 constructs that do not exist in Mule 4: `<spring:beans>`
  `PropertyPlaceholderConfigurer`, `<set-property>`, `<dw:transform-message>`,
  `<http:request-builder>`, and flat `host`/`port` attributes on HTTP configs.
- MUnit mocks are keyed on `doc:name`. The previous suite matched on `#[attributes.method]`, which
  cannot discriminate mocks — MUnit matches a processor's configured attributes, not runtime message
  attributes — and registered three mocks against the same processor.
- `maxReplicas` is a number from configuration rather than the string literal `"5"`.

### Migration

If you were running either previous file:

| | Before | Now |
|---|---|---|
| Endpoint | `/autoscale` or `/autoscale/webhook` | `/autoscale/webhook` |
| Auth | none | `X-Autoscaler-Secret` header required |
| Config file | `config.properties` (root) or `mule-app.properties` | `src/main/resources/config.properties` |
| Credential keys | `client.id` / `client.secret` | `anypoint.client.id` / `anypoint.client.secret` |
| Threshold key | `scale.threshold` | `scale.up.threshold` + `scale.down.threshold` |
| Target ids | `org.id` / `env.id` / `app.id` | taken from the alert; `anypoint.org.id` / `anypoint.env.id` are fallbacks only |

Update your Anypoint Monitoring alert to send the shared-secret header, and add a second alert on
the low threshold if you want scale-down.

### Known open item

The exact Anypoint Monitoring webhook body has not been verified against a live alert. Field
mappings are tolerant of the shapes previously assumed by this project but should be confirmed
against a captured payload before production use. See the README for how to capture one.
