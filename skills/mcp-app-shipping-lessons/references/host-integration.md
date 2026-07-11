# Host integration traps: links and other "ambient browser" APIs

## The symptom

A component that renders a normal `<a href="..." target="_blank">` (or calls `window.open(url)`) for external navigation works perfectly when the widget is tested standalone in a browser tab, but silently does nothing when the same widget runs inside an MCP App host's sandboxed iframe. No error, no console warning by default — the click just doesn't navigate anywhere.

## Why

MCP App iframes are sandboxed by the host, and popup/new-tab permissions are not reliably granted to sandboxed content. This isn't a bug to route around with `rel="noopener"` tricks or a different `target` value — the sandbox genuinely doesn't guarantee that path works.

## The fix

The Apps SDK exposes a dedicated host RPC method for exactly this: something like `app.openLink({ url })` — "request the host to open an external URL in the default browser," handled outside the DOM/CSP model entirely (check your SDK's `app.d.ts` or equivalent for the exact method name and signature; it was `openLink` at time of writing for `@modelcontextprotocol/ext-apps`).

Concretely:

```tsx
function ExternalLink({ url, children }) {
  const handleClick = (e: React.MouseEvent) => {
    e.preventDefault();
    getApp()?.openLink({ url });
  };
  return <a href={url} onClick={handleClick}>{children}</a>;
}
```

Keep the `href` for accessibility/right-click-copy, but intercept the click so the host RPC — not the native anchor behavior — actually performs the navigation.

## Where this bites you more than once

If your widget UI reuses/wraps components from a larger design system, check for **every** place that renders a raw external link or calls a "new tab" style API — not just the one you noticed first. A generic `Link` component fixed this way is a good start, but any component that composes its *own* separate link/button (rather than delegating to your fixed `Link`) will reintroduce the same silent failure. See `component-reuse.md` for the specific "relative import bypasses the fix" version of this problem.

## The general pattern

Before writing a workaround for something that "doesn't work" in the sandbox, check whether the host SDK has a purpose-built RPC for it. Camera, microphone, geolocation, clipboard writes, and external navigation are all the kind of ambient capability that a sandbox intentionally gates behind an explicit host-mediated call rather than a raw browser API — look for the SDK's permission/capability list before assuming something is simply broken or impossible.
