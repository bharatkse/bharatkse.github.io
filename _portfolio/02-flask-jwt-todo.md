---
layout: single
title: "Secure REST API with JWT Authentication & Authorization"
permalink: /projects/flask-jwt-auth/
excerpt: "Reference implementation for stateless authentication and role-based access control using JWT."

order: 4
featured: false

categories: [backend, security, authentication]
tags: [jwt, flask, rbac, auth, api, security]

problem: >
  Session-based authentication does not scale well for APIs,
  especially in distributed and microservice-based architectures.

solution: >
  Implemented JWT-based stateless authentication with role-based
  access control (RBAC) to enable scalable and secure API access.

outcome: >
  Delivered a production-ready API foundation that supports
  horizontal scaling, secure authorization, and clean separation
  of concerns.

repo_description: >
  A secure Flask REST API for managing users and ToDo tasks using
  JWT-based authentication. Includes user registration, login,
  logout, password reset, RBAC enforcement, and full CRUD support,
  powered by Flask, SQLAlchemy, and PostgreSQL.

metrics:
  - "Stateless authentication"
  - "Role-based access control (RBAC)"
  - "Horizontally scalable API"
  - "Secure token lifecycle management"

github:
  repo: "https://github.com/bharatkse/flask_jwt_todo"
  repo_name: "flask-jwt-todo"

blog:
  title: "Secure REST APIs with JWT Authentication"
  url: "/blog/flask-jwt-auth/"

sidebar:
  - title: "Role"
    text: "Backend Engineer"

  - title: "Tech Stack"
    text: "Python, Flask, JWT, SQLAlchemy, PostgreSQL"

  - title: "Repository"
    text: "[View on GitHub](https://github.com/bharatkse/flask-jwt-todo)"

  - title: "Related Write-up"
    text: "[Read Blog →](/blog/jwt-authentication-flask/)"
---



## Problem

Many APIs mix authentication, authorization, and business logic,
leading to insecure and hard-to-scale systems.

---

## Solution

Built a **clean reference API** demonstrating:

- Stateless JWT authentication
- Short-lived access tokens
- Role-based authorization
- Explicit security boundaries

---

## Architecture Overview

- Flask REST API
- JWT token validation
- SQLAlchemy ORM
- Stateless request handling

---

## Outcome & Impact

- 🔐 Clear auth boundaries
- 🚀 Horizontally scalable
- 📚 Reusable security reference
- 🧱 Suitable for microservices

---

## Related Writing

- [Secure REST APIs with JWT Authentication](/blog/secure-rest-api-jwt/)
