# Framework-Specific Guide for Catalyst AppSail (Python)

All Python frameworks are supported. No templates or framework-specific restrictions apply.

---

## Flask

**Startup command:** `python app.py`

```python
import os
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def index():
    return jsonify({"status": "ok"})

if __name__ == '__main__':
    port = int(os.environ.get('X_ZOHO_CATALYST_LISTEN_PORT', 5000))
    app.run(host='0.0.0.0', port=port, debug=False)
```

**requirements.txt:**
```
flask>=3.0.0
```

**With Gunicorn (recommended for production):**
Startup command: `sh -c "gunicorn -b 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT app:app"`

```
flask>=3.0.0
gunicorn>=21.0.0
```

---

## Django

**Startup command:** `sh -c "python manage.py runserver 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT"`

Key `settings.py` changes:
```python
import os

# Allow Catalyst's AppSail domain
ALLOWED_HOSTS = ['*']  # Or restrict to your AppSail domain

# Static files (serve via WhiteNoise or external CDN)
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
```

**requirements.txt:**
```
django>=5.0
gunicorn>=21.0.0
whitenoise>=6.6.0
```

**With Gunicorn:**
Startup command: `sh -c "gunicorn -b 0.0.0.0:$X_ZOHO_CATALYST_LISTEN_PORT myproject.wsgi:application"`

**Important:** Run `python manage.py collectstatic` before deploying if you have static files.

---

## FastAPI + Uvicorn

**Startup command:** `sh -c "uvicorn main:app --host 0.0.0.0 --port $X_ZOHO_CATALYST_LISTEN_PORT"`

```python
import os
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"status": "ok"}

# No if __name__ == '__main__' needed when using uvicorn command
```

**requirements.txt:**
```
fastapi>=0.110.0
uvicorn[standard]>=0.29.0
```

---

## Tornado

**Startup command:** `python server.py`

```python
import os
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write({"status": "ok"})

def make_app():
    return tornado.web.Application([
        (r"/", MainHandler),
    ])

if __name__ == "__main__":
    app = make_app()
    port = int(os.environ.get('X_ZOHO_CATALYST_LISTEN_PORT', 8888))
    app.listen(port)
    tornado.ioloop.IOLoop.current().start()
```

**requirements.txt:**
```
tornado>=6.4
```

---

## Bottle

**Startup command:** `python app.py`

```python
import os
from bottle import Bottle, run, response

app = Bottle()

@app.route('/')
def index():
    response.content_type = 'application/json'
    return '{"status": "ok"}'

if __name__ == '__main__':
    port = int(os.environ.get('X_ZOHO_CATALYST_LISTEN_PORT', 8080))
    run(app, host='0.0.0.0', port=port, debug=False)
```

**requirements.txt:**
```
bottle>=0.12.25
```

---

## CherryPy

**Startup command:** `python app.py`

```python
import os
import cherrypy

class Root:
    @cherrypy.expose
    @cherrypy.tools.json_out()
    def index(self):
        return {"status": "ok"}

if __name__ == '__main__':
    port = int(os.environ.get('X_ZOHO_CATALYST_LISTEN_PORT', 8080))
    cherrypy.config.update({
        'server.socket_host': '0.0.0.0',
        'server.socket_port': port,
    })
    cherrypy.quickstart(Root())
```

**requirements.txt:**
```
cherrypy>=18.9.0
```

---

## app-config.json Template

```json
{
  "name": "my-python-app",
  "stack": "python",
  "command": "python app.py",
  "port": null,
  "memory": 256,
  "env_variables": {
    "ZOHO_CLIENT_ID": "",
    "ZOHO_CLIENT_SECRET": "",
    "ZOHO_REFRESH_TOKEN": "",
    "ZOHO_BOOKS_ORG_ID": "",
    "ZOHO_ACCOUNTS_URL": "https://accounts.zoho.com"
  }
}
```

---

## Custom Runtime (Docker)

For frameworks or Python versions not natively supported by Catalyst, use a custom Docker runtime.

**Dockerfile example (Python 3.11 + FastAPI):**
```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV X_ZOHO_CATALYST_LISTEN_PORT=8080
EXPOSE 8080

CMD ["sh", "-c", "uvicorn main:app --host 0.0.0.0 --port $X_ZOHO_CATALYST_LISTEN_PORT"]
```

**Deploy as Docker image:**
```bash
docker build -t my-python-app .
catalyst appsail:add  # Select "Docker Image" when prompted
catalyst deploy
```

> Catalyst only supports OCI-compliant images built for **Linux AMD64 (x86-64)** platform.
> Supported registries: Docker Hub, AWS ECR, Google Artifact Registry, local Docker registry.
