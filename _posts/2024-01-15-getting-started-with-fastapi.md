---
title: "Getting Started with FastAPI"
date: 2024-01-15 10:00:00 +0530
categories: [Python, FastAPI]
tags: [fastapi, python, api, backend, tutorial]
excerpt: "Learn how to build high-performance APIs with FastAPI - from setup to deployment"
author: Bharat Kumar
---

## Introduction

FastAPI is a modern, fast web framework for building APIs with Python. It's built on top of Starlette and Pydantic, making it incredibly fast and easy to use.

### Key Features

- ✅ Fast to code - Write less, code faster
- ✅ Fast to run - High performance, comparable to NodeJS and Go
- ✅ Intuitive - Great editor support and auto-completion
- ✅ Easy - Easy to learn and use
- ✅ Robust - Production-ready code with automatic interactive docs
- ✅ Standards-based - Based on OpenAPI and JSON Schema standards

## Quick Start

### Installation
```bash
pip install fastapi uvicorn
```

### Create Your First API
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def read_root():
    return {"Hello": "World"}

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}
```

### Run the Server
```bash
uvicorn main:app --reload
```

Visit `http://127.0.0.1:8000` to see your API in action!

## Interactive Documentation

FastAPI automatically generates interactive API documentation:
- **Swagger UI**: http://127.0.0.1:8000/docs
- **ReDoc**: http://127.0.0.1:8000/redoc

## Conclusion

FastAPI makes building high-performance APIs simple and enjoyable. Give it a try today!