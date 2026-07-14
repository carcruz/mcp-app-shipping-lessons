---
name: mcp-app-shipping-lessons
description: Lessons for hardening and debugging MCP Apps (tools with interactive iframe UIs) once a prototype already exists. Covers sandboxed-iframe data fetching via CSP instead of server-side prefetch relays, why external links need a host RPC call instead of raw anchor tags, a reuse-vs-stub decision framework for widget UI components (including bundler-level import interception and safe stub return values), container-build gotchas (including runtime-read files the Dockerfile forgot to COPY), auditing whether a multi-widget migration is actually complete, diagnosing generic "unable to reach" host errors, and why in-memory MCP sessions need a min-instances floor on autoscaled/serverless deploys. Use this whenever debugging why an MCP App widget can't fetch data, can't open a link, looks visually "off" from the rest of an app, silently does nothing for some inputs, when building or deploying the server as a container image, when a host shows a generic connectivity error, or when checking whether "all the widgets" really are all of them. Does not cover initial scaffolding — pair with create-mcp-app, add-app-to-server, convert-web-app, or migrate-oai-app for that.
---

# MCP App shipping lessons

Practical, field-tested lessons from taking an MCP App widget server from prototype to production. Each reference file below is self-contained — read only the one relevant to the current problem.

## When something looks wrong, check these first

1. **Widget can't load data, or you're tempted to build a server-side data relay** → read `references/direct-fetch-csp.md`. Sandboxed MCP App iframes usually *can* fetch external APIs directly; the fix is almost always a CSP config field, not a new server endpoint.

2. **A link/button inside the widget silently does nothing** (works in a normal browser tab, dead inside the MCP host) → read `references/host-integration.md`. This is almost always a raw DOM API (`<a target="_blank">`, `window.open()`) that the sandbox doesn't grant, when a host RPC method exists for it.

3. **Widget UI looks subtly different from the "real" app it's borrowing components from, or a feature silently does nothing/renders empty for every input** (wrong colors, missing icons, broken styling, a hook that never surfaces data) → read `references/component-reuse.md` for the decision framework on when to reuse a real component vs. keep a stub, three variants of an import-path gotcha that causes "I fixed this already" bugs (and the bundler-level fix that handles all three at once), how to design a stub's return value so it doesn't silently gate a feature off, and how to verify a stub fix actually reached the built bundle.

4. **Building the server as a container image** → read `references/docker-build.md` for the client-bundle/build-context/platform gotchas that cost real debugging time, including runtime-read config files the Dockerfile forgot to `COPY` (works in dev, 404s only on the one request path that reads them in production).

5. **Checking whether a multi-widget migration (one MCP tool per feature of a larger app) is actually complete** → read `references/migration-completeness.md`. A hand-maintained "excluded features" list drifts out of date in both directions — stale exclusion reasons, and similar-looking features wrongly assumed to be "already covered" by each other. Diff against the source app's own feature list programmatically instead of trusting the doc.

6. **A host shows a generic "Unable to reach `<server>`" error** → read `references/host-error-diagnosis.md` before touching infrastructure. That message covers everything from a genuinely down server to a normal application-level JSON-RPC error dressed up as a connectivity failure — check the client-side transport log first to see which specific call is actually failing.

7. **The server works, then randomly stops responding after being idle, on a serverless/autoscaled deploy target** → read `references/serverless-scaling.md`. In-memory MCP session state doesn't survive the platform scaling the service down to zero between requests — this needs a `min-instances` floor, not a session/auth fix.

## Core mental model

MCP App hosts (Claude Desktop, ChatGPT, etc.) run widget UIs in a **sandboxed iframe**, not a full browser tab. That sandbox:
- Restricts network access via CSP fields the server declares per-resource — it does not silently block everything, and it does not silently allow everything.
- Does not grant ambient browser capabilities (opening a new tab/window, for one) — anything like that goes through an explicit host RPC method instead.

Most "it doesn't work in the MCP host but works standalone" bugs trace back to one of these two restrictions being unaccounted for, not a fundamental limitation of what the sandbox can do. Before designing a workaround (a server-side relay, a custom postMessage bridge, etc.), check whether the SDK already has a sanctioned, documented way to do it — it usually does.
