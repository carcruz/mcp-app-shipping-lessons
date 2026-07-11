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
