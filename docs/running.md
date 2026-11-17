# Building and running

[Back to the README](../README.md)

> **Note:** Local builds use `target.tmp` as the Cargo target directory (instead of the default `target/`) to avoid conflicts with Docker builds. The `target.tmp/` directory is gitignored.

## Local

Default build (listens on `localhost:3000` with auto-detected parallelism):

```sh
cd rust-server
cargo run
```

Custom configuration via build-time env vars:

```sh
PORT=8080 WORKERS=4 cargo run
```

CLI flags (run `cargo run -- --help` for the full list):

```sh
cargo run -- --summary    # Log aggregated req/s every 5s instead of per-request details
```

The server listens on the configured hostname and port:
- `http://localhost:3000/` — plain HTTP
- `https://localhost:3000/` — TLS (HTTP/1.1 or HTTP/2, and HTTP/3 if built in)

## Docker

The Docker build cross-compiles a fully static binary with musl, then compresses it with [UPX](https://upx.github.io/) for minimal image size (`FROM scratch`, ~1.2 MB). A self-signed TLS certificate is generated at build time (or picked up from `certs/` if present).

```sh
docker build -f rust-server/Dockerfile -t app-rust .
docker run -p 3000:3000 --rm app-rust
```

The Dockerfile's only build arg is `TARGETARCH`, which Docker sets automatically (see [Configuration](configuration.md)). The server's build-time env vars are **not** forwarded by the current Dockerfile — to bake them in, declare ARGs and pass them to the `cargo build` step:

```dockerfile
ARG HOSTNAME=myhost.local
ARG PORT=8080
# ... then use: HOSTNAME=$HOSTNAME PORT=$PORT cargo build --release --target "$RUST_TARGET"
```

## go.sh

`go.sh` builds the React app in `react-app/` into `public/` first (if it is
there), then:

```sh
./go.sh            # run the tests, then build and run the server locally, with logging on
./go.sh --docker   # build and run the Docker image
./go.sh --nginx    # the same site served by nginx, for comparison
./go.sh --node     # the same site served by Node, for comparison
```

## Tests

```sh
cd rust-server && cargo test   # unit and integration tests
./test-env-params.sh           # every build-time variable, set to a non-default value
./verify.sh                    # the full verification: tests, miri, size checks and load tests
```

## Logging

### Detailed mode (default)

Logs are batched and flushed once per second with column-aligned output:

```
PR  METHOD   PATH            STA  SIZE  TIME
h2  GET      /                  200   123B  76µs
h1  GET      /about             200   456B  89µs
h3  GET      /style.a1b2.js     200   789B  49µs
```

### Summary mode (`--summary`)

A lightweight counter that prints req/s every 5 seconds:

```
127 requests in the last 5s (25.4 req/s)
```

## Project structure

```
├── certs/           # Optional custom TLS certificates
├── public/          # Your static assets (HTML, CSS, JS, SVG, etc.)
├── react-app/       # An example React app, built into public/ by go.sh
├── rust-server/     # The Rust server
│   ├── Cargo.toml
│   ├── Dockerfile
│   ├── build.rs     # Build script (config, minification, SRI, codegen, TLS)
│   ├── build_helpers/
│   ├── benches/
│   ├── tests/
│   └── src/         # Runtime source code
├── nginx-server/    # nginx serving the same files, for comparison
├── node-server/     # Node serving the same files, for comparison
├── docs/            # These documents
├── go.sh            # Run locally, or in Docker (Rust, nginx or Node)
├── test-env-params.sh
├── verify.sh
└── README.md
```

## Compared with nginx and Node

The same site served three ways (`./go.sh --docker`, `--nginx` and `--node`):

| Runtime | RAM (MB)  | Image Size (MB) | Index load (ms) |
| :--- | ---: | ---: | ---: |
| Rust-server |  8 | 1.46 | 2 |
| Nginx-server |   15 | 13.4 | 4 |
| Node-server  | 282 | 307 | 10 |
