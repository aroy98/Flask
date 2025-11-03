# 🧭 Flask Mastery Series – Part III: Routing, Blueprints & Application Structure

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Routing Fundamentals](#routing-fundamentals)

   * [Defining Routes](#defining-routes)
   * [Dynamic Routes](#dynamic-routes)
   * [URL Building with `url_for`](#url-building-with-url_for)
   * [Redirects & HTTP Methods](#redirects--http-methods)
3. [Error Handling & Custom Responses](#error-handling--custom-responses)
4. [Blueprints — Modular Application Design](#blueprints--modular-application-design)

   * [Creating and Registering Blueprints](#creating-and-registering-blueprints)
   * [Organizing API and Web Modules](#organizing-api-and-web-modules)
5. [The Application Factory Pattern](#the-application-factory-pattern)
6. [Config Management with Environments](#config-management-with-environments)
7. [Project Folder Structure Example](#project-folder-structure-example)
8. [Best Practices for Large Flask Apps](#best-practices-for-large-flask-apps)
9. [Next Steps](#next-steps)

---

## 🧠 Introduction

Flask’s flexibility allows small apps to scale into large modular systems. In this part, you’ll learn **routing best practices**, **blueprint modularization**, and the **application factory pattern** — the foundation for scalable Flask architectures.

---

## 🛣️ Routing Fundamentals

### Defining Routes

A route binds a URL to a function.

```py
@app.route('/')
def home():
    return 'Welcome to Flask Mastery!'
```

### Dynamic Routes

Capture URL variables using angle brackets:

```py
@app.route('/user/<username>')
def profile(username):
    return f'Hello, {username}!'

@app.route('/post/<int:post_id>')
def post(post_id):
    return f'Viewing post #{post_id}'
```

> You can specify types: `string`, `int`, `float`, `path`, `uuid`.

---

### URL Building with `url_for`

Never hardcode URLs — use Flask’s `url_for()` to generate them dynamically.

```py
from flask import url_for, redirect

@app.route('/redirect-home')
def redirect_home():
    return redirect(url_for('home'))
```

This ensures that if route names or paths change, references stay valid.

In templates:

```html
<a href="{{ url_for('profile', username='akash') }}">Visit Profile</a>
```

---

### Redirects & HTTP Methods

Specify allowed HTTP methods:

```py
@app.route('/submit', methods=['POST'])
def submit():
    return 'Form submitted!'
```

Redirect example:

```py
from flask import redirect

@app.route('/old')
def old():
    return redirect(url_for('new'))
```

---

## ⚠️ Error Handling & Custom Responses

Use decorators to catch specific HTTP errors.

```py
@app.errorhandler(404)
def not_found(e):
    return {'error': 'Not Found'}, 404

@app.errorhandler(500)
def server_error(e):
    return {'error': 'Internal Server Error'}, 500
```

Custom error templates:

```
templates/
 ├── 404.html
 └── 500.html
```

```py
@app.errorhandler(404)
def not_found(e):
    return render_template('404.html'), 404
```

---

## 🧩 Blueprints — Modular Application Design

Blueprints allow splitting large apps into components (auth, API, admin, etc.).

### Creating and Registering Blueprints

**1. Create Blueprint:**

```py
# app/auth/__init__.py
from flask import Blueprint

auth_bp = Blueprint('auth', __name__, url_prefix='/auth')

from . import routes
```

**2. Define routes:**

```py
# app/auth/routes.py
from . import auth_bp
from flask import render_template, redirect, url_for

@auth_bp.route('/login')
def login():
    return render_template('auth/login.html')

@auth_bp.route('/logout')
def logout():
    return redirect(url_for('auth.login'))
```

**3. Register Blueprint in main app:**

```py
# app/__init__.py
from flask import Flask
from app.auth import auth_bp

def create_app():
    app = Flask(__name__)
    app.register_blueprint(auth_bp)
    return app
```

Run app:

```bash
flask --app app:create_app run
```

---

### Organizing API and Web Modules

Typical blueprint-based separation:

```
app/
├── __init__.py
├── api/
│   ├── __init__.py
│   └── routes.py
├── web/
│   ├── __init__.py
│   └── routes.py
└── auth/
    ├── __init__.py
    └── routes.py
```

Each blueprint handles a clear domain (API endpoints, templates, or authentication).

---

## 🏗️ The Application Factory Pattern

Factory pattern dynamically creates app instances with configuration flexibility.

```py
# app/__init__.py
def create_app(config_name='config.DevConfig'):
    app = Flask(__name__)
    app.config.from_object(config_name)

    from app.api import api_bp
    from app.auth import auth_bp
    app.register_blueprint(api_bp)
    app.register_blueprint(auth_bp)

    return app
```

Advantages:

* Environment-specific apps (dev/stage/prod)
* Easy testing and configuration overrides
* Lazy initialization of extensions

Run:

```bash
flask --app app:create_app run --debug
```

---

## ⚙️ Config Management with Environments

```py
# config.py
class BaseConfig:
    SECRET_KEY = 'supersecret'
    DEBUG = False

class DevConfig(BaseConfig):
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = 'sqlite:///dev.db'

class ProdConfig(BaseConfig):
    SQLALCHEMY_DATABASE_URI = 'postgresql://user:pass@db:5432/app'
```

Switch environment dynamically:

```bash
export FLASK_CONFIG=config.ProdConfig
flask run
```

---

## 📁 Project Folder Structure Example

```
flask-mastery/
├── app/
│   ├── __init__.py
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes.py
│   ├── auth/
│   │   ├── __init__.py
│   │   └── routes.py
│   ├── web/
│   │   ├── __init__.py
│   │   └── routes.py
│   ├── templates/
│   └── static/
├── config.py
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

---

## 🧱 Best Practices for Large Flask Apps

✅ Use blueprints for modularization.
✅ Always use an app factory for flexible config loading.
✅ Keep routes lean — move logic to service or model layers.
✅ Use environment-based configs and `.env` files.
✅ Follow a consistent naming convention for blueprints and folders.

---

## 🚀 Next Steps

Continue to **Part IV – Databases & ORM (SQLAlchemy + Flask-Migrate)**, where you’ll integrate a database layer, manage models, and handle migrations.

---

*Prepared for the Flask Mastery Series — exportable in `.md`, `.pdf`, and `.docx` formats.*
