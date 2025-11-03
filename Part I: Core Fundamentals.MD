# 🐍 Flask Mastery Series – Part I: Core Fundamentals

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Quick Setup — Local + Docker (Recommended)](#quick-setup---local--docker-recommended)

   * [Prerequisites](#prerequisites)
   * [Create a Python virtual environment (local)](#create-a-python-virtual-environment-local)
   * [Install Flask and create first app (`app.py`)](#install-flask-and-create-first-app-apppy)
   * [Run with `flask run` (development)](#run-with-flask-run-development)
   * [Dockerize the app (Dockerfile + docker-compose)](#dockerize-the-app-dockerfile--docker-compose)
3. [Flask App Anatomy & Request Lifecycle](#flask-app-anatomy--request-lifecycle)
4. [Routing & View Functions](#routing--view-functions)
5. [Templates & Static Files (Jinja2)](#templates--static-files-jinja2)
6. [Configuration & Environments](#configuration--environments)
7. [Project Structure (Factory Pattern & Blueprints)](#project-structure-factory-pattern--blueprints)
8. [Development Tips & Debugging](#development-tips--debugging)
9. [Production Considerations (Gunicorn, Reverse Proxy)](#production-considerations-gunicorn-reverse-proxy)
10. [Next Steps & Suggested Learning Path](#next-steps--suggested-learning-path)

---

## 🧠 Introduction

Welcome to **Flask Mastery — Part I**. This part gets you from zero to a working, dockerized Flask app ready for local development and production-grade deployment patterns. We'll start with a quick environment setup, show the minimal app, then cover core concepts like routing, templates, configuration, and structuring your project for growth.

---

## ⚙️ Quick Setup — Local + Docker (Recommended)

### Prerequisites

* Python 3.10+ installed. Verify with:

```bash
python --version
```

* Docker & Docker Compose installed (for the docker section). Verify with:

```bash
docker --version
docker-compose --version
```

> Tip: Use `pyenv` on macOS/Linux to manage Python versions if you need multiple versions.

### Create a Python virtual environment (local)

```bash
# create project folder
mkdir flask-mastery && cd flask-mastery

# create venv
python -m venv .venv

# activate (macOS / Linux)
source .venv/bin/activate

# activate (Windows PowerShell)
.\.venv\Scripts\Activate.ps1

# upgrade pip
pip install --upgrade pip
```

### Install Flask and create first app (`app.py`)

```bash
pip install Flask gunicorn
```

Create `app.py`:

```py
# app.py
from flask import Flask, jsonify, request, render_template

app = Flask(__name__)

@app.route('/')
def index():
    return render_template('index.html', message='Hello from Flask!')

@app.route('/api/hello', methods=['GET'])
def hello_api():
    name = request.args.get('name', 'World')
    return jsonify({
        'message': f'Hello, {name}!'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

Create minimal template `templates/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>Flask Mastery - Part I</title>
  </head>
  <body>
    <h1>{{ message }}</h1>
    <p>Try the API: <a href="/api/hello?name=Akash">/api/hello?name=Akash</a></p>
  </body>
</html>
```

### Run with `flask run` (development)

Set the `FLASK_APP` env var and run:

```bash
# macOS / Linux
export FLASK_APP=app.py
export FLASK_ENV=development  # enables debug & reloader (deprecated in newer Flask; use --debug)
flask run --host=0.0.0.0 --port=5000

# Windows PowerShell
$env:FLASK_APP = 'app.py'
$env:FLASK_ENV = 'development'
flask run --host=0.0.0.0 --port=5000
```

Or run the module directly for development (recommended for simple projects):

```bash
python app.py
```

Visit `http://localhost:5000` and `http://localhost:5000/api/hello`.

---

## 🐳 Dockerize the app (Dockerfile + docker-compose)

Dockerizing gives you consistent environments and easy deployment. Add the following files.

### `Dockerfile`

```dockerfile
# Use an official lightweight Python runtime as a parent image
FROM python:3.11-slim

# Set environment
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Create and set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
  && rm -rf /var/lib/apt/lists/*

# Copy only requirements first for cache
COPY requirements.txt /app/
RUN pip install --upgrade pip
RUN pip install -r requirements.txt

# Copy app code
COPY . /app

# Expose port
EXPOSE 5000

# Run the app with gunicorn (production ready)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app", "--workers", "2"]
```

Create `requirements.txt`:

```
Flask>=2.3
gunicorn
```

### `docker-compose.yml`

```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/app
    environment:
      - FLASK_ENV=development
    command: python app.py
```

Run with:

```bash
docker-compose up --build
```

Notes:

* For development, mounting the project (`volumes`) enables live code editing.
* For production, build a static image (no volumes) and run `gunicorn` directly.

---

## 🔁 Flask App Anatomy & Request Lifecycle

High-level flow for each HTTP request:

1. **Client** sends HTTP request → reaches web server (Nginx, or Docker port mapping)
2. **WSGI server** (Gunicorn/uWSGI) receives request and calls Flask app
3. **Flask** matches the request URL to a view (route)
4. **View function** executes, may access DB or call other services
5. **Response** (HTML/JSON/redirect) is returned to WSGI server
6. **WSGI server** returns HTTP response to client

Key components:

* **Werkzeug**: HTTP utilities & request/response objects
* **Jinja2**: templating engine
* **WSGI**: interface to connect Flask app with servers like Gunicorn

---

## 🧭 Routing & View Functions

Basic route examples:

```py
@app.route('/items', methods=['GET'])
def list_items():
    # return an HTML template or JSON
    return jsonify(items=[1, 2, 3])

@app.route('/items/<int:item_id>', methods=['GET'])
def get_item(item_id):
    return jsonify({ 'id': item_id, 'name': f'Item {item_id}' })

@app.route('/submit', methods=['POST'])
def submit():
    data = request.get_json() or request.form
    # validate and process
    return jsonify({'status': 'ok'})
```

Best practices:

* Use explicit HTTP methods (`methods=['GET', 'POST']`) rather than relying on defaults.
* Validate inputs early and return consistent error responses (see `errorhandler` below).
* Keep view functions thin — delegate business logic to services/modules.

Error handler example:

```py
from flask import jsonify

@app.errorhandler(404)
def not_found(e):
    return jsonify({ 'error': 'not_found' }), 404

@app.errorhandler(500)
def internal_error(e):
    return jsonify({ 'error': 'internal_server_error' }), 500
```

---

## 🖼️ Templates & Static Files (Jinja2)

* Templates go into `templates/` and static assets into `static/`.
* Use template inheritance to centralize layout.

Example layout `templates/base.html`:

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>{% block title %}Flask Mastery{% endblock %}</title>
</head>
<body>
  <header>
    <h1>My App</h1>
  </header>

  <main>
    {% block content %}{% endblock %}
  </main>

  <footer>
    <small>© 2025</small>
  </footer>
</body>
</html>
```

Then `templates/index.html`:

```html
{% extends 'base.html' %}

{% block title %}Home{% endblock %}

{% block content %}
  <h2>{{ message }}</h2>
{% endblock %}
```

Serving static files:

```html
<link rel="stylesheet" href="{{ url_for('static', filename='css/main.css') }}">
```

---

## ⚙️ Configuration & Environments

Use a configuration pattern to separate environments:

```py
# config.py
class BaseConfig:
    SECRET_KEY = 'replace-me'
    DEBUG = False

class DevConfig(BaseConfig):
    DEBUG = True

class ProdConfig(BaseConfig):
    DEBUG = False
```

Factory pattern to load config:

```py
# factory_app.py
from flask import Flask

def create_app(config_class='config.DevConfig'):
    app = Flask(__name__)
    app.config.from_object(config_class)

    # register blueprints here

    return app
```

Load with environment variable:

```bash
export FLASK_CONFIG='config.ProdConfig'
```

Secrets management:

* Don’t store secrets in source control.
* Use environment variables, a vault, or platform-specific secret stores (AWS Secrets Manager, GCP Secret Manager).

---

## 🏗️ Project Structure (Factory Pattern & Blueprints)

Recommended structure for medium/large apps:

```
flask-mastery/
├─ app/
│  ├─ __init__.py        # create_app factory
│  ├─ routes.py
│  ├─ models.py
│  ├─ extensions.py      # db, migrate, login, etc.
│  ├─ api/               # blueprint for API endpoints
│  └─ web/               # blueprint for web (templates)
├─ migrations/
├─ tests/
├─ requirements.txt
├─ Dockerfile
├─ docker-compose.yml
└─ README.md
```

Basic blueprint example:

```py
# app/api/__init__.py
from flask import Blueprint

api_bp = Blueprint('api', __name__, url_prefix='/api')

from . import views

# app/api/views.py
from . import api_bp
from flask import jsonify

@api_bp.route('/hello')
def hello():
    return jsonify({ 'message': 'hello from api' })
```

Register blueprint in factory:

```py
# app/__init__.py
from flask import Flask
from .api import api_bp

def create_app(config_class='config.DevConfig'):
    app = Flask(__name__)
    app.config.from_object(config_class)

    app.register_blueprint(api_bp)

    return app
```

---

## 🐞 Development Tips & Debugging

* Use `flask shell` for REPL access to app context:

```bash
flask shell
>>> from app import create_app
>>> app = create_app()
>>> app.app_context().push()
```

* Use the Flask Debug Toolbar during development:

```bash
pip install flask-debugtoolbar
```

* Helpful extensions:

  * `Flask-Migrate` (Alembic) for DB migrations
  * `Flask-SQLAlchemy` for ORM
  * `Flask-WTF` for forms and CSRF protection
  * `Flask-Login` for session management

---

## 🚀 Production Considerations (Gunicorn, Reverse Proxy)

Recommended production stack:

* **Gunicorn** as WSGI server
* **Nginx** as reverse proxy (TLS termination, static serving, caching)
* **Supervisor / systemd** for process management (on VMs)

Example `gunicorn` command:

```bash
gunicorn --workers 4 --bind 0.0.0.0:5000 app:app
```

Nginx simple config:

```nginx
server {
    listen 80;
    server_name example.com;

    location /static/ {
        alias /var/www/app/static/;
        expires 30d;
    }

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Scaling notes:

* Tune Gunicorn worker count based on CPU/memory and sync vs async worker class.
* Use a process manager and health checks.
* For heavier workloads, use multiple app instances behind a load balancer.

---

## ➕ Next Steps & Suggested Learning Path

After finishing Part I, continue with these focused parts:

* Part II – Templates & Forms (Jinja2, Flask-WTF)
* Part III – Databases & Migrations (SQLAlchemy, Flask-Migrate)
* Part IV – REST APIs & Validation (Marshmallow, Flask-Smorest)
* Part V – Auth & Security (JWT, OAuth2, CSRF)
* Part VI – Testing & CI/CD (pytest, GitHub Actions)
* Part VII – Background Tasks & Real-time (Celery, Flask-SocketIO)
* Part VIII – Deployment & Observability (Docker, Kubernetes, Prometheus)

---

## ✅ Summary

Part I covered:

* Local Python environment setup (virtualenv) and running Flask
* Basic app structure with routes and templates
* Dockerizing the app with `Dockerfile` and `docker-compose`
* Request lifecycle, configuration, blueprints, and production tips

You're now ready to build feature-focused parts that expand on databases, testing, auth, async tasks, and deployment.

---

*Prepared in the style of the Flask Mastery Series. Exportable to `.md`, `.pdf`, and `.docx`.*
