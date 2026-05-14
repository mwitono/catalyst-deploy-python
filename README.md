# catalyst-deploy-python

A Cowork skill for deploying Python web applications to Zoho Catalyst AppSail.

## What this skill does

Guides you through the complete process of deploying any Python web app (Flask, Django, FastAPI, Tornado, Bottle, CherryPy, or any custom framework) to Zoho Catalyst AppSail — including:

- First-time deployments and redeployments
- Port binding configuration (the #1 cause of failed deploys)
- Environment variable management
- Promoting from development to production
- Integrating with Zoho Books, CRM, and Desk via REST API or Catalyst Connections
- Troubleshooting common deployment errors

## Skill Structure

```
catalyst-deploy-python/
├── SKILL.md                        # Main skill — workflows, checklist, quick reference
└── references/
    ├── zoho-integration.md         # Zoho Books/CRM/Desk API integration guide
    ├── frameworks.md               # Framework-specific startup commands & code examples
    └── troubleshooting.md          # Error solutions and debug tips
```

## Trigger phrases

This skill triggers when you say things like:

- "Deploy my Python app to Catalyst"
- "How do I host a Flask app on Catalyst AppSail?"
- "Deploy Django to Zoho Catalyst"
- "My Catalyst Python deployment is failing"
- "Set up Zoho Books integration in my Catalyst app"

## Supported Frameworks

| Framework | Startup command |
|---|---|
| Flask | `python app.py` |
| Django | `sh -c "python manage.py runserver 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT"` |
| FastAPI + Uvicorn | `sh -c "uvicorn main:app --host 0.0.0.0 --port $X_ZOHO_CATALYST_LISTEN_PORT"` |
| Tornado | `python server.py` |
| Bottle | `python app.py` |
| CherryPy | `python app.py` |
| Gunicorn | `sh -c "gunicorn -b 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT app:app"` |

## Critical Note

All Python apps on Catalyst AppSail **must** bind to the port provided by the `X_ZOHO_CATALYST_LISTEN_PORT` environment variable. Hardcoded ports will cause the app to fail.

```python
import os
port = int(os.environ.get('X_ZOHO_CATALYST_LISTEN_PORT', 5000))
app.run(host='0.0.0.0', port=port)
```

## Based on

- [Zoho Catalyst AppSail Documentation](https://docs.catalyst.zoho.com/en/serverless/help/appsail/introduction/)
- [Zoho Books API](https://www.zoho.com/books/api/v3/)
- Last synced: May 2026
