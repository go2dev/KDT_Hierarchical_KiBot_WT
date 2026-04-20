# KDT KiBot Automation — Minimum Viable Intervention Plan

> **Scope**: A CLI/TUI tool (`kdt`) for config validation, editing, and KiCad
> file synchronisation, plus release-please integration that writes changelogs
> in the exact format `get_changelog.py` already expects.
>
> **Non-goals**: KiCad plugins, GUI applications, changes to KiBot itself.

---

## Table of Contents

1. [Current Pain Points (Summary)](#1-current-pain-points-summary)
2. [Deliverable 1 — `kdt` CLI / TUI Tool](#2-deliverable-1--kdt-cli--tui-tool)
3. [Deliverable 2 — Release-Please Integration](#3-deliverable-2--release-please-integration)
4. [Deliverable 3 — CI Workflow Refactor](#4-deliverable-3--ci-workflow-refactor)
5. [Changelog Format Compatibility — Critical Detail](#5-changelog-format-compatibility--critical-detail)
6. [File-by-File Change Map](#6-file-by-file-change-map)
7. [Open Questions & Decisions](#7-open-questions--decisions)
8. [Phasing & Dependencies](#8-phasing--dependencies)

---

## 1. Current Pain Points (Summary)

Ranked by frequency × impact. These are the problems we are solving.

| # | Pain Point | Where | Frequency | Failure Mode |
|---|-----------|-------|-----------|-------------|
| P1 | **5-way version sync** — CHANGELOG, kibot YAML, schematic text vars, git tag, and CI variant must all agree on the version string | Across 5 files | Every release | Inconsistent documents, blank revision history |
| P2 | **CHANGELOG.md is fully manual** — user writes entries by hand, get_changelog.py parses them into schematic text variables | CHANGELOG.md | Every release | Wrong/missing revision history in output PDFs |
| P3 | **Changelog YAML entries** — user must uncomment and add version-specific variable blocks in kibot_pre_set_text_variables.yaml per release | kibot_pre_set_text_variables.yaml | Every release | Blank changelog on schematic pages |
| P4 | **CI variant must be hand-edited** — DRAFT → PRELIMINARY → CHECKED in ci.yaml env block | .github/workflows/ci.yaml | 3-4× per project | Skipped DRC/ERC, incomplete outputs |
| P5 | **30+ config fields on init** — project metadata, BOM field names, layer names, theme, paths all need manual editing | kibot_main.yaml definitions block | Once per project | Wrong names on all documents, empty BOM columns, missing PDF pages |
| P6 | **BOM field name mismatch** — MPN_FIELD/MAN_FIELD must exactly match schematic custom field names; silent failure if wrong | kibot_main.yaml | Once per project | Empty BOM columns with zero warnings |
| P7 | **PCB layer name mismatch** — 10 layer name definitions must match user-defined layers in the PCB; silent failure if wrong | kibot_main.yaml | Once per project | Missing fabrication/assembly PDF pages |
| P8 | **Impedance table / fab notes** — entirely manual, project-specific rewrite required | kibot_resources/templates/*.txt | Once per project | Wrong manufacturing specifications |

---

## 2. Deliverable 1 — `kdt` CLI / TUI Tool

### 2.1 Purpose

A standalone Python CLI tool (no KiCad dependency at runtime) that:
- Reads/writes a **single source-of-truth** project config file
- **Introspects KiCad files** (`.kicad_sch`, `.kicad_pcb`, `.kicad_pro`) by parsing their text formats directly (S-expression for KiCad 8+, no pcbnew API needed)
- **Generates** the scattered YAML definitions from that single config
- **Validates** consistency across all config files before build
- **Syncs** changelog versions into kibot_pre_set_text_variables.yaml

### 2.2 The Single Source of Truth: `.kdt-project.yaml`

This file replaces the need to manually edit kibot_main.yaml definitions, ci.yaml
env vars, and kibot_pre_set_text_variables.yaml changelog entries.

```yaml
# .kdt-project.yaml — canonical project config
# All other configs are generated/validated from this file.

project:
  name: ""          # → kibot_main.yaml → PROJECT_NAME
  board_name: ""    # → kibot_main.yaml → BOARD_NAME
  company: ""       # → kibot_main.yaml → COMPANY
  designer: ""      # → kibot_main.yaml → DESIGNER
  logo: ""          # → kibot_main.yaml → LOGO
  git_url: ""       # → kibot_main.yaml → GIT_URL

kicad:
  version: 9        # → ci.yaml → kicad_version
  color_theme: ""   # → kibot_main.yaml → COLOR_THEME
  worksheet: ""     # → kibot_main.yaml → SHEET_WKS

bom:
  mpn_field: ""     # → kibot_main.yaml → MPN_FIELD
  man_field: ""     # → kibot_main.yaml → MAN_FIELD

fabrication:
  check_zone_fills: false    # → CHECK_ZONE_FILLS
  stackup_note: ""           # → STACKUP_TABLE_NOTE
  scaling: 1                 # → FAB_SCALING
  plot_refs: true            # → PLOT_REFS
  group_round_slots: true    # → GROUP_ROUND_SLOTS
  group_pth_npth: "no"       # → GROUP_PTH_NPTH
  group_pth_npth_drl: false  # → GROUP_PTH_NPTH_DRL

assembly:
  scaling: 1                 # → ASSEMBLY_SCALING
  exclude_refs: "[MB*]"      # → EXCLUDE_REFS

rendering:
  rotation: [2, -1, 1]      # → 3D_VIEWER_ROT_X/Y/Z
  zoom: -1                   # → 3D_VIEWER_ZOOM
  key_color: "#00FF00"       # → KEY_COLOR

layers:
  # These are auto-detected from .kicad_pcb by `kdt init`
  title_page: "TitlePage"
  drill_map: "DrillMap"
  tp_list_top: "F.TestPointList"
  tp_list_bottom: "B.TestPointList"
  assembly_text_top: "F.AssemblyText"
  assembly_text_bottom: "B.AssemblyText"
  dnp_top: "F.DNP"
  dnp_bottom: "B.DNP"

workflow:
  variant: DRAFT   # → ci.yaml → kibot_variant (the ONLY place variant lives)
```

Every field has a comment showing exactly which downstream value it maps to.
The tool never edits kibot_main.yaml's *structure* — only the `definitions:` block
at the bottom, which is cleanly separated from the rest of the config.

### 2.3 Commands

#### `kdt init`

**Purpose**: Interactive project setup. Run once when starting from the template.

**Behaviour**:

1. **Find KiCad files** — glob for `*.kicad_pro` in the working directory. If
   multiple found, ask the user to pick one. Derive `.kicad_sch` and
   `.kicad_pcb` paths from the project file.

2. **Parse `.kicad_pro`** (JSON in KiCad 8+) — extract:
   - KiCad version from `meta.version` field
   - Text variables already defined in the project (these are candidates for
     cross-referencing)

3. **Parse `.kicad_sch`** (S-expression) — extract:
   - All unique custom field names across every symbol instance (property
     entries beyond the standard Value/Reference/Footprint/Datasheet set)
   - Present these as a selectable list for MPN_FIELD and MAN_FIELD with
     fuzzy matching (auto-suggest "MPN", "Manufacturer Part Number", etc.)
   - Count total symbols and sheets (for validation later)

4. **Parse `.kicad_pcb`** (S-expression) — extract:
   - User-defined layer names from the `(layers ...)` section (User.1 through
     User.9, renamed layers)
   - Present these for mapping to the 8 expected layer roles (title_page,
     drill_map, tp_list_top, etc.)
   - If the template's expected names (TitlePage, DrillMap, etc.) are found
     verbatim, auto-map them
   - **Board dimensions** from the Edge.Cuts layer bounding box (for rendering
     defaults)

5. **Prompt for remaining fields** — project name, board name, company,
   designer, logo path, git_url. Offer to auto-detect git_url from
   `git remote get-url origin`.

6. **Detect KiCad version** — from the `.kicad_pro` meta section, or fall back
   to parsing the file format version header. Map to `8` or `9`.

7. **Write `.kdt-project.yaml`** with all collected values.

8. **Run `kdt generate`** (see below) to propagate values downstream.

**KiCad file parsing notes**:
- `.kicad_pro` is JSON — trivial to parse with stdlib `json`.
- `.kicad_sch` and `.kicad_pcb` are S-expression format. Write a minimal
  S-expression tokeniser (parenthesis-balanced, string-aware) rather than
  importing a full parser. We only need to extract specific top-level forms:
  - From `.kicad_sch`: `(symbol ... (property "FieldName" "Value" ...))` —
    collect the set of all property keys
  - From `.kicad_pcb`: `(layers (N "LayerName" user) ...)` — collect user
    layer names
- This avoids any dependency on KiCad's Python API and works on any machine
  regardless of KiCad installation.

#### `kdt generate`

**Purpose**: Propagate `.kdt-project.yaml` values into the downstream config
files. Idempotent — safe to run repeatedly.

**Behaviour**:

1. Read `.kdt-project.yaml`.
2. **Update `kibot_main.yaml` definitions block** — locate the `definitions:`
   key at the end of the file, replace values for all mapped fields. Preserve
   comments and formatting in the rest of the file. Only touch lines that
   correspond to known `.kdt-project.yaml` keys.
3. **Update `.github/workflows/ci.yaml` env block** — update `kibot_variant`
   and `kicad_version` values from the `workflow:` and `kicad:` sections.
4. **Update `kibot_pre_set_text_variables.yaml` changelog entries** — see
   section 2.5 below.

The update strategy for YAML files should be **line-based find-and-replace**
(like `file_edit`), not a full YAML parse-and-dump, to preserve comments,
ordering, and formatting.

#### `kdt validate`

**Purpose**: Pre-build consistency check. Can run locally and in CI.

**Checks** (each produces a PASS/WARN/FAIL with a clear message):

| Check | Level | What it verifies |
|-------|-------|-----------------|
| Placeholder detection | FAIL | No field in `.kdt-project.yaml` is still empty or contains "Project Name" / "Board Name" / "Company Name" defaults |
| BOM field existence | FAIL | `bom.mpn_field` and `bom.man_field` values exist as property names in the `.kicad_sch` file |
| Layer name existence | FAIL | Every layer name in `layers:` exists as a user layer in the `.kicad_pcb` file |
| CHANGELOG format | FAIL | CHANGELOG.md parses successfully — has a valid `## [Unreleased]` or `## [X.Y.Z] - YYYY-MM-DD` heading structure |
| Version consistency | WARN | If latest CHANGELOG version ≠ latest git tag, warn (expected during development; error on release) |
| Changelog YAML sync | WARN | Every versioned section in CHANGELOG.md has a corresponding uncommented entry pair in kibot_pre_set_text_variables.yaml |
| Variant appropriateness | WARN | If variant is DRAFT but `.kicad_pcb` has meaningful content (copper layers, components), suggest promotion |
| KiCad version match | WARN | kicad.version in project config matches the format version in `.kicad_pro` |
| Worksheet existence | WARN | The file at `kicad.worksheet` path exists on disk (relative to project root) |
| Color theme existence | INFO | Check if theme name appears in common KiCad config paths (best-effort, non-blocking) |
| Git URL format | INFO | `project.git_url` looks like a valid GitHub/GitLab URL |

**Exit codes**: 0 = all pass, 1 = any FAIL, 2 = only WARNs (configurable
with `--strict` to treat WARNs as errors).

**CI integration**: add as a step before KiBot runs:
```yaml
- name: Validate KDT config
  run: kdt validate --strict
```

#### `kdt promote <variant>`

**Purpose**: Change the project phase. Shorthand with guardrails.

**Behaviour**:
1. Validate that the target variant is one of: DRAFT, PRELIMINARY, CHECKED.
   (RELEASED is never set manually — it's automatic on tag events.)
2. Run `kdt validate` — if any FAIL checks, refuse to promote.
3. If promoting to CHECKED, additionally verify:
   - CHANGELOG.md has at least one entry under `[Unreleased]` that isn't a
     placeholder
   - ERC/DRC reports don't exist or aren't stale (warn if last report is
     older than the latest `.kicad_sch`/`.kicad_pcb` modification time)
4. Update `workflow.variant` in `.kdt-project.yaml`.
5. Run `kdt generate` to propagate the change to ci.yaml.
6. Stage the changed files and offer to commit with message
   `chore: promote variant to <VARIANT>`.

#### `kdt sync-changelog`

**Purpose**: Read CHANGELOG.md and regenerate the changelog variable entries in
`kibot_pre_set_text_variables.yaml`. This is the command that eliminates P3.

**Behaviour**: See section 2.5.

#### `kdt status`

**Purpose**: Quick summary of project state.

**Output example**:
```
Project:    High-Efficiency Buck Converter (HEBC-001)
Variant:    CHECKED
KiCad:      9
Version:    1.2.0+ (Unreleased)
Last tag:   1.1.0
Changelog:  3 released versions, Unreleased section has 4 entries
Validation: 11/11 checks passed
```

### 2.4 TUI Mode

If `kdt init` is run in an interactive terminal, use a TUI library (e.g.
[textual](https://github.com/Textualize/textual) or
[questionary](https://github.com/tmbo/questionary)) for:
- Selectable lists for BOM fields and layer mappings (with search/filter)
- Colour-coded validation results
- Editable form for project metadata

If non-interactive (CI), fall back to reading `.kdt-project.yaml` directly with
no prompts.

### 2.5 Changelog Sync Logic (Detail)

This is the mechanism that bridges CHANGELOG.md → kibot_pre_set_text_variables.yaml.

**Current state of the problem**:

The template's `kibot_pre_set_text_variables.yaml` has a commented-out block
with placeholder version entries:
```yaml
# - variable: '@RELEASE_TITLE_VAR@1.0.0'
#   command: '@GET_TITLE_CMD@ 1.0.0'
# - variable: '@RELEASE_BODY_VAR@1.0.0'
#   command: '@GET_BODY_CMD@ 1.0.0'
```

For each release, the user must manually uncomment and add new version pairs.
The variable names must match exactly. This is P3.

**Automated approach**:

`kdt sync-changelog` will:

1. Parse CHANGELOG.md to extract all version headings:
   - `## [Unreleased]` → version label `UNRELEASED`
   - `## [1.2.0] - 2025-01-15` → version label `1.2.0`
   - `## [1.1.0] - 2024-11-03` → version label `1.1.0`
   - etc.

2. For each version found, generate two YAML entries:
   ```yaml
   - variable: 'RELEASE_TITLE_1.2.0'
     command: 'python3 kibot_resources/scripts/get_changelog.py -f CHANGELOG.md --title-only --version 1.2.0'
   - variable: 'RELEASE_BODY_1.2.0'
     command: 'python3 kibot_resources/scripts/get_changelog.py -f CHANGELOG.md --extra-spaces --separators 35 --version 1.2.0'
   ```

3. **Replace** the changelog section in kibot_pre_set_text_variables.yaml.
   Use markers to delimit the auto-generated block:
   ```yaml
   # --- BEGIN AUTO-GENERATED CHANGELOG ENTRIES (do not edit) ---
   ...generated entries...
   # --- END AUTO-GENERATED CHANGELOG ENTRIES ---
   ```
   On first run, insert the markers around the existing commented-out block.
   On subsequent runs, replace everything between the markers.

4. Always include the UNRELEASED entry (it's always present in the template).

**Integration with release-please**: When release-please updates CHANGELOG.md
via a Release PR, the CI runs `kdt sync-changelog` as a post-step to keep the
YAML in sync. This completely eliminates manual editing of the text variables
file for changelog purposes.

### 2.6 Technology Choices

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | Python 3.9+ | Already in KiBot's CI Docker images, no extra deps for CI. KDT template already has Python scripts. |
| S-expression parsing | Custom minimal tokeniser | ~100 lines. Avoids sexpdata or other deps. Only needs to extract top-level forms, not round-trip. |
| YAML editing | Line-based string manipulation | Preserve comments and formatting. Full YAML parse→dump destroys comments. Use regex to find and replace specific `key: value` lines. |
| TUI library | questionary (optional dep) | For interactive init. Falls back to basic input() if not installed. |
| Packaging | Single-file script or pip-installable package | Distribute in the template repo under `kibot_resources/scripts/kdt.py` for zero-install use, with optional `pip install kdt-config` for system-wide. |
| Config format | YAML | Matches the rest of the KiBot ecosystem. |

### 2.7 KiCad File Parsing — Specific Fields to Extract

#### From `.kicad_sch` (S-expression)

Target: the set of all unique property names used across symbol instances.

```
; We need to find all (property "NAME" "VALUE" ...) inside (symbol ...) blocks
; Standard properties to ignore: "Reference", "Value", "Footprint", "Datasheet",
;   "Description", "ki_keywords", "ki_description", "ki_fp_filters"
; Everything else is a custom field candidate for BOM field mapping.
```

Walk the S-expression tree looking for `(symbol (lib_id ...) ... (property
"FieldName" ...))` at the sheet-instance level (not inside lib_symbols).
Collect the set of field names. Present sorted by frequency (most common first).

#### From `.kicad_pcb` (S-expression)

Target 1: user-defined layer names.
```
; In the (layers ...) block:
;   (0 "F.Cu" signal)
;   (31 "B.Cu" signal)
;   ...
;   (44 "User.1" user "TitlePage")    ← renamed user layer
;   (45 "User.2" user "DrillMap")     ← renamed user layer
; The 4th element (when present) is the custom name.
```

Target 2: board bounding box (for rendering defaults).
```
; Parse (gr_rect ...) or (gr_line ...) on Edge.Cuts layer,
; or use (setup (aux_axis_origin ...)) as a simpler reference.
; Just need approximate dimensions for zoom/rotation defaults.
```

#### From `.kicad_pro` (JSON)

```json
{
  "meta": { "version": 1, "filename": "project.kicad_pro" },
  "text_variables": { "COMPANY": "Acme", ... }
}
```

Extract `meta.version` for KiCad version detection and `text_variables` for
cross-referencing with existing project settings.

---

## 3. Deliverable 2 — Release-Please Integration

### 3.1 Goal

Replace the current manual workflow:
```
human writes CHANGELOG.md → human creates git tag → CI promotes [Unreleased]
→ CI creates GitHub Release
```

With:
```
developer writes conventional commits → release-please auto-generates
CHANGELOG.md → release-please opens Release PR → human merges PR →
release-please creates tag + GitHub Release → CI generates KiBot outputs
```

### 3.2 Conventional Commit Types for Hardware

Define a hardware-specific commit convention. These types map to the
release-please `changelog-sections` config and to SemVer bump rules.

| Commit Prefix | SemVer Bump | Changelog Section | When to use |
|--------------|-------------|-------------------|-------------|
| `feat:` | minor | Added | New functional capability (new subcircuit, interface, connector) |
| `fix:` | patch | Fixed | Corrects an error (wrong value, bad trace, footprint error) |
| `schematic:` | minor | Changed | Schematic modifications that aren't new features or fixes |
| `layout:` | patch | Changed | PCB layout changes (routing, placement, copper pours) |
| `bom:` | patch | Changed | Component substitutions, alternate sources |
| `mechanical:` | minor | Changed | Physical/mechanical changes (mounting, outline, keepout) |
| `test:` | patch | Changed | DFT changes (testpoints, test pads, debug headers) |
| `docs:` | — (hidden) | *(not shown)* | Documentation only (README, assembly notes) |
| `chore:` | — (hidden) | *(not shown)* | Build/config changes (CI, KiBot YAML, scripts) |
| Any type with `!` suffix | **major** | *(prepended with ⚠️)* | Breaking change (connector pinout, stackup, footprint change affecting physical compatibility) |

Note: release-please allows any commit type via `changelog-sections`. Types
not listed in the config are hidden from the changelog by default.

### 3.3 Release-Please Configuration

#### `release-please-config.json`

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "packages": {
    ".": {
      "release-type": "simple",
      "changelog-path": "CHANGELOG.md",
      "include-v-in-tag": false,
      "bump-minor-pre-major": true,
      "changelog-sections": [
        { "type": "feat",       "section": "Added" },
        { "type": "fix",        "section": "Fixed" },
        { "type": "schematic",  "section": "Changed" },
        { "type": "layout",     "section": "Changed" },
        { "type": "bom",        "section": "Changed" },
        { "type": "mechanical", "section": "Changed" },
        { "type": "test",       "section": "Changed" },
        { "type": "perf",       "section": "Changed" },
        { "type": "refactor",   "section": "Changed" },
        { "type": "docs",       "section": "Documentation", "hidden": true },
        { "type": "chore",      "section": "Miscellaneous", "hidden": true }
      ]
    }
  }
}
```

**Section name choice**: Using Keep a Changelog standard names (Added, Fixed,
Changed, Removed) so that the existing `get_changelog.py` parser works with
minimal modification (see section 5).

#### `.release-please-manifest.json`

```json
{
  ".": "0.0.0"
}
```

This is the version tracking file. Release-please updates it on each release.
Start at 0.0.0 for new projects; it will auto-increment.

### 3.4 Commit Message Enforcement

Add a commit lint step to CI that validates PR commit messages follow the
convention. Two options:

**Option A — GitHub Action (lightweight)**:
Use `amannn/action-semantic-pull-request` to validate PR titles (which become
the squash-merge commit message):

```yaml
# .github/workflows/lint-pr.yaml
name: Lint PR Title
on:
  pull_request:
    types: [opened, edited, synchronize]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: amannn/action-semantic-pull-request@v5
        with:
          types: |
            feat
            fix
            schematic
            layout
            bom
            mechanical
            test
            docs
            chore
          requireScope: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Option B — commitlint (heavier, local + CI)**:
Requires Node.js. Provides local git hook enforcement via husky.

Recommend **Option A** for minimum viable intervention — it requires no local
tooling and catches issues at PR time.

---

## 4. Deliverable 3 — CI Workflow Refactor

### 4.1 Split Into Two Workflows

Currently everything is in a single `ci.yaml`. Split into:

#### `ci.yaml` — Development Builds (unchanged trigger, simplified logic)

```
Triggers: push to main/dev (excluding *.md), workflow_dispatch
Purpose:  Generate outputs at the current variant level
Changes:
  - Read variant from .kdt-project.yaml instead of hardcoded env var
  - Add `kdt validate` pre-step
  - Remove all changelog/tag/release logic (moved to release.yaml)
```

Key change to the env block:
```yaml
env:
  kibot_config: kibot_yaml/kibot_main.yaml
  # Variant is now read from .kdt-project.yaml at runtime
  # kibot_variant: DRAFT  ← REMOVE this hardcoded value
  kicad_version: 9  # Still here for Docker image selection, but validated
```

Add a step early in the job:
```yaml
- name: Read project config
  id: config
  run: |
    variant=$(python3 -c "
    import yaml
    with open('.kdt-project.yaml') as f:
        cfg = yaml.safe_load(f)
    print(cfg['workflow']['variant'])
    ")
    kicad_v=$(python3 -c "
    import yaml
    with open('.kdt-project.yaml') as f:
        cfg = yaml.safe_load(f)
    print(cfg['kicad']['version'])
    ")
    echo "variant=$variant" >> $GITHUB_OUTPUT
    echo "kicad_version=$kicad_v" >> $GITHUB_OUTPUT
```

Then reference `${{ steps.config.outputs.variant }}` throughout.

#### `release.yaml` — Release Management (new file)

```
Triggers: push to main only
Purpose:  Run release-please; on release, generate RELEASED outputs
```

Workflow structure:

```
Job 1: release-please
  → Runs release-please-action
  → Outputs: release_created (bool), version, tag_name

Job 2: generate-release-outputs (needs: release-please, if: release_created)
  → Checkout
  → Run `kdt sync-changelog` to update YAML from new CHANGELOG.md
  → Commit the sync'd YAML back to main
  → Run KiBot with variant=RELEASED and REVISION=version
  → Commit generated outputs to main
  → Upload release assets to the GitHub Release
```

This means:
- **Daily development**: push to main/dev triggers `ci.yaml`, which reads the
  variant from `.kdt-project.yaml` and generates outputs accordingly. No
  release logic runs.
- **Release flow**: developer merges the Release PR (created by release-please).
  This triggers both workflows, but `release.yaml` creates the tag and
  generates the RELEASED variant outputs. `ci.yaml` also runs but produces
  the normal variant outputs (which is fine — they get overwritten by the
  release outputs).

### 4.2 Eliminating the Tag-Push Pattern

The current template uses manual `git tag X.Y.Z && git push --tags` to trigger
releases. With release-please:

- **Tags are created automatically** by release-please after the Release PR is
  merged.
- The CI no longer needs `on: push: tags:` trigger patterns.
- The `release.yaml` workflow uses release-please's output to know a release
  happened, not a tag event.
- This eliminates the race condition in the current CI where the tag push
  triggers a build that tries to commit back to main from a detached HEAD.

### 4.3 Variant Auto-Promotion on Release

Currently the CI has special logic:
```yaml
if [[ "${{ github.ref_type }}" == "tag" ]]; then
  echo "kibot_variant=RELEASED" >> $GITHUB_ENV
fi
```

This is replaced by the `release.yaml` workflow explicitly setting
`variant: RELEASED` when `release_created` is true. The logic is cleaner
because it's a separate job with a clear conditional, not an inline shell
override.

---

## 5. Changelog Format Compatibility — Critical Detail

This is the most important technical constraint in the entire plan.

### 5.1 The Problem

The existing `get_changelog.py` script uses this regex to parse versioned
entries:

```python
version_pattern = re.compile(
    rf"## \[{version}\] - (\d{{4}}-\d{{2}}-\d{{2}})\n"
    r"(.*?)"
    r"(?=## \[|\[Unreleased\]:|\[\d+\.\d+\.\d+\]:|$)",
    re.DOTALL
)
```

This expects **exactly** the Keep a Changelog heading format:
```markdown
## [1.2.0] - 2025-01-15
```

Release-please generates a **different** heading format by default:
```markdown
## [1.2.0](https://github.com/owner/repo/compare/v1.1.0...v1.2.0) (2025-01-15)
```

| Aspect | KDT expects (Keep a Changelog) | release-please generates |
|--------|-------------------------------|------------------------|
| **Version heading** | `## [1.2.0] - 2025-01-15` | `## [1.2.0](https://...) (2025-01-15)` |
| **Date separator** | ` - ` (space-dash-space) | ` ` (space, in parentheses) |
| **Comparison link** | Not present | Inline in heading |
| **Entry format** | `-   Description` (3-space indent) | `* description ([hash](link))` |
| **Section names** | Added, Fixed, Changed, Removed | Configurable via changelog-sections ✓ |
| **Unreleased section** | `## [Unreleased]` (maintained) | Not maintained (only versioned entries) |

**Three incompatibilities** must be resolved:
1. Heading format (link and date placement)
2. Entry format (bullet style and commit links)
3. Unreleased section management

### 5.2 Resolution Strategy: Post-Processor Script

The cleanest approach that avoids modifying `get_changelog.py` (which belongs
to the upstream template) and avoids writing a custom release-please plugin
(which requires TypeScript and maintaining a fork):

**Write a `normalise_changelog.py` script** that runs after release-please
updates CHANGELOG.md but before KiBot reads it. This script converts
release-please format → Keep a Changelog format in-place.

**Transformations needed**:

1. **Heading rewrite**:
   - Input:  `## [1.2.0](https://github.com/owner/repo/compare/v1.1.0...v1.2.0) (2025-01-15)`
   - Output: `## [1.2.0] - 2025-01-15`
   - Regex: `r'## \[(\d+\.\d+\.\d+)\]\([^)]*\) \((\d{4}-\d{2}-\d{2})\)'`
   - Replace: `r'## [\1] - \2'`

2. **Entry rewrite**:
   - Input:  `* correct timing in clock circuit ([abc1234](https://...))`
   - Output: `-   Correct timing in clock circuit`
   - Regex: `r'^\* (.+?)(?:\s*\(\[[a-f0-9]+\]\([^)]*\)\))?$'`
   - Replace: capitalise first letter, strip commit link, change `*` to `-   `

3. **Unreleased section maintenance**:
   - Release-please does not maintain an `[Unreleased]` section.
   - After the post-processor rewrites versioned entries, it should ensure
     an `## [Unreleased]` section exists at the top (below `# Changelog`).
   - If one already exists, leave it. If not, insert an empty one:
     ```markdown
     ## [Unreleased]

     ### Fixed

     ### Added

     ### Changed

     ### Removed
     ```
   - This preserves compatibility with the template's expectations and with
     the `get_changelog_version.py` script which reads the Unreleased heading.

4. **Bottom-of-file link references** (optional, nice-to-have):
   - Keep a Changelog uses reference-style links at the bottom:
     ```markdown
     [Unreleased]: https://github.com/owner/repo/compare/v1.2.0...HEAD
     [1.2.0]: https://github.com/owner/repo/compare/v1.1.0...v1.2.0
     ```
   - Generate these from the git URL in `.kdt-project.yaml`.

**Where this runs in CI**:

In `release.yaml`, after the release-please action and before KiBot:
```yaml
- name: Normalise changelog format
  if: ${{ steps.release.outputs.release_created }}
  run: python3 kibot_resources/scripts/normalise_changelog.py CHANGELOG.md

- name: Sync changelog to KiBot vars
  if: ${{ steps.release.outputs.release_created }}
  run: python3 kibot_resources/scripts/kdt.py sync-changelog
```

### 5.3 Alternative Considered: Modify `get_changelog.py`

Instead of post-processing CHANGELOG.md, we could modify the regex in
`get_changelog.py` to accept both formats:

```python
# Before (strict):
rf"## \[{version}\] - (\d{{4}}-\d{{2}}-\d{{2}})"

# After (flexible):
rf"## \[{version}\](?:\([^)]*\))?\s*[-( ]*(\d{{4}}-\d{{2}}-\d{{2}})"
```

**Pros**: Simpler, one change, no extra script.
**Cons**: Modifies upstream template code. If the user updates the template
from upstream, the change gets overwritten. Also doesn't fix the entry format
(commit links would appear in schematic text).

**Recommendation**: Use the post-processor approach. It's a clean boundary —
release-please writes its native format, the normaliser converts it, and all
downstream tools see the format they expect. If the upstream template ever adds
native release-please support, the normaliser becomes a no-op and can be
removed.

### 5.4 The Unreleased Section Question

With release-please managing the changelog, the semantics of `[Unreleased]`
change:

- **Before**: User manually writes entries under `[Unreleased]`. On tag push,
  CI promotes it to a versioned heading.
- **After**: Release-please auto-generates versioned entries from commits. There
  are no "unreleased" entries in the changelog — they exist only as unparsed
  commits in the git log.

This means the `RELEASE_TITLE_UNRELEASED` and `RELEASE_BODY_UNRELEASED` text
variables will always be empty (or show "Version Unreleased not found.")
**during development builds**.

This is actually acceptable for most workflows — the unreleased changelog on
the schematic is a nice-to-have, not a requirement. During development, the
schematic shows the current tag + "(Unreleased)" via the REVISION variable.

However, if populating the unreleased section is important for the team:

**Option**: Add a CI step in the development build (`ci.yaml`) that generates a
pseudo-unreleased section from conventional commits since the last tag:

```bash
# Generate unreleased changelog preview from git log
git log $(git describe --tags --abbrev=0)..HEAD --pretty=format:"- %s" \
  | grep -E "^- (feat|fix|schematic|layout|bom|mechanical|test):" \
  >> /tmp/unreleased_preview.md
```

Then inject this into a temporary CHANGELOG.md copy before KiBot runs. This
way the development build shows a preview of what the next release changelog
will contain, without modifying the committed CHANGELOG.md.

**Recommendation**: Defer this. Start with the simpler model where Unreleased
is empty during dev builds. Add the preview feature later if the team wants it.

---

## 6. File-by-File Change Map

Summary of every file that needs to be created, modified, or deleted.

### New Files

| File | Purpose |
|------|---------|
| `.kdt-project.yaml` | Single source of truth for project config |
| `release-please-config.json` | Release-please configuration |
| `.release-please-manifest.json` | Release-please version tracker |
| `.github/workflows/release.yaml` | Release management workflow |
| `.github/workflows/lint-pr.yaml` | PR title / commit message linting |
| `kibot_resources/scripts/kdt.py` | CLI tool (or separate package) |
| `kibot_resources/scripts/normalise_changelog.py` | Changelog format converter |
| `.commitlintrc.yaml` *(optional)* | Commit type definitions for local linting |

### Modified Files

| File | What changes | How |
|------|-------------|-----|
| `kibot_yaml/kibot_main.yaml` | `definitions:` block values only | `kdt generate` writes values from `.kdt-project.yaml` |
| `.github/workflows/ci.yaml` | Remove hardcoded env vars for variant/version; add `kdt validate` step; read variant from `.kdt-project.yaml` at runtime; remove all release/tag/changelog logic | Manual refactor |
| `kibot_yaml/kibot_pre_set_text_variables.yaml` | Changelog section replaced with auto-generated entries between markers | `kdt sync-changelog` |
| `CHANGELOG.md` | Managed by release-please + normaliser instead of manual edits | Automated |

### Unchanged Files

| File | Why unchanged |
|------|--------------|
| All `kibot_yaml/kibot_out_*.yaml` | These reference `@DEFINITIONS@` which resolve from kibot_main.yaml — no changes needed |
| `kibot_yaml/kibot_globals.yaml` | Values already flow through from kibot_main.yaml definitions |
| `kibot_resources/templates/*.txt` | Still require manual per-project editing (P8) — out of scope for this intervention |
| `kibot_resources/scripts/get_changelog.py` | Unchanged — the normaliser ensures it sees the format it expects |
| `kibot_resources/scripts/get_changelog_version.py` | Unchanged — [Unreleased] heading preserved by normaliser |
| `kibot_launch.sh` | Unchanged — local builds continue to work; could optionally add `kdt validate` call but not required |

---

## 7. Open Questions & Decisions

These need resolution before or during implementation.

| # | Question | Options | Recommendation | Notes |
|---|----------|---------|----------------|-------|
| Q1 | **Where does `kdt` live?** | (a) Single script in template repo, (b) Separate pip package, (c) Both | (c) Both — ship `kdt.py` in the template for zero-dep use, publish to PyPI for ergonomics | If pip package, name it `kdt-config` to avoid conflicts |
| Q2 | **Does `kdt generate` auto-run in CI?** | (a) Explicit CI step, (b) Git pre-commit hook, (c) Both | (a) Explicit CI step | Pre-commit hooks are opt-in and unreliable. CI step guarantees it runs. |
| Q3 | **Should we validate on every push or only on PRs?** | (a) Every push, (b) PRs only, (c) PRs + tag pushes | (a) Every push | Fast validation (~2s) prevents broken builds early. |
| Q4 | **How to handle the Unreleased section in dev builds?** | (a) Leave empty, (b) Generate preview from git log, (c) Maintain manually alongside release-please | (a) Leave empty initially | Can add (b) later. (c) defeats the purpose of automation. |
| Q5 | **Should `kdt init` create the `.kdt-project.yaml` in the repo root?** | (a) Repo root, (b) kibot_yaml/ directory | (a) Repo root | It's a project-level config, not a KiBot-specific config. Visible in the top-level directory listing. |
| Q6 | **Should `kdt` handle the `kibot_launch.sh` local build script?** | (a) Yes — read variant from .kdt-project.yaml, (b) No — leave as-is | (b) No for now | kibot_launch.sh already reads version from CHANGELOG.md via get_changelog_version.py. Can integrate later. |
| Q7 | **Tag format: `1.2.0` or `v1.2.0`?** | (a) No prefix (current template), (b) v prefix (release-please default) | (a) No prefix | Template CI already uses `[0-9]+.[0-9]+.[0-9]+` pattern. Set `include-v-in-tag: false` in release-please config. |
| Q8 | **Squash merge or merge commits for PRs?** | (a) Squash (single conventional commit per PR), (b) Merge (preserves individual commits) | (a) Squash merge | Single commit per PR means the PR title IS the conventional commit message. Simpler enforcement via `action-semantic-pull-request`. |
| Q9 | **What happens to existing projects adopting this?** | Need a migration path | `kdt init` should detect existing kibot_main.yaml definitions and pre-populate .kdt-project.yaml from them | One-time migration, then the YAML is the source of truth going forward. |
| Q10 | **Should `normalise_changelog.py` be idempotent?** | (a) Yes — safe to run on already-normalised files, (b) No — only run once | (a) Yes, must be idempotent | It will run on every release. If CHANGELOG.md is already in KaC format (e.g., user manually edited), it should be a no-op. |

---

## 8. Phasing & Dependencies

### Phase 1 — Foundation (week 1-2)

**Goal**: Get release-please working end-to-end with correct changelog format.

| Task | Depends on | Effort |
|------|-----------|--------|
| Write `normalise_changelog.py` | — | S |
| Write `release-please-config.json` and manifest | — | S |
| Write `.github/workflows/release.yaml` | normalise_changelog.py | M |
| Write `.github/workflows/lint-pr.yaml` | — | S |
| Test release-please end-to-end on a fork | All above | M |
| Verify `get_changelog.py` parses normalised output correctly | normalise_changelog.py | S |

**Milestone**: Push a conventional commit, see a Release PR appear with a
correctly formatted changelog, merge it, see a tag + GitHub release created
automatically. Verify `get_changelog.py` can parse the resulting CHANGELOG.md.

### Phase 2 — CLI Core (week 3-4)

**Goal**: `kdt init`, `kdt validate`, `kdt generate` working.

| Task | Depends on | Effort |
|------|-----------|--------|
| Design `.kdt-project.yaml` schema (finalise all fields) | — | S |
| Write S-expression tokeniser for .kicad_sch/.kicad_pcb | — | M |
| Write `kdt init` (interactive prompts + KiCad file parsing) | S-expr parser | L |
| Write `kdt generate` (propagate config to YAML files) | Schema | M |
| Write `kdt validate` (all checks from section 2.3) | S-expr parser, Schema | M |
| Test on the template project and one real project | All above | M |

**Milestone**: Run `kdt init` on a real KiCad project, see `.kdt-project.yaml`
generated with correct BOM fields and layer names auto-detected. Run
`kdt validate` and see all checks pass.

### Phase 3 — Changelog Sync & Promotion (week 5)

**Goal**: `kdt sync-changelog` and `kdt promote` working. Full integration.

| Task | Depends on | Effort |
|------|-----------|--------|
| Write `kdt sync-changelog` | Phase 1 (normalise_changelog.py) | M |
| Write `kdt promote` | kdt validate, kdt generate | S |
| Write `kdt status` | Schema, git integration | S |
| Refactor `ci.yaml` to read variant from `.kdt-project.yaml` | kdt generate | M |
| Integration test: full release cycle on a fork | All above | L |
| Write documentation (README section on the new workflow) | All above | M |

**Milestone**: Complete release cycle with zero manual file editing —
conventional commit → Release PR → merge → tag → KiBot RELEASED outputs →
GitHub Release with assets. The only human actions are: writing the commit
message and merging the Release PR.

### Phase 4 — Polish & Migration (week 6+)

| Task | Depends on | Effort |
|------|-----------|--------|
| Migration path for existing projects (`kdt init --from-existing`) | kdt init | M |
| TUI mode with selectable lists | kdt init | M |
| pip packaging and PyPI publish | kdt.py stable | S |
| Upstream PR to KDT template repo | All above | L |
| Documentation: conventional commit cheat sheet for hardware teams | — | S |

**Effort key**: S = small (< 1 day), M = medium (1-3 days), L = large (3-5 days)

---

## Appendix A — Mapping: `.kdt-project.yaml` → Downstream Files

Complete field-level mapping for the `kdt generate` command.

```
.kdt-project.yaml field          → Target file                              → Target field/line
─────────────────────────────────────────────────────────────────────────────────────────────────
project.name                     → kibot_main.yaml definitions              → PROJECT_NAME
project.board_name               → kibot_main.yaml definitions              → BOARD_NAME
project.company                  → kibot_main.yaml definitions              → COMPANY
project.designer                 → kibot_main.yaml definitions              → DESIGNER
project.logo                     → kibot_main.yaml definitions              → LOGO
project.git_url                  → kibot_main.yaml definitions              → GIT_URL
kicad.version                    → .github/workflows/ci.yaml env            → kicad_version
kicad.color_theme                → kibot_main.yaml definitions              → COLOR_THEME
kicad.worksheet                  → kibot_main.yaml definitions              → SHEET_WKS
bom.mpn_field                    → kibot_main.yaml definitions              → MPN_FIELD
bom.man_field                    → kibot_main.yaml definitions              → MAN_FIELD
fabrication.check_zone_fills     → kibot_main.yaml definitions              → CHECK_ZONE_FILLS
fabrication.stackup_note         → kibot_main.yaml definitions              → STACKUP_TABLE_NOTE
fabrication.scaling              → kibot_main.yaml definitions              → FAB_SCALING
fabrication.plot_refs            → kibot_main.yaml definitions              → PLOT_REFS
fabrication.group_round_slots    → kibot_main.yaml definitions              → GROUP_ROUND_SLOTS
fabrication.group_pth_npth       → kibot_main.yaml definitions              → GROUP_PTH_NPTH
fabrication.group_pth_npth_drl   → kibot_main.yaml definitions              → GROUP_PTH_NPTH_DRL
assembly.scaling                 → kibot_main.yaml definitions              → ASSEMBLY_SCALING
assembly.exclude_refs            → kibot_main.yaml definitions              → EXCLUDE_REFS
rendering.rotation[0]            → kibot_main.yaml definitions              → 3D_VIEWER_ROT_X
rendering.rotation[1]            → kibot_main.yaml definitions              → 3D_VIEWER_ROT_Y
rendering.rotation[2]            → kibot_main.yaml definitions              → 3D_VIEWER_ROT_Z
rendering.zoom                   → kibot_main.yaml definitions              → 3D_VIEWER_ZOOM
rendering.key_color              → kibot_main.yaml definitions              → KEY_COLOR
layers.title_page                → kibot_main.yaml definitions              → LAYER_TITLE_PAGE
layers.drill_map                 → kibot_main.yaml definitions              → LAYER_DRILL_MAP
layers.tp_list_top               → kibot_main.yaml definitions              → LAYER_TP_LIST_TOP
layers.tp_list_bottom            → kibot_main.yaml definitions              → LAYER_TP_LIST_BOTTOM
layers.assembly_text_top         → kibot_main.yaml definitions              → LAYER_ASSEMBLY_TEXT_TOP
layers.assembly_text_bottom      → kibot_main.yaml definitions              → LAYER_ASSEMBLY_TEXT_BOTTOM
layers.dnp_top                   → kibot_main.yaml definitions              → LAYER_DNP_TOP
layers.dnp_bottom                → kibot_main.yaml definitions              → LAYER_DNP_BOTTOM
workflow.variant                 → .github/workflows/ci.yaml env            → kibot_variant
```

## Appendix B — Changelog Format: Before and After Normalisation

### Release-please raw output:
```markdown
# Changelog

## [1.2.0](https://github.com/acme/hebc-001/compare/1.1.0...1.2.0) (2025-03-20)


### Added

* add testpoint TP15 on I2C bus ([abc1234](https://github.com/acme/hebc-001/commit/abc1234))
* new analog frontend for sensor input ([def5678](https://github.com/acme/hebc-001/commit/def5678))


### Fixed

* correct timing on clock distribution network ([789abcd](https://github.com/acme/hebc-001/commit/789abcd))


### Changed

* increase power trace width from 10mil to 15mil ([111aaaa](https://github.com/acme/hebc-001/commit/111aaaa))
* substitute TPS63001 with TPS63051 (EOL) ([222bbbb](https://github.com/acme/hebc-001/commit/222bbbb))
```

### After `normalise_changelog.py`:
```markdown
# Changelog

## [Unreleased]

## [1.2.0] - 2025-03-20

### Added

-   Add testpoint TP15 on I2C bus
-   New analog frontend for sensor input

### Fixed

-   Correct timing on clock distribution network

### Changed

-   Increase power trace width from 10mil to 15mil
-   Substitute TPS63001 with TPS63051 (EOL)
```

The normalised output is exactly what `get_changelog.py` expects to parse.
