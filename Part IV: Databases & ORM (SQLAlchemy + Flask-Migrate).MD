# 🗄️ Flask Mastery Series – Part IV: Databases & ORM (SQLAlchemy + Flask-Migrate)

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Choosing a Database](#choosing-a-database)
3. [Flask-SQLAlchemy – Setup & Configuration](#flask-sqlalchemy--setup--configuration)
4. [Defining Models & Relationships](#defining-models--relationships)
5. [Migrations with Flask-Migrate (Alembic)](#migrations-with-flask-migrate-alembic)
6. [CRUD Patterns & Querying Efficiently](#crud-patterns--querying-efficiently)
7. [Transactions, Sessions & Concurrency](#transactions-sessions--concurrency)
8. [Pagination, Filtering & Sorting](#pagination-filtering--sorting)
9. [Indexing, Performance & Optimization](#indexing-performance--optimization)
10. [Testing Database Code](#testing-database-code)
11. [Connection Pooling & Deployment Notes](#connection-pooling--deployment-notes)
12. [Security & Data Protection](#security--data-protection)
13. [Best Practices & Pitfalls to Avoid](#best-practices--pitfalls-to-avoid)
14. [Next Steps](#next-steps)

---

## 🧠 Introduction

This part covers integrating relational databases with Flask using **Flask-SQLAlchemy** and **Flask-Migrate (Alembic)**. You’ll learn how to design models, handle relationships, write efficient queries, run migrations, and prepare your app for production database workloads.

---

## 🥣 Choosing a Database

Common choices:

* **SQLite** — Great for local dev and small prototypes.
* **PostgreSQL** — Recommended for production: reliable, powerful, supports JSON, full-text search.
* **MySQL / MariaDB** — Popular, widespread hosting support.

Use PostgreSQL for most production apps due to advanced features and stability.

---

## ⚙️ Flask-SQLAlchemy – Setup & Configuration

Install packages:

```bash
pip install Flask-SQLAlchemy Flask-Migrate psycopg2-binary
```

Example `app/__init__.py` using factory pattern:

```py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()

def create_app(config_object='config.DevConfig'):
    app = Flask(__name__)
    app.config.from_object(config_object)

    db.init_app(app)
    migrate.init_app(app, db)

    # register blueprints, etc.
    return app
```

Example `config.py`:

```py
class BaseConfig:
    SQLALCHEMY_TRACK_MODIFICATIONS = False

class DevConfig(BaseConfig):
    SQLALCHEMY_DATABASE_URI = 'sqlite:///dev.db'

class ProdConfig(BaseConfig):
    SQLALCHEMY_DATABASE_URI = 'postgresql://user:pass@db:5432/app'
```

**Important:** set `SQLALCHEMY_TRACK_MODIFICATIONS=False` to avoid extra overhead.

---

## 🧱 Defining Models & Relationships

Basic model example:

```py
from . import db
from datetime import datetime

class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(255), unique=True, nullable=False, index=True)
    name = db.Column(db.String(120), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    posts = db.relationship('Post', backref='author', lazy='dynamic')

class Post(db.Model):
    __tablename__ = 'posts'
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    body = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    published = db.Column(db.Boolean, default=False, index=True)
```

**Relationship loading strategies:**

* `lazy='select'` (default): loads when accessed
* `lazy='joined'`: joins at query time (eager)
* `lazy='subquery'`: eager load via subquery
* `lazy='dynamic'`: returns query object for further filtering

Choose strategy based on access patterns.

---

## 🔁 Migrations with Flask-Migrate (Alembic)

Initialize migrations:

```bash
flask db init
flask db migrate -m "initial"
flask db upgrade
```

Common workflow:

1. Modify models
2. `flask db migrate -m "describe change"`
3. Review the generated migration file in `migrations/`
4. `flask db upgrade` to apply

Rollback example:

```bash
flask db downgrade <revision>
```

**Tips:**

* Keep migration filenames descriptive.
* Review autogeneration; complex schema changes may need manual edits.

---

## 🔍 CRUD Patterns & Querying Efficiently

Create:

```py
u = User(email='a@example.com', name='Akash')
db.session.add(u)
db.session.commit()
```

Read:

```py
user = User.query.filter_by(email='a@example.com').first()
post = Post.query.get(1)
```

Update:

```py
user.name = 'New'
db.session.commit()
```

Delete:

```py
db.session.delete(user)
db.session.commit()
```

Bulk operations with care (use `session.bulk_save_objects` or raw SQL for high-volume inserts).

**Avoid N+1 problem:**

```py
# N+1 - bad
users = User.query.all()
for u in users:
    print(u.posts.count())

# Better: eager load posts
users = User.query.options(db.joinedload(User.posts)).all()
```

---

## 🔐 Transactions, Sessions & Concurrency

SQLAlchemy's session manages transactions. By default, session is in transaction; `commit()` commits, `rollback()` undoes.

Safe pattern in view:

```py
try:
    # DB work
    db.session.add(obj)
    db.session.commit()
except Exception:
    db.session.rollback()
    raise
finally:
    db.session.close()
```

For request-scoped apps, Flask-SQLAlchemy handles session lifecycle automatically, but explicit handling is needed in background jobs.

**Concurrency notes:**

* Use short transactions to reduce lock contention.
* Avoid holding transactions across network calls.

---

## 📜 Pagination, Filtering & Sorting

Use `limit()` and `offset()` or SQLAlchemy `paginate` helpers.

Example simple pagination:

```py
page = int(request.args.get('page', 1))
per_page = min(int(request.args.get('per_page', 20)), 100)
posts = Post.query.filter_by(published=True).order_by(Post.created_at.desc()).paginate(page=page, per_page=per_page)
```

Return pagination metadata in API responses (total, page, per_page, pages).

---

## ⚡ Indexing, Performance & Optimization

1. **Indexes:** Add indexes on frequently filtered/sorted columns (e.g., `email`, `created_at`, `published`).
2. **EXPLAIN** queries in DB to identify slow queries.
3. **Use connection pooling** and tune pool size for your WSGI workers.
4. **Denormalize** selectively for heavy read workloads.
5. **Materialized views** (Postgres) for expensive aggregations.
6. **Avoid SELECT *;** only fetch necessary columns. Use `with_entities()` to select columns.

Example index:

```py
email = db.Column(db.String(255), unique=True, index=True)
```

---

## 🧪 Testing Database Code

Use a separate test DB or an in-memory SQLite (with caveats). Use `pytest` fixtures for setup/teardown.

Example `conftest.py` fixture:

```py
import pytest
from app import create_app, db

@pytest.fixture
def app():
    app = create_app('config.TestConfig')
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()
```

**Tip:** Autouse transactions (savepoint rollback) speed tests and isolate state.

---

## 🔗 Connection Pooling & Deployment Notes

Default connection pooling settings may need tuning on production.

In `SQLALCHEMY_DATABASE_URI`, you can pass engine options via `create_engine` or SQLALCHEMY_ENGINE_OPTIONS in Flask config:

```py
SQLALCHEMY_ENGINE_OPTIONS = {
    'pool_size': 10,
    'max_overflow': 20,
    'pool_timeout': 30,
}
```

For serverless deployments (e.g., AWS Lambda) use RDS Proxy or pooling strategies to avoid connection exhaustion.

---

## 🔐 Security & Data Protection

* **Use parameterized queries** (SQLAlchemy does this by default) to avoid SQL injection.
* **Encrypt sensitive data at rest** (DB-level or application-level encryption).
* **Mask PII** in logs — avoid logging raw sensitive fields.
* **Use least-privilege DB user** for your application.

---

## 🧭 Best Practices & Pitfalls to Avoid

✅ Index columns used in WHERE/ORDER BY.
✅ Keep transactions short.
✅ Use migrations; don’t hand-edit production schemas blindly.
✅ Avoid heavy ORM loops for millions of rows — use bulk or streaming queries.
✅ Profile queries regularly (pg_stat_statements for Postgres).

Pitfalls:

* Relying on SQLite for production.
* Long-running transactions that lock tables.
* Not monitoring connection usage (leading to timeouts).

---

## 🚀 Next Steps

Proceed to **Part V – RESTful API Development** where you’ll build JSON APIs, request validation, error handling, and API versioning.

---

*Prepared for the Flask Mastery Series — exportable in `.md`, `.pdf`, and `.docx` formats.*
