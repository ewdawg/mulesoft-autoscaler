# MuleSoft Auto-Scaler

A Mule 4 application that scales a **CloudHub 2.0** deployment up and down in response to load
signals reported over HTTP.

## Why this exists

CloudHub 2.0 has native autoscaling, but it is gated behind the **Titanium** subscription tier.
Everyone below that tier has the same problem — load varies, replicas do not — and only two
sanctioned answers: pay for the upgrade, or provision permanently for peak and leave the headroom
idle around the clock.

This app is the third answer: **an emulation of the Titanium autoscaling feature**, built out of the
parts available underneath it — the Application Manager API, which any tier can call, plus a signal
your own application reports about itself.

> **Read [Where alerts come from](#where-alerts-come-from) before adopting this.** Earlier versions
> of this README claimed the trigger was an Anypoint Monitoring alert with a webhook action. **No
> such capability exists, at any tier.** Every Anypoint alerting system delivers by email only. The
> caller has to be something you control; `examples/autoscaler-emitter.xml` is a working one.

The premise is that an application should track its own load as a matter of course, and that
holding replicas at peak allocation twenty-four hours a day to serve a few hours of traffic is a
poor default. On a fixed vCore contract the saving is headroom: replicas you are not holding are
vCores other applications can use.

**Scope note.** This does what native autoscaling does, not more. If you are on Titanium, use the
native feature — it is faster, supported, and does not consume vCores of its own to run.

Where it goes further is the signal it scales on. Because the caller is yours (see
[Where alerts come from](#where-alerts-come-from)), you can scale on queue depth, orders per minute,
in-flight requests, or anything else you can measure — signals native autoscaling cannot see.
`normalize-alert` never interprets the metric, only its value.

## How it works

Two control paths, deliberately different in kind.

**Edge-triggered — reacts to alerts:**

```
your alert source  (see examples/autoscaler-emitter.xml)
        │
        ▼
POST /autoscale/webhook          ← shared-secret header required
        │
        ├─ verify shared secret ──────────────────► 401 if absent or wrong
        ├─ normalize alert  ──────────────────────► 400 if no deployment id
        ├─ metric within band? ───────────────────► no action (no API calls made)
        ├─ cooldown window open? ─────────────────► suppressed
        │
        ├─ OAuth client_credentials → access token
        ├─ GET  deployment           → current replicas
        ├─ clamp(current ± step, min, max)
        │       └─ unchanged? ────────────────────► no action (already at bound)
        └─ PATCH deployment → new count, open cooldown, track for decay
```

**Level-triggered — converges on elapsed time:**

```
every decay.interval.seconds
        │
        └─ for each tracked deployment
              └─ idle > decay.idle.seconds, and not in cooldown?
                    └─ remove one step, until back at scale.min.replicas
```

Scaling is bidirectional: a metric at or above `scale.up.threshold` adds replicas, at or below
`scale.down.threshold` removes them, and anything between is left alone. The band between the two
thresholds prevents a metric hovering near a single value from oscillating the deployment.

### Why the decay flow exists

The alert path only acts when an alert arrives, which makes its expensive failure mode a silent
one. If a scale-down alert is never configured, is dropped, or the webhook path breaks after a
scale-up, the deployment stays elevated indefinitely and quietly bills for it — the exact outcome
this app is meant to prevent.

Decay is the safety net. It converges on **elapsed time rather than on load**, which matters
because without the top tier there is no metrics API to poll — time is a signal you can always get.
A deployment that has gone `decay.idle.seconds` without any scaling action loses one step,
repeatedly, until it is back at the minimum. Any alert in the meantime resets the clock, so a
genuinely busy application is never decayed.

It fails safe: if the entire webhook path breaks, deployments drift down to minimum rather than
sticking at maximum.

## Safety properties

Each is covered by the test suite:

| Guarantee | Mechanism | Test |
|---|---|---|
| Only authorized callers can scale | `X-Autoscaler-Secret` checked before any other work | `e2e-unauthenticated-call-returns-401`, `rejects-wrong-webhook-secret` |
| Replicas stay within bounds | `scale.min.replicas` / `scale.max.replicas` clamp applied every time | `clamps-at-max-replicas`, `clamps-at-min-replicas` |
| A flapping alert cannot ratchet replicas | Persistent cooldown keyed by deployment id | `e2e-cooldown-suppresses-repeat-alert`, `starts-cooldown-after-scaling` |
| Costs come back down on alert | Scale-down path on the low threshold | `scales-down-by-one-step` |
| Costs come back down even if alerts fail | Decay flow, on elapsed time | `decay-removes-a-replica-when-idle`, `decay-stops-tracking-at-minimum` |
| Decay never fights a live workload | Idle clock resets on every action; cooldown respected | `decay-skips-recently-active-deployment`, `decay-respects-an-open-cooldown` |
| Existing sizing is never clobbered | `PATCH` sends only `target.replicas`, nothing else | — |
| No wasted API calls | Token fetched only after threshold and cooldown checks pass | `e2e-value-within-band-takes-no-action` |
| Malformed alerts fail closed | Missing deployment id raises `AUTOSCALER:BAD_REQUEST` | `rejects-alert-without-deployment-id` |

## Endpoint contract

### Request

```
POST /autoscale/webhook
X-Autoscaler-Secret: <value of webhook.secret>
Content-Type: application/json
```

**This contract belongs to this project.** It is not a vendor format being matched — nothing MuleSoft
ships can call a webhook (see [Where alerts come from](#where-alerts-come-from)), so the caller is
always something you control and the shape below is simply what this app accepts.

The canonical body:

```json
{
  "alert": {
    "id": "emitter-20260902T0431",
    "metric": "requests-per-minute",
    "currentValue": 240,
    "resource": {
      "id": "1f63a0a8-b904-4954-8f8a-fdb544d19820",
      "name": "orders-api",
      "environment": { "id": "0cb6f230-c36b-4be6-82f9-15db9c466ee1" }
    },
    "organization": { "id": "b448e279-ea6b-4e79-9c52-ab9269073cb4" }
  }
}
```

Only two fields are load-bearing: **`resource.id`**, which must be the CloudHub 2.0 *deployment id*
of the app to scale, and **`currentValue`**, the number compared against the thresholds. Everything
else is labelling or has a fallback.

`normalize-alert` accepts several alternative spellings, kept because they cost nothing and make the
endpoint forgiving of hand-written callers:

| Internal field | Read from, in order |
|---|---|
| `alertId` | `alert.id`, `alert.alertId` |
| `applicationName` | `alert.resource.name`, `alert.appName`, `alert.applicationName` |
| `deploymentId` | `alert.resource.id`, `alert.deploymentId`, `alert.appId` |
| `organizationId` | `alert.organization.id`, `alert.resource.organizationId`, `anypoint.org.id` |
| `environmentId` | `alert.resource.environment.id`, `alert.environmentId`, `anypoint.env.id` |
| `metric` | `alert.metric` (default `"cpu"`) |
| `currentValue` | `alert.currentValue`, `alert.value` |

A body may also be nested under `event` instead of `alert`; both are accepted. `metric` is a label
only — the app never interprets it, which is what lets you scale on any signal you can measure.

### Responses

| Status | Body `status` | Meaning |
|---|---|---|
| `200` | `scaled` | Replica count changed. Body carries `previousReplicas` and `replicas`. |
| `200` | `skipped` | No action: within band, cooldown open, or already at a bound. |
| `400` | `bad_request` | Alert carried no deployment id. |
| `401` | `unauthorized` | Missing or invalid `X-Autoscaler-Secret`. |
| `500` | `error` | Unexpected failure. `detail` carries the description. |

Example success body:

```json
{
  "status": "scaled",
  "application": "orders-api",
  "direction": "UP",
  "previousReplicas": 2,
  "replicas": 3
}
```

### Diagnostic endpoints

Both are inert unless `capture.enabled` is `true`, in which case they answer as below. When it is
`false` they return `404` with `{"status":"disabled"}`. See
[Verifying your alert source](#3-verifying-your-alert-source).

| Endpoint | Auth | Behaviour |
|---|---|---|
| `POST ${capture.path}` | **none** — protected only by the random path segment | Records the raw request. Always `200` `{"status":"captured"}`, even if the capture fails, so an alert action is never disabled by an error response. |
| `GET /autoscale/captures` | `X-Autoscaler-Secret` | Returns `{"count":n,"captures":[…]}`, each entry carrying `receivedAt`, `method`, `requestPath`, `queryString`, `headers` and the raw `body`. |

## Requirements

- JDK 17 and Maven 3.9+
- An Anypoint Platform **Connected App** (`client_credentials`) able to read and modify deployments
  in the target environment
- A target application on **CloudHub 2.0**
- **Something to call the webhook.** Anypoint itself cannot — see
  [Where alerts come from](#where-alerts-come-from). `examples/autoscaler-emitter.xml` is a
  ready-made caller you drop into the application you want scaled.

## Configuration

```bash
cp src/main/resources/config.properties.example src/main/resources/config.properties
```

`src/main/resources/config.properties` is gitignored. **Do not commit real credentials.**

| Property | Meaning |
|---|---|
| `http.port` | Listener port |
| `webhook.secret` | Value the caller must send in `X-Autoscaler-Secret` |
| `anypoint.host` | Control-plane host (`anypoint.mulesoft.com`, or the EU equivalent) |
| `anypoint.client.id` / `anypoint.client.secret` | Connected App credentials |
| `scale.up.threshold` | Metric value at or above which replicas are added |
| `scale.down.threshold` | Metric value at or below which replicas are removed |
| `scale.min.replicas` / `scale.max.replicas` | Hard bounds, never exceeded |
| `scale.step` | Replicas added or removed per action |
| `scale.cooldown.seconds` | Minimum gap between actions for the same deployment |
| `decay.enabled` | Whether the safety-net sweep runs |
| `decay.interval.seconds` | How often the sweep runs |
| `decay.start.delay.seconds` | Delay before the first sweep after startup |
| `decay.idle.seconds` | Idle time before a deployment loses a step |
| `anypoint.org.id` / `anypoint.env.id` | Fallbacks used only if the alert omits them |
| `capture.enabled` | Whether the diagnostic capture path records raw requests. Off by default |
| `capture.path` | Path of the unauthenticated capture endpoint. Give it a long random segment |

Supply the two secrets at deploy time rather than in the file — any property can be overridden by a
system property of the same name:

```
-M-Danypoint.client.secret=...  -M-Dwebhook.secret=...
```

`mule-artifact.json` declares both as `secureProperties`, keeping them out of logs and diagnostics.

### Tuning

- Leave a **wide gap** between `scale.up.threshold` and `scale.down.threshold`. A narrow band with a
  short cooldown will scale up, immediately satisfy the low threshold, and scale back down.
- Keep `decay.idle.seconds` **well above** `scale.cooldown.seconds` so decay never races a fresh
  scale-up. The defaults (1800 vs 600) leave a wide margin.
- Decay is a backstop, not the primary scale-down path. If it is doing most of the work, your
  scale-down alert is probably misconfigured.

## Where alerts come from

Not from Anypoint. This is the most important thing to know about running this app, and earlier
versions of this README had it wrong.

**Anypoint has three alerting systems, and all of them deliver by email only — at every subscription
tier, including Titanium. None can call a webhook.**

| System | What it does when it fires |
|---|---|
| Anypoint Monitoring, custom dashboard alerts *(Titanium)* | *"trigger email notifications"* |
| Runtime Manager alerts | Email. On CloudHub 2.0, only on deployment success/failure — not on metrics |
| API Manager alerts | *"select Business Group users to receive email notifications"* |

The evidence, in case you are about to go looking yourself:

- MuleSoft's [monitoring docs](https://docs.mulesoft.com/monitoring/alerts) say alerts deliver
  notifications as emails, and the [CloudHub 2.0 alert docs](https://docs.mulesoft.com/cloudhub-2/ch2-config-app-alerts)
  document email recipients only.
- In the docs *source*, non-email destinations appear exactly once — commented out, under a literal
  `//TODO: VERIFY THAT ALL THESE FEATURES ARE ACTUALLY IMPLEMENTED`:

  ```
  * Channel Integrations: Ability to specify a channel (such as
    PagerDuty, SMS, or Slack) to which notifications are sent.
  ```

  MuleSoft's own writers could not confirm it shipped, so they hid it.
- On a non-Titanium org the Monitoring **Alerts** page redirects away to the dashboards view; per the
  docs source that page is Titanium-gated, and lower tiers get links out to API/Runtime alerts.
- Confirmed directly in the product UI: email is the only delivery channel offered.

### What this means

The premise "Titanium gates autoscaling, so emulate it with alert webhooks available lower down" does
not hold, because alert webhooks are not available lower down *or higher up*. Buying Titanium would
get you native autoscaling; it would never have made this design's input path work.

That sounds worse than it is. This app was never really coupled to Anypoint Monitoring — it needs
*something* to POST a value at it, and that something is now explicitly yours. Two consequences, both
good:

- **The inbound contract is this project's own**, so it is defined rather than guessed at. Earlier
  releases carried an open risk that the assumed payload shape was wrong; that risk is gone, because
  there is no vendor shape to be wrong about.
- **You can scale on anything you can measure.** Queue depth, orders per minute, in-flight requests
  — signals native CloudHub autoscaling cannot see. The README used to list this as a bonus. It is
  now the main event.

`examples/autoscaler-emitter.xml` is a working caller. [Setting up Anypoint](#2-the-alert-source)
covers wiring it in.

## Setting up Anypoint

### 1. Connected App

In **Access Management → Connected Apps**, create an app with the *App acts on its own behalf*
(`client_credentials`) grant. Scoped to the environment holding the target app, grant it:

- **Read Applications** — for the `GET`
- **Manage Applications** — for the `PATCH`

### 2. The alert source

There is nothing to configure in Anypoint here, because **Anypoint cannot call a webhook.** See
[Where alerts come from](#where-alerts-come-from) for the evidence. Instead, the application you want
scaled reports on itself.

Copy `examples/autoscaler-emitter.xml` into the **monitored** application's `src/main/mule/`, add one
line to each flow whose traffic should count:

```xml
<flow-ref name="autoscaler-count-request"/>
```

then configure it and redeploy that application:

```
autoscaler.host                  <this-app>.<region>.cloudhub.io
autoscaler.secret                <same value as this app's webhook.secret>
autoscaler.emit.interval.seconds 60
autoscaler.up.threshold          240
autoscaler.down.threshold        30
autoscaler.deployment.id         <the monitored app's CloudHub 2.0 deployment id>
autoscaler.application.name      orders-api
autoscaler.organization.id       <org id>
autoscaler.environment.id        <env id>
```

The emitter measures **requests per minute** and reports only when that is outside the band. Its full
rationale — including why it does not measure CPU or heap, and how to substitute your own signal —
is in the file's header comment.

A deployed CloudHub 2.0 app cannot discover its own deployment id, so `autoscaler.deployment.id` has
to be supplied. Find it with:

```bash
anypoint-cli-v4 runtime-mgr application list \
  --organization "$ORG_ID" --environment 'Sandbox' --output json
```

The emitter is a reference, not a requirement. Anything that can POST JSON works — a CI job, an
external uptime monitor, a load test, or your own code. The contract is in
[Endpoint contract](#endpoint-contract).

### 3. Verifying your alert source

When you point a new caller at this app and it does not behave, the first question is always *what
did it actually send?* The capture path answers that without log scraping — it records raw requests
verbatim, including every header, and hands them back over HTTP.

It was built to capture a real Anypoint Monitoring alert. That turned out to be impossible for the
reasons above, but the tool is if anything more useful now: the callers are yours, so they are the
ones that need debugging.

Deploy with capture on and a random path segment:

```
--property "capture.enabled:true"
--property "capture.path:/autoscale/capture/8f3aa1c9d24b"
```

Point your caller at `https://<this-app>.<region>.cloudhub.io/autoscale/capture/8f3aa1c9d24b`
instead of `/autoscale/webhook`, run it, then read back what arrived:

```bash
curl -s https://<this-app>.<region>.cloudhub.io/autoscale/captures \
  -H "X-Autoscaler-Secret: $WEBHOOK_SECRET"
```

Each record carries the raw body, the method, the path, and **every header**. Headers matter as much
as the body: the usual cause of a caller getting `401` is that its secret header never arrived.
Capture also records bodies that are not JSON at all, so a caller sending form-encoded data shows up
as itself rather than as a parse error.

When you are done, point the caller back at `/autoscale/webhook` and set `capture.enabled=false`.

**Why the capture endpoint checks no shared secret.** `verify-webhook-secret` runs before
`normalize-alert`, so a caller with a missing or wrong header is rejected with `401` and its body —
the thing you are trying to inspect — is never seen. An endpoint that only captures correctly
authenticated requests would be useless for debugging authentication. What protects it instead is
`capture.path`: a caller's URL is always configurable even when its headers are not. Put a long
random segment on it, and leave `capture.enabled=false` outside of an investigation.

If the read-back endpoint is unreachable, the same record is written to the log with the marker
`AUTOSCALER-CAPTURE`, so `anypoint-cli-v4 runtime-mgr application download-logs` and a `grep` is the
fallback. Note that log retrieval through Anypoint **Monitoring** is itself subscription-gated (see
[Reading logs](#reading-logs)).

## Verified against a live environment

This app has been **deployed to CloudHub 2.0 and exercised end to end** against a real sandbox — not
only unit-tested against mocks.

**Platform API contract:**

| Assumption | Result |
|---|---|
| `GET /amc/application-manager/api/v2/organizations/{org}/environments/{env}/deployments/{id}` | **200 OK** |
| Replica count lives at `target.replicas` | **Confirmed** |
| `PATCH` accepts a partial `{"target":{"replicas":n}}` | **Accepted, and the new replica actually started** |
| Token endpoint + `client_credentials` form body | **Works** |
| Replica count independent of vCore sizing | **Confirmed** — `application.vCores` unchanged |

**Running behaviour**, tested by POSTing alerts at the deployed app's public URL and watching a real
target application:

| Behaviour | Result |
|---|---|
| No `X-Autoscaler-Secret` | **401**, `unauthorized` |
| Wrong secret | **401**, `unauthorized` |
| CPU 92 with valid secret | **200**, `scaled` — target went **1 → 2 replicas** |
| Immediate repeat alert | **`skipped`** — cooldown suppressed it |
| CPU 55 (between thresholds) | **`skipped`**, `direction: NONE`, no platform API calls |
| Alert with no deployment id | **400**, `bad_request` |
| Left idle past `decay.idle.seconds` | **Decay fired unprompted — target reclaimed 2 → 1** |

That last row is the important one. It confirms the scheduler runs, and that
`os:retrieve-all-keys` works against **Object Store v2** on CloudHub 2.0 — the assumption the whole
decay design rests on, and the one that could not be checked without deploying.

**The capture path**, exercised over HTTP against the same deployment:

| Behaviour | Result |
|---|---|
| JSON body to `capture.path` | **200** `captured` |
| Form-encoded (non-JSON) body | **200**, body stored **verbatim** — capture cannot fail closed |
| Guessable path without the random segment | **404** — the path token is what protects it |
| `GET /autoscale/captures` without the secret | **401**, and nothing recorded |
| `GET /autoscale/captures` with the secret | **200**, raw bodies and all headers returned |
| Custom header through CloudHub's ingress | **Survives unmodified** |
| Capture hook on the authenticated webhook path | Records the alert; a rejected `401` records nothing |

Two things worth keeping from that. CloudHub's EDGE ingress passes custom headers through untouched,
so if a caller can set a header it will arrive. And an unauthorized request is never stored, so the
capture store cannot be filled by an unauthenticated caller hitting `/autoscale/webhook`.

The **inbound contract** is no longer an open question, but not because it was measured — because it
turned out there was nothing to measure. No Anypoint alerting system can call a webhook, so no vendor
payload was ever going to arrive. The contract is now this project's own and is documented in
[Endpoint contract](#endpoint-contract). See [Where alerts come from](#where-alerts-come-from).

## Build and test

```bash
mvn clean package     # target/mulesoft-autoscaler-1.2.0-mule-application.jar
mvn test              # MUnit suite (20 tests)
```

Tests read `src/test/resources/config.properties`, which is committed with fake values and takes
classpath precedence over the main config — so the suite is deterministic and runs on a fresh clone
without any local setup.

First run needs network: the Mule runtime and connectors resolve from `repository.mulesoft.org` and
`maven.anypoint.mulesoft.com`, both declared in `pom.xml`.

## Deploying

CloudHub 2.0 deploys **from Exchange**, not from a local file, so this is a two-step process. The
commands below are the ones actually used to deploy and validate this app.

**A note on what does not work:** the `mule-maven-plugin` `cloudhub2Deployment` block is the
commonly documented route, but in practice it failed here — it does not publish the artifact to
Exchange first, so the deployment fails with *"Failed to retrieve artifact information from
Exchange."* Publishing separately via `deploy:deploy-file` also failed (connection reset). The
Anypoint CLI path below is what worked, so that is what is documented.

**1. Package and publish to Exchange.** The Exchange asset id must be the **organization ID** as
the group, and the file key must be exactly `mule-application.jar`:

```bash
mvn clean package

anypoint-cli-v4 exchange asset upload \
  --organization "$ORG_ID" \
  --name "mulesoft-autoscaler" \
  --type app \
  --files '{"mule-application.jar":"/abs/path/target/mulesoft-autoscaler-1.2.0-mule-application.jar"}' \
  "$ORG_ID/mulesoft-autoscaler/1.2.0"
```

**2. Deploy from Exchange.** Positional arguments must come **before** the flags — the variadic
`--property` flags will otherwise swallow them:

```bash
anypoint-cli-v4 runtime-mgr application deploy \
  'mulesoft-autoscaler' 'cloudhub-us-west-2' '4.9.20' 'mulesoft-autoscaler' \
  --organization "$ORG_ID" --environment 'Sandbox' \
  --groupId "$ORG_ID" --assetVersion '1.2.0' \
  --replicas 1 --replicaSize 0.1 \
  --releaseChannel LTS --javaVersion 17 \
  --property "http.port:8081" \
  --property "anypoint.host:anypoint.mulesoft.com" \
  --property "scale.up.threshold:80" \
  --property "scale.down.threshold:30" \
  --property "scale.min.replicas:1" \
  --property "scale.max.replicas:3" \
  --property "scale.cooldown.seconds:600" \
  --property "decay.enabled:true" \
  --property "decay.interval.seconds:300" \
  --property "decay.idle.seconds:1800" \
  --property "anypoint.client.id:$CLIENT_ID" \
  --secureProperty "anypoint.client.secret:$CLIENT_SECRET" \
  --secureProperty "webhook.secret:$WEBHOOK_SECRET"
```

Secrets go in as `--secureProperty`, so they are masked in Runtime Manager and excluded from logs.

### Runtime version

Deploy on an **LTS** channel runtime (`4.9.20` at time of writing) rather than EDGE. `mule-artifact.json`
declares `minMuleVersion: 4.9.0` to allow this. Note the tests run on 4.11.6, because MUnit 3.7
cannot create an embedded container for 4.9.20 — a mismatch worth knowing about, though the app uses
no feature newer than 4.9.

To find valid targets and their supported runtimes:

```bash
curl -s "https://anypoint.mulesoft.com/runtimefabric/api/organizations/$ORG_ID/targets" \
  -H "Authorization: Bearer $TOKEN"
```

The deploy target argument is the target **`id`** (e.g. `cloudhub-us-west-2`).

### Replica count

All three object stores are persistent, so on CloudHub 2.0 they use Object Store v2 and survive
restarts. Run this app as a **single replica** — see [Known limitations](#known-limitations).

### Reading logs

Worth knowing before you go looking, because it bites in exactly the tier this project targets:
**log retrieval through Anypoint Monitoring is subscription-gated.** On an org without it, tools that
route through the Monitoring log-search API — including the MuleSoft MCP server's log retrieval —
fail with:

```
Required monitoringCenter subscription one of "Premium, Trial Titanium Monitoring" (1),
"Premium, Paid Titanium Monitoring" (2), and "Premium, Advanced Monitoring" (4) not found.
Current value: 3
```

The CloudHub 2.0 deployment log API is **not** gated, and the CLI reaches it:

```bash
anypoint-cli-v4 runtime-mgr application describe "$APP_ID" \
  --organization "$ORG_ID" --environment 'Sandbox' --output json   # take desiredVersion as SPEC_ID

anypoint-cli-v4 runtime-mgr application download-logs "$APP_ID" "$SPEC_ID" ./logs \
  --organization "$ORG_ID" --environment 'Sandbox'
```

Prefer `download-logs` over `logs`. The `logs` subcommand tails, and after its first batch it crashes
on an upstream bug in `anypoint-cli-ch1-plugin`
(`TypeError: Cannot read properties of undefined (reading 'timestamp')`). The first batch still
prints, so it is usable in a pinch, but it is not something to script against.

## Project layout

```
pom.xml                                  Mule 4 application build
mule-artifact.json                       runtime descriptor, secure property declarations
src/main/mule/global-config.xml          listener, request config, object stores
src/main/mule/autoscaler.xml             flows and sub-flows
src/main/resources/config.properties.example
src/test/resources/config.properties     committed test values (fake)
src/test/munit/autoscale-flow-test.xml   MUnit suite
examples/autoscaler-emitter.xml          reference alert source, for the MONITORED app
```

`examples/` is documentation. It is not compiled, packaged or tested by this project, because it
belongs to a different application.

## Flow reference

| Flow | Responsibility |
|---|---|
| `autoscale-webhook-flow` | Entry point. Orchestrates the alert path and owns the error handler mapping failures onto HTTP status codes. |
| `verify-webhook-secret` | Compares `X-Autoscaler-Secret` against `webhook.secret`; raises `AUTOSCALER:UNAUTHORIZED`. Runs first. |
| `normalize-alert` | Maps the inbound body onto the internal alert shape. **Single point of truth for the webhook contract.** Raises `AUTOSCALER:BAD_REQUEST` if no deployment id. |
| `apply-scaling` | Fetches the deployment, computes the clamped target, patches only when it differs, then opens the cooldown and records the deployment for decay. |
| `autoscale-decay-flow` | Scheduled sweep over tracked deployments. Failures are isolated per deployment so one bad entry cannot stop the sweep. |
| `decay-one-deployment` | Decides whether one deployment is idle enough to lose a step, reusing `apply-scaling`; drops it from tracking once at the minimum. |
| `anypoint-get-token` | OAuth `client_credentials` exchange. |
| `anypoint-get-deployment` | `GET` the deployment into `vars.deployment`. |
| `anypoint-patch-replicas` | `PATCH` carrying only `target.replicas`. |
| `build-skipped-response` | The `skipped` response body shared by all no-action paths. |
| `capture-request` | Records one raw request — body, method, path, all headers — without interpreting it. |
| `autoscale-capture-flow` | Unauthenticated capture endpoint on `capture.path`. Always answers `200`. |
| `autoscale-captures-read-flow` | `GET /autoscale/captures`. Authenticated read-back of captured requests. |

Custom error types: `AUTOSCALER:UNAUTHORIZED` (→ 401), `AUTOSCALER:BAD_REQUEST` (→ 400).

Object stores: `Cooldown_Store` (TTL'd debounce), `Managed_Deployments` (decay tracking, no TTL —
entries live until the deployment is back at minimum), and `Alert_Captures` (diagnostic, capped at 25
entries with a 7-day TTL because it holds unvalidated bodies from an unauthenticated endpoint).

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Every call returns `401` | The caller is not sending `X-Autoscaler-Secret`, or it does not match the running config. Turn on capture and inspect the headers it actually sent. |
| Every call returns `400` | The body carried no deployment id at any accepted path. Capture the payload and compare it against [Endpoint contract](#endpoint-contract). |
| `500` with `HTTP:UNAUTHORIZED` | Connected App credentials wrong, or missing Read/Manage Applications on that environment. |
| `500` with `HTTP:NOT_FOUND` | The `deploymentId` is not a CloudHub 2.0 deployment id in the resolved org/environment. |
| `skipped` when you expect scaling | Within the band, cooldown open, or already at a bound. The log line says which. |
| Replicas creep down during steady traffic | `decay.idle.seconds` is shorter than your quiet periods. Raise it. |
| Decay never fires | `decay.enabled` is false, nothing has been scaled up yet (nothing is tracked), or every entry is still inside its cooldown. |
| Replicas stuck high | Check the decay sweep is running and the deployment is in `Managed_Deployments`. A deployment scaled up before this app was deployed is not tracked and will not decay. |

## Known limitations

- **Anypoint cannot drive this app.** No Anypoint alerting system can call a webhook at any tier, so
  you must supply the caller yourself. See [Where alerts come from](#where-alerts-come-from) and the
  reference emitter in `examples/`. This is the single biggest thing to understand before adopting
  it.
- **The emitter requires editing the monitored application.** A `flow-ref` has to be added to each
  flow whose traffic should count, and the app's deployment id supplied as a property, because a
  CloudHub 2.0 app cannot discover its own. If you cannot modify the monitored app, you need a
  different caller — an external monitor or scheduled job.
- **The emitter's counter is not atomic.** Read-modify-write on a shared store undercounts under
  concurrency. Adequate for a scaling signal, unsuitable for anything that must be exact.
- **`examples/autoscaler-emitter.xml` is not built or tested by this project.** It is a
  documentation artifact for a different application. Its DataWeave was verified by execution on
  Mule 4.11.6 and 4.12.2, but nothing guards it against drift.
- **The capture endpoint is unauthenticated when enabled.** That is deliberate and explained above,
  but it means `capture.enabled=true` opens a public write path guarded only by the randomness of
  `capture.path`. Turn it off when the investigation is done.
- **Only deployments this app has scaled are tracked for decay.** A deployment already sitting at an
  elevated count when this app is first deployed will not be reclaimed until an alert scales it once.
- **Run a single replica.** Two replicas of *this* app would sweep concurrently and could double-step
  a deployment. The object stores are shared on CloudHub 2.0, which narrows the window, but nothing
  here takes a lock.
- **The listener speaks plain HTTP**, intended to sit behind CloudHub's TLS termination. The build
  warns about this. Put TLS in front if you expose it directly.
- **The shared-secret check is a plain string comparison**, so not constant-time. Given a long random
  secret this is minor, but it is not a MAC. Use an HMAC over the body if you need more.
- **Only replica count is adjusted.** vCore size per replica is untouched by design, so a scaling
  action can never silently resize an application.
- **No dry-run mode.** Every qualifying alert results in a real `PATCH`. Test against a
  non-production environment first.
