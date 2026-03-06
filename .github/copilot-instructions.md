# Copilot Instructions

## Project Overview

`caddy-l4` is a Layer 4 (TCP/UDP) routing and proxy plugin for [Caddy](https://caddyserver.com/). It enables composable handling of raw TCP/UDP connections based on connection properties or the beginning of the stream, supporting use cases like TLS termination, protocol detection (HTTP, TLS, SSH, SOCKS4/5, XMPP), proxying, echoing, throttling, and more.

The project is a fork of **Project Conncept** and is designed to run alongside other Caddy apps (e.g., the HTTP server or TLS certificate manager) via Caddy's native JSON configuration.

## Project Structure

```
caddy-l4/
├── .github/
│   └── workflows/
│       └── default.yaml       # CI/CD pipeline (delegates to shared pipeline)
├── layer4/                    # Core Layer 4 app framework
│   ├── app.go                 # Caddy app registration and lifecycle
│   ├── connection.go          # Connection abstraction (buffered read, context)
│   ├── handlers.go            # Handler interface and middleware chain
│   ├── matchers.go            # Matcher interface and logic
│   ├── routes.go              # Route matching and chaining
│   └── server.go              # TCP/UDP server implementation
├── modules/                   # Pluggable matchers and handlers
│   ├── l4echo/                # Echo handler
│   ├── l4http/                # HTTP matcher and handler
│   ├── l4proxy/               # Reverse proxy handler (with load balancing)
│   ├── l4proxyprotocol/       # HAProxy PROXY protocol (matcher + handler)
│   ├── l4socks/               # SOCKS4/SOCKS5 matcher and handler
│   ├── l4ssh/                 # SSH matcher
│   ├── l4subroute/            # Sub-route handler
│   ├── l4tee/                 # Tee (branch) handler
│   ├── l4throttle/            # Throttle handler
│   ├── l4tls/                 # TLS termination handler and matcher
│   └── l4xmpp/               # XMPP matcher
├── imports.go                 # Blank imports to register all standard modules
├── go.mod                     # Go module definition
├── go.sum                     # Go module checksums
├── CHANGELOG.md               # Version history
└── CONTRIBUTING.md            # Contribution guide
```

## Build & Development Commands

### Prerequisites

- [Go](https://go.dev/dl/) 1.25+
- [xcaddy](https://github.com/caddyserver/xcaddy): `go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest`
- [Git](https://git-scm.com/downloads) 2.30+

### Common Commands

```bash
# Download dependencies
go mod download

# Build Caddy with this plugin using local source
xcaddy build --with github.com/mholt/caddy-l4=.

# Verify layer4 modules are loaded
./caddy list-modules --versions | grep layer4

# Run all tests
go test ./...

# Format code
go fmt ./...

# Vet code
go vet ./...
```

## Architecture

The plugin follows Caddy's module system and a **middleware chain** pattern:

- **App** (`layer4.App`): Top-level Caddy app that manages one or more servers.
- **Server**: Listens on TCP/UDP addresses; dispatches accepted connections through a route list.
- **Routes**: Ordered list of `{match, handle}` pairs. A route matches when all its matchers return `true`; once matched, the handlers in that route are executed in sequence.
- **Connection** (`layer4.Connection`): Wraps a `net.Conn` with buffered peeking so matchers can inspect bytes without consuming them. Carries a `context.Context` for passing values between handlers.
- **Matchers**: Implement `layer4.ConnMatcher`. They inspect connection bytes (without consuming) to decide whether a route applies.
- **Handlers**: Implement `layer4.NextHandler`. Each handler receives the connection and a `Next` handler to call to continue the chain. Terminal handlers (e.g., `echo`, `proxy`) do not call `Next`.

Each module in `modules/` self-registers with Caddy using `func init() { caddy.RegisterModule(...) }`.

## Coding Conventions

- **Language**: Go — follow standard Go idioms and the [Effective Go](https://go.dev/doc/effective_go) guide.
- **Naming**: Use `CamelCase` for exported types and functions; `camelCase` for unexported. Matcher structs follow the pattern `MatchXxx` or `XxxMatcher` (e.g., `MatchHTTP`, `MatchTLS`, `Socks5Matcher`). Handler structs are typically named `Handler` within their own package (e.g., `l4echo.Handler`, `l4proxy.Handler`), or `XxxHandler` when multiple handlers coexist in a package (e.g., `Socks5Handler`).
- **Module IDs**: Caddy module IDs follow `layer4.matchers.<name>` and `layer4.handlers.<name>`. These are returned from the `CaddyModule()` method.
- **File organization**: One module per file where practical; tests live alongside source (`_test.go` suffix in the same package or `_test` package).
- **Error handling**: Use `fmt.Errorf("context: %w", err)` wrapping; avoid panics in non-init code.
- **Logging**: Use `go.uber.org/zap` via Caddy's provisioned logger (`caddy.Log()` or module-provisioned `zap.Logger`).
- **JSON tags**: All configuration fields use `json:"fieldName"` tags for Caddy's JSON config format; use `caddy:"..."` struct tags for documentation and validation.
- **No Caddyfile support**: This plugin only supports Caddy's native JSON configuration format.

## Testing

- **Framework**: Standard Go testing (`testing` package) with `go test ./...`.
- **Test files**: Named `*_test.go`, located alongside the code under test.
- **Pattern**: Tests are table-driven where appropriate. Use `t.Run(name, func)` for subtests.
- **Approach**: Unit tests focus on individual matchers and handlers. Integration-style tests spin up in-process listeners and connections.
- **No external test frameworks**: Avoid third-party assertion libraries; use `t.Errorf`, `t.Fatalf`, and standard comparisons.

Example:

```go
func TestMatchHTTP(t *testing.T) {
    // Given: a connection with HTTP bytes
    // When: the matcher is applied
    // Then: it returns true
}
```

## CI/CD

The CI pipeline is defined in `.github/workflows/default.yaml` and delegates to the shared reusable workflow at `rios0rios0/pipelines/.github/workflows/go.yaml@main`.

It runs on:
- Pushes to `main`
- Tag pushes
- Pull requests targeting `main`
- Manual dispatch (`workflow_dispatch`)

The pipeline typically includes:
- Dependency download (`go mod download`)
- Build verification
- Linting (`go vet`, and potentially `golangci-lint`)
- Tests (`go test ./...`)
- Security/SAST scanning

### Local Quality Gates

Before opening a pull request, run these locally:

```bash
go fmt ./...
go vet ./...
go test ./...
```

For more details, see the [Development Guide](https://github.com/rios0rios0/guide/wiki) and [CONTRIBUTING.md](../CONTRIBUTING.md).
