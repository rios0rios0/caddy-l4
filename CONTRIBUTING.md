# Contributing

Contributions are welcome. By participating, you agree to maintain a respectful and constructive environment.

For coding standards, testing patterns, architecture guidelines, commit conventions, and all
development practices, refer to the **[Development Guide](https://github.com/rios0rios0/guide/wiki)**.

## Prerequisites

- [Go](https://go.dev/dl/) 1.25+
- [xcaddy](https://github.com/caddyserver/xcaddy) (`go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest`)
- [Git](https://git-scm.com/downloads) 2.30+

## Development Workflow

1. Fork and clone the repository
2. Create a branch: `git checkout -b feat/my-change`
3. Install dependencies:
   ```bash
   go mod download
   ```
4. Build Caddy with this plugin using the local source:
   ```bash
   xcaddy build --with github.com/mholt/caddy-l4=.
   ```
5. Verify the layer4 modules are loaded:
   ```bash
   ./caddy list-modules --versions | grep layer4
   ```
6. Run tests:
   ```bash
   go test ./...
   ```
7. Format and vet the code:
   ```bash
   go fmt ./...
   go vet ./...
   ```
8. Update `CHANGELOG.md` under `[Unreleased]`
9. Commit following the [commit conventions](https://github.com/rios0rios0/guide/wiki/Life-Cycle/Git-Flow)
10. Open a pull request against `main`
