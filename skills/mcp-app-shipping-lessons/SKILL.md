---
name: mcp-app-shipping-lessons
description: Lessons for hardening and debugging MCP Apps (tools with interactive iframe UIs) once a prototype already exists. Covers sandboxed-iframe data fetching via CSP instead of server-side prefetch relays, why external links need a host RPC call instead of raw anchor tags, a reuse-vs-stub decision framework for widget UI components, and container-build gotchas. Use this whenever debugging why an MCP App widget can't fetch data, can't open a link, looks visually "off" from the rest of an app, or when building the server as a container image. Does not cover initial scaffolding — pair with create-mcp-app, add-app-to-server, convert-web-app, or migrate-oai-app for that.
---

# MCP App shipping lessons

Practical, field-tested lessons from taking an MCP App widget server from prototype to production. Each reference file below is self-contained — read only the one relevant to the current problem.

## When something looks wrong, check these first

1. **Widget can't load data, or you're tempted to build a server-side data relay** → read `references/direct-fetch-csp.md`. Sandboxed MCP App iframes usually *can* fetch external APIs directly; the fix is almost always a CSP config field, not a new server endpoint.

2. **A link/button inside the widget silently does nothing** (works in a normal browser tab, dead inside the MCP host) → read `references/host-integration.md`. This is almost always a raw DOM API (`<a target="_blank">`, `window.open()`) that the sandbox doesn't grant, when a host RPC method exists for it.

3. **Widget UI looks subtly different from the "real" app it's borrowing components from** (wrong colors, missing icons, broken styling) → read `references/component-reuse.md` for the decision framework on when to reuse a real component vs. keep a stub, plus a specific import-path gotcha that causes "I fixed this already" bugs.

4. **Building the server as a container image** → read `references/docker-build.md` for the client-bundle/build-context/platform gotchas that cost real debugging time.

## Core mental model

MCP App hosts (Claude Desktop, ChatGPT, etc.) run widget UIs in a **sandboxed iframe**, not a full browser tab. That sandbox:
- Restricts network access via CSP fields the server declares per-resource — it does not silently block everything, and it does not silently allow everything.
- Does not grant ambient browser capabilities (opening a new tab/window, for one) — anything like that goes through an explicit host RPC method instead.

Most "it doesn't work in the MCP host but works standalone" bugs trace back to one of these two restrictions being unaccounted for, not a fundamental limitation of what the sandbox can do. Before designing a workaround (a server-side relay, a custom postMessage bridge, etc.), check whether the SDK already has a sanctioned, documented way to do it — it usually does.
