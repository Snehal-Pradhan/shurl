<h1 align="center" >shURL — URL Shortener</h1>
<div align="center">
  <picture>
    <img alt="shurl logo" src="public/logo.png" width="20%">
  </picture>
  <p align="center">
    A lightweight URL shortener built with Go's standard library. Shorten long URLs and redirect with a single <b>net/http</b> server — zero external dependencies, zero framework magic, just clean Go.
  </p>
  <p align="center">
    <a href="#features">Features</a> ·
    <a href="#architecture">Architecture</a> ·
    <a href="#tech-stack">Tech Stack</a> ·
    <a href="#quick-start">Quick Start</a> ·
    <a href="#project-structure">Structure</a> ·
    <a href="#patterns">Patterns</a>
  </p>
</div>

&nbsp;

A hands-on exploration of Go web server patterns — layered architecture, custom HTTP handler middleware, typed errors, standardized JSON responses, and dependency injection. Built to learn and demonstrate how to structure a Go HTTP service that scales in complexity without collapsing into spaghetti.

## Features

| Area | |
|------|--|
| **Layered architecture** | Models, services, repositories, and handlers separated by concern |
| **Custom HTTP handler** | `func(w, r) error` — errors caught in one place, not scattered across handlers |
| **Standardized responses** | Every reply uses the `{success, message, data}` envelope |
| **Typed errors** | `AppError` maps directly to HTTP 400 / 404 — never hard-code status codes |
| **Repository pattern** | Data access behind an interface — swap in-memory for PostgreSQL without touching business logic |
| **Dependency injection** | Repository → Service → Handler, wired at startup with zero globals |

## Architecture

```text
main.go
  │
  └── server.Server()
        │
        ├── net/http mux              ──  route matching (no third-party router)
        ├── handlers.Handler          ──  error-catching middleware
        │     └── url.handler         ──  HTTP handlers
        │           ├── decoder       ──  JSON request decoding
        │           └── responses     ──  JSON response formatting
        │
        └── services/url
              ├── service.go          ──  business logic
              └── repository/         ──  data access interface
                    └── inmemory.go   ──  map-backed implementation
```

### Request lifecycle

```text
Client  ──►  net/http mux  ──►  handlers.Handler.ServeHTTP()
                                      │
                                      ├── POST /shorten
                                      │     ├── decoder.DecodeJSON()
                                      │     ├── url.Service.CreateShortURL()
                                      │     │     └── repository.Save()
                                      │     └── responses.OK().ToJSON()
                                      │
                                      ├── GET /{slug}
                                      │     ├── url.Service.ResolveURL()
                                      │     │     └── repository.Get()
                                      │     └── http.Redirect(302)
                                      │
                                      └── on error:
                                            ├── AppError?  ──► 400 / 404
                                            └── unknown?   ──► 500
```

## Tech Stack

| Technology | Purpose |
|------------|---------|
| [Go](https://go.dev/) | Language — pure `net/http` for the server |
| `net/http` | HTTP routing, serving, and lifecycle |
| In-memory map | URL storage (behind a `Repository` interface) |

## Quick Start

```bash
# Clone and run
git clone <repo-url>
cd shurl
go run main.go

# Shorten a URL
curl -s http://localhost:8080/shorten \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/very/long/url"}'

# Use the short URL
curl -s http://localhost:8080/<slug-from-response>
# → 302 redirect to the original URL
```

## Project Structure

```
shurl/
├── main.go                              # Entry point
├── go.mod / go.sum
├── public/                              # Static assets
│   └── image.svg                        # Logo
├── errors/
│   └── error.go                         # Typed AppError (BadRequest, NotFound)
├── models/
│   └── url.go                           # URL data structures
├── services/
│   └── url/
│       ├── service.go                   # Service interface + business logic
│       └── repository/
│           ├── repository.go            # Repository interface
│           └── inmemory.go              # In-memory map implementation
└── server/
    ├── server.go                        # Router, DI wiring, mux setup
    ├── decoder/
    │   └── decoder.go                   # JSON decode helper
    ├── responses/
    │   └── response.go                  # Standardized JSON envelope
    └── handlers/
        ├── handler.go                   # Custom Handler type (error wrapper)
        └── url/
            ├── handler.go               # URL HTTP handlers
            └── entity.go                # Request/response DTOs
```

## Patterns

### Custom error-returning handler

Instead of writing `if err != nil` in every handler, define a function type that returns `error` and implement `http.Handler` once:

```go
type Handler func(w http.ResponseWriter, r *http.Request) error

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := h(w, r); err != nil {
        respondWithErr(err, w)  // centralized error handling
    }
}
```

### Typed errors → HTTP status codes

```go
// errors/error.go
type AppError struct {
    text    string
    errType Type  // TypeBadRequest or TypeNotFound
}

// In respondWithErr():
errors.As(err, &appError)  // true → 400/404, false → 500
```

### Standardized response envelope

```go
responses.OK("message", data)   // → { "success": true,  "message": "...", "data": {...} }
responses.Fail("message", 400)  // → { "success": false, "message": "..." }
```

### Repository abstraction

```go
type Repository interface {
    Get(slug string) (*models.URL, error)
    Save(models.URL) error
}
// inmemory.go implements it with a sync.Map
// postgres.go would implement it with PostgreSQL — swap without changing services
```

## Detailed Breakdown

### Handler layer — error centralization

Every HTTP handler returns `error`. The custom `handlers.Handler` type (which implements `http.Handler`) catches that error in `ServeHTTP` and routes it through `respondWithErr`. This means zero error-handling boilerplate in your handler functions — you just return an error and it gets formatted as the right JSON response.

### Service layer — business logic

The service is an interface (`url.Service`) with a single implementation. It generates unique slugs, timestamps, and delegates persistence to the repository. The handler never touches the database — it only calls the service.

### Repository layer — data access

`repository.Repository` is an interface with two methods: `Get` and `Save`. The in-memory implementation uses a `sync.Map` (safe for concurrent access). URLs are stored under keys like `url::<slug>`. Because the service depends on the interface, swapping to PostgreSQL or Redis requires zero changes to business logic.

### Error types — semantic HTTP mapping

`AppError` carries a `Type` field (`TypeBadRequest` or `TypeNotFound`). The central `respondWithErr` function uses `errors.As` to unwrap the error, checks its type, and writes the correct HTTP status code. Unknown/unexpected errors always return 500 with a generic message — no information leakage.

### Responses — consistent envelope

Every endpoint writes through `responses.OK()` or `responses.Fail()`. The JSON shape is always `{ "success": bool, "message": string, "data": ... }`. The `ToJSON()` method sets `Content-Type: application/json` and writes the status code in one place — no handler can accidentally omit the header or use a different format.

&nbsp;

<div align="center">
  <sub>Built with Go</sub>
</div>
