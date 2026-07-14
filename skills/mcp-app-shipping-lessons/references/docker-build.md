# Building an MCP App server image

An MCP App server usually ships as two things bolted together: a set of client-side widget bundles (built separately, e.g. via Vite) and the server itself (HTTP/MCP transport) that serves them. That split is where most of the build gotchas live — not in Docker itself.

## 1. Build client bundles before the image build

If your widgets are bundled separately from the server (e.g. Vite IIFE builds copied into a `dist/` the server serves statically), build them **first**, and make sure your Dockerfile doesn't try to (re)generate that output — most Dockerfiles for this kind of server just `COPY` a pre-built `dist/` in rather than running the whole widget build inside the image. If that output directory is gitignored, it's easy to silently ship a stale or missing bundle by forgetting this step — the image builds "successfully" and serves widgets that are empty, outdated, or missing entirely.

```bash
yarn build:widgets   # or your project's equivalent
```

## 2. Build from the right context

If the server lives inside a monorepo and its Dockerfile needs sibling packages (a shared lockfile, internal workspace packages), build from the **monorepo root**, not the server's own subdirectory. Docker only sees files inside the build context you give it — build from the wrong folder and any `COPY` referencing a sibling package just can't find it. This fails at build time if you're lucky (missing file error) and silently produces a broken image if you're not (an optional/fallback code path swallows the miss).

```bash
cd <monorepo-root>
docker buildx build --platform linux/amd64 \
  -t <image-name>:vX.Y.Z \
  --load \
  -f <path-to-server>/Dockerfile .
```

Note the context is the trailing `.` — it's relative to wherever you run the command from, not to the Dockerfile's own location (`-f` can point anywhere; the context still comes from `.`).

## 3. Watch for platform mismatches

`--platform linux/amd64` matters if you're building on Apple Silicon (arm64) and the image is meant to run somewhere x86-only. Without it, `docker buildx` will happily build a native arm64 image that builds fine, runs fine locally, and then fails to start (or silently gets emulated at a real performance cost) wherever it's deployed. If a container "works on my machine" but misbehaves after deploy, check the platform flag before anything else.

## 4. Every runtime-read file needs its own `COPY` line — not just source and bundles

A Dockerfile's `COPY` lines are a hand-maintained list, and nothing keeps that list in sync with what the server actually reads off disk at request time. It's easy to remember to `COPY` the server's source and the built widget bundles, and forget a smaller runtime dependency added later — a config file, a per-environment script, a template, anything read via `readFile`/`require` outside of `node_modules` and the two obvious directories.

This bug is nasty specifically because of *when* it surfaces: the image builds cleanly (Docker doesn't know or care that a runtime `readFile` call will 404), the container starts and passes health checks (nothing touches the missing file at startup), and it only breaks the exact request path that reads that file — which, for an MCP App server, usually means the resource-read handler that serves the widget's HTML shell, not the tool-call handler. So `tools/list` and `tools/call` succeed, and the failure looks like a connectivity/session problem in the host (see `host-error-diagnosis.md`) rather than a missing file, because the host only sees a generic JSON-RPC error, not the underlying `ENOENT`.

When a feature is added that makes the server read a *new* file or directory at runtime (a profile/config script, a template, anything outside the paths the Dockerfile already copies), treat updating the Dockerfile as part of that change, not a separate deploy concern. To verify a rebuilt image actually contains everything it needs before pushing:

```bash
docker run --rm --entrypoint sh <image>:<tag> -c "find / -iname '<expected-file-or-dir>' 2>/dev/null"
```

An empty result means the Dockerfile is missing a `COPY` line for it — fix that before pushing, not after a user hits the broken code path in production.
