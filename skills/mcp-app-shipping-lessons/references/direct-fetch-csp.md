# Direct fetch from the widget iframe, via CSP

## The trap

It's tempting to assume a sandboxed MCP App iframe can't reach external APIs, and to build a workaround: the server prefetches data and injects it into the tool result, or the widget calls a server-side tool that proxies the real request. Both add latency, duplicate fetch logic, and break naturally-occurring client-driven queries (e.g. a click-to-expand detail panel that fires its own GraphQL query) — those need hand-rolled "fetch everything up front and filter client-side" logic to work around the relay.

## What actually works

The sandboxed iframe *can* fetch directly, as long as the server declares the target origins in the resource's CSP metadata. In the `@modelcontextprotocol/ext-apps` SDK this is set per-resource, typically in the resource's `_meta`:

```ts
_meta: {
  "ui/csp": {
    connectDomains: [apiUrl, ...anyExtraOrigins],
  },
}
```

`connectDomains` governs `fetch`/`XHR`/WebSocket-style network access from inside the iframe. Once it's set, a normal Apollo Client / `fetch()` call inside the widget component works exactly like it would in a regular web app — no interceptor, no relay tool, no server-side prefetch step.

If a widget needs data from more than one origin (e.g. the primary GraphQL API plus a public data CDN for a 3D viewer or file preview), just list every origin it needs:

```ts
extraConnectDomains: ["https://some-external-cdn.example", "https://another-api.example"],
```

## The easy-to-miss sibling: `resourceDomains`

`connectDomains` only covers fetch/XHR. If your widget's HTML shell also loads *static* cross-origin resources — web fonts via `<link>`, images, external `<script>` tags — those need `resourceDomains` set too. Per spec, an omitted `resourceDomains` defaults to blocking cross-origin static resource loads. This is easy to miss because "data fetching" (the exciting part) works fine while an unrelated `<link rel="stylesheet">` to a font host fails silently in the background — check your host's dev console for CSP violations, not just broken data.

## When you actually do need a server-side relay

Keep the server-side path only for data the client genuinely can't or shouldn't fetch itself: secrets/API keys that can't be exposed to the iframe, requests that must be authenticated as the *server* rather than the end user, or origins you deliberately don't want to CSP-allowlist. Don't default to it — verify the direct-fetch path doesn't work first.
