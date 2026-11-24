# ADR 001: Architecture Style

## Status

Accepted

## Context

PayLink is a bill payment and digital wallet system.
The system needs a clear structure that separates responsibilities
and makes the application easy to maintain, test, and extend.

## Decision

We decided to use a Layered Architecture implemented as a Modular Monolith.

The application will be organized into the following layers:

1. API Layer: Handles incoming HTTP requests and responses.
2. Service Layer: Contains business logic and application operations.
3. Repository Layer: Handles database access and data operations.
4. Domain Layer: Contains core business entities and rules.
5. Core Layer: Contains shared configuration and application settings.

## Consequences

### Positive

- Clear separation of responsibilities.
- Easier testing and maintenance.
- Simple deployment as one application.
- Modules can be developed independently.

### Negative

- Modules are part of the same application.
- Tight coupling may occur between layers or modules if dependencies are not    managed properly.
- Changes in one module may affect other modules if proper separation is not maintained.    
