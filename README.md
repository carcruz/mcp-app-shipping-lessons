# mcp-app-shipping-lessons

A Claude Code skill distilling lessons learned from taking an [MCP App](https://modelcontextprotocol.io/) widget server from a working prototype to a hardened, deployed production service.

It's **complementary, not a replacement**, for scaffolding-focused skills like `create-mcp-app`, `add-app-to-server`, `convert-web-app`, and `migrate-oai-app`. Use those to build the thing; use this once you're hardening, debugging, or deploying it.

## What's in here

- Why sandboxed widget iframes usually don't need a server-side data relay — and the CSP fields (`connectDomains`, `resourceDomains`) that make direct fetch work
- Host-integration traps: why raw `<a target="_blank">` / `window.open()` silently fail inside an MCP App host, and the RPC method that actually works
- A decision framework for whether to reuse a real UI component (with a thin adapter) or keep a stub, plus a specific gotcha around components that bypass your stub via relative imports
- Container-build gotchas for an MCP App server: build-context location in a monorepo, stale/missing client bundles, and platform mismatches

## Install

**As a plugin (recommended):**

```
/plugin marketplace add carcruz/mcp-app-shipping-lessons
/plugin install mcp-app-shipping-lessons
```

**Manually:**

Copy `skills/mcp-app-shipping-lessons/` into your `~/.claude/skills/` directory (or your project's `.claude/skills/`).

## License

MIT
