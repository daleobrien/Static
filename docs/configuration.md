# Configuration

[Back to the README](../README.md)

All configuration is resolved at **build time** by `build.rs` and baked into the binary. The server performs zero environment variable reads and zero config file I/O at runtime.

Set variables on the command line when building, or persist them in `.cargo/config.toml`:

```toml
[env]
HOSTNAME = "myhost.local"
PORT = "8080"
WORKERS = "4"
MAX_CONNS = "2048"
SHUTDOWN_TIMEOUT_SECS = "60"
NOT_FOUND_FILENAME = "not-found.html"
DISABLE_SRI = "true"
ALLOW_INLINE_STYLES = "true"
FORM_ACTION = "https://auth.example.com"
CONNECT_SRC = "https://api.example.com https://something.else:3443"
```

| Variable | Default | Description |
| --- | --- | --- |
| `HOSTNAME` | `localhost` | Hostname displayed in the startup banner and used in auto-generated TLS certificate SANs |
| `PORT` | `3000` | Server port (TCP, and UDP for HTTP/3) |
| `WORKERS` | *available parallelism* (floor 4) | Number of worker threads (one TCP listener and QUIC endpoint each) |
| `MAX_CONNS` | `4096` | Maximum concurrent TCP connections (enforced via a shared semaphore) |
| `TCP_HANDLERS_PER_WORKER` | `max(MAX_CONNS / WORKERS, 64)` | Number of pre-spawned TCP handler tasks per worker |
| `H3_HANDLERS_PER_CONNECTION` | `8` | Number of pre-spawned h3 handler tasks per QUIC connection |
| `H2_CONN_WINDOW` | `16777216` (16 MiB) | HTTP/2 connection-level flow-control window in bytes
| `H2_STREAM_WINDOW` | `4194304` (4 MiB) | HTTP/2 per-stream flow-control window in bytes
| `H2_MAX_FRAME_SIZE` | `65535` | HTTP/2 max frame size (range: 16384–16777215)
| `H2_MAX_SEND_BUF` | `1048576` (1 MiB) | HTTP/2 max per-stream write buffer before backpressure
| `SHUTDOWN_TIMEOUT_SECS` | `30` | Graceful shutdown timeout — how long to wait for in-flight requests after SIGINT/SIGTERM |
| `NOT_FOUND_FILENAME` | `404.html` | Name of the file in `public/` used as the custom 404 page |
| `DISABLE_SRI` | `false` | Set to `1` or `true` to disable Subresource Integrity (content-hashed filenames, `integrity` attributes in HTML, and CSP hash allowlisting). When disabled, CSP uses `'self' 'unsafe-inline'` for scripts and styles to allow the inline version-check script. |
| `ALLOW_INLINE_STYLES` | `false` | Set to `1` or `true` to add `'unsafe-inline'` to the `style-src` CSP directive while keeping SRI hashes for all other resources. Use this when React packages (e.g. styled-components, emotion) need to inject `<style>` elements dynamically at runtime. |
| `FORM_ACTION` | *(not set)* | Space-separated list of additional external origins for the `form-action` CSP directive. Set this when your app submits `<form>` POSTs to external endpoints (e.g. SAML authentication callbacks). `'self'` is always included automatically. |
| `CONNECT_SRC` | *(not set)* | Space-separated list of additional external origins for the `connect-src` CSP directive. Set this when your app makes fetch/XHR calls to external backends. `'self'` is always included automatically. |
| `DISABLE_HTTP3` | `true` | HTTP/3 (QUIC) is left out by default, since static sites rarely need it. Set to `false` to build it in, with a QUIC endpoint on the same port as TCP. |
| `EMBED_GZIP` | `true` | Embed a gzip copy of every file. |
| `EMBED_BROTLI` | `true` | Embed a Brotli copy of every file. |
| `EMBED_ZSTD` | `true` | Embed a zstd copy of every file. |
| `DISABLE_LOGGING` | `true` | Compile out all stderr output (request logs, error messages, startup banner, and `--help` text). Set to `false` to re-enable logging. |

In addition to the user-configurable variables above, the build consumes two environment variables that are set automatically and are not meant to be configured manually:

| Variable | Set by | Purpose |
| --- | --- | --- |
| `OUT_DIR` | Cargo | Output directory for the build script; `build.rs` writes `generated.rs` and `config_constants.rs` there, and the runtime reads them back via `include!`/`include_bytes!` |
| `TARGETARCH` | Docker | BuildKit-provided build arg that selects the musl target triplet (`x86_64-unknown-linux-musl` vs `aarch64-unknown-linux-musl`) in the Dockerfile |
