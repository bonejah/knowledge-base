# Types of Layered Architectures

## Layered Architecture

```text
Client
   │
   ▼
Controller
   │
   ▼
Service
   │
   ▼
Repository
   │
   ▼
Database
```

### Responsibilities

- **Controller** → Receives HTTP requests and returns HTTP responses.
- **Service** → Contains the application's business logic.
- **Repository** → Handles communication with the database.
- **Entity** → Represents a database table (JPA).
- **DTO (Data Transfer Object)** → Object used for API requests and responses.
- **Config** → Spring configuration classes.
- **Utility** → Reusable helper functions.

---

## Feature-based Architecture (Package by Feature)

```text
user/
 ├── UserController
 ├── UserService
 ├── UserRepository
 ├── UserDTO
 └── User

product/
 ├── ProductController
 ├── ProductService
 ├── ProductRepository
 ├── ProductDTO
 └── Product
```

Each feature contains all the classes related to a specific business capability.

---

## Hexagonal Architecture (Ports & Adapters)

Commonly used in large enterprise applications, fintechs, and banks.

```text
Controller
      │
      ▼
Application Service
      │
      ▼
Domain
      ▲
      │
Repository Interface
      ▲
      │
JPA Repository
```

The domain layer does not depend on Spring, HTTP, or the database.

### Advantages

- Easy to test
- Loosely coupled
- Easy to replace the database
- Easy to replace REST with Kafka, GraphQL, or gRPC

---

## Clean Architecture

An evolution of Hexagonal Architecture.

```text
Controller
      │
      ▼
Use Cases
      │
      ▼
Domain
      │
      ▼
Infrastructure
```