# 🧪 Flask Mastery Series – Part VII: Testing & CI/CD (pytest, Coverage, GitHub Actions)

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Testing Pyramid & Strategy](#testing-pyramid--strategy)
3. [Setting Up pytest for Flask](#setting-up-pytest-for-flask)

   * [Fixtures & App Factory Integration](#fixtures--app-factory-integration)
   * [Database Fixtures & Transactions](#database-fixtures--transactions)
4. [Unit Tests](#unit-tests)
5. [Integration Tests](#integration-tests)
6. [End-to-End (E2E) Testing (Cypress / Playwright)](#end-to-end-e2e-testing-cypress--playwright)
7. [Contract Testing & OpenAPI Validation](#contract-testing--openapi-validation)
8. [Test Coverage & Static Analysis](#test-coverage--static-analysis)
9. [Security Scanning & Dependency Checks](#security-scanning--dependency-checks)
10. [CI Pipelines with GitHub Actions (Example)](#ci-pipelines-with-github-actions-example)
11. [Dockerized Test Environments](#dockerized-test-environments)
12. [Flaky Tests & Reliability Strategies](#flaky-tests--reliability-strategies)
13. [Publishing Test Artifacts & Reporting](#publishing-test-artifacts--reporting)
14. [Next Steps & Checklist](#next-steps--checklist)

---

## 🧠 Introduction

Testing and CI/CD are essential for confidence and repeatable releases. This part shows how to create fast unit tests, meaningful integration tests, reliable E2E suites, and CI pipelines that run them automatically with quality gates (linting, type checks, security scans, coverage thresholds).

---

## 🧭 Testing Pyramid & Strategy

* **Unit tests** — small, fast, pure logic (use mocks). Aim for many.
* **Integration tests** — database, extensions, and modules working together.
* **End-to-end tests** — full user flows (slowest). Use sparingly.

Balance speed and coverage: make unit tests fast; run heavier tests in nightly or pre-release pipelines.

---

## ⚙️ Setting Up pytest for Flask

Install:

```bash
pip install pytest pytest-flask pytest-cov factory-boy
```

Create `tests/conftest.py` to provide app and client fixtures.

### Fixtures & App Factory Integration

```py
import pytest
from app import create_app, db

@pytest.fixture
def app():
    app = create_app('config.TestConfig')
    with app.app_context():
        yield app

@pytest.fixture
def client(app):
    return app.test_client()
```

### Database Fixtures & Transactions

Use a transactional test pattern:

```py
@pytest.fixture(autouse=True)
def _db_session(app):
    db.create_all()
    yield
    db.session.remove()
    db.drop_all()
```

For faster isolation, use savepoints and rollback on each test.

---

## ✅ Unit Tests

Unit tests focus on isolated functions and modules. Mock external dependencies using `unittest.mock` or `pytest-mock`.

Example:

```py
from unittest.mock import patch

def test_send_email(mocker):
    mocker.patch('app.email.send')
    result = mymodule.trigger_email(1)
    assert result is True
    app.email.send.assert_called_once()
```

Tips:

* Test validation logic, serializers, and helper functions.
* Avoid DB and network calls — mock them out.

---

## 🔗 Integration Tests

Integration tests exercise DB, Flask extensions, and blueprints.

Example:

```py
def test_create_user(client):
    res = client.post('/api/users', json={'name': 'A', 'email': 'a@example.com'})
    assert res.status_code == 201
    assert 'id' in res.get_json()
```

Use real DB migrations in CI or a fresh test DB for realistic behavior.

---

## 🌐 End-to-End (E2E) Testing (Cypress / Playwright)

E2E tests simulate user behavior in a running environment.

**Cypress** — great DX, runs in the browser.
**Playwright** — cross-browser automation, supports Python/JS.

Basic Cypress test:

```js
describe('Login', () => {
  it('logs in', () => {
    cy.visit('/login')
    cy.get('input[name=email]').type('user@example.com')
    cy.get('input[name=password]').type('password')
    cy.get('button[type=submit]').click()
    cy.url().should('include', '/dashboard')
  })
})
```

Run E2E in CI against a real deployed staging environment or a docker-compose setup that starts the app and DB.

---

## 🧾 Contract Testing & OpenAPI Validation

Use **schemathesis** to generate tests from your OpenAPI schema and detect contract regressions.

```bash
pip install schemathesis
schemathesis run http://localhost:5000/openapi.json
```

Run contract tests on CI to prevent API/consumer mismatches.

---

## 📊 Test Coverage & Static Analysis

Measure coverage with `pytest-cov`:

```bash
pytest --cov=app --cov-report=term-missing
```

Use tools:

* **flake8** / **pylint** for linting
* **mypy** for type checking (if using type hints)
* **black** for consistent formatting

Set coverage thresholds in CI to fail builds if coverage falls below a threshold.

---

## 🔒 Security Scanning & Dependency Checks

Integrate dependency scanning:

* **pip-audit** (Python dependency vulnerability scanner)
* **safety** or **Snyk** for deeper vulnerability checks

Example pip-audit run:

```bash
pip install pip-audit
pip-audit
```

Also run SAST rules via `bandit` for Python security checks:

```bash
pip install bandit
bandit -r app
```

---

## ⚙️ CI Pipelines with GitHub Actions (Example)

Create `.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres" --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov
      - name: Run linters
        run: |
          pip install flake8
          flake8
      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
        run: |
          pytest --cov=app --cov-report=xml
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml
```

Add jobs for E2E, security scans, and deploy gates as needed.

---

## 🐳 Dockerized Test Environments

Use `docker-compose.ci.yml` for integration tests with DB & services:

```yaml
version: '3.8'
services:
  web:
    build: .
    command: pytest --maxfail=1 --disable-warnings -q
    depends_on:
      - db
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/test_db
  db:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: test_db
```

Run:

```bash
docker-compose -f docker-compose.ci.yml up --build --abort-on-container-exit
```

This provides parity between local and CI environments.

---

## 🧭 Flaky Tests & Reliability Strategies

* Mark flaky tests with `@pytest.mark.flaky` or xfail when necessary.
* Use retries in CI (careful — hide real issues).
* Isolate tests and avoid shared global state.
* Use test seeds to reproduce randomness deterministically.

---

## 📎 Publishing Test Artifacts & Reporting

Upload artifacts in GitHub Actions:

```yaml
- name: Upload coverage report
  uses: actions/upload-artifact@v4
  with:
    name: coverage-report
    path: coverage.xml
```

Publish HTML reports, test result JUnit XML, and linter outputs for PR feedback.

---

## ✅ Next Steps & Checklist

* [ ] Add unit tests for all critical business logic.
* [ ] Add integration tests for API endpoints and DB interactions.
* [ ] Add at least one E2E scenario for core user flow.
* [ ] Enforce linting, type checks, and security scans in CI.
* [ ] Publish coverage and test artifacts for PRs.
* [ ] Run contract tests against OpenAPI spec in CI.

---

*Prepared for the Flask Mastery Series — exportable in `.md`, `.pdf`, and `.docx` formats.*
