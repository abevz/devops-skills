---
name: dockerfile-review
description: Use when reviewing a Dockerfile or container image build for size, security, and reproducibility. Mention "review this dockerfile", "container image review", "why is my image huge" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Dockerfile Review

## TL;DR checklist

- [ ] Inspect base image, build stages, final user, and copied artifacts.
- [ ] Check reproducibility, cache behavior, image size, and secret exposure.
- [ ] Report concrete changes to the final image or build.

## Key read-only checks

- Read the Dockerfile, build context, lockfiles, and supplied image metadata.

## Common pitfalls

- Do not copy build-only tools or secrets into the final stage.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/examples.md](references/examples.md)

Last verified: unverified

## When to use

Use when reviewing a Dockerfile (or Containerfile) before it ships — new images, base image
changes, or "the image is too big / too slow to build / flagged by the scanner" complaints.

## Goal

Catch image-build mistakes that cause bloated images, broken caching, security findings, or
non-reproducible builds — with the concrete fix for each.

## Workflow

1. **Base image** — pinned to a specific version (ideally a digest), not `latest`; minimal
   variant where practical (`-slim`, distroless, alpine — noting musl caveats for Go/CGO and
   native deps); official or org-approved source.
2. **Multi-stage builds** — build toolchain (compilers, dev headers, npm/go caches) stays in a
   builder stage; final stage copies only artifacts. A Go/Java/Node app shipping its compiler is
   the most common bloat finding.
3. **Non-root user** — final stage sets `USER` to a non-root UID; file ownership (`COPY --chown`)
   matches. Flag images that only work as root.
4. **Layer hygiene** — related `RUN` commands chained where it saves layers; package-manager
   caches cleaned in the same layer they're created (`apt-get clean`/`rm -rf /var/lib/apt/lists/*`,
   `--no-cache` for apk); no "delete in a later layer" pattern (the data is still in the earlier
   layer).
5. **Cache ordering** — dependency manifests (`go.mod`, `package.json`, `requirements.txt`)
   copied and installed *before* copying the full source, so code changes don't bust the
   dependency cache.
6. **Secrets** — no secrets via `ARG`/`ENV`/`COPY` (all end up readable in image history or
   layers); build-time secrets use BuildKit `--mount=type=secret`; `.dockerignore` excludes
   `.env`, `.git`, credentials.
7. **COPY vs ADD** — `COPY` unless the auto-extract/URL behavior of `ADD` is genuinely needed
   (it almost never is).
8. **Runtime contract** — `EXPOSE` documents the port; `HEALTHCHECK` present where the
   orchestrator doesn't provide probes (plain Docker/Compose); `ENTRYPOINT`/`CMD` use exec form
   (JSON array) so signals reach the process (PID 1 must handle SIGTERM for graceful shutdown).
9. **Reproducibility** — dependency versions pinned (lockfiles used in the build); no
   `curl | sh` of unpinned installers mid-build.

## References

- `references/examples.md` — bad → good multi-stage pairs for Go/Node/Python with the
  mechanics behind each finding (signal handling, layer additivity, BuildKit secret/cache
  mounts, digest pinning, HEALTHCHECK applicability).
- Optional tooling: `hadolint` (Dockerfile lint) and `trivy image` / `grype` (CVE scan of the
  built image) can supplement this review if the user already has them — suggest, never
  install, and only run them with the user's confirmation.

## Safety rules

- This is a review skill: do not build, push, or publish images.
- Never echo a secret discovered in a Dockerfile, `.env`, or image history — describe the
  exposure and the fix without quoting the value.
- Alpine/distroless recommendations must note compatibility caveats (musl vs glibc, no shell for
  debugging) rather than being suggested unconditionally.

## Output format

```
## Critical (security / broken behavior)
## Major (bloat, cache-busting, reproducibility)
## Minor / style
## Suggested Dockerfile changes
<concrete diff-style snippets for the top findings>
```

## Quality checklist

- [ ] Base image pinning and final-stage `USER` were checked
- [ ] Build-time secret exposure (ARG/ENV/history) was checked
- [ ] Cache ordering (deps before source) was checked
- [ ] Signal handling (exec-form ENTRYPOINT) was checked
- [ ] No image was built or pushed
