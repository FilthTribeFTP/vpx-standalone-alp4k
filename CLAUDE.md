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

## HARD RULES

- **Never commit table binaries, ROMs, or any DRM-protected file.** `.vpx`
  files are DRM-injected at install time; `.elf` launchers and lu-tablemanager
  files are also protected. PRs contain configs/metadata only.
- Safe (no DRM) file types this repo does carry per table: `.yml`, `.ini`,
  `.vbs`, `.nv`, `.png`/`.webp`, `alias.txt`, `buttons.ini`, `VPReg.ini`.
- Never commit to `main`. See branch discipline below.

## Repo topology and branch discipline

- **Upstream:** `LegendsUnchained/vpx-standalone-alp4k` — fixes PR there.
- **Staging:** evilwraith's (Wraith's) repo — full Wizard **table submissions**
  go there first, before official merge into LegendsUnchained main.
- **This fork's `main` stays exactly synced with upstream main — zero local
  commits.** All work goes on a new branch per PR. When creating a PR branch,
  base it on **upstream** main (`git fetch` the upstream remote and branch from
  `upstream/main`), never on a fork branch that might carry local commits, so
  the PR diff vs upstream stays clean.
- Fix-branch naming pattern: `fix-<table>-<what>`
  (e.g. `fix-goldeneye-checksum-3.1`).
- A branch may hold multiple commits if related or batched the same day(s).

## Wizard table submission conventions (Wraith's preferences)

- **1–2 tables max per PR** for full Wizard table submissions.
- New folder under `external/`, named `vpx-<tablename>` — lowercase, no
  brackets (e.g. `vpx-tz` for Twilight Zone).
- Files in the table folder, named exactly:
  - `README.md` (follow `table-template_README.md` in the repo root)
  - `table.yml`
  - `launcher.png` (500x750)
  - Optionally, if applicable: `table.ini`, `table.vbs`, `nvram.nv`
- Plus one new file in `images/`: a `.webp` of the table's playfield,
  referenced by path from the table's README so it renders on GitHub.
- `table.yml` is authored with the YML generator tool at
  https://wizardyml.legendsunchained.com/ (by Mox / moxassault); the full
  field reference is `table.yml.example` and `doc/advanced/wizardtable.adoc`.

## PR mechanics

- Checksums are **lowercase MD5**. Brandon generates them on Windows with
  `(Get-FileHash <file> -Algorithm MD5).Hash.ToLower()`.
- PR bodies: short, factual, reference the originating Discord bug report or
  GitHub issue. No drama, no file requests.
- `table.yml` changes are validated by CI (`validate-table-yaml.yml`) against
  VPSDB; run the repo's pre-commit validation locally before pushing when
  possible. Fix reported errors — never bypass the hook.
- Data Claude cannot derive and must get from Brandon per table: VPS IDs, MD5
  hashes of the actual files, measured FPS, tester names, and binary assets
  (`launcher.png`, `nvram.nv`, patched `.vbs`, playfield `.webp`).
- Fork-as-config-source testing (when pointing Table Manager at this fork):
  enable Actions on the fork → push tag → workflow builds artifact → publish
  release. Table Manager reads the release artifact via the `configrepo` key
  in `external/lu-tablemanager/settings.json` on the cabinet drive. An empty
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
