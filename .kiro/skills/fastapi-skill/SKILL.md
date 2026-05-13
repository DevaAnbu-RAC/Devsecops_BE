---
name: fastapi-skill
description: "Use this skill when building FastAPI applications for agentic AI. This skill provides best practices for creating and managing FastAPI applications, including dependency injection, exception handling, routing, and observability."

metadata:
  reference-name: FastAPI
---

## FastAPI Best Practices
- Use lifespan context managers for startup/shutdown logic
- Register exception handlers globally for consistent error responses
- Use dependency injection with `Depends()` for services
- Implement health check endpoints (/healthz, /readyz)
- Use Pydantic models for request/response validation
- Instrument with OpenTelemetry for observability
- Use routers to organize endpoints by domain
- Configure CORS, middleware, and security headers appropriately
- Use lazy loading only when needed, not as default
- Use async SQLAlchemy with `async_sessionmaker` for FastAPI applications
- Maintain the dependencies in following path,`src/<project_name>/services/dependencies.py` - Dependency injection configuration for FastAPI
- Maintain the route definitions in following path,`src/<project_name>/routes/` - API endpoints (FastAPI)
- Implement global exception handlers for FastAPI applications