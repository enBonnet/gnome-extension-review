# GNOME Extension Review — Agent Skill

An AI agent skill that reviews, audits, and **releases** GNOME Shell extensions
written in GJS against the official
[extensions.gnome.org](https://extensions.gnome.org) (EGO) review guidelines
and [gjs.guide](https://gjs.guide) best practices.

It bundles the authoritative EGO review rules (R1–R35, plus GNOME 49/50/51
removed-API checks), a lifecycle-audit method, a `js/ui` API map, a full
release workflow (packaging, translations, EGO submission and versioning), and
a dependency-free Python static scanner — so any compatible agent can find
rejection risks (memory leaks, forbidden imports, missing `disable()` cleanup,
blocking IO, metadata/schema problems) before you submit, then package and
validate the bundle for upload. Rule ids are cross-referenced with
[Shexli](https://gitlab.gnome.org/Infrastructure/extensions-web), the static
analyzer EGO runs on uploads.

**Scope:** GNOME Shell **45+** (ESModules, `Extension` class): code review and
the release path (pack → validate → test → upload → review handling). Pre-45
code is flagged as legacy, not reviewed.

## Requirements

- An agent harness that supports skills (Claude Code or OpenCode)
- [Python 3](https://www.python.org/) — for the static scanner (stdlib only,
  no pip packages)

## Install

### Claude Code

Clone the skill into your personal skills directory:

```sh
git clone https://github.com/enBonnet/gnome-extension-review.git \
  ~/.claude/skills/gnome-extension-review
```

### OpenCode

Clone it globally…

```sh
git clone https://github.com/enBonnet/gnome-extension-review.git \
  ~/.config/opencode/skill/gnome-extension-review
```

…or into a single project:

```sh
git clone https://github.com/enBonnet/gnome-extension-review.git \
  .opencode/skill/gnome-extension-review
```

### Update

```sh
git -C ~/.claude/skills/gnome-extension-review pull
```

(Adjust the path if you installed it elsewhere.)

### Uninstall

```sh
rm -rf ~/.claude/skills/gnome-extension-review
```

## Usage

Once installed, just ask your agent in natural language:

> Review the GNOME extension in ~/projects/my-extension against the EGO guidelines

> Release my extension: package it, validate the zip, and walk me through uploading it

The skill triggers on words like *review*, *audit*, *check*, or *rate* for
GNOME Shell extensions, when asking to *fix* extension code, and on
*release/publish/package/zip/submit/upload* for the release workflow.

### Standalone scanner

The static scanner works without an agent and has no dependencies:

```sh
python3 scripts/static_checks.py /path/to/your-extension
python3 scripts/static_checks.py my-extension@example.github.io.zip
```

- `--json` — machine-readable output
- Directory mode scans the source tree; **zip mode** validates a built bundle
  (required files at the zip root, no compiled schemas, no `.po`/`.pot`, no
  binaries — see `references/release-guide.md`)
- Exit code `0` = no blocking candidates, `1` = blocking candidates found,
  `2` = usage/IO error

> Findings are **candidates, not verdicts** — verify each one in context before
> acting on it.

## What it checks

| Area | Examples |
|---|---|
| Lifecycle | constructor hygiene, `enable()`/`disable()` symmetry, leak-free cleanup order |
| Signals & sources | connect/disconnect balance, timeout creation/removal, orphaned sources |
| Forbidden imports | `Gtk`/`Gdk`/`Adw` in the shell process, `St`/`Clutter` in prefs (R6/R7) |
| Blocking IO | sync file/subprocess APIs in the shell process (R28/R29), Soup sessions left un-aborted (R35) |
| Clipboard | `St.Clipboard` use → declaration + scrutiny checklist (R13) |
| Legacy patterns | `imports.*`, `Lang.bind`, `Mainloop`, `imports._gi`, lookup helpers |
| Prefs API | `fillPreferencesWindow` vs `getPreferencesWidget` (R31), `close-request` cleanup (R34) |
| Version compat | GNOME 49/50 removed APIs, gated on `shell-version` (C49/C50) |
| Packaging & legal | binaries in the zip, compiled schemas (R25/EGO-P-006), unreachable modules, license, AI notice |
| Metadata & schemas | uuid, `shell-version`, gschema id/path/filename, session-modes, unlock-dialog `disable()` comment placement (R18/EGO-M-008) |
| Version compat | GNOME 49/50/51 removed APIs, gated on `shell-version` (C49/C50/C51) |
| Extension system | imports of `extensionSystem`/`extensionDownloader`, `Main.extensionManager` (R8) |
| Release & packaging | zip validation (files at root, no `gschemas.compiled`/`.po`/binaries), `version-name` format, pack guidance |

Findings are severity-tagged:

- 🔴 **Blocking** — would likely be rejected by EGO
- 🟡 **Should fix** — SHOULD rules and anti-patterns
- 🟢 **Nice-to-have** — recommendations

## Repository layout

```
├── SKILL.md                     # Skill definition and review/release workflows
├── references/
│   ├── review-guidelines.md     # EGO review rules (R1–R35, C49/C50/C51) + Shexli cross-ref
│   ├── best-practices.md        # gjs.guide anti-pattern benchmark
│   ├── lifecycle-audit.md       # enable/disable symmetry method
│   ├── shell-ui-map.md          # js/ui module map + live API verification
│   ├── metadata-and-schemas.md  # metadata.json / gschema / packaging
│   └── release-guide.md         # pack, test, translations, EGO upload, versioning
└── scripts/
    └── static_checks.py         # Candidate-findings scanner (+ zip validation)
```

## License

[MIT](LICENSE) © Ender Bonnet
