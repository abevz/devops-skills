# Dockerfile bad → good examples

Self-authored reference (no third-party source). Language-specific multi-stage pairs and the
mechanics behind each finding.

## Go — the full multi-stage pattern

❌ Bad (ships the toolchain, runs as root, busts cache on every code change):

```dockerfile
FROM golang:1.24
WORKDIR /app
COPY . .                          # any file change invalidates everything below
RUN go build -o server ./cmd/server
CMD go run ./cmd/server           # shell form + running source, toolchain in prod
```

✅ Good:

```dockerfile
FROM golang:1.24-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download               # cached until go.mod/go.sum change
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
USER nonroot
ENTRYPOINT ["/server"]
```

Mechanics worth citing in review: `CGO_ENABLED=0` + `distroless/static` avoids the
musl-vs-glibc question entirely; `nonroot` variant = UID 65532; exec-form ENTRYPOINT means
SIGTERM reaches the process (shell form wraps in `/bin/sh -c`, PID 1 is the shell, signals die
there — graceful shutdown breaks).

## Node.js

❌ Bad: `FROM node:latest`, `COPY . .` before `npm install`, `npm install` (with devDeps),
running as root, `CMD npm start` (npm swallows SIGTERM).

✅ Good:

```dockerfile
FROM node:22-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-slim
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --chown=node:node . .
USER node
ENTRYPOINT ["node", "server.js"]     # not "npm start" — npm doesn't forward signals
```

`npm ci` (not `install`) = lockfile-exact, fails on drift. The built-in `node` user exists in
official images — no need to create one.

## Python

✅ Key lines beyond the same multi-stage shape:

```dockerfile
FROM python:3.13-slim AS deps
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.13-slim
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1
COPY --from=deps /install /usr/local
RUN useradd -r -u 10001 app
USER app
ENTRYPOINT ["python", "-m", "myapp"]
```

`PYTHONUNBUFFERED=1` or logs vanish on crash (stdout buffering). Alpine + Python = compiling
wheels from source (musl) — slim-Debian is almost always the right base here.

## Secrets in builds — what actually leaks

- ❌ `ARG NPM_TOKEN` used in `RUN` — persists in `docker history` output for that layer.
- ❌ `COPY .env .` / `COPY id_rsa .` then `RUN rm` — still in the earlier layer, extractable
  from any registry pull.
- ✅ BuildKit secret mount — exists only during that RUN, never in a layer:

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
# build with: docker build --secret id=npmrc,src=$HOME/.npmrc .
```

- ✅ SSH for private repos: `RUN --mount=type=ssh git clone ...` + `docker build --ssh default`.
- `.dockerignore` review floor: `.git`, `.env*`, `*.pem`, `id_*`, `node_modules`, build output
  dirs. Missing `.git` alone can leak the entire history including previously-committed secrets.

## Cache and layer mechanics behind the findings

- Deletion in a later layer doesn't shrink the image — layers are additive; cleanup must happen
  in the same `RUN` that created the data:
  `RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*`
- Layer order = cache hierarchy: least-changing first (base config → deps manifests → dep
  install → source copy → build). A `COPY . .` above dependency install turns every commit into
  a full rebuild.
- `RUN --mount=type=cache,target=/root/.cache/go-build` (BuildKit) persists compiler/package
  caches across builds without baking them into layers — recommend for slow CI builds.
- Pinning: `FROM node:22-slim@sha256:<digest>` is reproducible; `:22-slim` alone moves under
  you (patch releases, even rebuilt same-version images). Digest-pin at least for prod images,
  with automation (Renovate/Dependabot) to update the pin.

## HEALTHCHECK — when it belongs

- Kubernetes ignores Dockerfile `HEALTHCHECK` (probes replace it). Including it anyway costs a
  periodic exec; harmless but not "missing = finding" on k8s-only images.
- Docker/Compose/Swarm deployments: it IS the health signal —
  `HEALTHCHECK --interval=30s --timeout=3s CMD ["/bin/grpc_health_probe", "-addr=:8080"]` or an
  HTTP check binary. Avoid `curl`-based checks in distroless (no curl) — ship a tiny probe
  binary or use the app's own flag.
