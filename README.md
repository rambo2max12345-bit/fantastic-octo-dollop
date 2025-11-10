```markdown
# LINE Webhook Proxy (Cloud Run) — deploy guide

This proxy verifies LINE X-Line-Signature (HMAC-SHA256) then forwards events to your Apps Script Web App as:
{ proxySecret: "...", events: [ ... ] }

Files:
- index.js
- package.json
- Dockerfile
- .dockerignore

Required environment variables (do NOT hardcode secrets):
- LINE_CHANNEL_SECRET      (from LINE Developers)
- APPS_SCRIPT_WEBAPP_URL   (Apps Script Web App URL, set to POST endpoint deployed as Web App)
- PROXY_SECRET             (random secret string; must match Script Property in Apps Script)

Important:
- Rotate any leaked CHANNEL_ACCESS_TOKEN immediately.
- Store secrets in Secret Manager and inject into Cloud Run where possible.

Quick deploy to Cloud Run (gcloud CLI)
1. Build and push image (Cloud Build automatic build or Docker + gcloud)
   Replace <PROJECT_ID> and <SERVICE_NAME> as needed.

   # using gcloud builds submit (recommended)
   gcloud builds submit --tag gcr.io/<PROJECT_ID>/line-webhook-proxy:latest

   # or build locally and push (requires docker authenticated)
   docker build -t gcr.io/<PROJECT_ID>/line-webhook-proxy:latest .
   docker push gcr.io/<PROJECT_ID>/line-webhook-proxy:latest

2. Deploy to Cloud Run
   gcloud run deploy <SERVICE_NAME> \
     --image gcr.io/<PROJECT_ID>/line-webhook-proxy:latest \
     --platform managed \
     --region us-central1 \
     --allow-unauthenticated \
     --set-env-vars "LINE_CHANNEL_SECRET=<LINE_CHANNEL_SECRET>,APPS_SCRIPT_WEBAPP_URL=<APPS_SCRIPT_WEBAPP_URL>,PROXY_SECRET=<PROXY_SECRET>"

   Notes:
   - Replace region with chosen region.
   - --allow-unauthenticated is required for LINE webhook to reach the service.
   - For better security, use Secret Manager and --set-secrets instead of --set-env-vars.

Use Secret Manager (recommended)
1. Create secrets:
   gcloud secrets create LINE_CHANNEL_SECRET --data-file=- <<< "your_line_channel_secret"
   gcloud secrets create PROXY_SECRET --data-file=- <<< "random_proxy_secret"
   gcloud secrets create APPS_SCRIPT_WEBAPP_URL --data-file=- <<< "https://script.google.com/macros/s/XXX/exec"

2. Deploy Cloud Run and mount secrets as environment variables:
   gcloud run deploy <SERVICE_NAME> \
     --image gcr.io/<PROJECT_ID>/line-webhook-proxy:latest \
     --platform managed \
     --region us-central1 \
     --allow-unauthenticated \
     --set-secrets LINE_CHANNEL_SECRET=LINE_CHANNEL_SECRET:latest,PROXY_SECRET=PROXY_SECRET:latest,APPS_SCRIPT_WEBAPP_URL=APPS_SCRIPT_WEBAPP_URL:latest

Testing
- After deploy, set the webhook URL in LINE Developers to:
  https://<CLOUD_RUN_HOST>/webhook
- Enable webhook and test by sending messages in the group
- Apps Script must be deployed as Web App and have Script Property for PROXY_SECRET (value must match)

Apps Script config reminders
- Deploy Apps Script (separate project) as Web App (Execute as: Me, Who has access: Anyone)
- Save PROXY_SECRET in Script Properties (same value used by Cloud Run)
- Save CHANNEL_ACCESS_TOKEN (LINE channel access token) in Script Properties for Apps Script to call LINE API
- Set SPREADSHEET_ID in Apps Script or keep it in existing system; the proxy does NOT modify your original project

Security tips
- Rotate CHANNEL_ACCESS_TOKEN if leaked.
- Use Secret Manager instead of plain env vars.
- Restrict Cloud Run invoker to only services/accounts you trust, if you place an authentication layer (but LINE requires public webhook).
- Log suspicious requests and monitor.

If you want, I can:
- provide cloudbuild.yaml for automatic build/deploy,
- add GitHub Actions workflow to build and deploy on push,
- show step-by-step screenshots for Cloud Console.
```