# Project Constitution

> This document defines the non-negotiable principles and governance rules for **gooof**.
> All agents and workflows must respect these principles.

## Project Overview

- **Name**: gooof
- **Type**: greenfield
- **Track**: team

## Architecture: Simple Layered

### Core Principles
- Keep it simple
- Routes handle HTTP
- Services contain logic
- Models handle data

### Layer Responsibilities

**Routes/Controllers**: HTTP request handling
- Contains: routes, controllers, middleware
- HTTP only
- Delegate to services

**Services**: Business logic
- Contains: services, helpers
- All business logic here
- No HTTP concerns

**Models/Data**: Data access
- Contains: models, repositories, database
- Data access only

## Anti-Patterns to AVOID

- **Route with Business Logic**: Business logic directly in route handlers
  - Fix: Extract logic into service layer

## Security Principles

### Security

- Never commit secrets, API keys, or credentials to source control
- Validate all inputs at system boundaries (user input, external APIs)
- Use parameterized queries to prevent SQL injection
- Apply principle of least privilege for all access controls
- Sanitize output to prevent XSS attacks

## Quality Principles

### Code Quality

- Write self-documenting code with clear naming
- Keep functions focused and small (single responsibility)
- Prefer composition over inheritance
- Avoid premature optimization
- Delete dead code rather than commenting it out

### Testing

- Write tests for all new functionality
- Test behavior, not implementation details
- Maintain meaningful test coverage
- Use descriptive test names that explain the expected behavior
- Keep tests independent and isolated

### Error Handling

- Handle errors at appropriate boundaries
- Provide meaningful error messages to users
- Log errors with sufficient context for debugging
- Use structured error types (not generic Error)
- Never swallow errors silently

## Governance Principles

### Governance

- All architecture decisions require team review
- Breaking changes require ADR documentation
- Security issues take priority over feature work
- Code review required before merging to main
- Document non-obvious design decisions
- This constitution is **immutable** during the project lifecycle
- All architectural decisions must reference this document
- Exceptions require explicit documentation and approval
