# MuleSoft Auto-Scaler

A Mule 4 application that scales a **CloudHub 2.0** deployment up and down in response to Anypoint
Monitoring alerts.

## Why this exists

CloudHub 2.0 has native autoscaling, but it is gated behind the **Titanium** subscription tier.
Everyone below that tier has the same problem — load varies, replicas do not — and only two
sanctioned answers: pay for the upgrade, or provision permanently for peak and leave the headroom
idle around the clock.

This app is the third answer. It uses a capability you already have at lower tiers — monitoring
alerts with webhook actions — to synthesize the capability you do not. **It is an emulation of the
Titanium autoscaling feature**, built out of the parts available underneath it.

The premise is that an application should track its own load as a matter of course, and that
holding replicas at peak allocation twenty-four hours a day to serve a few hours of traffic is a
poor default. On a fixed vCore contract the saving is headroom: replicas you are not holding are
vCores other applications can use.

**Scope note.** This does what native autoscaling does, not more. If you are on Titanium, use the
native feature — it is faster, supported, and does not consume vCores of its own to run. The one
place this app goes further is scaling on signals native autoscaling cannot see: queue depth,
orders per minute, or any other business metric you can raise an alert on. `normalize-alert` is
metric-agnostic, so that path is already open.

## How it works

Two control paths, deliberately different in kind.

**Edge-triggered — reacts to alerts:**

```
Anypoint Monitoring alert
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

The body is an Anypoint Monitoring alert. `normalize-alert` maps it onto an internal shape,
tolerating several field layouts (see [Known limitations](#known-limitations)):

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

## Requirements

- JDK 17 and Maven 3.9+
- An Anypoint Platform **Connected App** (`client_credentials`) able to read and modify deployments
  in the target environment
- A target application on **CloudHub 2.0**
- Anypoint Monitoring alerts with webhook actions — **the one capability everything here depends
  on.** Confirm it is available at your tier before building on this.

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

## Setting up Anypoint

### 1. Connected App

In **Access Management → Connected Apps**, create an app with the *App acts on its own behalf*
(`client_credentials`) grant. Scoped to the environment holding the target app, grant it:

- **Read Applications** — for the `GET`
- **Manage Applications** — for the `PATCH`

### 2. Monitoring alerts

Create a CPU alert on the target application with a **webhook** action pointing at:

```
https://<this-app>.<region>.cloudhub.io/autoscale/webhook
```

Add a custom header `X-Autoscaler-Secret` matching `webhook.secret`. Without it every call is
rejected with `401`.

Create a **second alert** on the low threshold for scale-down. The decay flow will reclaim replicas
even if you skip this, but far more slowly than a real signal would.

### 3. Capturing the real payload

The exact webhook body Anypoint Monitoring sends has **not** been verified against a live alert.
Before production, point one alert at a request-capture endpoint (or temporarily log the raw
payload in `normalize-alert`) and confirm the mappings above. All mapping lives in that one
sub-flow.

## Build and test

```bash
mvn clean package     # target/mulesoft-autoscaler-1.1.0-mule-application.jar
mvn test              # MUnit suite (20 tests)
```

Tests read `src/test/resources/config.properties`, which is committed with fake values and takes
classpath precedence over the main config — so the suite is deterministic and runs on a fresh clone
without any local setup.

First run needs network: the Mule runtime and connectors resolve from `repository.mulesoft.org` and
`maven.anypoint.mulesoft.com`, both declared in `pom.xml`.

## Deploying

Deploy the packaged artifact to CloudHub 2.0 via Runtime Manager, or add a `cloudhub2Deployment`
block to `pom.xml` and run `mvn deploy -DmuleDeploy`. Supply secrets as system properties at deploy
time rather than baking them into the artifact.

Both object stores are persistent, so on CloudHub 2.0 they use Object Store v2 and survive restarts.
Run this app as a **single replica** — see [Known limitations](#known-limitations).

## Project layout

```
pom.xml                                  Mule 4 application build
mule-artifact.json                       runtime descriptor, secure property declarations
src/main/mule/global-config.xml          listener, request config, object stores
src/main/mule/autoscaler.xml             flows and sub-flows
src/main/resources/config.properties.example
src/test/resources/config.properties     committed test values (fake)
src/test/munit/autoscale-flow-test.xml   MUnit suite
```

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

Custom error types: `AUTOSCALER:UNAUTHORIZED` (→ 401), `AUTOSCALER:BAD_REQUEST` (→ 400).

Object stores: `Cooldown_Store` (TTL'd debounce) and `Managed_Deployments` (decay tracking, no TTL —
entries live until the deployment is back at minimum).

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Every call returns `401` | The alert is not sending `X-Autoscaler-Secret`, or it does not match the running config. |
| Every call returns `400` | The real webhook body matches no layout `normalize-alert` expects. Capture the payload and correct the mapping. |
| `500` with `HTTP:UNAUTHORIZED` | Connected App credentials wrong, or missing Read/Manage Applications on that environment. |
| `500` with `HTTP:NOT_FOUND` | The `deploymentId` is not a CloudHub 2.0 deployment id in the resolved org/environment. |
| `skipped` when you expect scaling | Within the band, cooldown open, or already at a bound. The log line says which. |
| Replicas creep down during steady traffic | `decay.idle.seconds` is shorter than your quiet periods. Raise it. |
| Decay never fires | `decay.enabled` is false, nothing has been scaled up yet (nothing is tracked), or every entry is still inside its cooldown. |
| Replicas stuck high | Check the decay sweep is running and the deployment is in `Managed_Deployments`. A deployment scaled up before this app was deployed is not tracked and will not decay. |

## Known limitations

- **The inbound webhook contract is unverified.** The exact body Anypoint Monitoring posts has not
  been confirmed against a live alert. `normalize-alert` tolerates the shapes this project has
  previously assumed and is the single place to correct.
- **The Application Manager API shape is unverified.** The `PATCH` assumes a partial
  `{"target":{"replicas":n}}` is accepted, and the `GET` assumes replicas live at `target.replicas`.
  Confirm both against a sandbox before production.
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
