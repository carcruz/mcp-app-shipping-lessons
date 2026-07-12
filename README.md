# mcp-app-shipping-lessons

A Claude Code skill distilling lessons learned from taking an [MCP App](https://modelcontextprotocol.io/) widget server from a working prototype to a hardened, deployed production service.

It's **complementary, not a replacement**, for scaffolding-focused skills like `create-mcp-app`, `add-app-to-server`, `convert-web-app`, and `migrate-oai-app`. Use those to build the thing; use this once you're hardening, debugging, or deploying it.

## What's in here

- Why sandboxed widget iframes usually don't need a server-side data relay — and the CSP fields (`connectDomains`, `resourceDomains`) that make direct fetch work
- Host-integration traps: why raw `<a target="_blank">` / `window.open()` silently fail inside an MCP App host, and the RPC method that actually works
- A decision framework for whether to reuse a real UI component (with a thin adapter) or keep a stub — three variants of the "component bypasses your stub via a relative import" gotcha and the bundler-level fix that handles all of them at once, how to design a stub's return value so it doesn't silently disable a feature instead of crashing, and how to verify a stub fix actually reached the built bundle
- Container-build gotchas for an MCP App server: build-context location in a monorepo, stale/missing client bundles, and platform mismatches
- Auditing whether a multi-widget migration (one MCP tool per feature of a larger app) is actually complete — diffing against the source app's own feature list instead of trusting a hand-maintained exclusion doc that drifts out of date

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
