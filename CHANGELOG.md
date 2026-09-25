# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

This is the changelog of the workspace root (`locorda_workspace`,
`publish_to: none`). Per-package history lives in `packages/*/CHANGELOG.md`.

## [Unreleased]

### Added

- **Documentation**: root `BACKLOG.md` created. It carries open/pending items
  only — the build and analysis problems found by the Windows-lane audit
  (Windows 11, Dart SDK 3.13.1 stable, Flutter 3.47.1 stable, HEAD `fceb240`)
  and every divergence from the `staylorx/dart-flutter-bible` standard
  (bible @ `d4d2ff1`), the latter flagged for later review and not auto-fixed.
- **Documentation**: root `CHANGELOG.md` created (this file).

### Notes

- Decisions recorded by this pass:
  - `BACKLOG.md` holds open/pending items only; anything decided and done is
    recorded here instead.
  - Every divergence from `staylorx/dart-flutter-bible` is recorded in
    `BACKLOG.md` verbatim in the required
    `Deviation: <path> - <what diverges and why>` form and left unfixed — the
    bible itself may be the side that is wrong on several of them.
  - No source file was changed by this pass. Generated blobs (the committed
    l10n files, `*.g.dart`, `pubspec.lock`, `pubspec_overrides.yaml`) were read
    only, never hand-edited.
  - The repo has no `AGENTS.md` at HEAD `fceb240`; the conventions that do exist
    (`CLAUDE.md`, `GEMINI.md`, `IMPLEMENTATION.md`) are followed where they do
    not conflict with the bible. Adding `AGENTS.md` is open as item B12.
  - Open question left in `BACKLOG.md`: `melos.yaml` sets
    `workspaceChangelog: true`, so a future `melos version` run would rewrite
    this hand-written root changelog. Decide whether melos owns the root
    changelog or this file does.
- Audit record for the working version `0.9.15-dev` (not a cut release) at HEAD
  `fceb240`, Windows lane: root `dart pub get` exit 0; `pub get` exit 0 for all
  7 packages and the example — no broken dependency; `analyze` exit 0 with
  "No issues found!" in 7 packages and the example, exit 2 in
  `locorda_annotations` (single `invalid_dependency` warning — BACKLOG.md B1);
  test suites pass — `locorda` 69, `locorda_core` 148, `locorda_drift` 32,
  `example` 11 (260 tests, 0 failures, 0 skips; four packages have no `test/`
  directory); `dart pub publish --dry-run` for `locorda_annotations` exit 65
  with 3 errors, 4 potential issues and 2 hints (BACKLOG.md B3–B9). No build
  error and no broken dependency was found.
- `BACKLOG.md` also records the 92 divergences from
  `staylorx/dart-flutter-bible` @ `d4d2ff1`, plus three latent code defects
  found while auditing (`OrSet.remove()` tombstone mismatch, `SolidBackend`
  shipping `UnimplementedError` contract members, `LocordaSync.save()` never
  persisting).
