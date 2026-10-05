# CLAUDE.md — Fork workflow context (FilthTribeFTP/vpx-standalone-alp4k)

Context for Claude Code sessions working in this fork. Owner: Brandon Griffin
(VPXS Wiz Tester). Goal: automate fork maintenance, `table.yml` config work,
and PR preparation so Brandon's time goes to physically testing tables on the
cabinet.

## What this repo is

Per-table wizard configs for VPX Standalone (Linux build) on the AtGames
Legends Pinball 4K (ALP4K, RK3588 SoC), run via the Legends Unchained Loader.
The repo holds **configs and metadata only**: `table.yml`, INIs, VBS scripts,
docs, and images. The LU Table Manager does not read YAML from the repo
directly — it consumes a build artifact attached to GitHub Releases, built by
GitHub Actions.

VPXS is now public and stable. Release tracks: stable (public), beta testers,
and wizards (Brandon's track, above beta). Upstream cuts prereleases with
"Create Testing Release" and promotes them with "Promote Testing Release".

Repo layout: table folders live in `tables/vpx-<name>/` (formerly
`external/`). Root JSON files: `team_favorites.json` (Team Favorites page,
see README) and `editors_picks.json` (plain array of table folder names).

## HARD RULES

- **Never commit table binaries, ROMs, or any DRM-protected file.** `.vpx`
  files are DRM-injected at install time; `.elf` launchers and lu-tablemanager
  files are also protected. PRs contain configs/metadata only.
- Safe (no DRM) file types this repo does carry per table: `.yml`, `.ini`,
  `.vbs`, `.nv`, `.png`/`.webp`, `alias.txt`, `buttons.ini`, `VPReg.ini`.
- Never commit to `main`. See branch discipline below.

## Repo topology and branch discipline

- **Upstream:** `LegendsUnchained/vpx-standalone-alp4k` — **every** PR (fixes
  and full Wizard table submissions) targets its `main`. There is no longer a
  staging repo in between (evilwraith's repo was used for that during beta).
- PRs are opened from the fork branch via GitHub's compare page:
  `https://github.com/LegendsUnchained/vpx-standalone-alp4k/compare/main...FilthTribeFTP:vpx-standalone-alp4k:<branch>`
- **This fork's `main` stays exactly synced with upstream main — zero local
  commits.** All work goes on a new branch per PR. When creating a PR branch,
  base it on **upstream** main (`git fetch` the upstream remote and branch from
  `upstream/main`), never on a fork branch that might carry local commits, so
  the PR diff vs upstream stays clean.
- Fix-branch naming pattern: `fix-<table>-<what>`
  (e.g. `fix-goldeneye-checksum-3.1`).
- A branch may hold multiple commits if related or batched the same day(s).

## Wizard table submission conventions

- **1–2 tables max per PR** for full Wizard table submissions.
- New folder under `tables/`, named `vpx-<tablename>` — lowercase, no
  brackets (e.g. `vpx-tz` for Twilight Zone).
- Files in the table folder, named exactly:
  - `README.md` — for Wizard tables this is now just a two-line pointer to
    the Table Manager catalog (copy `tables/vpx-hpgof/README.md`, swap the
    folder name). The long `table-template_README.md` format and the
    `images/` playfield preview are obsolete for Wizard tables.
  - `table.yml`
  - `launcher.png` (640x960)
  - Optionally, if applicable: `table.ini`, `table.vbs`, `nvram.nv`
- Image standards enforced by the hook and CI: every `*.png` must be a real,
  non-animated PNG; `launcher.png` 640x960, `backglass.png` 1920x1080,
  `dmd.png` 1920x1200. Check real pixel dimensions before committing.
- Converting an existing manual-install table to Wizard: replace the whole
  folder (old README becomes the catalog pointer; drop files the new set
  doesn't include).
- Keep `table.vbs` line endings matching the previous file / repo majority
  (LF) so the PR diff shows only real script changes.
- `table.yml` is authored with the YML generator tool at
  https://wizardyml.legendsunchained.com/ (by Mox / moxassault); the full
  field reference is `table.yml.example` and `doc/advanced/wizardtable.adoc`.

## PR mechanics

- Checksums are **lowercase MD5**. Brandon generates them on Windows with
  `(Get-FileHash <file> -Algorithm MD5).Hash.ToLower()`.
- PR bodies: short, factual, reference the originating Discord bug report or
  GitHub issue. No drama, no file requests.
- Data Claude cannot derive and must get from Brandon per table: VPS IDs, MD5
  hashes of the actual files, measured FPS, tester names, and binary assets
  (`launcher.png`, `nvram.nv`, patched `.vbs`).
- Binary files must reach Claude **zipped**: the chat app re-encodes any
  attached image to a lossy WebP. Brandon zips the PNG (or the whole table
  folder) and attaches the `.zip`; text files (yml/ini/vbs) attach fine as-is.

## Pre-commit hook (required before every commit)

The repo's `.pre-commit-config.yaml` runs the same checks CI runs on a PR:
`table.yml` metadata validation against VPSDB + yamllint, and image standards
for `*.png`. It only checks **staged** files.

- Hooks live in `.git/`, so every fresh clone/container needs a one-time
  install (`.venv` is already gitignored):
  ```sh
  python3 -m venv .venv
  .venv/bin/python -m pip install pre-commit
  .venv/bin/python -m pre_commit install
  ```
- Check staged files: `.venv/bin/python -m pre_commit run`. For specific
  files: `... pre_commit run --files <paths>`.
- **Never** use `--all-files` (validates 400+ tables against VPSDB).
- **Never** bypass with `--no-verify`. A failure gets fixed, not skipped.
- The validator fetches `virtualpinballspreadsheet.github.io`; in Claude cloud
  sessions that host must be on the environment's network allowlist, or the
  metadata check fails with a proxy 403 (a network problem, not a YAML one).

## Fork releases (testing this fork on the cabinet)

Branch model (same as pinballwizard2023's fork):
- One branch per table, created from `upstream/main`, named
  `add-vpx-<tablename>` — this is the branch the PR is opened from.
- `Testing-Main` = `upstream/main` + a `--no-ff` merge of every table branch
  under test. Releases are cut **from `Testing-Main`**, so every table in
  testing ships together. Never PR from `Testing-Main`.
- Changes are made on the table branch first, then merged into
  `Testing-Main` again. To pick up upstream, merge `upstream/main` into
  `Testing-Main` (merge, never rebase). After a table's PR is merged
  upstream, its branch is done; the next upstream merge absorbs it.

- Use the **Create Fork Release** workflow (Actions tab →
  Create Fork Release → Run workflow, "Use workflow from" = `Testing-Main`). Leave the tag blank to auto-bump the
  patch version (first release is `v1.0.0`). It refuses to run on upstream.
- It builds the release assets and publishes a normal, non-prerelease
  release marked latest: Table Manager reads a non-upstream config repo via
  `/releases/latest`, which ignores prereleases and drafts.
- Actions must be enabled on the fork first (forks have them off by default).
- Table Manager is pointed at the fork via the `configrepo` key in
  `external/lu-tablemanager/settings.json` on the cabinet drive. An empty
  Wizard means the release has no build artifact.

## Config hierarchy (VPXS on ALP4K)

1. `Default_VPinballX.ini` at USB root — global config for all tables.
2. Per-table INI (e.g. `vpx-tz.ini` inside the table folder) — overrides the
   global for that table.
3. Anything undefined in either falls back to the canonical Default in this
   repo (contains ALP4K-specific tuning: `PUPStaggerStartup = 1`, RKMPP/GBM
   producer flags, `PUPDMDWindowX=3840` geometry, ALP-4KP button mapping).
- `VPinballX.ini` inside a table folder is an auto-generated clone of the root
  Default — it does NOT override the per-table INI. Never edit it; it
  regenerates if deleted.
- `table.ini.base` (newer): full settings list for a table, so custom settings
  survive table updates (updates overwrite custom `.vbs` and `.ini`).
- `PauseResumeCountdownSeconds` (`[Player]` block) is INI-configurable;
  Brandon runs 3 (default 5). No user-configurable playfield-present watchdog
  timeout exists.

## Wheel image pipeline (Python + Pillow)

- Target: 640x960 PNG wheel images.
- Standard wheels: crop to content bounding box (PIL `getbbox()` — mandatory,
  source padding varies wildly), scale to 640x640, pad 160px transparent
  top+bottom.
- LU-themed composites: 640x960 playfield PNG background → wheel ring cropped
  via `getbbox()`/ring detection, scaled to 549px diameter, pasted at
  (40, 195) → `lu_logo_layer.png` at (0,0) on top. Reference:
  `vpx-burner.png`.
- Never scale full canvases to fixed sizes — crop to content first, always.
  Verify every output, not a sample.
- For tiny pixelated logos: alpha mask → potrace → SVG → cairosvg raster at
  10x.

## Debugging knowledge base (VPXS Linux crash/script patterns)

Common script-level crash causes:
- Windows-only COM calls (`shell.application`, `WScript.Shell`,
  `Scripting.FileSystemObject`) — stub or branch around them.
- Undefined VBS variables (`PinCab_Blades`, `Bulb15`, `B2SOn`,
  `BallShadowA0-A10`, etc.).
- Compile-time syntax issues (`Then.GetImage` missing space, `Elseif`
  spacing).
- Wrong object references (`Plunger.interval` vs `PlungerTimer.interval`).
- Cache poisoning: mis-tagged texture in `used_textures.xml` (colorspace
  mismatch crash).
- Missing asset folders (e.g. a `<table>.FlexDMD` folder must sit beside the
  `.vpx`).
- Segfaults below script level = native VPX crash; report to devs, not
  script-fixable.

Platform-level (RK3588) known issues — NOT fixable via script:
- Playfield-present watchdog fires from KMS/DRM contention between CRTC
  outputs during multi-display init.
- Post-pause FPS degradation: KMS page-flip timeout during Reflection Probe
  prerender burst disables plane-driven presentation for the session
  (60fps→~47fps); only fix is table restart.

PUP pack routing:
- Content routes by `ScreenNum`, not `ScreenDes` labels (cosmetic only).
- `CustomPos` source screen numbers must point to physically active screens.
- DMD screen (1) needs blank `CustomPos` to resolve as a standalone window via
  `PUPDMDWindowX=3840`; FullDMD packs use screen 5 as content hub.
- Linux is case-sensitive: watch filename case mismatches
  (`balllocked.mp4` vs `BallLocked.mp4`) — cross-check log errors against
  actual folder listings.

## Working style

- Brandon is a 16+ year sysadmin — skip foundational explanations.
- Verify with actual data (real byte counts, real pixel scans, real hashes)
  before asserting; never derive targets from rounded/estimated numbers.
- Bug reports go through Discord channels or GitHub issues, not dev DMs.
- Discord-facing writeups aimed at players: friendly, factual, non-technical.
- PowerShell is Brandon's native tooling on Windows; Python/Pillow for image
  work.
