# Koqeto customizations over StacDev/flow

This repository is the **fork of [StacDev/flow](https://github.com/StacDev/flow)
that Koqeto consumes as a git dependency**. The goal: keep Koqeto's own
customizations isolated on `koqeto-custom`, preserve upstream history
verbatim, and keep future upstream updates mergeable without losing them.

## Remote and branch layout

| Remote / branch | Role |
|---|---|
| `upstream` | https://github.com/StacDev/flow — the original repository |
| `origin` | https://github.com/jadl456/flow — this fork |
| `main` | Mirror of upstream/main. **Never commit Koqeto changes here**; fast-forward it from upstream only |
| `koqeto-custom` | Koqeto's customizations, branched from the upstream tag the current patch set is based on. **All Koqeto commits land here** |
| `v<N>-koqeto.<n>` tags | One stable, immutable ref per consumable revision. Koqeto's pubspec pins the tag; `pubspec.lock` pins the resolved SHA |

## Patch ledger

Every Koqeto commit on top of its upstream base is listed here, so an
upstream merge can be audited hunk by hunk.

### Base: upstream tag `v0.3.0` (commit `611197a`)

> Note: at `v0.3.0` the repo was single-package — `flow_ui` lives at the
> repository ROOT. Upstream restructured into a pub workspace
> (`packages/flow_ui`) on `main` afterwards. While `koqeto-custom` is
> based on `v0.3.0`, consumers point at the repo root; after the first
> merge of upstream `main`, consumers must add `path: packages/flow_ui`.

| Commit | File | What and why |
|---|---|---|
| `35b90f1` | `lib/src/widgets/flow_composer.dart` | Forward `contextMenuBuilder` (9 added lines, 3 hunks) from `FlowComposer` to the `material_ui.TextField` it already builds — the parameter existed there but was never exposed. Needed so the host can build the composer's selection toolbar itself: the SDK toolbar's Paste calls `EditableTextState._pasteTextWithReporting` directly and cannot be intercepted from outside, which blocked clipboard-image paste in the Koqeto chat. Null keeps upstream behavior identical. |

## Upstream sync procedure

```bash
cd <flow clone>

# 1. Bring in upstream
git fetch upstream --tags
git checkout main
git merge --ff-only upstream/main
git push origin main

# 2. Integrate into koqeto-custom
git checkout koqeto-custom
git merge upstream/main
#    Conflicts, if any, live in the files of the patch ledger above —
#    resolve by re-applying the intent (the ledger's "what and why"),
#    not by discarding either side.

# 3. Validate before publishing
flutter analyze
flutter test

# 4. Publish the revision
git push origin koqeto-custom
git tag v<upstream-version>-koqeto.<n>   # bump <n>, or bump the base on real upgrades
git push origin v<upstream-version>-koqeto.<n>
```

Then, in the Koqeto repository:

```bash
# Point the dependency at the new tag, then re-resolve so the lock
# records the new resolved-ref/SHA.
#   dependencies: flow_ui: git: { url, ref: v<upstream-version>-koqeto.<n> }
# (after the first upstream-main merge also add path: packages/flow_ui)
flutter pub upgrade flow_ui
flutter analyze && flutter test
```

## Rules that keep this maintainable

1. **One concern per commit** on `koqeto-custom`, documented in this ledger.
2. **Prefer pass-through parameters over behavior changes** — minimal
   divergence means minimal merge conflicts.
3. **Never rewrite published history**: tags are immutable refs that
   Koqeto's builds depend on; use merge (not rebase) to integrate upstream.
4. **Base upgrades are deliberate**: moving from `v0.3.0` to a newer
   upstream base follows Koqeto's own dependency-upgrade review
   (flow_ui releases carry breaking changes; Koqeto also pins
   `material_ui` exact).
