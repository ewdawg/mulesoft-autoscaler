# MuleSoft Auto-Scaler

A Mule 4 application that adjusts a **CloudHub 2.0** deployment's replica count in response to
Anypoint Monitoring alert webhooks. It exists for organizations on fixed vCore contracts, or with
native autoscaling disabled, that still want bounded, auditable scaling without standing up
additional infrastructure.

- [How it works](#how-it-works)
- [Safety properties](#safety-properties)
- [Endpoint contract](#endpoint-contract)
- [Requirements](#requirements)
- [Configuration](#configuration)
- [Setting up Anypoint](#setting-up-anypoint)
- [Build and test](#build-and-test)
- [Deploying](#deploying)
- [Project layout](#project-layout)
- [Flow reference](#flow-reference)
- [Troubleshooting](#troubleshooting)
- [Known limitations](#known-limitations)

## How it works

```
Anypoint Monitoring alert
        │
        ▼
POST /autoscale/webhook          ← shared-secret header required
        │
        ├─ verify shared secret ──────────────────► 401 if absent or wrong
        │
        ├─ normalize alert  ──────────────────────► 400 if no deployment id
        │
        ├─ metric within band? ───────────────────► no action (no API calls made)
        │
        ├─ cooldown window open? ─────────────────► suppressed
        │
        ├─ OAuth client_credentials → access token
        ├─ GET  deployment           → current replicas
        ├─ clamp(current ± step, min, max)
        │       └─ unchanged? ────────────────────► no action (already at bound)
        └─ PATCH deployment          → new replica count, then open cooldown
```

Scaling is **bidirectional**: a metric at or above `scale.up.threshold` adds replicas, at or below
`scale.down.threshold` removes them, and anything between the two is left alone. The band between
the two thresholds is deliberate — it prevents a metric hovering near a single threshold from
oscillating the deployment up and down.

## Safety properties

These are the guarantees the flow is built to provide. Each is covered by the test suite:

| Guarantee | Mechanism | Test |
|---|---|---|
| Only authorized callers can scale | `X-Autoscaler-Secret` checked before any other work | `e2e-unauthenticated-call-returns-401`, `rejects-wrong-webhook-secret` |
| Replicas stay within bounds | `scale.min.replicas` / `scale.max.replicas` clamp applied every time | `clamps-at-max-replicas`, `clamps-at-min-replicas` |
| A flapping alert cannot ratchet replicas | ObjectStore cooldown keyed by deployment id, TTL `scale.cooldown.seconds` | `e2e-cooldown-suppresses-repeat-alert`, `starts-cooldown-after-scaling` |
| Costs can come back down | Scale-down path on the low threshold | `scales-down-by-one-step` |
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

The body is an Anypoint Monitoring alert. The `normalize-alert` sub-flow maps it onto an internal
shape, tolerating several field layouts (see [Known limitations](#known-limitations)):

| Internal field | Read from, in order |
|---|---|
| `alertId` | `alert.id`, `alert.alertId` |
| `applicationName` | `alert.resource.name`, `alert.appName`, `alert.applicationName` |
| `deploymentId` | `alert.resource.id`, `alert.deploymentId`, `alert.appId` |
| `organizationId` | `alert.organization.id`, `alert.resource.organizationId`, `anypoint.org.id` |
| `environmentId` | `alert.resource.environment.id`, `alert.environmentId`, `anypoint.env.id` |
| `metric` | `alert.metric` (default `"cpu"`) |
| `currentValue` | `alert.currentValue`, `alert.value` |

A body may also be nested under `event` instead of `alert`; both are accepted.

### Responses

| Status | Body `status` | Meaning |
|---|---|---|
| `200` | `scaled` | Replica count was changed. Body carries `previousReplicas` and `replicas`. |
| `200` | `skipped` | No action: metric within band, cooldown open, or already at a bound. |
| `400` | `bad_request` | Alert carried no deployment id. |
| `401` | `unauthorized` | Missing or invalid `X-Autoscaler-Secret`. |
| `500` | `error` | Unexpected failure. `detail` carries the error description. |

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

## Requirements

- JDK 17 and Maven 3.9+
- An Anypoint Platform **Connected App** (`client_credentials` grant) able to read and modify
  deployments in the target environment
- A target application deployed on **CloudHub 2.0**
- Anypoint Monitoring alerts configured to POST to this app's endpoint

## Configuration

Copy the template and fill it in:

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
| `anypoint.org.id` / `anypoint.env.id` | Fallbacks used only if the alert omits them |

For real deployments, supply the two secrets at deploy time rather than in the file — any property
can be overridden by a system property of the same name:

```
-M-Danypoint.client.secret=...  -M-Dwebhook.secret=...
```

`mule-artifact.json` declares `anypoint.client.secret` and `webhook.secret` as `secureProperties`,
which keeps them out of logs and diagnostics.

### Choosing thresholds

Leave a wide gap between `scale.up.threshold` and `scale.down.threshold`. A narrow band combined
with a short cooldown will scale up, immediately satisfy the low threshold, and scale back down.
The shipped defaults (80 / 30, with a 600-second cooldown) are a conservative starting point.

## Setting up Anypoint

### 1. Connected App

In **Access Management → Connected Apps**, create an app with the *App acts on its own behalf*
(`client_credentials`) grant type. Grant it, scoped to the environment holding the target app:

- **Read Applications** — for the `GET` deployment call
- **Manage Applications** — for the `PATCH`

Copy the client id and secret into your configuration.

### 2. Monitoring alert

In **Anypoint Monitoring → Alerts**, create a CPU alert on the target application and add a
**webhook** action pointing at this app's public URL:

```
https://<this-app>.<region>.cloudhub.io/autoscale/webhook
```

Add a custom header `X-Autoscaler-Secret` with the same value as `webhook.secret`. Without it every
call is rejected with `401`.

Create a **second alert** on the low threshold if you want scale-down — this app decides direction
from the metric value it receives, so it needs to actually receive an alert when utilization drops.

### 3. Capturing the real payload

The exact webhook body Anypoint Monitoring sends has **not** been verified against a live alert.
Before trusting this in production, point one alert at a request-capture endpoint (or temporarily
raise `normalize-alert`'s logger to log the raw payload), then confirm the field mappings in the
table above. All mapping lives in that one sub-flow.

## Build and test

```bash
mvn clean package     # produces target/mulesoft-autoscaler-1.0.0-mule-application.jar
mvn test              # runs the MUnit suite (15 tests)
```

The build is offline-hostile on first run: it resolves the Mule runtime and connectors from
`repository.mulesoft.org` and `maven.anypoint.mulesoft.com`, both declared in `pom.xml`.

## Deploying

Deploy the packaged artifact to CloudHub 2.0 as you would any Mule application — via Runtime
Manager, or by adding a `cloudhub2Deployment` block to `pom.xml` and running
`mvn deploy -DmuleDeploy`. Supply the two secrets as system properties at deploy time rather than
baking them into the artifact.

Run this app as a **single replica** unless you switch the cooldown to a persistent store; see
[Known limitations](#known-limitations).

## Project layout

```
pom.xml                                  Mule 4 application build
mule-artifact.json                       runtime descriptor, secure property declarations
src/main/mule/global-config.xml          listener, request config, cooldown object store
src/main/mule/autoscaler.xml             the flow and its sub-flows
src/main/resources/config.properties.example
src/test/munit/autoscale-flow-test.xml   MUnit suite
```

## Flow reference

| Flow | Responsibility |
|---|---|
| `autoscale-webhook-flow` | Entry point. Orchestrates the sequence and owns the error handler that maps failures onto HTTP status codes. |
| `verify-webhook-secret` | Compares `X-Autoscaler-Secret` against `webhook.secret`; raises `AUTOSCALER:UNAUTHORIZED` on mismatch. Runs first. |
| `normalize-alert` | Maps the inbound body onto the internal alert shape. **The single point of truth for the webhook contract.** Raises `AUTOSCALER:BAD_REQUEST` if no deployment id is present. |
| `apply-scaling` | Fetches the deployment, computes the clamped target replica count, patches only when it differs, then opens the cooldown window. |
| `anypoint-get-token` | OAuth `client_credentials` exchange. |
| `anypoint-get-deployment` | `GET` the deployment; stores it in `vars.deployment`. |
| `anypoint-patch-replicas` | `PATCH` carrying only `target.replicas`. |
| `build-skipped-response` | Builds the `skipped` response body used by all three no-action paths. |

Custom error types: `AUTOSCALER:UNAUTHORIZED` (→ 401) and `AUTOSCALER:BAD_REQUEST` (→ 400).

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Every call returns `401` | The alert's webhook action is not sending `X-Autoscaler-Secret`, or its value does not match `webhook.secret` in the running config. |
| Every call returns `400` | The real webhook body does not match any layout `normalize-alert` expects. Capture the payload and correct the mapping — see [Capturing the real payload](#3-capturing-the-real-payload). |
| `500` with an `HTTP:UNAUTHORIZED` detail | Connected App credentials are wrong, or lack Read/Manage Applications on that environment. |
| `500` with `HTTP:NOT_FOUND` | The `deploymentId` from the alert is not a CloudHub 2.0 deployment id in the resolved org/environment. |
| Responses say `skipped` when you expect scaling | Metric is between the thresholds, the cooldown is still open, or the deployment is already at a bound. The log line for each case says which. |
| Scaling happens once then never again | Expected inside `scale.cooldown.seconds`. If it persists beyond that, check that this app has not been restarted — the cooldown store is in-memory and does not survive a restart. |
| Replicas ratchet up across restarts | Multiple replicas of *this* app, each with its own cooldown state. Run one replica or use a persistent store. |

## Known limitations

- **The inbound webhook contract is unverified.** The exact body Anypoint Monitoring posts has not
  been confirmed against a live alert. `normalize-alert` tolerates the shapes this project has
  previously assumed and is the single place to correct once you have captured a real payload.
- **The listener speaks plain HTTP.** This is intended to sit behind CloudHub's TLS termination. If
  you expose it directly, put TLS in front of it. The build emits a warning to this effect.
- **The shared-secret check is a plain string comparison**, so it is not constant-time. Given the
  secret is a long random string this is a minor concern, but it is not a hardened MAC. If you need
  stronger assurance, replace it with an HMAC over the request body.
- **The cooldown store is in-memory and non-persistent.** It does not survive a restart, and with
  more than one replica of *this* app each replica keeps its own state. Run a single replica, or
  switch `Cooldown_Store` to a persistent store.
- **Only replica count is adjusted.** vCore size per replica is left untouched by design, so that a
  scaling action can never silently resize an application.
- **No dry-run mode.** Every qualifying alert results in a real `PATCH`. Test against a non-production
  environment first.
