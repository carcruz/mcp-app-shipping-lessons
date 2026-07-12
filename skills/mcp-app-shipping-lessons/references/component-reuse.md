# Reuse vs. stub: a decision framework for widget UI components

When a widget bundle is built by extracting components out of a larger app (rather than writing a widget from scratch), the extraction step usually leaves behind hand-written stand-in "stub" versions of shared components — a fake `Link`, a hardcoded theme object, a simplified icon component — because the real ones assume things a standalone bundle doesn't have (routing context, a full design-system provider, dev-only dependencies).

Stubs drift in two ways: the component itself goes stale (wrong colors, dropped props like `className`/`onClick`/`ariaLabel`, missing icon variants), and the *justification* for stubbing it goes stale too — a comment saying "stubbed because X doesn't exist yet" can outlive X actually being added to the codebase. Before trusting an old stub's stated reason, check whether the blocking dependency still doesn't exist. It's not obvious from a glance which stubs are "fine to leave" and which are actively wrong either way.

## Two categories

**1. Broken duplicates of something that works fine standalone.** Theme/color tokens, a generic `Link`, a presentational component like a star-rating or badge, a hover-preview card that just fires its own query — these usually have no real dependency on the "full app" context. They're stubbed only because nobody's revisited them since the original extraction. Fix: **reuse the real component/module directly**, adding a minimal adapter only for what's genuinely contextually different (e.g. navigation semantics — see `host-integration.md` for the specific link-opening adapter).

**2. Structural stubs that exist because the real thing depends on something genuinely absent** in a standalone widget — a heavy dev-only dependency (a GraphQL playground UI, say), a missing upstream service, or a UX that only makes sense with a full page shell (a persistent download-manager drawer, a page-level tooltip provider, shared cross-component viewer state that only a full provider tree sets up). These should **stay stubbed** — this isn't unfinished work, it's a legitimate design choice given the widget's constraints.

## How to tell which is which

Before touching any stub, ask two questions:
1. Are the real component's dependencies actually resolvable in a standalone bundle (no giant dev-only packages, no missing service)?
2. Does its behavior still make sense without the full app shell (real routes, real hover/preview UX, a real download flow, shared cross-component state)?

If both answers are yes, reuse the real thing. If either is no, leave it stubbed — and don't feel obligated to "fix" it. Category 2 also covers "similar-looking but architecturally different" components: two features that both render a 3D viewer, say, aren't necessarily the same amount of work to un-stub — one might be self-contained (easy, category 1) while the other depends on a shared state-management system that only a full provider tree populates (category 2, stays special-cased). Check what each one *actually* imports before assuming they're interchangeable.

## The gotcha: imports can bypass barrel-level fixes in more than one way

If you fix a stub by re-exporting the real component from your shared UI barrel/index file, that only helps callers that import through the barrel. There are at least three ways a real component's own source can route around that fix entirely:

1. **A sibling relative import.** The real component you're re-exporting internally imports *another* real component via a relative path (`import Link from "./Link"`) instead of through the barrel. Re-exporting the outer component naively (assuming "it's just a presentational wrapper, should be safe") silently reintroduces whatever bug you already fixed in the barrel's version of the inner one — because the inner import never goes through your barrel at all.
2. **A relative import into the barrel's own file.** A real component imports something via a relative path that happens to resolve to your own package's barrel/index file itself (e.g. `import { Foo } from "../.."` from two directories under the barrel). If your build tooling intercepts the *resolved barrel file path* (common when stubbing via a bundler plugin — see below), this import lands on your stub too, and fails if the stub doesn't export `Foo`.
3. **Multiple distinct real components all reaching the same third dependency.** Two unrelated components both import the same shared helper by relative path; fixing the barrel export doesn't touch either call site.

**The fix that generalizes across all three:** don't chase down every "just re-export it" candidate one at a time. Instead, intercept the *exact resolved module file path* at the bundler level (e.g. a Vite/webpack plugin `load`/resolve hook keyed on the real component's absolute path) and substitute your fixed implementation there. This catches every import style — bare-specifier through the barrel, a relative sibling import, or a relative import that happens to land back on the barrel file — because they all resolve to the same underlying file id. It's more work up front than "mirror the rendered output" but it's the only approach that doesn't require re-auditing every future caller by hand.

If bundler-level interception isn't available or worth the setup for a one-off case, the narrower fallback still works: mirror the composite component's exact rendered output (same icon, same text, same structure) but wire it to *your* already-fixed version of the sub-component, rather than re-exporting the real composite wholesale.

This is worth checking for every "just re-export it" candidate, not only ones that look link-related — any composite component, provider, or context hook can have this shape.

## Designing what a stub *returns*, not just what it renders

For stubbed hooks (not components), don't stop at "doesn't crash" — trace how the *return value* is actually consumed downstream. A hook that fed real ancestor-provided data in the full app, stubbed to return `null` or an empty object in the widget, might satisfy every destructure without throwing, but still silently break the feature if a caller uses the shape as a gate (e.g. `if (!hasData) return null` based on a field inside it). An empty/neutral default there doesn't produce a visible error — it produces a widget that renders nothing, for every input, forever, which is worse than a crash because nobody notices until someone reports "this feature just doesn't work." Read every call site's control flow, not just its destructuring, before deciding what a stub should return.

## Verifying the fix actually landed

A clean build (or a passing typecheck) only proves the code compiles — it doesn't prove the right module was intercepted or that the fix is actually reachable at runtime. After changing a stub or adding a bundler-level interception, grep the *built output* for a fingerprint of the old behavior (an error string the old code threw, a DOM attribute the old code set, an identifier that should no longer appear) and confirm it's gone. This catches the case where your fix compiles fine but silently didn't apply — e.g. because a *different* file (a second stub file, a per-widget build variant) needed the same fix and was missed.
