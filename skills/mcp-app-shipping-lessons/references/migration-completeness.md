# Auditing whether "all the widgets" really are all the widgets

## The trap

When a widget server exposes a subset of a larger app's features one at a time (one MCP tool per section/page/panel of the source app), it's common to keep a hand-maintained list of what's been migrated and what's deliberately excluded, along with a reason for each exclusion ("needs a two-step query," "uses a component that doesn't exist yet," "covered by this other widget"). That list is a claim about the source app *at the time it was written* — both the source app and the migration's own capabilities move on, and the list doesn't update itself.

Two ways this goes wrong:

1. **A stated exclusion reason stops being true.** A feature was skipped because it needed some capability the migration pipeline didn't support yet; months later that capability gets added for a different feature, but nobody revisits the original "excluded" entry to see if it now qualifies.
2. **Two similarly-named or similar-looking features get conflated.** A migrated widget and an unmigrated one both render "the same kind of thing" (e.g. both show a 3D structure viewer, both show a bibliography list) at a glance, so the unmigrated one gets marked "already covered by the other one" — without checking that they're actually the same underlying component, data, and code path. If they're structurally distinct features that just look similar, this silently drops real functionality from the migration while the tracking doc claims full coverage.

Both failure modes produce the same symptom: the exclusion list *looks* authoritative and complete, but isn't, and nothing about reading it tells you which entries are stale.

## The fix

Don't audit a migration's completeness by reading the exclusion list — audit it against the actual source of truth. Concretely:

1. Enumerate every real feature/section/page in the source app (a directory listing, a config file, whatever the source app's own registration mechanism is).
2. Enumerate every feature the migration has actually wired up (the migration's own registry/config, not its documentation).
3. Diff the two lists programmatically. Anything in (1) not in (2) is either a genuine, currently-valid exclusion, or a gap — and you don't know which until you check.

For each gap the diff turns up, re-derive the reason from the current code, not from an old comment: read the actual component, check whether it depends on something the migration pipeline still doesn't support, and check whether it's still live/enabled in the *source* app itself (a feature can be fully coded in the source app's codebase while being disabled/commented-out at the point where the source app actually renders it — check the source app's own live routing/rendering config, not just whether the code exists on disk).

When two features look similar, verify they're the same component before accepting "already covered": check whether they share the same source file, the same data-fetching logic, and the same external dependencies. Two components that both use a 3D-viewer library, for instance, might do so through completely different internal state-management systems — one wired into a shared cross-component provider, the other fully self-contained — which changes both how much work migrating it is and whether an existing migrated widget genuinely already covers it.

## Aftermath

After finding and fixing a stale exclusion, don't just fix that one entry — leave a note for the next audit to start with the programmatic diff again rather than trusting the (now presumably-fixed, but still hand-maintained) list at face value. The failure mode is structural (a document going stale relative to two moving codebases), not a one-time mistake, so the next drift is just as likely as the last one.
