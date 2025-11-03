# 🔌 Flask Mastery Series – Part XI: Advanced Integrations (Celery, WebSockets, AI, Microservices)

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Background Tasks with Celery](#background-tasks-with-celery)

   * [Why use background workers?](#why-use-background-workers)
   * [Celery setup & configuration](#celery-setup--configuration)
   * [Task patterns & retries](#task-patterns--retries)
   * [Monitoring workers (Flower)](#monitoring-workers-flower)
3. [Real-time Communication with Flask-SocketIO](#real-time-communication-with-flask-socketio)

   * [Setup & basic chat example](#setup--basic-chat-example)
   * [Scaling Socket.IO with message brokers](#scaling-socketio-with-message-brokers)
   * [Authentication & Rooms](#authentication--rooms)
4. [Integrating AI & LLMs into Flask](#integrating-ai--llms-into-flask)

   * [Use-cases: chatbots, summarization, classification](#use-cases-chatbots-summarization-classification)
   * [Serving model inference vs proxying external APIs](#serving-model-inference-vs-proxying-external-apis)
   * [Streaming responses to clients (SSE / WebSockets)](#streaming-responses-to-clients-sse--websockets)
   * [Prompt engineering & safety considerations](#prompt-engineering--safety-considerations)
5. [Microservices & Service Mesh Patterns](#microservices--service-mesh-patterns)

   * [When to split into microservices](#when-to-split-into-microservices)
   * [API gateways & versioning](#api-gateways--versioning)
   * [Inter-service communication (REST, gRPC, message brokers)](#inter-service-communication-rest-grpc-message-brokers)
6. [Event-Driven Architecture & Message Queues](#event-driven-architecture--message-queues)

   * [Designing events & schemas (Avro/JSON Schema)](#designing-events--schemas-avrojson-schema)
   * [Using Kafka or RabbitMQ for durable streams](#using-kafka-or-rabbitmq-for-durable-streams)
7. [Observability for Distributed Systems](#observability-for-distributed-systems)

   * [Distributed tracing with OpenTelemetry](#distributed-tracing-with-opentelemetry)
   * [Correlation IDs across services](#correlation-ids-across-services)
8. [Security & Compliance in Integrations](#security--compliance-in-integrations)

   * [Throttling, quotas, and cost control for LLMs](#throttling-quotas-and-cost-control-for-llms)
   * [Data privacy when sending user data to external models](#data-privacy-when-sending-user-data-to-external-models)
9. [Testing & Local Development Patterns](#testing--local-development-patterns)

   * [Local emulators for Kafka/RabbitMQ/Redis](#local-emulators-for-kafkarabbitmqredis)
   * [Integration tests for async workflows](#integration-tests-for-async-workflows)
10. [Best Practices & Anti-Patterns](#best-practices--anti-patterns)
11. [Next Steps & Learning Resources](#next-steps--learning-resources)

---

## 🧠 Introduction

Advanced integrations let your Flask app handle background processing, real-time communication, AI features, and scale horizontally via microservices. This part shows practical patterns and code examples to connect Flask with Celery, Socket.IO, LLMs, message brokers, and observability tooling.

---

## 🔁 Background Tasks with Celery

### Why use background workers?

* Offload long-running tasks (email, image processing, ML inference).
* Improve request latency by delegating heavy work.
* Schedule periodic jobs (cron-like tasks).

### Celery setup & configuration

Install:

```bash
pip install celery[redis]
```

Create `celery_app.py`:

```py
from celery import Celery

def make_celery(app=None):
    celery = Celery(
        app.import_name if app else __name__,
        broker=app.config.get('CELERY_BROKER_URL', 'redis://redis:6379/0'),
        backend=app.config.get('CELERY_RESULT_BACKEND', 'redis://redis:6379/1')
    )
    celery.conf.update(app.config or {})
    return celery
```

Task example:

```py
@celery.task(bind=True, max_retries=3, default_retry_delay=60)
def send_email(self, user_id, template):
    try:
        # send mail logic
        pass
    except Exception as exc:
        raise self.retry(exc=exc)
```

Start worker:

```bash
celery -A celery_app.celery worker --loglevel=info
```

For scheduled tasks use Celery Beat or alternative schedulers.

### Task patterns & retries

* Use `retry()` for transient failures.
* Use idempotent tasks or track task IDs to avoid duplicate effects.
* Use `acks_late` for reliable processing.

### Monitoring workers (Flower)

Install and run Flower for UI monitoring:

```bash
pip install flower
celery -A celery_app.celery flower
```

---

## 🔊 Real-time Communication with Flask-SocketIO

Install:

```bash
pip install flask-socketio[asyncio] python-socketio[asyncio_client]
```

Basic chat example:

```py
from flask_socketio import SocketIO, send, emit
socketio = SocketIO(app, cors_allowed_origins='*')

@socketio.on('message')
def handle_message(msg):
    send(msg, broadcast=True)

if __name__ == '__main__':
    socketio.run(app, host='0.0.0.0', port=5000)
```

### Scaling Socket.IO with message brokers

Use Redis or RabbitMQ as message queue for Socket.IO to scale across processes:

```py
socketio = SocketIO(app, message_queue='redis://redis:6379/0')
```

### Authentication & Rooms

Authenticate socket connections via token in query string or `Authorization` header during handshake. Use `join_room()` to isolate rooms.

---

## 🤖 Integrating AI & LLMs into Flask

### Use-cases: chatbots, summarization, classification

* Chat assistants (conversational UI)
* Document summarization
* Text classification & routing
* Code generation & analysis

### Serving model inference vs proxying external APIs

**Proxying External APIs (easier):** Call OpenAI/Anthropic/etc. from server — fast to implement, cost/control tradeoffs.
**Self-hosting models (harder):** Run models on GPU instances (e.g., using Docker + CUDA) or use managed inference services. Requires model lifecycle, scaling, and cost management.

### Streaming responses to clients (SSE / WebSockets)

Use Server-Sent Events (SSE) or WebSockets for progressive token streaming.

**SSE example:**

```py
from flask import Response

def stream_tokens(generator):
    return Response(generator(), mimetype='text/event-stream')

@app.route('/ai/stream')
def stream_endpoint():
    def gen():
        for token in llm_stream(prompt):
            yield f'data: {token}\n\n'
    return stream_tokens(gen)
```

**WebSocket streaming**: send tokens over Socket.IO events for interactive UIs.

### Prompt engineering & safety considerations

* Sanitize user inputs before sending to LLMs.
* Implement content filters and rate limits to avoid abuse.
* Monitor model outputs and have human-in-the-loop moderation for high-risk domains.

---

## 🧩 Microservices & Service Mesh Patterns

### When to split into microservices

* When teams need independent deployability.
* When scaling requirements differ across components.
* When fault isolation is important.

Microservices complexity costs: distributed tracing, transactions, service discovery.

### API gateways & versioning

Use API Gateway (Kong, Ambassador, AWS API Gateway) to centralize auth, rate-limiting, and routing. Offload TLS termination and routing from services.

### Inter-service communication (REST, gRPC, message brokers)

* **REST**: straightforward, human-readable.
* **gRPC**: high-performance, strong typing (Protobuf).
* **Message brokers**: async decoupling (Kafka, RabbitMQ).

Choose based on latency, throughput, and operational complexity.

---

## 🗂️ Event-Driven Architecture & Message Queues

### Designing events & schemas (Avro/JSON Schema)

Define stable event contracts and version them. Use schema registry (Confluent) for Kafka.

### Using Kafka or RabbitMQ for durable streams

Kafka suits high-throughput streaming; RabbitMQ is simpler for task queues and routing patterns.

Pattern example — user signup event:

```
service-auth -> emits user.created event -> service-welcome listens and sends email
```

Ensure idempotent consumers and at-least-once processing semantics when appropriate.

---

## 🔎 Observability for Distributed Systems

### Distributed tracing with OpenTelemetry

Instrument requests and background tasks to trace across services.

```bash
pip install opentelemetry-api opentelemetry-sdk opentelemetry-instrumentation-flask
```

### Correlation IDs across services

Generate request IDs and propagate via headers (`X-Request-ID`) to link logs and traces.

---

## 🔐 Security & Compliance in Integrations

### Throttling, quotas, and cost control for LLMs

* Limit per-user tokens.
* Add budgets & alerts for model usage.
* Cache model outputs where possible.

### Data privacy when sending user data to external models

* Remove PII when possible.
* Use customer-managed keys or enterprise products that support data residency.
* Document and obtain consent when required by regulations (GDPR, CCPA).

---

## 🧪 Testing & Local Development Patterns

### Local emulators for Kafka/RabbitMQ/Redis

* Use `docker-compose` to run Kafka (or `confluentinc/cp-kafka`) and RabbitMQ locally.
* Use Redis Docker image for Celery broker/backends.

### Integration tests for async workflows

* Use test doubles or in-memory brokers where feasible.
* For end-to-end tests, spin up a docker-compose environment in CI and run consumer/producer workflows.

---

## ⚠️ Best Practices & Anti-Patterns

Best practices:

* Keep services small and cohesive.
* Use durable, versioned event schemas.
* Monitor and alert on queue lag and retry rates.

Anti-patterns:

* Over-splitting into too many microservices early.
* Tight coupling via synchronous calls causing chain failures.
* Not having observability for async flows.

---

## 📚 Next Steps & Learning Resources

* Celery docs: [https://docs.celeryq.dev](https://docs.celeryq.dev)
* Flask-SocketIO docs: [https://flask-socketio.readthedocs.io](https://flask-socketio.readthedocs.io)
* OpenTelemetry Python: [https://opentelemetry.io](https://opentelemetry.io)
* Kafka fundamentals: [https://kafka.apache.org](https://kafka.apache.org)

---
