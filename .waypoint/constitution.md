# Project Constitution: gooof

> This is your project's constitution — customize it to match your team's principles.
> WayPoint will not overwrite this file after initial creation.

## Project Overview

- **Name**: gooof
- **Type**: greenfield
- **Track**: team

## Architecture: Simple Layered

### Principles

- Keep it simple
- Routes handle HTTP
- Services contain logic
- Models handle data

### Layer Responsibilities

**Routes/Controllers**: HTTP request handling
- HTTP only
- Delegate to services

**Services**: Business logic
- All business logic here
- No HTTP concerns

**Models/Data**: Data access
- Data access only

### Anti-Patterns to Avoid

- **Route with Business Logic**: Business logic directly in route handlers

## Quality Standards

- Write tests for all new functionality
- Keep functions focused and small
- Prefer composition over inheritance
- Delete dead code rather than commenting it out

## Security Principles

1. **No secrets in code** — Use environment variables
2. **Validate all inputs** — At system boundaries
3. **Parameterized queries only** — Never concatenate SQL
4. **Principle of least privilege** — Minimal permissions

## Governance

- All architecture decisions require documentation
- Breaking changes require team review
- Exceptions require explicit approval
