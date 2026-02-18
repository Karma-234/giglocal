# Copilot Instructions — giglocal

## Project Overview

**giglocal** is a local gig/services platform built in Go 1.25. The codebase is in early stage — most packages are scaffolded but not yet implemented.

## Architecture

- **Modular monorepo** — single Go module (`github.com/karma-234/giglocal`) with domain packages under `internal/` and runnable entrypoints under `app/`.
- `app/api-gateway/` — HTTP entrypoint; intended to route requests to internal domain packages. Package name is `apigateway` (not `main` — see conventions below).
- `internal/auth/` — authentication/authorization domain logic.
- `internal/job/` — job/gig listing domain logic.
- `internal/common/` — shared types, utilities, and helpers used across domains.

### Key structural decisions

- Domain logic lives in `internal/` to prevent external imports — keep it that way.
- Each domain package (`auth`, `job`) should be self-contained with its own types, handlers, and storage interfaces.
- `internal/common/` is for truly cross-cutting concerns (errors, middleware, config). Avoid dumping domain-specific code here.

### API style — REST + gRPC

- **REST**: External-facing HTTP API served by the API gateway using `net/http` with Go 1.25 enhanced ServeMux patterns (e.g. `mux.HandleFunc("GET /jobs/{id}", ...)`).
- **gRPC**: Internal service-to-service communication. Proto definitions should live alongside their domain package (e.g. `internal/job/*.proto`). Use `protoc` with `protoc-gen-go` and `protoc-gen-go-grpc` for code generation.
- The API gateway acts as the REST→gRPC translation layer: it receives HTTP requests and calls internal gRPC services.
- Keep proto-generated code in the same domain package or a `pb` sub-package (e.g. `internal/job/pb/`).

## Conventions

- **Package naming**: The API gateway uses package `apigateway`, not `main`. If adding new entrypoints under `app/`, follow this pattern (package name = directory name, no underscores).
- **Go version**: 1.25 — use modern Go features (range-over-func, enhanced servemux patterns, etc.) when appropriate.
- **Dependencies**: Prefer stdlib (`net/http`, `encoding/json`, `database/sql`, `log/slog`) for REST. gRPC requires `google.golang.org/grpc` and `google.golang.org/protobuf` — these are the expected exceptions to the stdlib-first rule.
- **Directory layout**: New domain areas get their own package under `internal/`. New runnable services get their own directory under `app/`.

## Build & Run

```sh
# Run the API gateway
go run ./app/api-gateway

# Run all tests
go test ./...
```

## When Adding New Features

1. Define proto service/messages in `internal/<domain>/` and generate Go code.
2. Implement the gRPC service server in the domain package, depending on interfaces (not concrete storage).
3. Add REST handlers in the API gateway that translate HTTP ↔ gRPC calls.
4. Wire gRPC servers and REST routes in `app/api-gateway/main.go`.
5. Shared request/response types, middleware, or error helpers go in `internal/common/`.
