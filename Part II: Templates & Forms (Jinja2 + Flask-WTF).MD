# 🧩 Flask Mastery Series – Part II: Templates & Forms (Jinja2 + Flask-WTF)

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Jinja2 Templating Deep Dive](#jinja2-templating-deep-dive)

   * [Basic Syntax & Expressions](#basic-syntax--expressions)
   * [Control Structures](#control-structures)
   * [Template Inheritance](#template-inheritance)
   * [Macros & Filters](#macros--filters)
3. [Static Assets & URL Building](#static-assets--url-building)
4. [Forms in Flask](#forms-in-flask)

   * [Flask-WTF Setup](#flask-wtf-setup)
   * [Defining WTForms](#defining-wtforms)
   * [Rendering Forms in Templates](#rendering-forms-in-templates)
   * [Validating User Input](#validating-user-input)
5. [CSRF Protection & Security](#csrf-protection--security)
6. [File Uploads with Flask-WTF](#file-uploads-with-flask-wtf)
7. [Flash Messages & Form Feedback](#flash-messages--form-feedback)
8. [Best Practices & Common Pitfalls](#best-practices--common-pitfalls)
9. [Next Steps](#next-steps)

---

## 🧠 Introduction

This chapter focuses on **Flask’s front-end layer**: the **Jinja2 templating engine** and the **Flask-WTF form handling extension**. You’ll learn to create dynamic, secure forms and reusable templates, integrating front-end HTML with Python logic safely.

---

## 🪄 Jinja2 Templating Deep Dive

### Basic Syntax & Expressions

Jinja2 is Flask’s default templating engine.

**Variables:**

```html
<h2>Hello, {{ user.name }}!</h2>
```

**Filters:**

```html
<p>{{ name|capitalize }}</p>
<p>{{ amount|floatformat(2) }}</p>
```

**Comments:**

```html
{# This will not be rendered #}
```

**Escaping:**
By default, Jinja2 escapes HTML to prevent XSS. Use `|safe` to allow HTML intentionally:

```html
{{ html_content|safe }}
```

---

### Control Structures

```html
{% if user %}
  <p>Welcome, {{ user.name }}</p>
{% else %}
  <p>Please log in</p>
{% endif %}

<ul>
{% for item in items %}
  <li>{{ item }}</li>
{% endfor %}
</ul>
```

---

### Template Inheritance

Use a **base layout** with `block` placeholders.

**base.html:**

```html
<!doctype html>
<html>
<head>
  <title>{% block title %}My Site{% endblock %}</title>
</head>
<body>
  <header>Header</header>
  {% block content %}{% endblock %}
  <footer>Footer</footer>
</body>
</html>
```

**home.html:**

```html
{% extends 'base.html' %}
{% block title %}Home{% endblock %}
{% block content %}
  <h1>Welcome!</h1>
{% endblock %}
```

---

### Macros & Filters

**Macros** help you reuse template snippets:

```html
{% macro input_field(name, label) %}
  <label>{{ label }}</label>
  <input type="text" name="{{ name }}" />
{% endmacro %}

<form>
  {{ input_field('username', 'Username') }}
</form>
```

**Custom filters (Python):**

```py
@app.template_filter('reverse')
def reverse_filter(s):
    return s[::-1]
```

Use in template:

```html
<p>{{ 'Flask'|reverse }}</p> <!-- Output: ksalF -->
```

---

## 🖼️ Static Assets & URL Building

Serve static files from `/static`.

**Directory structure:**

```
project/
├── static/
│   ├── css/
│   │   └── main.css
│   └── img/
└── templates/
```

Link assets dynamically:

```html
<link rel="stylesheet" href="{{ url_for('static', filename='css/main.css') }}">
<img src="{{ url_for('static', filename='img/logo.png') }}" alt="Logo">
```

---

## 🧾 Forms in Flask

### Flask-WTF Setup

Install dependencies:

```bash
pip install Flask-WTF WTForms email-validator
```

Configure secret key for CSRF:

```py
# app/__init__.py
app.config['SECRET_KEY'] = 'supersecretkey'
```

---

### Defining WTForms

Create `forms.py`:

```py
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField
from wtforms.validators import DataRequired, Email, Length

class LoginForm(FlaskForm):
    email = StringField('Email', validators=[DataRequired(), Email()])
    password = PasswordField('Password', validators=[DataRequired(), Length(min=6)])
    submit = SubmitField('Login')
```

---

### Rendering Forms in Templates

```py
# routes.py
from flask import render_template, flash, redirect, url_for
from .forms import LoginForm

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        flash(f"Welcome, {form.email.data}")
        return redirect(url_for('dashboard'))
    return render_template('login.html', form=form)
```

**Template (`login.html`):**

```html
{% extends 'base.html' %}
{% block content %}
  <h2>Login</h2>
  <form method="POST">
    {{ form.hidden_tag() }}
    <p>{{ form.email.label }} {{ form.email(size=32) }}</p>
    <p>{{ form.password.label }} {{ form.password(size=32) }}</p>
    <p>{{ form.submit() }}</p>
  </form>
{% endblock %}
```

---

### Validating User Input

WTForms automatically handles validation.
Use custom validators for complex rules:

```py
from wtforms.validators import ValidationError

def must_be_gmail(form, field):
    if not field.data.endswith('@gmail.com'):
        raise ValidationError('Only Gmail addresses allowed.')
```

Then apply:

```py
email = StringField('Email', validators=[DataRequired(), Email(), must_be_gmail])
```

---

## 🔒 CSRF Protection & Security

Flask-WTF includes built-in CSRF protection using `SECRET_KEY`.
Ensure each POST form includes:

```html
{{ form.hidden_tag() }}
```

Best practices:

* Never disable CSRF in production.
* Use HTTPS for secure cookie transmission.
* Rotate `SECRET_KEY` if compromised.

---

## 📁 File Uploads with Flask-WTF

```py
from flask_wtf.file import FileField, FileAllowed, FileRequired
from wtforms import SubmitField

class UploadForm(FlaskForm):
    file = FileField('Upload Image', validators=[
        FileRequired(),
        FileAllowed(['jpg', 'png'], 'Images only!')
    ])
    submit = SubmitField('Upload')
```

**Route:**

```py
import os
from flask import current_app

@app.route('/upload', methods=['GET', 'POST'])
def upload():
    form = UploadForm()
    if form.validate_on_submit():
        filename = form.file.data.filename
        path = os.path.join(current_app.root_path, 'uploads', filename)
        form.file.data.save(path)
        flash('File uploaded!')
        return redirect(url_for('upload'))
    return render_template('upload.html', form=form)
```

**Template:**

```html
<form method="POST" enctype="multipart/form-data">
  {{ form.hidden_tag() }}
  {{ form.file.label }} {{ form.file() }}
  {{ form.submit() }}
</form>
```

---

## 💬 Flash Messages & Form Feedback

Use `flash()` to display transient messages:

```py
flash('Login successful!', 'success')
flash('Invalid credentials', 'error')
```

In templates:

```html
{% with messages = get_flashed_messages(with_categories=true) %}
  {% if messages %}
    <ul class="flashes">
      {% for category, msg in messages %}
        <li class="{{ category }}">{{ msg }}</li>
      {% endfor %}
    </ul>
  {% endif %}
{% endwith %}
```

---

## 🧩 Best Practices & Common Pitfalls

✅ Use `hidden_tag()` for CSRF every time.
✅ Keep form logic in separate `forms.py` file.
✅ Validate both client- and server-side.
✅ Avoid directly injecting user data — always use `{{ variable }}` (auto-escaped).
✅ Use macros for repeated form components.

---

## 🚀 Next Steps

Proceed to **Part III – Routing, Blueprints & Application Structure**, where you’ll modularize your Flask app and scale its organization.

---

*Prepared for the Flask Mastery Series — exportable in `.md`, `.pdf`, and `.docx` formats.*
