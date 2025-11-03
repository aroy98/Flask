# ⚡ Flask Mastery Series – Part VIII: Performance & Caching (Redis, Profiling, Gunicorn Tuning)

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Performance Goals & Metrics](#performance-goals--metrics)
3. [Profiling & Benchmarking](#profiling--benchmarking)

   * [Locally: cProfile & pyinstrument](#locally-cprofile--pyinstrument)
   * [Endpoint-level: Flask-Profiler & Werkzeug profiler](#endpoint-level-flask-profiler--werkzeug-profiler)
   * [Load testing: k6 & locust](#load-testing-k6--locust)
4. [Caching Strategies](#caching-strategies)

   * [Levels of caching](#levels-of-caching)
   * [Cache invalidation patterns](#cache-invalidation-patterns)
5. [Using Redis with Flask (Flask-Caching)](#using-redis-with-flask-flask-caching)

   * [Setup & configuration](#setup--configuration)
   * [Examples & decorator usage](#examples--decorator-usage)
6. [Database Query Optimization](#database-query-optimization)

   * [Indexing & EXPLAIN](#indexing--explain)
   * [Avoiding N+1 & using eager loading](#avoiding-n1--using-eager-loading)
   * [Connection Pooling](#connection-pooling)
7. [Gunicorn & WSGI Tuning](#gunicorn--wsgi-tuning)

   * [Worker types & counts](#worker-types--counts)
   * [Worker timeouts & graceful restarts](#worker-timeouts--graceful-restarts)
8. [Asynchronous Workloads & Background Jobs](#asynchronous-workloads--background-jobs)

   * [Celery + Redis/RabbitMQ patterns](#celery--redismq-patterns)
   * [Using asyncio / Quart for async endpoints](#using-asyncio--quart-for-async-endpoints)
9. [Edge & CDN Caching](#edge--cdn-caching)
10. [HTTP Caching: ETag, Cache-Control, Conditional Requests](#http-caching-etag-cache-control-conditional-requests)
11. [Observability: Metrics, Tracing & APM](#observability-metrics-tracing--apm)
12. [Autoscaling & Load Balancing](#autoscaling--load-balancing)
13. [Performance Checklist & Best Practices](#performance-checklist--best-practices)
14. [Next Steps](#next-steps)

---

## 🧠 Introduction

This part focuses on making Flask applications fast and resilient under load. We'll cover profiling, query tuning, caching at multiple layers (Redis, CDN, HTTP), WSGI server tuning (Gunicorn), background processing, and observability to ensure you can identify and fix bottlenecks.

---

## 🎯 Performance Goals & Metrics

Track metrics that matter:

* **Latency** (P50, P95, P99)
* **Throughput** (requests/sec)
* **Error rate**
* **CPU / Memory usage**
* **DB query times**

Use these to set SLAs and performance budgets (e.g., P95 < 500ms for API endpoints).

---

## 🧪 Profiling & Benchmarking

### Locally: cProfile & pyinstrument

Use `cProfile` (built-in) and `pyinstrument` for lightweight profiling.

```bash
python -m cProfile -o out.prof app.py
pyinstrument app.py
```

Visualize with `snakeviz`:

```bash
pip install snakeviz
snakeviz out.prof
```

### Endpoint-level: Flask-Profiler & Werkzeug profiler

`Flask-Profiler` or `werkzeug`'s built-in profiler middleware can profile requests and show slow endpoints.

```bash
pip install flask-profiler
```

### Load testing: k6 & locust

**k6** (scriptable JS) and **locust** (Python) simulate traffic.

k6 example:

```bash
k6 run script.js
```

Locust example:

```bash
pip install locust
locust -f locustfile.py
```

Run load tests in CI for performance regression detection.

---

## 🧠 Caching Strategies

### Levels of caching

1. **Client-side** (HTTP caching): Cache-Control, ETag.
2. **CDN / Edge**: Cache whole responses for static or cacheable API results.
3. **Application cache**: Redis-based caching for computed values.
4. **Database caching**: Materialized views or denormalized tables.

### Cache invalidation patterns

* **Time-based TTL** (simple)
* **Explicit purge** (on write events)
* **Cache versioning / namespacing** (e.g., `users:v2:id`)
* **Write-through / write-behind caching** (advanced)

Avoid stale data by choosing TTL and invalidation strategy fitting your consistency needs.

---

## 🧰 Using Redis with Flask (Flask-Caching)

Install Redis and Flask-Caching:

```bash
pip install Flask-Caching redis
```

Config:

```py
app.config['CACHE_TYPE'] = 'RedisCache'
app.config['CACHE_REDIS_URL'] = 'redis://redis:6379/0'
from flask_caching import Cache
cache = Cache(app)
```

Decorator usage:

```py
@cache.cached(timeout=60, key_prefix='all_users')
def get_users():
    return User.query.all()

@cache.memoize(timeout=300)
def user_profile(user_id):
    return User.query.get(user_id)
```

Per-view caching:

```py
@app.route('/reports')
@cache.cached(timeout=120)
def reports():
    # expensive aggregation
    return render_template('reports.html')
```

Invalidate cache:

```py
cache.delete('all_users')
cache.delete_memoized(user_profile, user_id)
```

---

## 🗃️ Database Query Optimization

### Indexing & EXPLAIN

Add indexes on filtered/sorted columns.
Use `EXPLAIN ANALYZE` to inspect query plans.

```sql
EXPLAIN ANALYZE SELECT * FROM posts WHERE published = true ORDER BY created_at DESC LIMIT 10;
```

### Avoiding N+1 & using eager loading

Use `joinedload` or `selectinload` in SQLAlchemy to prefetch relationships.

```py
from sqlalchemy.orm import joinedload
posts = Post.query.options(joinedload(Post.author)).all()
```

### Connection Pooling

Tune `pool_size` and `max_overflow` in `SQLALCHEMY_ENGINE_OPTIONS` for Gunicorn worker count.

---

## 🏃 Gunicorn & WSGI Tuning

### Worker types & counts

* **sync** workers (default) — best for CPU-bound Python code.
* **gevent/eventlet** (async workers) — for IO-bound concurrency.
* **uvicorn/gunicorn using uvloop** — for ASGI apps.

Rule of thumb for worker count:

```
workers = (2 x cores) + 1
```

Tune for available memory.

### Worker timeouts & graceful restarts

Set `--timeout` to avoid long-running workers blocking; use `--graceful-timeout` for reloads.

Example:

```bash
gunicorn app:app --workers 3 --worker-class gthread --threads 4 --timeout 30
```

Use health checks and supervise workers with systemd or container orchestrator.

---

## 🔁 Asynchronous Workloads & Background Jobs

### Celery + Redis/RabbitMQ patterns

Use Celery for CPU/IO-intensive background jobs.

```bash
pip install celery redis
```

Basic task:

```py
from celery import Celery
celery = Celery(__name__, broker='redis://redis:6379/0')

@celery.task
def send_welcome_email(user_id):
    # send email
    pass
```

Run workers separately and monitor queue depth.

### Using asyncio / Quart for async endpoints

If you need async handlers, consider **Quart** (Flask-compatible ASGI) or migrate endpoints to FastAPI for high-concurrency async I/O.

---

## 🌐 Edge & CDN Caching

Cache static assets and cacheable API responses at the CDN layer (Cloudflare, Fastly, CloudFront).
Configure cache keys, TTLs, and purge rules. Use Cache-Control headers to control CDN behavior.

---

## 📡 HTTP Caching: ETag, Cache-Control, Conditional Requests

Use ETag and `If-None-Match` to serve `304 Not Modified` when responses unchanged.

```py
from flask import make_response
resp = make_response(jsonify(payload))
resp.set_etag('v1-'+hash)
return resp
```

Set `Cache-Control` headers:

```py
resp.headers['Cache-Control'] = 'public, max-age=60, s-maxage=300'
```

`s-maxage` is respected by shared caches/CDN.

---

## 📈 Observability: Metrics, Tracing & APM

Collect metrics and traces to find bottlenecks.

* **Prometheus**: expose `/metrics` using `prometheus_client`.
* **Grafana**: visual dashboards.
* **OpenTelemetry**: distributed tracing (Jaeger/Tempo).
* **APM**: Datadog, New Relic for high-level insights.

Example Prometheus metric:

```py
from prometheus_client import Counter, generate_latest
REQUESTS = Counter('http_requests_total', 'Total HTTP Requests')

@app.before_request
def before():
    REQUESTS.inc()

@app.route('/metrics')
def metrics():
    return generate_latest()
```

Trace requests with OpenTelemetry to correlate across services.

---

## ☁️ Autoscaling & Load Balancing

* Use container orchestrators (Kubernetes / Cloud Run / ECS) for autoscaling.
* Use horizontal pod autoscalers (HPA) based on CPU, memory, or custom metrics.
* Ensure statelessness for easy horizontal scaling (store sessions in Redis or use JWTs).
* Use a managed database with read replicas for high read throughput.

---

## ✅ Performance Checklist & Best Practices

* [ ] Profile and measure before optimizing.
* [ ] Add caching at multiple layers where appropriate.
* [ ] Tune DB queries and add necessary indexes.
* [ ] Configure Gunicorn workers and timeouts.
* [ ] Move long-running tasks to Celery.
* [ ] Add Prometheus metrics and traces.
* [ ] Use CDN for static and cacheable content.
* [ ] Run load tests and set performance budgets.

---

## 🚀 Next Steps

Proceed to **Part IX – Security, Validation & Middleware**, which will cover CSRF, rate limiting, secure headers, input sanitization, and logging.

---

*Prepared for the Flask Mastery Series — exportable in `.md`, `.pdf`, and `.docx` formats.*
