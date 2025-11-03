# 🌐 Flask Mastery Series – Part V: RESTful API Development (Validation, Versioning, Docs)

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Designing RESTful APIs — Principles](#designing-restful-apis--principles)
3. [Flask Tooling for APIs (Flask-RESTful, Flask-Smorest, Flask-RESTX)](#flask-tooling-for-apis-flask-restful-flask-smorest-flask-restx)
4. [Request Parsing & Validation (Marshmallow)](#request-parsing--validation-marshmallow)
5. [Serializers & Response Formatting](#serializers--response-formatting)
6. [Error Handling & Consistent Responses](#error-handling--consistent-responses)
7. [Pagination, Filtering & Sorting for APIs](#pagination-filtering--sorting-for-apis)
8. [Versioning Strategies](#versioning-strategies)
9. [Authentication & Rate Limiting for APIs](#authentication--rate-limiting-for-apis)
10. [Documentation & OpenAPI (Swagger)](#documentation--openapi-swagger)
11. [Testing & Contract Tests](#testing--contract-tests)
12. [API Performance & Caching](#api-performance--caching)
13. [Hypermedia (HATEOAS) & Best Practices](#hypermedia-hateoas--best-practices)
14. [Next Steps](#next-steps)

---

## 🧠 Introduction

This part covers building robust, well-documented RESTful APIs with Flask. You’ll learn validation with **Marshmallow**, API frameworks like **Flask-Smorest**, documentation via **OpenAPI/Swagger**, versioning strategies, testing practices, and performance patterns essential for production APIs.

---

## 🏛️ Designing RESTful APIs — Principles

* Use **nouns** for resources (`/users`, `/orders`), not verbs.
* Use HTTP verbs to describe actions (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`).
* Use proper **status codes** (`200`, `201`, `204`, `400`, `401`, `404`, `429`, `500`).
* Make APIs **discoverable** and **documented**.
* Keep responses **consistent** (same envelope structure for errors/success).

---

## 🧰 Flask Tooling for APIs (Flask-RESTful, Flask-Smorest, Flask-RESTX)

Popular libraries:

* **Flask-RESTful** — lightweight resource-based routing.
* **Flask-Smorest** — Marshmallow-friendly, auto OpenAPI generation.
* **Flask-RESTX** — fork of Flask-RESTPlus for Swagger docs.

### Example with Flask-Smorest

Install:

```bash
pip install flask-smorest marshmallow
```

Basic usage:

```py
from flask import Flask
from flask_smorest import Api, Blueprint
from marshmallow import Schema, fields

app = Flask(__name__)
api = Api(app)

class UserSchema(Schema):
    id = fields.Int(dump_only=True)
    name = fields.Str(required=True)

bp = Blueprint('users', 'users', url_prefix='/api/users', description='Users operations')

@bp.route('/')
class UsersResource:
    @bp.response(UserSchema(many=True))
    def get(self):
        return []

api.register_blueprint(bp)
```

Flask-Smorest auto-generates OpenAPI specs based on schemas and decorators.

---

## 🧾 Request Parsing & Validation (Marshmallow)

Define input/output schemas with Marshmallow.

```py
from marshmallow import Schema, fields, validate

class CreateUserSchema(Schema):
    name = fields.Str(required=True, validate=validate.Length(min=1))
    email = fields.Email(required=True)
```

Use in a route:

```py
@bp.route('/')
class UsersResource:
    @bp.arguments(CreateUserSchema)
    @bp.response(UserSchema)
    def post(self, new_user):
        # new_user is validated dict
        user = create_user(new_user)
        return user, 201
```

Benefits:

* Single place for validation and serialization.
* Clear error messages returned to clients.

---

## 🔁 Serializers & Response Formatting

Keep one serializer per resource: `UserSchema`, `PostSchema`.
Use `dump_only` and `load_only` for controlling fields.

Example:

```py
class UserSchema(Schema):
    id = fields.Int(dump_only=True)
    password = fields.Str(load_only=True)
```

Response envelope pattern (optional):

```json
{ "status": "ok", "data": {...} }
{ "status": "error", "error": {"message": "...", "code": 400} }
```

---

## 🚨 Error Handling & Consistent Responses

Standardize error format and handle HTTP exceptions:

```py
from werkzeug.exceptions import HTTPException

@app.errorhandler(HTTPException)
def handle_http_exception(e):
    response = e.get_response()
    response.data = json.dumps({
        'status': 'error',
        'error': { 'code': e.code, 'message': e.description }
    })
    response.content_type = 'application/json'
    return response
```

Return validation errors with `400` and explain fields.

---

## 📚 Pagination, Filtering & Sorting for APIs

API should support pagination parameters: `page`, `per_page`, `limit`, `offset`.

Example with Flask-SQLAlchemy paginate:

```py
page = int(request.args.get('page', 1))
per_page = min(int(request.args.get('per_page', 20)), 100)
q = User.query
q = q.filter(User.active.is_(True))
q = q.order_by(User.created_at.desc())
page_obj = q.paginate(page=page, per_page=per_page)

return {
  'items': UserSchema(many=True).dump(page_obj.items),
  'meta': {
    'page': page_obj.page,
    'per_page': page_obj.per_page,
    'total': page_obj.total,
    'pages': page_obj.pages
  }
}
```

For filtering, use query params or an advanced query language (RQL, GraphQL).

---

## 🧭 Versioning Strategies

Common approaches:

* **URI versioning**: `/v1/users` (simple, explicit)
* **Header versioning**: `Accept: application/vnd.myapp.v1+json`
* **Media type versioning**: similar to header versioning
* **Query param**: `?version=1` (less common)

Recommendation: URI versioning for public REST APIs.

---

## 🔐 Authentication & Rate Limiting for APIs

Auth options:

* **JWT** (Flask-JWT-Extended)
* **OAuth2** (Auth0, Google, Okta)
* **API Keys** for service-to-service

Rate limiting (protect from abuse):

```bash
pip install Flask-Limiter
```

Example:

```py
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
limiter.init_app(app)

@app.route('/api/login')
@limiter.limit('5 per minute')
def login():
    pass
```

Also implement throttling and monitoring for abusive patterns.

---

## 📖 Documentation & OpenAPI (Swagger)

Auto-generate OpenAPI with Flask-Smorest / Flask-RESTX.
Provide an interactive Swagger UI for developers.

Example Flask-Smorest config:

```py
app.config.update({
    'API_TITLE': 'My API',
    'API_VERSION': 'v1',
    'OPENAPI_VERSION': '3.0.2'
})
api = Api(app)
```

Host Swagger UI for easy testing and client generation.

---

## 🧪 Testing & Contract Tests

* Unit test schemas & serialization.
* Integration tests using test client against a test DB.
* Contract tests ensure API matches spec (use `schemathesis` or `pytest-openapi`).

Example test with pytest:

```py
def test_create_user(client):
    res = client.post('/api/users', json={'name':'A', 'email':'a@example.com'})
    assert res.status_code == 201
    data = res.get_json()
    assert data['name'] == 'A'
```

Use CI to run contract tests and fail if API diverges from OpenAPI spec.

---

## ⚡ API Performance & Caching

* Use **HTTP caching headers** (`Cache-Control`, `ETag`) for cacheable endpoints.
* Use **Redis** or CDN for caching expensive responses.
* Implement conditional GETs with `If-None-Match`/`ETag` to save bandwidth.

Example ETag usage:

```py
from flask import make_response

resp = make_response(jsonify(payload))
resp.set_etag('some-hash')
return resp
```

---

## 🔗 Hypermedia (HATEOAS) & Best Practices

HATEOAS adds navigational links to responses:

```json
{
  "id": 1,
  "name": "Alice",
  "links": [
    { "rel": "self", "href": "/api/users/1" },
    { "rel": "posts", "href": "/api/users/1/posts" }
  ]
}
```

While powerful, HATEOAS is not required for all APIs — weigh complexity vs benefit.

---

## ✅ Best Practices Summary

* Validate inputs and use clear error messages.
* Version your API early.
* Provide OpenAPI docs and run contract tests.
* Protect endpoints with auth and rate limits.
* Cache carefully and add HTTP caching headers.

---

## 🚀 Next Steps

Proceed to **Part VI – Authentication & Authorization** for deep coverage of JWTs, OAuth2, RBAC, and secure session management.

---

*Prepared for the Flask Mastery Series — exportable in `.md`, `.pdf`, and `.docx` formats.*
