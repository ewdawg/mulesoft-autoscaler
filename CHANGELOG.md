# Changelog

## 1.3.0 — The alert source does not exist, and the contract is now ours

This release resolves the question that has been open since 1.0.0 — the inbound webhook contract —
but not by measuring it. By discovering there was nothing to measure.

### The finding

**No Anypoint alerting system can call a webhook, at any subscription tier, including Titanium.**
All three deliver by email only:

| System | On trigger |
|---|---|
| Anypoint Monitoring, custom dashboard alerts *(Titanium)* | "trigger email notifications" |
| Runtime Manager alerts | Email; on CloudHub 2.0, deployment success/failure only, not metrics |
| API Manager alerts | "select Business Group users to receive email notifications" |

Established from four independent directions: MuleSoft's rendered docs; the docs *source*, where
non-email channels appear only inside a commented-out block headed
`//TODO: VERIFY THAT ALL THESE FEATURES ARE ACTUALLY IMPLEMENTED`; the behaviour of a non-Titanium
org, where the Monitoring Alerts page redirects away to dashboards; and direct observation in the
product UI.

This invalidates the premise carried since 1.0.0. The README described Monitoring alerts with
webhook actions as "the one capability everything here depends on" and told readers to confirm it
was available at their tier. It is available at no tier. Buying Titanium would deliver native
autoscaling; it would never have made this design's input path work.

The org's entitlements also confirm the project's underlying rationale is sound — `autoscaling:
false` — so the problem this app solves is real even though the trigger it assumed was not.

### Why this closes the gap rather than opening a worse one

The app was never coupled to Anypoint Monitoring; it needs *something* to POST a value at it. With
the caller now explicitly yours, **the inbound contract is this project's own** — defined rather
than reverse-engineered. The risk carried through three releases, that the assumed payload shape
was wrong, is gone: there is no vendor shape to be wrong about.

It also promotes what the README treated as a footnote. Scaling on queue depth, orders per minute,
or in-flight requests — signals native autoscaling cannot see — is now the main event rather than a
bonus.

### Added

- **`examples/autoscaler-emitter.xml`** — a reference alert source: a drop-in flow for the
  *monitored* application that counts its own requests and reports rate to the autoscaler. Needs no
  Titanium and no change to the autoscaler.

  It measures request rate rather than CPU or heap because a Mule app cannot portably observe its
  own machine metrics, and both obvious routes were tried and rejected on evidence:
  `java.lang.Runtime` through DataWeave's `java!` interop returns null, since DataWeave's dot
  notation does JavaBean getter access rather than method invocation; and
  `ManagementFactory::getMemoryMXBean().heapMemoryUsage` resolves the getter but then fails on the
  Java 17 module system, because `java.management` does not export `sun.management` and DataWeave
  reflects on the implementation class. Request rate needs no reflection and survives runtime
  upgrades. Its DataWeave was verified by execution on 4.11.6 and 4.12.2.

  The flow swallows all errors: a monitoring sidecar that can take down the application it monitors
  is worse than no monitoring.

- **Diagnostic capture path**, off by default (`capture.enabled=false`):
  - `capture-request` records one raw request — body, method, path, and **every header** — without
    interpreting it, so a form-encoded, XML or malformed body survives instead of being lost to a
    parse error.
  - `autoscale-capture-flow`, an endpoint on `capture.path` that **checks no shared secret**, since
    an endpoint that only captures correctly authenticated requests cannot help you debug
    authentication. It is protected by a random segment in `capture.path`, and always answers `200`,
    including when capture itself fails.
  - `autoscale-captures-read-flow` (`GET /autoscale/captures`), authenticated, because it hands data
    out rather than taking it in.
  - A capture hook in `normalize-alert`, after the secret check.
- `Alert_Captures` object store: persistent, capped at 25 entries with a 7-day TTL, because it holds
  unvalidated request bodies from an endpoint that is deliberately unauthenticated.
- Properties `capture.enabled` and `capture.path`.
- Four tests, taking the suite to 24, including verbatim capture of a non-JSON body.

### Verified on the live CloudHub 2.0 deployment

| Behaviour | Result |
|---|---|
| JSON body to `capture.path` | 200 `captured` |
| Form-encoded body | 200, stored verbatim |
| Guessable path without the random segment | 404 |
| `GET /autoscale/captures` without secret | 401, nothing recorded |
| `GET /autoscale/captures` with secret | 200, raw bodies and all headers |
| Custom header through CloudHub ingress | Survives unmodified |
| Scaling path, rotated secret | 1 → 2 replicas |
| Decay | Reclaimed 2 → 1 unprompted |

CloudHub's EDGE ingress passes custom headers through untouched, so a caller that can set a header
will have it arrive. An unauthorized request is never captured, so the store cannot be filled
through `/autoscale/webhook`.

### Documentation

- README rewritten around the finding: a new **Where alerts come from** section carrying the
  evidence, the setup section replaced with emitter instructions, the endpoint contract restated as
  this project's own with a canonical body, and the opening corrected — it previously described the
  trigger as an Anypoint Monitoring alert.
- **Log retrieval is subscription-gated**, which bites in exactly the tier this project targets.
  Tools routing through the Monitoring log-search API — including the MuleSoft MCP server's — fail
  with `Required monitoringCenter subscription ... Current value: 3`. The CloudHub 2.0 deployment
  log API is not gated and the CLI reaches it. Prefer `download-logs` over `logs`: the latter tails
  and then crashes on an upstream bug in `anypoint-cli-ch1-plugin`
  (`Cannot read properties of undefined (reading 'timestamp')`).
- Known limitations now lead with the fact that Anypoint cannot drive this app.

### Security

- The webhook shared secret was rotated during this release's deployment.

---

## 1.2.0 — Deployed and validated end to end on CloudHub 2.0

The application has now actually been run. Until this release everything was verified either by
unit tests with mocked HTTP, or by calling the platform API directly — the app itself had never
been deployed, its listener had never served a request, and its scheduler had never fired.

### Validated on a live deployment

Deployed to a CloudHub 2.0 sandbox and driven by POSTing alerts at its public URL:

| Behaviour | Result |
|---|---|
| No / wrong `X-Autoscaler-Secret` | 401 `unauthorized` |
| CPU 92 with valid secret | 200 `scaled`, target application went **1 → 2 replicas** |
| Immediate repeat alert | `skipped` — cooldown suppressed it |
| CPU 55, between thresholds | `skipped`, `direction: NONE`, no platform API calls made |
| Alert with no deployment id | 400 `bad_request` |
| Left idle past `decay.idle.seconds` | **decay fired unprompted, target reclaimed 2 → 1** |

The decay result resolves the largest open risk in 1.1.0. That release made both object stores
persistent, which on CloudHub 2.0 means **Object Store v2** — a network-backed service whose
support for `os:retrieve-all-keys` was unverified. The whole decay sweep depends on that call, so
had it been unsupported, decay would have silently done nothing: precisely the failure it was added
to prevent. It works.

### Changed

- `mule-artifact.json` now declares `minMuleVersion: 4.9.0` (was 4.11.0), so the app can deploy on
  an **LTS** channel runtime. EDGE-channel deploys were rejected by this org, and the one existing
  EDGE application in it is not running, so LTS is the safer default.

### Documentation

- README now carries the deployment procedure that actually works, and says plainly which
  documented approach does *not*: the `mule-maven-plugin` `cloudhub2Deployment` block does not
  publish the artifact to Exchange, so the deploy fails with "Failed to retrieve artifact
  information from Exchange". Publishing separately with `deploy:deploy-file` also failed. The
  working path is `anypoint-cli-v4 exchange asset upload` followed by
  `runtime-mgr application deploy`, with two non-obvious details recorded: the Exchange file key
  must be exactly `mule-application.jar`, and positional arguments must precede the variadic
  `--property` flags or they are swallowed.
- No `cloudhub2Deployment` block is committed, since it does not work; carrying broken
  configuration would be worse than none.

### Known gap

Tests run on Mule 4.11.6 while the app deploys on 4.9.20, because MUnit 3.7 cannot create an
embedded container for 4.9.20. The app uses no feature newer than 4.9, but the runtimes differ.

### Still open

The **inbound webhook contract**. Every alert used above was synthetic, hand-built to match the
assumed shape. No real Anypoint Monitoring alert has been captured.

---

## 1.1.1 — Platform API contract verified against a live environment

Everything this app sends to Anypoint has now been exercised against a real CloudHub 2.0 sandbox
rather than against mocks alone. Three assumptions carried since 1.0.0 are resolved.

### Verified

- `GET /amc/application-manager/api/v2/organizations/{org}/environments/{env}/deployments/{id}`
  returns **200**. The URL `anypoint-get-deployment` builds is correct.
- Replica count lives at **`target.replicas`**, as `apply-scaling` assumes.
- A **partial** `PATCH` body of `{"target":{"replicas":n}}` is accepted and takes effect: a running
  application was scaled 1 → 2, the second replica started, and it was then restored to 1. This was
  the last open question about the write path.
- The token endpoint and `client_credentials` form body used by `anypoint-get-token` work as written.
- `application.vCores` is independent of `target.replicas`, confirming that a replica change cannot
  resize an application.

### Corrected

The 1.0.0 entry claimed the original code's `payload.replicas` was absent, so its `default 1` always
applied and the app scaled to exactly 2. **That was wrong**, and the live response shows why:
`payload.replicas` *does* exist at the top level, but it holds an **array of live replica
instances**, not a count:

```json
"replicas": [{"id":"...-55d54f8bd7-h5gzw","state":"STARTED", ...}]
```

`default` therefore never fired. `(current as Number) + 1` would have attempted to coerce a `List`
to a `Number` and thrown at runtime. The original would not have mis-scaled — it would have failed
outright on its first real alert.

The same response also vindicates sending a minimal `PATCH`: `target.deploymentSettings` carries a
large nested structure (sidecars, HTTP endpoints, JVM args, runtime channel). Reconstructing a full
body, as the superseded `autoscaler_updated.xml` did, would have had to reproduce all of it
faithfully or silently drop parts of it.

### Still open

The **inbound webhook contract** remains unverified — no live Anypoint Monitoring alert has been
captured. Everything the app *sends* is now confirmed; what Anypoint *sends the app* is not.

---

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

- Replica count is read from `target.replicas`. The original read `payload.replicas`.
  (See the 1.1.1 note below — the effect of that bug was worse than described here.)
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
