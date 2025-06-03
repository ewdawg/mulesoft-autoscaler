# MuleSoft Auto-Scaler via Webhook Alerts

This is an AI generated lightweight MuleSoft application that enables **auto-scaling behavior for CloudHub deployments** based on performance alerts — all built using MuleSoft itself.

It listens for **Anypoint Monitoring webhook alerts** (e.g., CPU > 80%) and uses the **Runtime Manager API** to scale your deployed MuleSoft application by increasing the number of replicas or allocated resources.

## 🔧 Use Case

Some organizations operate under fixed vCore contracts or have disabled native autoscaling due to budget, compliance, or control reasons. This solution offers a **self-hosted, Mule-native workaround** for smart scaling without additional services or infrastructure.

## 🚀 How It Works

1. Anypoint Monitoring alert sends a webhook with performance data.
2. This app receives the webhook at `/autoscale`, evaluates CPU usage.
3. If CPU usage exceeds the threshold, the app:
   - Authenticates with the Anypoint Platform
   - Issues a PATCH call to the Runtime Manager API
   - Increases the number of replicas or vCore allocation for the targeted app

## 🔐 Requirements

- MuleSoft Anypoint Platform credentials with access to Runtime Manager
- API client ID/secret for OAuth
- Anypoint Monitoring enabled
- A target app deployed in CloudHub 2.0

## 📁 File Structure

- `autoscaler.xml`: The core Mule flow
- `config.properties`: External configuration (credentials, target app IDs)

## 🛠 config.properties Template

```properties
client.id=REPLACE_WITH_CLIENT_ID
client.secret=REPLACE_WITH_CLIENT_SECRET
org.id=REPLACE_WITH_ORG_ID
env.id=REPLACE_WITH_ENV_ID
app.id=REPLACE_WITH_TARGET_APP_ID
```

Rename this file to `config.properties` and place it alongside `autoscaler.xml`
before running the Mule application.
