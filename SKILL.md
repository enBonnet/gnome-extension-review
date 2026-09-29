---
name: gnome-extension-review
description: >
  Review, audit, and release GNOME Shell extensions (GJS) against the official
  extensions.gnome.org (EGO) review guidelines and gjs.guide best practices.
  Use whenever the user asks to review, audit, check, rate, or find problems in
  a GNOME extension, shell extension, extension.js, prefs.js, or metadata.json;
  when preparing an extension for extensions.gnome.org submission or EGO
  review; when hunting memory leaks, missing disable() cleanup, forbidden
  imports (Gtk in shell, St in prefs), orphaned signals/timeouts, or lifecycle
  violations in GJS code; when asked to fix or refactor GNOME extension code,
  even if the word "review" is not used; and when asked to release, publish,
  package, zip, submit, or upload an extension to extensions.gnome.org
  (packaging, translations, versioning, EGO submission flow).
version: 1.2.0
---

# GNOME Shell Extension Review & Release

Audit existing extension code against the official EGO review guidelines, and
take passing extensions through packaging, testing, and EGO submission.
Scope: GNOME Shell **45+** (ESModules, `Extension` class). Pre-45 code: note it
is out of scope and point to the legacy docs; do not review it as-is.

Authoritative sources (all bundled in `references/`, plus live-verification
commands):

- `references/review-guidelines.md` — EGO review rules (R1–R35, severity-tagged)
- `references/best-practices.md` — official anti-pattern benchmark for AI code
- `references/lifecycle-audit.md` — enable/disable symmetry audit method
- `references/shell-ui-map.md` — js/ui module map + live API verification
- `references/metadata-and-schemas.md` — metadata.json / gschema / packaging
- `references/release-guide.md` — pack, test, translations, EGO upload,
  post-submission handling, release versioning

## Severity model

- **🔴 Blocking** — violates a MUST/MUST NOT rule → EGO rejection risk
- **🟡 Should fix** — SHOULD rules, case-by-case risks, anti-patterns
- **🟢 Nice-to-have** — recommendations (lint, HIG, file hygiene)

## Workflow

### 0. Identify the code and target version

Ask (or infer from code/metadata) which GNOME Shell version(s) the code
targets. Everything below assumes 45+; if `metadata.json` lists older versions
or the code uses `init()`/`imports.*`, flag the legacy pattern first and confirm
before reviewing against 45+ rules.

### 1. Discover the extension layout

Locate `extension.js`, `prefs.js`, `metadata.json`, `schemas/`, and JS modules
(recursive; ignore `node_modules/`, `.git/`, `locale/`, build dirs). If no
extension is present at the given path, report that and stop.

### 2. Run the static scanner (candidate findings)

```sh
python3 <skill-dir>/scripts/static_checks.py <extension-dir>
# add --json for machine-readable output; exit code 1 = blockers found
# point it at a built <bundle>.zip instead to validate release packaging
```

The scanner greps for rule violations: forbidden cross-process imports,
deprecated modules, metadata.json problems, schema problems, connect/disconnect
and timeout creation/removal count mismatches, `run_dispose`, lifecycle flags,
synchronous file/subprocess IO (R28/R29), clipboard usage (R13), privileged
spawns (R14), the 45+ prefs API (R31), `imports._gi`, lookup helpers, manual
stylesheet loading, Soup sessions, unreachable modules, compiled-schema
shipping, unlock-dialog `disable()` comment placement (R18/EGO-M-008),
overlong lines, binaries, AI notice, GNOME 49/50/51 removed APIs
(version-gated), extension-system interference (R8), and more.

Rule ids cross-reference EGO's [Shexli](https://gitlab.gnome.org/Infrastructure/extensions-web)
static analyzer (numeric ids `EGO001`–`EGO037`, display ids like `EGO-X-004`
on review pages) — see the mapping table in
`references/review-guidelines.md` (Appendix).

**These are candidates, not verdicts.** For each finding, read the surrounding
code and confirm or reject it (e.g. a `connect()` may be cleaned up through a
helper class; a count mismatch may have a legitimate explanation). Never report
a finding you have not verified in context. Also inspect what greps cannot see:
constructor-time GObjects, `Main.*` mutations, partial `disable()` cleanup.

### 3. Lifecycle audit (the core)

Follow `references/lifecycle-audit.md` §2: inventory every resource created in
imports/`constructor()`/`enable()` (signals, sources, widgets, Shell
registrations, injections, settings bindings, D-Bus) and match each one to a
release in `disable()`/`destroy()`. Check:

- constructor is static-only (R1)
- `enable()`/`disable()` are adjacent and symmetric
- destroy order: sources → signals → refs → `super.destroy()` last
- timeout removal immediately before re-creation
- per-class cleanup ownership (no spaghetti cleanup)
- no `_destroyed`/`_enabled` flags, no defensive try-catch/`?.()` around
  guaranteed cleanup calls (best practices)
- `unlock-dialog` checklist if session-modes includes it

### 4. Verify uncertain APIs against live docs

Never trust memory for version-sensitive APIs. Use the commands in
`references/shell-ui-map.md`:

- gnome-shell raw source (pin with `ref=<tag>` for the target version) to
  confirm a `js/ui` API/property exists and how it is used upstream
- gjs-docs JSON API (`docs/<slug>~<version>/index.json` + `db.json`) for
  GLib/Gio/St/... signatures
- the gjs.guide porting guide for the target version when code claims support
  across versions

### 5. Check metadata, schemas, packaging, legal

Per `references/metadata-and-schemas.md`: uuid, shell-version plausibility
(stable releases + at most one dev release), session-modes, donations keys,
settings-schema convention, gschema id/path/filename, no compiled
`gschemas.compiled` for 45+ targets (R25/EGO-P-006), unnecessary files,
binaries, licensing (GPL-compatible), attribution, CoC/political/trademark
content. If `unlock-dialog` is declared, the comment explaining it **MUST** be
inside the `disable()` body, not above the method (R18/EGO-M-008). If the
extension declares clipboard access, verify it is declared in the
metadata description (R13 checklist). If `shell-version` lists 49, 50 or 51,
run the removed-API checks (C49/C50/C51) from `references/review-guidelines.md`
§7; for 52, verify APIs against live sources (no rule set curated yet).

If the code carries the AI-generation notice ("Generated with AI..."), keep it
when handing the code back, and remind the author it must be removed before EGO
upload (R16 / best practices).

## Release workflow (release / publish / pack / submit triggers)

When the user asks to release, publish, package, zip, or submit the extension
to EGO, run the review workflow (steps 0–5) first and resolve 🔴 blockers.
Then follow `references/release-guide.md` end to end:

1. **Version checklist** — `version-name` bumped, no `version` field,
   `shell-version` only tested stable releases + at most one dev release.
2. **Pack** — `gnome-extensions pack` (or the manual zip fallback); files at
   the zip root; no `gschemas.compiled`, `.po`/`.pot`, build scripts, or
   binaries in the bundle (R12/R25).
3. **Validate** — run the scanner on the built zip
   (`static_checks.py <bundle>.zip`) and list its contents.
4. **Smoke test** — local `gnome-extensions install --force`, enable/disable
   cycles, lock/unlock, prefs open/close, clean `journalctl` (release-guide §4).
5. **Upload** — EGO submission is **web-only, no public API**: hand the zip to
   the user with the upload steps; do not claim to have uploaded anything.
6. **After submission** — fix Shexli auto-check warnings before human review,
   respond on the review thread, and note that re-uploads supersede a pending
   review (reviewers only look at the newest upload).

Finish a release interaction with the release checklist from
`release-guide.md` §9 so the user sees exactly what was done and what
remains manual.

## Report format

Always finish with a report using exactly this structure:

```markdown
# Extension Review: <name or uuid>

Target: GNOME Shell <version(s)> · Files reviewed: <n>

## Verdict
🟡 Not ready for EGO submission / 🔴 Would be rejected / 🟢 Ready
One-paragraph summary.

## 🔴 Blocking
- **<Rule id> — <short title>** `path/file.js:LINE`
  What was found (verified in context).
  ```js
  // fix snippet or corrected code
  ```

## 🟡 Should fix
(same format)

## 🟢 Nice-to-have
(same format)

## Verified clean
Areas explicitly checked and found correct (lifecycle table summary,
metadata, imports) — so the user knows the coverage.
```

Keep every finding tied to a rule id from `references/review-guidelines.md`
(R1–R35, C49-*/C50-*/C51-*) or an explicit "best-practices" tag. Include
file:line for each. If something is suspected but could not be verified (e.g.
runtime-only leak), list it under Should fix with the uncertainty stated.

## Handling fixes requested by the user

When the user asks to fix findings: apply minimal changes, keep the existing
code style, re-run the scanner after fixing, and re-verify the touched cleanup
paths. Do not restructure unrelated code.
