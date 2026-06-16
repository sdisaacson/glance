# Agent Guide for Glance

This document is intended for AI coding agents working on the Glance codebase. Glance is a self-hosted dashboard application written in Go. All comments, documentation, and user-facing text in the project are in English.

## Project Overview

Glance is a lightweight, highly customizable dashboard that aggregates feeds and data sources into a single-page web interface. It is distributed as a single static binary (under 20 MB) and as a Docker image. The application reads a YAML configuration file, renders a set of widgets into HTML on the server, and serves the resulting pages over HTTP.

Key characteristics:

- **Language**: Go (module version `go 1.26.3`, requires Go >= v1.23).
- **Entry point**: `main.go` delegates to `internal/glance.Main()`.
- **Embedded assets**: HTML templates, CSS, JavaScript, fonts, and icons are embedded via `//go:embed` from `internal/glance/static` and `internal/glance/templates`.
- **Server-side rendering**: Widgets fetch data at request time and are rendered into HTML templates using Go's `html/template`.
- **No JavaScript build step**: The frontend is vanilla JS/CSS. CSS is bundled at runtime from `internal/glance/static/css/main.css` by recursively inlining `@import` statements.
- **Configuration**: YAML file (`glance.yml` by default) with support for includes, environment variables, Docker secrets, and file-from-env substitutions.
- **Authentication**: Optional cookie-based sessions using HMAC-SHA256 tokens and bcrypt password hashes.

## Repository Layout

```
.
├── main.go                         # Tiny entry point: os.Exit(glance.Main())
├── internal/glance/                # All application logic
│   ├── main.go                     # CLI dispatch and serveApp lifecycle
│   ├── glance.go                   # application struct, page/widget init, HTTP router
│   ├── cli.go                      # Command-line parsing and utility commands
│   ├── config.go                   # YAML config parsing, includes watcher, validation
│   ├── config-fields.go            # Custom YAML unmarshalers (HSL colors, durations, icons, proxies)
│   ├── widget.go                   # Widget interface, widgetBase, cache/update logic
│   ├── widget-*.go                 # Individual widget implementations (one per widget type)
│   ├── widget-shared.go            # Shared types/helpers for forum-style widgets
│   ├── widget-utils.go             # Common widget helper functions
│   ├── templates.go                # Template parsing helpers and global funcs
│   ├── embed.go                    # go:embed declarations and CSS bundling
│   ├── theme.go                    # Theme preset handling and CSS generation
│   ├── auth.go                     # Session token logic and login handlers
│   ├── auth_test.go                # Tests for session tokens
│   ├── utils.go                    # General utility functions
│   ├── singleflight.go             # Singleflight request deduplication
│   ├── diagnose.go                 # Diagnostic CLI output
│   ├── static/                     # Embedded CSS, JS, fonts, icons, images
│   └── templates/                  # Embedded Go/HTML templates
├── pkg/sysinfo/                    # Cross-platform system metrics collection
│   └── sysinfo.go                  # CPU, memory, swap, disk, sensor, host info
├── docs/                           # User documentation
├── Dockerfile                      # Standard Docker build
├── Dockerfile.goreleaser           # Minimal Docker image used by GoReleaser
├── .goreleaser.yaml                # Release builds, archives, and Docker manifests
└── .github/workflows/release.yaml  # GitHub Actions release workflow
```

## Technology Stack

- **Go 1.26.3** with `gopkg.in/yaml.v3` for configuration parsing.
- **Standard library HTTP server** (`net/http`). No external router.
- **Embedded filesystem** (`embed`) for templates and static assets.
- **Dependencies** (kept minimal):
  - `github.com/fsnotify/fsnotify` - config file hot-reload.
  - `github.com/mmcdole/gofeed` - RSS/Atom feed parsing.
  - `github.com/refraction-networking/utls` - TLS fingerprinting for some external requests.
  - `github.com/shirou/gopsutil/v4` - system metrics.
  - `github.com/tidwall/gjson` - JSON path extraction for custom API widgets.
  - `golang.org/x/crypto` - bcrypt password hashing.
  - `golang.org/x/net` and `golang.org/x/text` - networking and i18n formatting.

## Build and Run Commands

Build a binary for the current platform:

```bash
go build -o build/glance .
```

Run directly without producing a binary:

```bash
go run .
```

Cross-compile (for example, Linux amd64):

```bash
GOOS=linux GOARCH=amd64 go build -o build/glance .
```

The binary looks for `glance.yml` in the working directory by default. Use `--config` to override:

```bash
./build/glance --config /etc/glance.yml
```

Build a Docker image locally:

```bash
docker build -t owner/glance:latest .
```

## CLI Commands

```
glance [options] command

Options:
  -config string   Set config path (default "glance.yml")

Commands:
  config:validate       Validate the config file
  config:print          Print the parsed config file with embedded includes
  password:hash <pwd>   Hash a password
  secret:make           Generate a random secret key
  sensors:print         List all sensors
  mountpoint:info       Print information about a given mountpoint path
  diagnose              Run diagnostic checks
```

## Testing

Run the test suite:

```bash
go test ./...
```

There is currently only one test file: `internal/glance/auth_test.go`, which covers session token generation, verification, regeneration, expiration, and tampering. When adding new functionality, add tests in the same package (`package glance`) alongside the code.

## Code Style Guidelines

Follow the existing Go style in the repository:

- Use standard Go formatting (`gofmt`).
- Keep the standard library as the default; avoid introducing new dependencies. The README explicitly says "Avoid introducing new dependencies" and "No package.json".
- Use `log` or `log/slog` for logging. Avoid third-party loggers.
- Prefer explicit, simple code over clever abstractions.
- Widget implementations live in `internal/glance/widget-<type>.go` and embed `widgetBase`.
- Custom YAML field types live in `internal/glance/config-fields.go`.
- Templates live in `internal/glance/templates/` and are parsed with `mustParseTemplate`.
- CSS files live in `internal/glance/static/css/`. Use `@import` for partials; the runtime bundler inlines them.
- Do not hard-code colors. Use the theme variables (`primary`, `positive`, `negative`) and existing CSS classes.
- For icons, prefer [Heroicons](https://heroicons.com/) where applicable.

## Widget Architecture

Widgets are the primary unit of functionality. Each widget:

1. Is registered in `newWidget()` in `internal/glance/widget.go` by its `type` string.
2. Embeds `widgetBase` and implements the `widget` interface.
3. Implements `initialize()` to validate config and set up HTTP clients.
4. Implements `update(context.Context)` to fetch data and populate fields.
5. Provides an HTML template (usually parsed in a package-level var) and a `Render()` method.
6. Uses `widgetBase.withCacheDuration()` or `withCacheOnTheHour()` to control caching.

Widgets are rendered server-side into the page template. The page content endpoint (`/api/pages/{page}/content/`) triggers `updateOutdatedWidgets()` to refresh stale widgets concurrently before rendering.

To add a new widget:

1. Add its type to the switch in `newWidget()`.
2. Create a struct in `internal/glance/widget-<type>.go` embedding `widgetBase`.
3. Implement `initialize()`, `update()`, and `Render()`.
4. Add its template to `internal/glance/templates/`.
5. Add documentation in `docs/configuration.md`.

## Configuration System

The config is a single YAML file parsed with `gopkg.in/yaml.v3`. Features:

- **Includes**: Use `!include: path/to/file.yml` or `$include:` to split config across files (see `parseYAMLIncludes`). Recursion depth is limited to 20.
- **Variables**:
  - `${API_KEY}` - environment variable.
  - `\${API_KEY}` - escaped literal.
  - `${secret:api_key}` - Docker secret from `/run/secrets/api_key`.
  - `${readFileFromEnv:PATH_TO_SECRET}` - read file whose absolute path is in the env var.
- **Hot reload**: `configFilesWatcher` watches the main file and its includes; changes debounce for 500 ms and trigger a server restart with the new config.
- **Validation**: `isConfigStateValid()` enforces layout rules (max columns, required full-width columns, page names, etc.).

See `docs/glance.yml` for a starter config and `docs/configuration.md` for all options.

## Theme System

Themes are defined in `themeProperties` with HSL colors and multipliers. The default theme is dark; a light preset is automatically injected unless `disable-picker` is true. Themes are compiled into CSS at startup by executing `theme-style.gotmpl`. Users can select themes via a cookie set by `POST /api/set-theme/{key}`.

When modifying styles:

- Edit CSS in `internal/glance/static/css/`.
- Avoid hard-coding colors; reference theme variables.
- Test both light and dark presets if your change affects color.

## Security Considerations

- **Authentication secrets**: `secret:make` generates a base64-encoded 64-byte key (32 bytes token secret + 32 bytes username hash). Store it in the config or via Docker secrets.
- **Passwords**: User passwords must be >= 6 characters. Hashes are generated with `password:hash` using bcrypt. Plaintext `password` fields in config are hashed in memory at startup and then zeroed.
- **Sessions**: HMAC-SHA256 signed cookies with a 14-day validity and 7-day regeneration window. Cookies are `HttpOnly` and `SameSite=Lax`.
- **Rate limiting**: Failed login attempts are rate-limited per IP (5 attempts per 5-minute window).
- **TLS**: The Go server does not terminate TLS itself; run behind a reverse proxy (set `proxied: true` to trust `X-Forwarded-For` and `X-Forwarded-Proto`).
- **HTML injection**: Widgets that render external HTML require opt-in (`allow-potentially-dangerous-html`) to avoid XSS.
- **Proxy config**: `allow-insecure` on proxy options disables TLS certificate verification; only use when necessary.
- **Do not commit secrets**: `glance*.yml` and `/assets`, `/build` are gitignored. Keep configs and credentials out of the repository.

## Deployment

### Docker

The Dockerfile builds a static binary with `CGO_ENABLED=0` and copies it into an `alpine:3.22` image. The container expects config at `/app/config/glance.yml` and exposes port 8080.

### Release

Releases are automated via GitHub Actions (`.github/workflows/release.yaml`) on tags matching `v*`. GoReleaser (`.goreleaser.yaml`) builds:

- Binaries for Linux, OpenBSD, FreeBSD, Windows, and Darwin on amd64, arm64, arm, and 386.
- Compressed archives (zip for Windows).
- Multi-arch Docker manifests for `amd64`, `arm64`, and `arm/v7` published to Docker Hub.

The build version is injected via ldflags: `-X github.com/glanceapp/glance/internal/glance.buildVersion={{ .Tag }}`.

## Common Development Tasks

### Add a config field

Add the field to the appropriate struct in `internal/glance/config.go` (or widget struct), use a custom type from `config-fields.go` if needed, and validate in `isConfigStateValid()` or the widget's `initialize()`.

### Add a custom YAML parser

Implement `UnmarshalYAML(node *yaml.Node) error` on the field type in `config-fields.go`. Keep error messages concise and consistent.

### Add static assets

Place files under `internal/glance/static/`. Reference them in templates through `app.StaticAssetPath(asset)`, which includes a content-hash prefix for cache busting. CSS is bundled from `css/main.css` automatically.

### Update dependencies

```bash
go get -u ./...
go mod tidy
```

Keep the dependency list small; new dependencies are discouraged unless they provide significant, hard-to-replicate value.

## Contributing Notes (from the project maintainers)

- Use `dev` as the base branch for new features or bug fixes; use `main` otherwise.
- Avoid backwards-incompatible configuration changes.
- Provide screenshots for UI-related changes when possible.
- Submit a feature request before working on a new feature, and avoid PRs for issues already tagged roadmap/backlog/icebox.

## Useful Files for Agents

- `README.md` - User-facing overview, install, build, and contribution guidelines.
- `docs/configuration.md` - Complete widget and server configuration reference.
- `docs/glance.yml` - Example starter configuration.
- `docs/themes.md` - Theme documentation.
- `docs/extensions.md` - Extension widget API (work in progress).
- `internal/glance/widget.go` - Widget interface and base behavior.
- `internal/glance/config.go` - Config parsing, includes, and validation.
- `internal/glance/glance.go` - HTTP routes and application setup.
