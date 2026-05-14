# Catalyst Python Deployment — Troubleshooting Guide

---

## App Deploys but Won't Start (502 / Not Responding)

**Most common cause: Port not bound correctly.**

Check that your app reads `X_ZOHO_CATALYST_LISTEN_PORT`:

```python
import os
port = int(os.environ.get('X_ZOHO_CATALYST_LISTEN_PORT', 5000))
app.run(host='0.0.0.0', port=port)
```

Also check:
- The startup command in `app-config.json` (`"command"` key) is correct
- Test the command locally: run it manually in your terminal and verify the app starts
- If using shell variable substitution (`$VAR`), prefix with `sh -c "..."`:
  ```json
  "command": "sh -c \"gunicorn -b 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT app:app\""
  ```

---

## ModuleNotFoundError at Runtime

Your `requirements.txt` is missing a package.

```bash
# Regenerate from your virtualenv
pip freeze > requirements.txt

# Or use pipreqs for only what's imported
pip install pipreqs && pipreqs . --force
```

Then redeploy:
```bash
catalyst deploy
```

---

## PermissionError / Read-only File System

Catalyst restricts write access to the app's current directory at runtime.

**Fix:** Write all temp/output files to `/tmp`:
```python
import tempfile, os

# Wrong:
with open('output.txt', 'w') as f:
    f.write(data)

# Correct:
tmp_path = os.path.join(tempfile.gettempdir(), 'output.txt')
with open(tmp_path, 'w') as f:
    f.write(data)
```

---

## `catalyst deploy` Fails — "catalyst.json not found"

You must run deploy from the root of your Catalyst project (where `catalyst.json` lives).

```bash
cd /path/to/my-catalyst-project  # not inside a subdirectory
catalyst deploy
```

If `catalyst.json` is missing entirely, reinitialize:
```bash
catalyst init
```

---

## App Works in Development, Fails in Production

**Most common cause:** Environment variables not set for production.

In Catalyst console:
1. Switch to **Production** environment (dropdown at top)
2. Go to **Serverless → AppSail → [your service] → Configurations**
3. Add all required environment variables for production
4. Redeploy or promote again

---

## `Invalid grant token` / OAuth Errors (Zoho Integration)

- Grant tokens expire in 10 minutes — generate a new one and exchange immediately
- Refresh tokens never expire unless revoked — keep them safe
- Ensure you used the correct scopes when generating the grant token
- Verify the `ZOHO_ACCOUNTS_URL` matches your Zoho data center

```bash
# Test token refresh manually:
curl -X POST https://accounts.zoho.com/oauth/v2/token \
  -d "refresh_token=YOUR_REFRESH_TOKEN" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "grant_type=refresh_token"
```

---

## Viewing Logs

### Via CLI
```bash
catalyst logs --app <your-appsail-service-name>
```

### Via Console
Go to: **Serverless → AppSail → [your service] → Logs**

---

## Memory / Performance Issues

Increase memory allocation in `app-config.json`:
```json
{
  "memory": 512
}
```

Valid values: `256`, `512`, `1024`, `2048` MB.

Redeploy after changing.

---

## Slow Cold Starts

AppSail scales to zero when idle. First request after idle period triggers a cold start.

Mitigations:
- Keep your startup time minimal (lazy-load heavy dependencies)
- Use Catalyst **Cron** to ping your app endpoint periodically (keeps it warm)
- Use Catalyst **Cache** for data that doesn't change often

---

## Standalone Deploy (No Prior Init)

If you want to deploy without running `catalyst init` first:

```bash
cd /path/to/your/python/app
catalyst appsail:deploy \
  --name my-service-name \
  --stack python \
  --command "python app.py"
```

---

## Useful CLI Commands

```bash
# Login
catalyst login

# Initialize new project
catalyst init

# Add AppSail to existing project
catalyst appsail:add

# Deploy all resources
catalyst deploy

# Deploy AppSail only
catalyst appsail:deploy

# Deploy client only
catalyst deploy --only client

# View logs
catalyst logs --app <service-name>

# Serve locally for testing
catalyst serve

# Pull project from console to local
catalyst pull

# List projects
catalyst project:list
```
