# Deploying an MCP App server to a container platform (Cloud Run example)

A verified build → push → deploy sequence for an MCP App server (widget bundles + Express/HTTP MCP transport) shipped as a container, plus the gotchas that actually cost debugging time. Commands below use Google Cloud Run as the concrete example; the shape (build a client asset bundle → build an image that copies it in → push to a registry → deploy) generalizes to other container platforms.

Replace `<gcp-project>`, `<region>`, `<artifact-registry-repo>`, `<service-name>`, `<api-url>` with your own values.

## 0. One-time environment prereq

`gcloud` (and the `docker-credential-gcloud` helper it provides) may not be on `PATH` in every shell, even if it's installed. `docker push` to a cloud registry fails with something like `docker-credential-<provider> executable file not found` without it. If pushes fail with a credential-helper error, check `PATH` before assuming an auth problem:

```bash
export PATH="/path/to/google-cloud-sdk/bin:$PATH"
```

## 1. Build client bundles before the image build

If your MCP App widgets are bundled separately from the server (e.g. Vite IIFE builds copied into a `dist/` the server serves statically), build them **first**, and make sure that output directory is not something your Dockerfile tries to (re)generate — most Dockerfiles for this kind of server just `COPY` a pre-built `dist/` in rather than running the whole widget build inside the image. If that output directory is gitignored, it's easy to deploy a stale or missing bundle by forgetting this step.

```bash
yarn build:widgets   # or your project's equivalent
```

## 2. Build the image from the right context

If the server lives inside a monorepo and its Dockerfile needs sibling packages (a shared lockfile, internal workspace packages), build from the **monorepo root**, not the server's own subdirectory — otherwise the build context won't include what the Dockerfile needs to `COPY`.

```bash
cd <monorepo-root>
docker buildx build --platform linux/amd64 \
  -t <region>-docker.pkg.dev/<gcp-project>/<artifact-registry-repo>/<service-name>:vX.Y.Z \
  --load \
  -f <path-to-server>/Dockerfile .
```

`--platform linux/amd64` matters if you're building on Apple Silicon and deploying to an x86 cloud runtime.

## 3. Push

```bash
docker push <region>-docker.pkg.dev/<gcp-project>/<artifact-registry-repo>/<service-name>:vX.Y.Z
```

Double-check the registry path segments match your actual Artifact Registry repo name before pushing — a copy-pasted command from an old note with the wrong repo segment will fail loudly here (registry doesn't exist), which is the safe failure mode. The *dangerous* failure mode is a wrong **service name** at deploy time — see below.

## 4. Deploy

```bash
gcloud run deploy <service-name> \
  --image <region>-docker.pkg.dev/<gcp-project>/<artifact-registry-repo>/<service-name>:vX.Y.Z \
  --platform managed \
  --region <region> \
  --port 8080 \
  --max-instances 1 \
  --allow-unauthenticated \
  --set-env-vars API_URL=<api-url>
```

**Watch the service name.** `gcloud run deploy <service-name>` does not error if `<service-name>` doesn't match any existing service — it just silently creates a brand-new one. If you've deployed before under one name and a stale copy of a deploy command uses a slightly different name (a typo, an old prefix), you won't get an error; you'll get an orphaned duplicate service running an old image, and no indication anything went wrong. Before running a deploy command, confirm the service name against your own notes/history, not against whatever was last pasted into a terminal.

**`--max-instances 1`** matters specifically if your MCP server keeps any in-memory per-session state (e.g. MCP session objects tied to a connection). Multiple instances behind a load balancer will route follow-up requests to a different instance than the one holding the session, breaking things in a way that looks intermittent and confusing rather than a clean failure.

**`<api-url>`** (or whatever env var selects your backend data source) — pick deliberately per environment; this is the actual observable behavior difference between a prod and staging/dev deploy, easy to get backwards.

## 5. Verify

```bash
curl -s https://<service-name>-<hash>.<region>.run.app/health
gcloud run services list --region <region>   # confirm only the services you expect exist
```

Two different URL formats can point at the same Cloud Run service (one shown by `gcloud run deploy`'s own output, another by `gcloud run services list`) — don't assume a different-looking URL means a duplicate service; check `services list` count instead.

## Redeploying with different config, same image

No need to rebuild/repush for a config-only change (e.g. switching which backend API a deploy points at). Rerun the deploy command with the same image tag and a different `--set-env-vars` value — Cloud Run creates a new revision and routes traffic to it immediately.
