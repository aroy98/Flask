# 🔐 Flask Mastery Series – Part VI: Authentication & Authorization (JWT, OAuth2, RBAC)

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Authentication vs Authorization](#authentication-vs-authorization)
3. [Flask-Login — Session-Based Auth](#flask-login--session-based-auth)

   * [Setup](#setup)
   * [User Model & Login Manager](#user-model--login-manager)
   * [Login & Logout Routes](#login--logout-routes)
4. [JWT Authentication with Flask-JWT-Extended](#jwt-authentication-with-flask-jwt-extended)

   * [Setup & Configuration](#setup--configuration)
   * [Creating Access & Refresh Tokens](#creating-access--refresh-tokens)
   * [Protecting Routes](#protecting-routes)
   * [Token Refresh & Revocation](#token-refresh--revocation)
5. [OAuth2 / OpenID Connect Integration](#oauth2--openid-connect-integration)

   * [Google OAuth Example](#google-oauth-example)
   * [Auth0 / Okta Setup](#auth0--okta-setup)
6. [Role-Based Access Control (RBAC)](#role-based-access-control-rbac)

   * [Role & Permission Models](#role--permission-models)
   * [Decorator-Based Access Control](#decorator-based-access-control)
7. [API Key Authentication (Service-to-Service)](#api-key-authentication-service-to-service)
8. [Security Best Practices](#security-best-practices)
9. [Logout & Session Management](#logout--session-management)
10. [Testing Auth Flows](#testing-auth-flows)
11. [Next Steps](#next-steps)

---

## 🧠 Introduction

This part explains how to secure Flask applications using **Flask-Login**, **JWT-based tokens**, **OAuth2**, and **role-based access control (RBAC)**. You’ll also learn session management, token revocation, and third-party authentication integrations.

---

## 🔑 Authentication vs Authorization

| Concept            | Description                   | Example                      |
| ------------------ | ----------------------------- | ---------------------------- |
| **Authentication** | Verifies user identity        | Login via email/password     |
| **Authorization**  | Determines user’s permissions | Only admins can delete users |

---

## 🧭 Flask-Login — Session-Based Auth

### Setup

Install:

```bash
pip install Flask-Login
```

Initialize in `app/__init__.py`:

```py
from flask_login import LoginManager

login_manager = LoginManager()
login_manager.login_view = 'auth.login'
login_manager.init_app(app)
```

### User Model & Login Manager

Add `UserMixin` to your model:

```py
from flask_login import UserMixin
from . import db

class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(255), unique=True, nullable=False)
    password = db.Column(db.String(255), nullable=False)
    role = db.Column(db.String(50), default='user')
```

Load users by ID:

```py
@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))
```

### Login & Logout Routes

```py
from flask import render_template, redirect, url_for, flash
from flask_login import login_user, logout_user, login_required
from werkzeug.security import check_password_hash

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        if user and check_password_hash(user.password, form.password.data):
            login_user(user)
            return redirect(url_for('dashboard'))
        flash('Invalid credentials', 'error')
    return render_template('login.html', form=form)

@app.route('/logout')
@login_required
def logout():
    logout_user()
    flash('Logged out successfully!')
    return redirect(url_for('auth.login'))
```

---

## 🧾 JWT Authentication with Flask-JWT-Extended

### Setup & Configuration

Install:

```bash
pip install Flask-JWT-Extended
```

In `app/__init__.py`:

```py
from flask_jwt_extended import JWTManager

app.config['JWT_SECRET_KEY'] = 'super-secret'
jwt = JWTManager(app)
```

---

### Creating Access & Refresh Tokens

```py
from flask_jwt_extended import create_access_token, create_refresh_token

@app.route('/api/login', methods=['POST'])
def login_api():
    data = request.get_json()
    user = User.query.filter_by(email=data['email']).first()
    if user and check_password_hash(user.password, data['password']):
        access_token = create_access_token(identity=user.id)
        refresh_token = create_refresh_token(identity=user.id)
        return jsonify(access_token=access_token, refresh_token=refresh_token)
    return jsonify({'error': 'Invalid credentials'}), 401
```

---

### Protecting Routes

```py
from flask_jwt_extended import jwt_required, get_jwt_identity

@app.route('/api/me')
@jwt_required()
def me():
    user_id = get_jwt_identity()
    user = User.query.get(user_id)
    return jsonify(id=user.id, email=user.email)
```

---

### Token Refresh & Revocation

```py
from flask_jwt_extended import jwt_refresh_token_required, create_access_token

@app.route('/api/refresh', methods=['POST'])
@jwt_refresh_token_required
def refresh():
    identity = get_jwt_identity()
    return jsonify(access_token=create_access_token(identity=identity))
```

To revoke tokens, maintain a denylist in Redis or DB.

---

## 🔓 OAuth2 / OpenID Connect Integration

Use **Authlib** or **Flask-Dance**.

Install:

```bash
pip install authlib
```

### Google OAuth Example

```py
from authlib.integrations.flask_client import OAuth

oauth = OAuth(app)
google = oauth.register(
    'google',
    client_id='YOUR_CLIENT_ID',
    client_secret='YOUR_CLIENT_SECRET',
    authorize_url='https://accounts.google.com/o/oauth2/auth',
    access_token_url='https://oauth2.googleapis.com/token',
    api_base_url='https://www.googleapis.com/oauth2/v1/',
    client_kwargs={'scope': 'openid email profile'}
)

@app.route('/login/google')
def login_google():
    redirect_uri = url_for('authorize_google', _external=True)
    return google.authorize_redirect(redirect_uri)

@app.route('/authorize/google')
def authorize_google():
    token = google.authorize_access_token()
    user_info = google.get('userinfo').json()
    # Create or find user in DB
    return jsonify(user_info)
```

---

### Auth0 / Okta Setup

For enterprise apps, integrate with **Auth0**, **Okta**, or **Azure AD**.
Use JWT verification middleware to validate ID tokens.

```bash
pip install python-jose
```

Validate JWT signature and claims using JWKS endpoints.

---

## 🧩 Role-Based Access Control (RBAC)

### Role & Permission Models

```py
class Role(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True)

class UserRole(db.Model):
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    role_id = db.Column(db.Integer, db.ForeignKey('role.id'))
```

### Decorator-Based Access Control

```py
from functools import wraps
from flask_jwt_extended import get_jwt_identity

def role_required(role_name):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            user_id = get_jwt_identity()
            user = User.query.get(user_id)
            if user.role != role_name:
                return jsonify({'error': 'Access denied'}), 403
            return fn(*args, **kwargs)
        return wrapper
    return decorator

@app.route('/api/admin')
@jwt_required()
@role_required('admin')
def admin_dashboard():
    return jsonify({'message': 'Welcome admin!'})
```

---

## 🔑 API Key Authentication (Service-to-Service)

Use API keys for internal services.

```py
from functools import wraps

def api_key_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        key = request.headers.get('X-API-KEY')
        if key != current_app.config['INTERNAL_API_KEY']:
            return jsonify({'error': 'Unauthorized'}), 401
        return fn(*args, **kwargs)
    return wrapper
```

Use `X-API-KEY` header or HMAC signature for stronger security.

---

## 🛡️ Security Best Practices

✅ Store passwords hashed (use `werkzeug.security.generate_password_hash`).
✅ Use HTTPS for all API endpoints.
✅ Implement CSRF protection for form-based sessions.
✅ Use short-lived access tokens with refresh tokens.
✅ Keep secret keys in environment variables or a vault.
✅ Rate-limit login endpoints.
✅ Use secure cookie settings (`HttpOnly`, `SameSite`, `Secure`).

---

## 🚪 Logout & Session Management

For JWT:

* Client deletes token on logout.
* Maintain a denylist for revoked tokens.

For Flask-Login sessions:

* `logout_user()` clears session.
* Invalidate session cookie via `session.clear()`.

---

## 🧪 Testing Auth Flows

Example pytest for login:

```py
def test_login_jwt(client, user):
    res = client.post('/api/login', json={'email': user.email, 'password': 'test'})
    assert res.status_code == 200
    assert 'access_token' in res.get_json()
```

Test expired token:

```py
def test_expired_token(client, expired_token):
    res = client.get('/api/me', headers={'Authorization': f'Bearer {expired_token}'})
    assert res.status_code == 401
```

---

## 🚀 Next Steps

Continue to **Part VII – Testing & CI/CD**, where you’ll automate tests, integrate CI pipelines, and enforce quality gates for secure, production-ready Flask applications.

---

*Prepared for the Flask Mastery Series — exportable in `.md`, `.pdf`, and `.docx` formats.*
