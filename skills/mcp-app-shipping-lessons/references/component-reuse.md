# Reuse vs. stub: a decision framework for widget UI components

When a widget bundle is built by extracting components out of a larger app (rather than writing a widget from scratch), the extraction step usually leaves behind hand-written stand-in "stub" versions of shared components — a fake `Link`, a hardcoded theme object, a simplified icon component — because the real ones assume things a standalone bundle doesn't have (routing context, a full design-system provider, dev-only dependencies).

Stubs drift. Months later they're subtly wrong — wrong colors, dropped props (`className`, `onClick`, `ariaLabel` silently swallowed), missing icon variants — and it's not obvious from a glance which stubs are "fine to leave" and which are actively broken.

## Two categories

**1. Broken duplicates of something that works fine standalone.** Theme/color tokens, a generic `Link`, a presentational component like a star-rating or badge — these usually have no real dependency on the "full app" context. They're stubbed only because nobody's revisited them since the original extraction. Fix: **reuse the real component/module directly**, adding a minimal adapter only for what's genuinely contextually different (e.g. navigation semantics — see `host-integration.md` for the specific link-opening adapter).

**2. Structural stubs that exist because the real thing depends on something genuinely absent** in a standalone widget — a heavy dev-only dependency (a GraphQL playground UI, say), a missing upstream service, or a UX that only makes sense with a full page shell (a persistent download-manager drawer, a page-level tooltip provider). These should **stay stubbed** — this isn't unfinished work, it's a legitimate design choice given the widget's constraints.

## How to tell which is which

Before touching any stub, ask two questions:
1. Are the real component's dependencies actually resolvable in a standalone bundle (no giant dev-only packages, no missing service)?
2. Does its behavior still make sense without the full app shell (real routes, real hover/preview UX, a real download flow)?

If both answers are yes, reuse the real thing. If either is no, leave it stubbed — and don't feel obligated to "fix" it.

## The gotcha: relative imports bypass barrel-level fixes

If you fix a stub by re-exporting the real component from your shared UI barrel/index file, that only helps callers that import through the barrel. If the *real* component's own source internally imports a sibling component via a **relative path** (`import Link from "./Link"`) rather than through the same barrel, that internal import bypasses whatever fix you made at the barrel level entirely.

Concretely: a `Navigate`-style "view details" link component composes an internal `Link` via a relative import. Naively re-exporting the real `Navigate` from the barrel (assuming "it's just a presentational wrapper, should be safe") silently reintroduces whatever bug you already fixed in your barrel's `Link` — because `Navigate`'s internal `Link` never goes through your barrel at all.

**Fix:** don't blindly re-export a "safe-looking" composite component. Check what it imports internally. If it composes another component via a relative path, either (a) mirror its exact rendered output (same icon, same text, same structure) but wire it to *your* already-fixed version of the sub-component, or (b) patch the barrel resolution (e.g. a build-time alias) so the relative import also resolves to your fixed version.

This is worth checking for every "just re-export it" candidate, not only ones that look link-related — any composite component can have this shape.
