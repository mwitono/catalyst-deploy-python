---
name: catalyst-deploy-python
description: >
  Deploy Python web applications (Flask, Django, Bottle, CherryPy, Tornado, FastAPI, or any Python framework) 
  to Zoho Catalyst AppSail. Use this skill whenever the user wants to deploy, publish, host, or push a Python 
  app to Catalyst — including first-time deployments, redeployments, production pushes, or troubleshooting 
  failed deploys. Also triggers when the user asks about "Catalyst AppSail Python", "hosting a Python app on 
  Catalyst", "deploying Flask/Django to Catalyst", or "how do I get my Python app live on Catalyst". 
  Always use this skill — do NOT attempt a Catalyst Python deployment without it.
---

# Catalyst Deploy — Python

A complete deployment guide for Python web apps on Zoho Catalyst AppSail.

## Quick Decision

| Situation | Path |
|---|---|
| New app, first deploy | → [Full Deploy Flow](#full-deploy-flow) |
| App already initialized, code changed | → [Redeploy](#redeploy) |
| Deploying to production | → [Promote to Production](#promote-to-production) |
| Deploy failing or app not starting | → [Troubleshooting](#troubleshooting) |
| Zoho Books / CRM integration needed | → Read [references/zoho-integration.md](references/zoho-integration.md) |

---

## Prerequisites Checklist

Before starting, confirm all of these:

- [ ] **Node.js** installed (for Catalyst CLI) — `node -v`
- [ ] **Catalyst CLI** installed — `npm install -g zcatalyst-cli`
- [ ] **Python** installed matching target runtime (3.9 recommended) — `python3 --version`
- [ ] **Zoho account** with Catalyst access at [catalyst.zoho.com](https://catalyst.zoho.com)
- [ ] App runs locally without errors
- [ ] App binds to port via `X_ZOHO_CATALYST_LISTEN_PORT` env var (see [Port Binding](#critical-port-binding))

---

## CRITICAL: Port Binding

**This is the #1 cause of failed deployments.** Your app MUST listen on the port Catalyst provides, not a hardcoded port.

### Flask
```python
import os
from flask import Flask

app = Flask(__name__)

if __name__ == '__main__':
    port = int(os.environ.get('X_ZOHO_CATALYST_LISTEN_PORT', 5000))
    app.run(host='0.0.0.0', port=port)
```

### Django (`manage.py` or `wsgi` startup command)
```bash
# Startup command in app-config.json:
python manage.py runserver 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT
```

### Tornado
```python
import os
import tornado.ioloop
import tornado.web

port = int(os.environ.get('X_ZOHO_CATALYST_LISTEN_PORT', 8888))
app.listen(port)
tornado.ioloop.IOLoop.current().start()
```

### Uvicorn / FastAPI
```bash
# Startup command:
uvicorn main:app --host 0.0.0.0 --port $X_ZOHO_CATALYST_LISTEN_PORT
```

> **Note:** Catalyst does not allow modifying the assigned port. You must always read from `X_ZOHO_CATALYST_LISTEN_PORT`.

---

## Full Deploy Flow

### Step 1 — Login to Catalyst
```bash
catalyst login
```
Follow the browser prompt to authenticate with your Zoho account.

### Step 2 — Initialize Project
```bash
mkdir my-python-app && cd my-python-app
catalyst init
```

CLI prompts:
- **Select project**: Choose your existing Catalyst project (or create one at console.catalyst.zoho.com first)
- **Select features**: Choose `AppSail`
- When asked for AppSail runtime type: choose `Catalyst-Managed Runtime`
- When asked for runtime: choose `Python`
- When asked for build path: point to your app's root directory (e.g., `.` or `./src`)

### Step 3 — Configure app-config.json

Catalyst creates `app-config.json` in your app directory. Edit it:

```json
{
  "name": "my-python-app",
  "stack": "python",
  "command": "python main.py",
  "port": null,
  "memory": 256,
  "env_variables": {
    "FLASK_ENV": "production",
    "ZOHO_ORG_ID": "your_org_id_here"
  }
}
```

**Startup command examples by framework:**

| Framework | `command` value |
|---|---|
| Flask | `python app.py` or `python main.py` |
| Django | `python manage.py runserver 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT` |
| Gunicorn + Flask | `gunicorn -b 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT app:app` |
| Uvicorn + FastAPI | `uvicorn main:app --host 0.0.0.0 --port $X_ZOHO_CATALYST_LISTEN_PORT` |
| Tornado | `python server.py` |
| Bottle | `python app.py` |

> **Shell commands note:** Startup commands in AppSail are executed directly without a shell. If your command uses `$VAR` substitution, wrap it: `sh -c "gunicorn -b 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT app:app"`

### Step 4 — Ensure Build Directory is Complete

Before deploying, verify your build directory contains:
- [ ] Main Python file (e.g., `app.py`, `main.py`, `wsgi.py`)
- [ ] `requirements.txt` with all dependencies
- [ ] Any config files (e.g., `settings.py` for Django, `.env` is NOT needed — use env_variables in app-config.json)
- [ ] Static files / templates if needed

> **File write restriction:** Catalyst restricts writes to the current directory at runtime. Write temp files to the OS temp directory (`/tmp`) instead.

### Step 5 — Deploy
```bash
# Deploy everything (functions + client + AppSail)
catalyst deploy

# Or deploy AppSail only
catalyst appsail:deploy
```

After deploy, the CLI prints your app's **endpoint URL**. Copy it.

### Step 6 — Verify
Open the endpoint URL in a browser or test with curl:
```bash
curl https://your-app-name-<hash>.zohoappsail.com/
```

---

## Redeploy

For subsequent code changes, just redeploy from your project directory:

```bash
cd my-python-app
catalyst deploy
```

Or AppSail only (faster if only app code changed):
```bash
catalyst appsail:deploy
```

---

## Promote to Production

CLI deploys go to **development environment** only. To go live:

1. Open [console.catalyst.zoho.com](https://console.catalyst.zoho.com)
2. Navigate to your project
3. Click **Deploy to Production** (top-right banner or Deployment menu)
4. Confirm the promotion

> Production restricts creating new functions/tables/configs. All changes must be done in development first, then promoted.

---

## Environment Variables

### In app-config.json (before deploy)
```json
{
  "env_variables": {
    "DB_HOST": "your-db-host",
    "ZOHO_CLIENT_ID": "your-client-id",
    "ZOHO_REFRESH_TOKEN": "your-refresh-token"
  }
}
```

### In the Console (after deploy)
Go to: **Serverless → AppSail → [your service] → Configurations → Environment Variables**

> Variables are environment-specific. Set different values for development and production separately.

### Access in Python code
```python
import os

zoho_token = os.environ.get('ZOHO_REFRESH_TOKEN')
db_host = os.environ.get('DB_HOST', 'localhost')
```

---

## requirements.txt

Always include a `requirements.txt`. Example for Flask + Zoho integration:

```text
flask==3.0.0
requests==2.31.0
gunicorn==21.2.0
python-dotenv==1.0.0
```

Generate from your current env:
```bash
pip freeze > requirements.txt
```

Or minimally with `pipreqs`:
```bash
pip install pipreqs
pipreqs . --force
```

---

## Troubleshooting

Read [references/troubleshooting.md](references/troubleshooting.md) for detailed solutions.

**Quick fixes:**

| Symptom | Likely cause | Fix |
|---|---|---|
| App deploys but won't start | Port not bound to `X_ZOHO_CATALYST_LISTEN_PORT` | Fix port binding — see [Port Binding](#critical-port-binding) |
| `ModuleNotFoundError` at runtime | Dependency missing from `requirements.txt` | Add missing package to `requirements.txt` and redeploy |
| `Permission denied` writing file | Writing to current directory | Write to `/tmp` instead |
| 502 Bad Gateway immediately | App crashes on startup | Check startup command; test locally first |
| `catalyst deploy` fails | `catalyst.json` missing or corrupt | Run `catalyst init` again from project root |
| App works in dev, fails in prod | Env variables not set for production | Set env vars in console for production environment |

---

## Deploy via Console (No CLI)

Alternative if CLI is not available:

1. Go to **console.catalyst.zoho.com → Serverless → AppSail**
2. Click **Deploy from Console**
3. Select **Catalyst-Managed Runtime** → **Python**
4. Enter a service name and startup command
5. Upload a ZIP of your build directory (do NOT include `app-config.json` — it's CLI-only)
6. Configure environment variables
7. Click **Deploy**

---

## Further Reading

- [references/zoho-integration.md](references/zoho-integration.md) — Connecting to Zoho Books, CRM, Desk from your app
- [references/troubleshooting.md](references/troubleshooting.md) — Detailed error solutions and debug tips
- [references/frameworks.md](references/frameworks.md) — Framework-specific startup commands and gotchas

---

*Based on Zoho Catalyst AppSail documentation — last synced May 2026*
