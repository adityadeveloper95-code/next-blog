# Backend Architecture Document

## 1. Overview

This backend is a **RESTful API service** built with Node.js.
It uses:

* **Session-based authentication**
* **Redis** for session storage
* **PostgreSQL** for persistent data

The system is designed to be:

* Simple and predictable
* Scalable horizontally
* Easy to reason about (Django-like mental model)

---

## 2. Tech Stack

### Core

* Node.js
* Express.js

### Database

* PostgreSQL
* Prisma ORM

### Caching / Sessions

* Redis
* express-session
* connect-redis

### Validation

* Zod

### Logging

* Pino

### Security & Middleware

* helmet
* cors
* express-rate-limit
* hpp
* compression

---

## 3. High-Level Architecture

```
Client (Browser / Mobile)
        ↓
   HTTP Request
        ↓
[ Middleware Layer ]
        ↓
[ Route Layer ]
        ↓
[ Controller Layer ]
        ↓
[ Service Layer ]
        ↓
[ Database / Cache ]
        ↓
   HTTP Response
```

---

## 4. Folder Structure

```
src/
  config/         # env, db, redis setup
  routes/         # route definitions
  controllers/    # request handlers
  services/       # business logic
  db/             # ORM / queries
  middlewares/    # custom middleware
  utils/          # helpers
  types/          # shared types (TS)
  index.ts        # entry point
```

---

## 5. Middleware Layer

### Order of execution

1. Security

   * helmet
   * cors
   * rate limiting
   * hpp

2. Parsing

   * express.json()
   * express.urlencoded()

3. Session

   * express-session
   * Redis store

4. Logging

   * pino-http

5. Compression

   * compression

---

## 6. Authentication Strategy

### Session-Based Authentication

* Client receives a **session cookie**
* Session ID stored in Redis
* User data stored server-side

### Flow

1. User logs in
2. Server validates credentials
3. Session created in Redis
4. Cookie sent to client
5. Subsequent requests use session

### Notes

* Cookies are HTTP-only
* `credentials: true` required in CORS
* CSRF protection should be considered

---

## 7. Request Lifecycle

1. Request enters Express
2. Passes through middleware
3. Route matches endpoint
4. Controller handles request
5. Validation using Zod
6. Service executes business logic
7. Database queried (Postgres)
8. Response returned (optionally compressed)

---

## 8. Data Layer

### PostgreSQL

* Primary source of truth
* Handles relational data

### Redis

* Session storage
* Optional caching layer

---

## 9. Error Handling

* Centralized error middleware
* Standard response format:

```
{
  "success": false,
  "error": {
    "message": "...",
    "code": "..."
  }
}
```

---

## 10. Logging & Observability

* Request/response logging via Pino
* Log levels:

  * info
  * warn
  * error

---

## 11. Security Considerations

* Helmet for headers
* CORS restrictions
* Rate limiting on sensitive routes
* Input validation (Zod)
* Secure session cookies
* Avoid storing sensitive data in sessions

---

## 14. Design Principles

* Keep layers thin and focused
* Prefer explicit over implicit behavior
* Validate all external input
* Avoid premature abstraction
* Optimize for readability and maintainability

---
