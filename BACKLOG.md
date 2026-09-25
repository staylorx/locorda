# BACKLOG — locorda

Open and pending items only. Anything already decided and done is recorded in
`CHANGELOG.md`, not here.

Repo: `staylorx/locorda`, default branch `main`, audited at HEAD `fceb240`
("refactor: update RDF dependencies and improve code generation patterns").
Sources: the Windows-lane build & analysis report (board task `t_42915a85`) and
the `staylorx/dart-flutter-bible` deviation audit (board task `t_d14138f3`,
bible @ `d4d2ff1` — `docs/00-compact.md` + `docs/01..12`). Neither pass changed a
source file, and nothing recorded below has been auto-fixed.

There is no `AGENTS.md` in this repo (see B12), so there is no repo-local
convention document to follow beyond `CLAUDE.md` / `GEMINI.md` /
`IMPLEMENTATION.md`; its rules are reflected in the deviation list below where
they conflict with the bible.

Measured baseline on the Windows lane (Windows 11, git-bash/MSYS; Dart SDK
3.13.1 stable windows_x64, Flutter 3.47.1 stable). Exit codes are real, captured
in this workspace under `audit2/`:

| Command (per package) | Exit | Result |
|---|---|---|
| root `dart pub get` | 0 | `Got dependencies!` |
| `locorda` — flutter pub get / analyze / test | 0 / 0 / 0 | 69 tests pass |
| `locorda_annotations` — dart pub get / analyze | 0 / **2** | 1 warning (`invalid_dependency`); no `test/` dir |
| `locorda_core` — dart pub get / analyze / test | 0 / 0 / 0 | 148 tests pass |
| `locorda_drift` — flutter pub get / analyze / test | 0 / 0 / 0 | 32 tests pass |
| `locorda_solid` — dart pub get / analyze | 0 / 0 | no `test/` dir |
| `locorda_solid_auth` — flutter pub get / analyze | 0 / 0 | no `test/` dir |
| `locorda_solid_ui` — flutter pub get / analyze | 0 / 0 | no `test/` dir |
| `example` — flutter pub get / analyze / test | 0 / 0 / 0 | 11 tests pass |
| `dart pub publish --dry-run` (`locorda_annotations`) | **65** | 3 errors, 4 potential issues, 2 hints |

So the monorepo builds, analyzes and tests clean on Windows with the committed
generated l10n files present, except for one analyzer finding and the
`locorda_annotations` publish blockers below.

## Build & analysis problems

Eighteen open items (B1–B18), plus one Windows-lane environment note (E1)
below. Raw messages are quoted verbatim from the captures behind the table
above; the full report is `FINDINGS_locorda_windows_audit.md`.

### B1 — analyzer gate fails: `invalid_dependency` in `locorda_annotations`

`packages/locorda_annotations/pubspec.yaml:12:5` declares `locorda_core` as a
`path:` dependency while the package is publishable. `dart analyze` → exit 2,
`melos analyze` → exit 1. Raw output:

```
warning - pubspec.yaml:12:5 - Publishable packages can't have 'path' dependencies.
Try adding a 'publish_to: none' entry to mark the package as not for publishing
or remove the path dependency. - invalid_dependency

1 issue found.
```

Pending: decide whether `locorda_annotations` is published or workspace-only;
either add `publish_to: none` or drop the path dependency and take `locorda_core`
from a hosted source. This is the only analyzer finding in the whole workspace.

### B2 — `publisher:` is not a pub key

`packages/locorda_annotations/pubspec.yaml:4` — raw message from
`dart pub publish --dry-run`:

```
* "publisher" is not a key recognized by pub - did you mean "publish_to"?
```

Pending: remove or rename the key; as written it is silently ignored.

### B3 — `locorda_annotations` cannot publish: no `LICENSE` file

`dart pub publish --dry-run` → exit 65, raw message:

```
* You must have a LICENSE file in the root directory.
```

The repo root has a `LICENSE`, the package root does not. Pending: add the
package-level `LICENSE` (or a symlink/`copy`) if the package is meant to ship.

### B4 — `locorda_annotations` cannot publish: `path:` dependency on `locorda_core`

Raw message:

```
* Don't depend on "locorda_core" from the path source. Use the hosted source instead.
```

Pending: same decision as B1 — publish order + hosted version, or keep the
package private.

### B5 — undeclared dependency: `rdf_core` used but not declared

`packages/locorda_annotations/lib/src/pod_resource.dart:5` imports
`package:rdf_core/rdf_core.dart`; `rdf_core` appears in no `dependencies`
section of that pubspec. Raw message:

```
* line 5, column 1 of .../locorda_annotations/lib/src/pod_resource.dart:
  This package does not have rdf_core in the `dependencies` section of `pubspec.yaml`.
```

It resolves today only transitively. Pending: declare `rdf_core` explicitly (and
decide `locorda_core` vs `rdf_core` as the dependency direction).

### B6 — no `homepage`/`repository` field in `locorda_annotations`

```
* It's strongly recommended to include a "homepage" or "repository" field in your pubspec.yaml
```

Pending: add the field (the root pubspec already carries `homepage:
https://locorda.dev/`, `repository: https://github.com/locorda/locorda` — note
that URL points at `locorda/locorda`, while this checkout's origin is
`staylorx/locorda`; settle that too).

### B7 — no `README.md` in `locorda_annotations`

```
* Please add a README.md file that describes your package.
```

Pending: add a README following the repo's documentation guidelines in
`CLAUDE.md` (package READMEs reference the main `locorda` narrative).

### B8 — non-dev dependencies are overridden in `pubspec_overrides.yaml`

```
* Non-dev dependencies are overridden in pubspec_overrides.yaml.
```

`melos bootstrap` injects the `locorda_core` path override plus dependency
overrides, so what CI tests is not what a published consumer resolves. Pending:
resolve via native pub workspaces instead (see the §2 deviation group below).

### B9 — version regression: `0.1.0+1` is older than the published `0.5.2`

`packages/locorda_annotations/pubspec.yaml:3` — raw message:

```
* The latest published version is 0.5.2.  Your version 0.1.0+1 is earlier than that.
```

Pending: settle the fork/version story before any publish; a real
`dart pub publish` of this tree would regress the published 0.5.x line.

### B10 — `rdf_mapper_annotations` is discontinued

Every package that resolves it reports `rdf_mapper_annotations 0.10.2` as
"discontinued, replaced by `locorda_rdf_mapper_annotations`". Pending: migrate
the annotation/codegen dependency chain, which also touches the §7 codegen
deviations below.

### B11 — committed generated l10n files are load-bearing (build breaks when absent)

`packages/locorda_solid_auth/lib/l10n/solid_auth_localizations{,_de,_en}.dart`
are generated (`l10n.yaml`, `flutter: generate: true`) but committed. When
missing, the build breaks hard:

```
lib/locorda_solid_auth.dart:12:8        uri_does_not_exist
lib/src/ui/login_page.dart:6:8          uri_does_not_exist
lib/src/ui/solid_status_widget.dart:5:8 uri_does_not_exist
login_page.dart:70,88,161 / solid_status_widget.dart:128,137,184
                                        undefined_identifier 'SolidAuthLocalizations'
example/lib/main.dart:114,116           undefined_identifier 'SolidAuthLocalizations'
example/test/widget_test.dart:41        non_constant_list_element + undefined_identifier
example `flutter test`                  "Compilation failed"
```

i.e. analyze exit 1 for `locorda_solid_auth` (9 issues) and `example` (4
issues), example test exit 1. Verified fix: `cd packages/locorda_solid_auth &&
flutter gen-l10n` regenerates the three files **byte-identical to HEAD** (git
status clean afterwards), after which analyze/test are exit 0. Root cause of the
disappearance observed in the audit: `flutter pub get` wipes the l10n output dir
and only regenerates it if resolution succeeds, so an interrupted `pub get`
leaves the tracked files deleted. Pending: add an explicit `flutter gen-l10n`
step to bootstrap/CI, or gitignore the l10n output and generate on build.

### B12 — `AGENTS.md`, `BACKLOG.md` and `.gitattributes` absent at the repo root

At HEAD `fceb240` the root carries `CLAUDE.md`, `GEMINI.md`,
`IMPLEMENTATION.md`, `CONTRIBUTING.md`, `TODO.md`, `melos.yaml` and `README.md`,
but no `AGENTS.md`, no `BACKLOG.md` (this file) and no `.gitattributes`. The
repo therefore has no agent-facing convention file and no line-ending
enforcement. Pending: add `AGENTS.md` (declared error style + local wiring) and
`.gitattributes` (`* text=auto eol=lf`).

### B13 — uncommitted `analyzer: exclude:` block in the example

`packages/locorda/example/analysis_options.yaml` was already modified in the
working tree before the audit ran, adding an `analyzer: exclude:` block for
`build/`, `android/`, `ios/`, `web/`, `windows/`, `macos/`, `linux/`. Not
produced by any command in that pass. Pending: decide whether that exclusion is
wanted and commit it, or drop it.

### B14 — 13 files dirtied by `melos bootstrap`

`packages/*/pubspec.lock` and `packages/*/pubspec_overrides.yaml` (13 files) show
as modified vs HEAD after bootstrap, and the repo does not commit the
bootstrapped overrides. Expected bootstrap fallout, but it means the working tree
is never clean after a bootstrap. Pending: decide what is tracked (see B8).

### B15 — documentation drift: `melos.yaml` / `CLAUDE.md` describe a tree that is not there

`CLAUDE.md` documents a `locorda_generator` package that does not exist under
`packages/`, and documents `packages/locorda/docs/adrs/`, `vocabularies/`,
`mappings/` and `spec/docs/...` paths that do not match the tree at HEAD.
Pending: reconcile the docs with the tree (or restore the missing paths).

### B16 — four packages have no tests

No `test/` directory in `locorda_annotations`, `locorda_solid`,
`locorda_solid_auth` or `locorda_solid_ui`. Only `locorda` (69),
`locorda_core` (148), `locorda_drift` (32) and `example` (11) have suites.
Pending: add at least the contract coverage the bible asks for (§6 below).

### B17 — dependency freshness

24–99 packages per package have newer versions incompatible with the current
constraints; `test` resolves 1.26.3 vs 1.32.0 available and `analyzer` 8.1.1 vs
14.4.0. Nothing is broken. Pending: refresh constraints when convenient
(`dart pub outdated`).

### B18 — no `dart format` gate anywhere, and the tree is not format-clean

`dart format --output=none --set-exit-if-changed` fails on nine files (3 in
`locorda_core/lib`, 2 in `locorda_solid_auth/lib`, 4 checked-in generated files
under `example/`). `.github/workflows/ci.yml` never runs a format check. Raw
evidence is in the `§2 Toolchain` deviation group below. Pending: format the
tree and add the gate to CI.

## Latent code defects (found while auditing, not bible deviations)

These are real defects surfaced by the audits, outside the bible's scope, and
were not fixed by either pass.

### L1 — `OrSet.remove()` tombstone cannot match non-`String` elements

`packages/locorda_core/lib/src/crdt/crdt_types.dart`: `remove()` stores
`element.toString()` into the tombstone set, while `value` does
`difference(_tombstones.cast<T>())`, so the removal path never matches elements
that are not `String`. Pending: fix the tombstone representation and add a
non-String removal test.

### L2 — `SolidBackend` ships unimplemented contract members

`packages/locorda_solid/lib/src/solid_backend.dart` publishes two of the three
`Backend` members as `throw UnimplementedError()` (lines 16, 19), so a runtime
`UnimplementedError` is reachable from a contract. Pending: implement them or
make the absence a typed failure.

### L3 — `LocordaSync.save()` never persists

The core `save()` only emits a hydration event and never writes through to
storage (its own `TODO` in `packages/locorda_core/lib/src/locorda_graph_sync.dart`
admits this). `TODO.md` already lists it as Priority 1. Pending: persist, then
delete the `TODO.md` entry.

## Deviations from `staylorx/dart-flutter-bible`

Each item below is a spot where the code differs from the bible standard while
the bible may itself be the side that is wrong. All are **flagged for later
review**: none was auto-fixed, and none is a build break. Entries are copied
verbatim, in the required `Deviation: <path> - <what diverges and why>` form,
from the audit report (`DEVIATIONS-locorda-vs-bible.md`, board task
`t_d14138f3`), which walked the whole workspace against
`staylorx/dart-flutter-bible` @ `d4d2ff1`.

Verification actually run by that audit, so the toolchain items are evidence and
not opinion:

```
dart --version                      -> Dart SDK version: 3.13.1 (stable) on "windows_x64"
dart format --output=none --set-exit-if-changed packages/locorda_core/lib       -> 3 changed (FAIL)
dart format --output=none --set-exit-if-changed packages/locorda_solid_auth/lib -> 2 changed (FAIL)
dart format --output=none --set-exit-if-changed packages/locorda/example        -> 4 changed (FAIL, generated .g.dart)
dart format --output=none --set-exit-if-changed <other packages lib/test>       -> 0 changed (PASS)
grep -rn "fpdart|equatable|shouldly|mocktail|dart_arch_test" (dart+yaml)        -> 0 hits
expect(...) in test/: locorda 159, locorda_core 279, locorda_drift 80           -> 518 total, 0 shouldly
analysis_options.yaml files: 1 (packages/locorda/example only)
```

92 deviations are recorded, grouped by the bible section they are filed under:
§1/§4 core semantics (35), §2 toolchain (26), §3 topology (5),
§5 persistence (3), §6 testing (6), §7 builders (5), §8 Flutter (10), §9/§10
conformance (2). §12 (sources of truth) is a bible-internal bibliography with no
repo-local surface.

### §1 Architecture / §4 Functional core — failure is not a value, no equatable, mixed layers

Deviation: packages/locorda_core/pubspec.yaml - depends on `logging`, `crypto`, `rdf_core` only: no `fpdart` and no `equatable` anywhere in the core package, so typed tuples and value semantics are impossible (§4 "fpdart ^1.2.0 pinned everywhere" + "equatable on every entity/value object").
Deviation: pubspec.yaml (root) and every packages/*/pubspec.yaml - `fpdart`, `equatable`, `shouldly`, `mocktail` and `dart_arch_test` are absent from the whole workspace's dependency graph (grep: 0 hits); the bible's pinned STACK is not installed (§2 STACK).
Deviation: packages/locorda_core/lib/src/storage/storage_interface.dart - the public `Storage` contract returns bare `Future<void>` / `Future<StoredDocument?>` / `Future<List<...>>`; the public seam is not `Future<Either<Failure, T>>`, so callers have no typed failure channel (§4 "the public seam is Future<Either<...>>").
Deviation: packages/locorda_core/lib/src/config/validation.dart - `ValidationResult.throwIfInvalid()` raises `SyncConfigValidationException`; failure is modelled as a thrown exception inside the core instead of a Left value (§1 law 4, §4 exception rule).
Deviation: packages/locorda_core/lib/src/sync/sync_engine.dart - `syncAll()` and `syncResourceType()` `throw StateError('Not authenticated - cannot sync')`; a business outcome is thrown from what is the application-ring orchestrator (§1 law 4, §4).
Deviation: packages/locorda_core/lib/src/locorda_graph_sync.dart - `setup()` throws `SyncConfigValidationException` (line 67), `hydrateStreaming()` throws `ArgumentError` (line 400) and the `ensure()` doc contracts a thrown `TimeoutException` (line 199): exceptions cross public core seams (§4).
Deviation: packages/locorda_core/lib/src/index/group_index_subscription_manager.dart - defines and throws `GroupIndexGraphSubscriptionException` for the two validation failures (lines 39, 55) instead of a closed, per-layer failure hierarchy returned as a value (§4 "Failure hierarchies are per-layer … closed").
Deviation: packages/locorda/lib/src/index/group_index_subscription_manager.dart - same pattern in the facade: `GroupKeyConverterException` thrown from `convertGroupKey()` and again from a catch block (lines 42, 68) (§4).
Deviation: packages/locorda/lib/src/config/sync_config_util.dart - hand-rolled `try { } on SerializerNotFoundException { return null; }` around a third-party registry call (line 21) in a non-adapter function; the only sanctioned conversion is `TaskEither.tryCatch` at an adapter boundary, returning Left (§4 "adapter boundary is a line, not a zone").
Deviation: packages/locorda/lib/src/config/sync_config_validator.dart - three hand-rolled try/catch blocks (lines 28, 84→103, 128→150) turn mapper exceptions into validation messages rather than Left values (§4).
Deviation: packages/locorda_core/lib/src/config/sync_config_base_validator.dart - `try { IriTerm.validated(...) } catch (e) {...}` inside a static validator (lines 22-25): hand-rolled try/catch outside an adapter boundary (§4).
Deviation: packages/locorda_core/lib/src/index/regex_transform_validation.dart - try/catch around `RegExp` compilation (lines 76-78) inside core logic (§4).
Deviation: packages/locorda/lib/src/config/sync_config_converter.dart - `throw ArgumentError('Unknown index type: ...')` (line 34) from a pure conversion function (§4).
Deviation: packages/locorda/lib/src/mapping/local_resource_iri_service.dart - throws `StateError` twice and `ArgumentError` once (lines 53, 83, 230) from a mapping service in the core ring (§4).
Deviation: packages/locorda_core/lib/src/hydration/hydration_stream_manager.dart - `emitToStream()` throws `StateError` when no controller exists (line 39) (§4).
Deviation: packages/locorda_solid/lib/src/solid_backend.dart - two of the three `Backend` members are published as `throw UnimplementedError()` (lines 16, 19); a contract is shipped with throwing, unimplemented members (§4, §3 contract-first).
Deviation: packages/locorda_core/lib/src/crdt/crdt_types.dart - `dynamic get value;` on `CrdtType` erases the value type at the centre of the domain (§1 law 2 purity, §2 "use Dart's type system effectively").
Deviation: packages/locorda_core/lib/src/crdt/hybrid_logical_clock.dart - `HybridLogicalClock` holds mutable `_logicalTime` / `_lastWallTime` and mutates them in `tick()` / `receive()`; a domain value object must be immutable and value-equal (§1 law 2, §4 equality).
Deviation: packages/locorda_core/lib/src/crdt/hybrid_logical_clock.dart - `HlcTimestamp` hand-writes `operator==`/`hashCode` instead of extending `Equatable` (§4).
Deviation: packages/locorda_core/lib/src/crdt/crdt_types.dart - `LwwRegister`, `FwwRegister` and `OrSet` have no value equality and `merge()` returns `this` by identity, so merged CRDT states cannot be compared by value in tests or pattern matching (§4 equality).
Deviation: packages/locorda_core/lib/src/hydration/type_local_name_key.dart - `TypeOrIndexKey` hand-writes `operator==`/`hashCode` rather than extending `Equatable` (§4).
Deviation: packages/locorda_core/lib/src/index/index_config_base.dart - `RegexTransform` hand-writes `operator==`/`hashCode`; the index-config value objects are not `Equatable` (§4).
Deviation: packages/locorda/lib/src/mapping/solid_mapping_context.dart - `SolidMappingContext` exposes three non-final public mutable fields (lines 18-20) with no `const` constructor and no equality (§4 immutability + equality).
Deviation: packages/locorda/lib/src/mapping/local_resource_iri_service.dart - `LocalResourceIriService` accumulates mutable setup state (`_isSetupComplete`, `_registeredTypes`, `_referencedTypes`, `_resourceTypeCache`, lines 30-32/163-164) and enforces its precondition by throwing; neither immutable nor value-typed (§1 law 2, §4).
Deviation: packages/locorda_drift/lib/src/drift_storage.dart - `DriftStorage` keeps mutable `bool _initialized` and mutates it in `initialize()`/`close()` (§4).
Deviation: packages/locorda/example/lib/models/note.dart - `Note` has non-final mutable fields, no `const` constructor and no `operator==`/`hashCode`: the app's own entity is neither immutable nor value-equal (§1 law 2, §4 equality).
Deviation: packages/locorda/example/lib/models/category.dart - same for `Category` (mutable fields, no Equatable).
Deviation: packages/locorda/example/lib/models/note_group_key.dart - `NoteGroupKey` hand-writes `operator==`/`hashCode` instead of extending `Equatable` (§4).
Deviation: packages/locorda/example/lib/utils/optional.dart - hand-rolled `Optional<T>` re-implements absence with hand-written equality instead of using the sanctioned `fpdart` `Option` (§4 "Value may be absent -> Option<T>", §11).
Deviation: packages/locorda_core/lib/src/locorda_graph_sync.dart (line 17) - `typedef IdentifiedGraph = (IriTerm id, RdfGraph graph);` is threaded through contracts as an opaque cargo tuple rather than discrete business parameters (§4 "business params, not cargo", §2 params).
Deviation: packages/locorda_core/ - no use-case layer exists; `locorda_graph_sync.dart` and `sync_engine.dart` mix entity definitions, datasource contracts and application orchestration in one package, and there is no `*_domain` / `*_usecases` / `*_datasource_*` split (§1 law 3, §3 package rules, §9 step 3).
Deviation: packages/locorda_core/lib/src/storage/storage_interface.dart - contracts (`Storage`, `RemoteStorage`, `Backend`, `ResourceLocator`) live in the mixed core package and no `I*Repository` contract exists at all, contrary to "contracts live in the domain and repos orchestrate, datasources do I/O" (§5, §3).
Deviation: packages/locorda/example/lib/storage/repositories.dart - `NoteRepository` / `CategoryRepository` are concrete classes that run drift queries and call the sync engine directly (lines 18, 152, 235-290); there is no repository contract and no datasource seam (§1 law 3, §5, §3).
Deviation: packages/locorda/lib/src/locorda_sync.dart - public methods take positional business inputs (`save<T>(T object)` line 189, `deleteDocument<T>(String id)` line 291, `ensure<T>(String id, {...})` line 253) against "named parameters everywhere, sole exceptions ref/message" (§2 Dart parameter style).
Deviation: packages/locorda_core/lib/src/locorda_graph_sync.dart - `configureGroupIndexSubscription(String indexName, RdfGraph groupKeyGraph, ItemFetchPolicy itemFetchPolicy)` is fully positional, with a `RdfGraph` passed where a business group-key param is meant (§4 business params, §2 named params).

### §2 Toolchain

Deviation: pubspec.yaml (root) and packages/*/pubspec.yaml - SDK constraints are `^3.6.0` (root, locorda_core, locorda_drift, locorda_solid, locorda_solid_auth, locorda_solid_ui, example) and `'>=3.5.0 <4.0.0'` (locorda, locorda_annotations); the bible pins `'>=3.10.0 <4.0.0'` in every pubspec (§2 SDK).
Deviation: melos.yaml - the repo orchestrates all packages with melos (`packages: packages/**`, workspace-wide `dependency_overrides`, the `analyze`/`test`/`format`/`lint` scripts) and the root pins `melos: ^6.1.0`; a `melos.yaml` is a bad smell (deleted in melos 7.0.0) and melos must never be the workspace orchestrator (§2 "Melos: optional — a script runner", §11).
Deviation: .github/workflows/ci.yml - CI runs `dart pub get` + `dart pub run melos bootstrap/analyze/test`; it never runs `dart analyze --fatal-infos --fatal-warnings`, so infos and warnings cannot fail the build (§2 "the gate is the fatal flags, not plain analyze", §9 step 10).
Deviation: pubspec.yaml (root) - there is no `workspace:` list, and no package declares `resolution: workspace` (grep: 0 hits); local linkage is `path:` dependencies plus seven committed `pubspec_overrides.yaml` files, i.e. exactly the mechanism pub workspaces replaced (§2 "Native pub workspaces"; §9 step 2/4).
Deviation: (repo root) - there is no root `analysis_options.yaml`; the only one in the tree is packages/locorda/example/analysis_options.yaml, so six of the eight packages analyze with no lint configuration, `public_member_api_docs` is off, and `todo: error` is not mapped (§2 Analyzer zero tolerance, §9 step 8, §10).
Deviation: packages/*/pubspec.yaml - `lints` is declared as a dev_dependency in every package but is never included by any `analysis_options.yaml` for those packages, so the declaration has no effect (§2 analyzer gate).
Deviation: (whole workspace test tree) - no `dart_arch_test` architecture/boundary test exists in any package; package-boundary direction and cycle-freedom are machine-unchecked (§2 "Package-boundary rules: dart_arch_test", §9 step 7, §10).
Deviation: packages/locorda_core/lib/src/locorda_graph_sync.dart, packages/locorda_core/lib/src/index/filesystem_safety.dart, packages/locorda_core/lib/src/storage/storage_interface.dart, packages/locorda_core/lib/locorda_core.dart - fail `dart format --set-exit-if-changed` (verified: 3 files change), so `dart format` is not clean in core (§2 Formatter).
Deviation: packages/locorda_solid_auth/lib/src/ui/login_page.dart, packages/locorda_solid_auth/lib/src/ui/solid_status_widget.dart - fail `dart format --set-exit-if-changed` (verified: 2 files change) (§2 Formatter).
Deviation: packages/locorda/example/lib/init_rdf_mapper.g.dart, packages/locorda/example/lib/models/{category,note,note_index_entry}.rdf_mapper.g.dart - 4 checked-in generated files also fail `dart format --set-exit-if-changed` (§2 Formatter).
Deviation: 28 files under packages/*/lib (excluding the 7 barrels) - open with a file-level `///` doc comment plus a `library;` directive (e.g. locorda_core/lib/src/hydration_result.dart:1-2, locorda_core/lib/src/storage/storage_interface.dart:1-5, locorda_core/lib/src/sync/sync_engine.dart:1-5, locorda_core/lib/src/locorda_graph_sync.dart:1-2, locorda/lib/src/locorda_sync.dart:1-2, locorda_annotations/lib/src/pod_resource.dart:1-2 …); the bible forbids file-header `///` and reserves `library;` for barrels (§2 Docs, §10).
Deviation: packages/locorda_core/lib/src/locorda_graph_sync.dart:27,137,161,237,281,310,409 and packages/locorda/example/lib/screens/notes_list_screen.dart:126,372 - nine `TODO:` comments in shipped lib code; TODOs are diagnostics (`todo: error`) and deferred work belongs in the roadmap doc (§2 Analyzer, §10, §11 Roadmap).
Deviation: packages/locorda_solid_auth/lib/l10n/solid_auth_localizations*.dart and packages/locorda/example/lib/main.dart:292,295,299 - `// ignore_for_file: type=lint` (ignore_for_file is banned outright) and bare `// ignore:` lines with no reason (§2 Suppression: per-line or nothing).
Deviation: packages/locorda_core/lib/src/locorda_graph_sync.dart:26 - `// ignore: unused_field` with no recorded reason (§2 Suppression).
Deviation: 16 files under packages/*/lib declare 2-9 public classes each (locorda_drift/lib/src/sync_database.dart 9; locorda_core/lib/src/index/index_config_base.dart 7; locorda_core/lib/src/config/sync_graph_config.dart 6; locorda/lib/src/config/sync_config.dart 6; locorda_core/lib/src/config/validation.dart 5; locorda_annotations/lib/src/crdt_annotations.dart 4; locorda_solid_auth/lib/src/providers/solid_provider_service.dart 3; locorda_core/lib/src/{storage/storage_interface,crdt/crdt_types}.dart 3; …) against one-class-per-file (§2 one-class-per-file discipline, §3 package rules).
Deviation: packages/locorda_solid_ui/lib/locorda_ui.dart - the barrel is not named after the package (`locorda_solid_ui.dart` is expected), so "one public door per package" naming is broken (§2 Barrel files).
Deviation: packages/locorda_core/lib/src/locorda_graph_sync.dart:57-123 and :177-226 - `///` blocks of 60+ lines with bullet lists, lifecycle prose and code examples hang off a single method; declaration docs must be 1-2 lines what+why (§2 Docs, §10).
Deviation: packages/locorda_annotations/lib/src/pod_resource.dart:49-119 - a ~70-line class doc comment with a usage example; same 1-2 line rule (§2 Docs, §10).
Deviation: packages/locorda/README.md - embeds a complete 30-line annotated model as documentation instead of pointing at tests/examples; example code must live in tests or `examples/`, never in prose (§2 Code placement, §1 D.R.Y.).
Deviation: packages/locorda_drift/README.md - publishes a "Database Schema" block (`rdf_documents`, `rdf_triples`, `crdt_metadata` tables) and names a `LocalStorage` interface that do not exist in the code (real tables are `SyncIris`/`SyncDocuments`/`SyncPropertyChanges`; the real interface is `Storage`); a second copy of a truth that has already drifted (§1 D.R.Y., §2 Code placement).
Deviation: packages/locorda_annotations/lib/src/pod_resource.dart:49-119 - the documented example API (`@SolidPodResource()`, `class Note extends RdfResource`, `@LwwRegister()`, `@Immutable()`, `late` fields) does not exist in this codebase (the annotation is `@PodResource`, the CRDT annotations are `@CrdtLwwRegister`/`@CrdtImmutable`, and the pattern is a plain class with `@RdfIriPart`); docs restate a wrong truth (§1 D.R.Y., §10).
Deviation: packages/locorda/lib/locorda.dart:36 - the barrel re-exports another package's barrel (`export 'package:locorda_core/locorda_core.dart' show HydrationSubscription;`) instead of only re-exporting its own `lib/src/` (§2 Barrel files).
Deviation: packages/locorda_core/lib/src/locorda_graph_sync.dart:6,7 - a `lib/src/` file imports its own package barrel (`package:locorda_core/locorda_core.dart`) and its own `package:locorda_core/src/...` paths; `src/` is private and the barrel is the only public door (§2 Barrel files).
Deviation: packages/locorda/lib/src/config/sync_config.dart:3 - a `lib/src/` file re-exports `package:locorda_core/locorda_core.dart` (show list), making a src file a cross-package door (§2 Barrel files).
Deviation: packages/locorda_annotations/lib/src/{pod_resource,pod_resource_ref}.dart - `lib/src/` files import their own barrel `package:locorda_annotations/locorda_annotations.dart` (self-import) (§2 Barrel files).
Deviation: (repo root) - no `AGENTS.md` and no `BACKLOG.md` exist at `fceb240`; the repo has nowhere recording its own deviations and local wiring, which is exactly what §1 D.R.Y. prescribes for a repo's AGENTS.md (§1 D.R.Y., §4 error-style declaration, §10).

### §3 Topology

Deviation: (repo root) - the topology is neither sanctioned shape: the one workspace repo holds the pure-Dart-ish core packages **and** the Flutter packages (`locorda_drift`, `locorda_solid_auth`, `locorda_solid_ui`) and the example app, so it is not Topology A (single delivery mechanism) nor Topology B (pure-Dart core repo + separate Flutter UI repo with a git dep); the core cannot be treated as a headless pure-Dart workspace (§3).
Deviation: packages/locorda/lib/src/locorda_sync.dart - the application layer/facade lives in the *Flutter* package `locorda`, not in a usecases package, so the shared facade is not consumable by a CLI/TUI sibling (§3 "the application layer (use cases) IS the facade").
Deviation: packages/locorda_solid_ui/pubspec.yaml + packages/locorda_drift/pubspec.yaml - platform-specific concerns (`path_provider`, `sqlite3_flutter_libs`, `drift_flutter`) are resolved inside workspace packages rather than in a UI composition root, and no `*_datasource_memory` package exists (§3 Topology B rules, §9 step 3).
Deviation: packages/locorda_drift/ + packages/locorda_core/ - only one `Storage` implementation exists (`DriftStorage`); there is no second adapter and no shared contract suite run against all adapters, so the at-least-two-adapter rule is unmet (§1 "the one we never skip", §5 in practice, §6 contract row).
Deviation: packages/locorda/example/lib/storage/database.dart + repositories.dart - the "datasource" is drift DAOs inside the app package, with repository logic, model mapping and sync coordination in the same classes; the domain-contract / datasource split does not exist (§5, §3).

### §5 Persistence

Deviation: packages/locorda_core/lib/src/storage/storage_interface.dart - no `IUnitOfWork? uow` parameter on any write method and no `IUnitOfWork` contract at all; `DriftStorage.saveDocument()` opens `_database.transaction(...)` privately, so callers cannot join an atomic unit (§5 UnitOfWork).
Deviation: packages/locorda_drift/lib/src/drift_storage.dart - the adapter exposes no transaction seam and its `close()` is gated on mutable `_initialized`, so "keep the native client on the txn for the whole `work()` run" cannot even be expressed (§5 UnitOfWork pitfalls).
Deviation: packages/locorda_drift/README.md - documents a schema and a `LocalStorage` interface that are not the code's (see §2 items); persistence documentation is a stale second truth (§5, §1 D.R.Y.).

### §6 Testing

Deviation: packages/locorda/{test}, packages/locorda_core/test, packages/locorda_drift/test - 518 `expect(...)` assertions (159 + 279 + 80) and zero `shouldly` usages; shouldly is the mandated assertion idiom and `expect()` may not be mixed in (§6).
Deviation: packages/locorda_core/test/**, packages/locorda/test/**, packages/locorda_drift/test/** - test names are descriptive/imperative ("generates simple group key from single property", "should fail with duplicate type IRIs", "basic functionality") rather than Given/When/Then groups (§6 Test names).
Deviation: packages/locorda/example/test/services/mock_note_repository.dart, mock_category_repository.dart, mock_solid_crdt_sync.dart - hand-rolled mock classes instead of `mocktail`, and they mock app-owned concrete classes rather than an own interface at the use-case seam (§6 Mocktail).
Deviation: packages/locorda/example/test/widget_test.dart - the widget test constructs `SolidAuth` and mock repositories/services by hand and asserts with `expect(...)`, with no provider container and no shouldly (§6 UI ring row).
Deviation: (all test dirs) - no test asserts both sides of an `Either` (there is no Either anywhere); failure paths are expressed as "throws something" expectations (§6 Coverage of failure paths).
Deviation: packages/locorda_drift/test/test_sync_database.dart - `TestSyncDatabase` (an in-memory drift database) lives inside the test tree of the datasource package; the second adapter that doubles as the test double is missing, so the in-memory double exists only as test scaffolding (§6 layer matrix, §5 two-adapter rule) — note the drift test technique itself (`NativeDatabase.memory`) conforms.

### §7 Builders / codegen

Deviation: packages/locorda/example/pubspec.yaml - dev_dependencies include `rdf_mapper_generator: ^0.10.5` with `build_runner`, and the repo commits its output (`lib/init_rdf_mapper.g.dart`, `lib/models/*.rdf_mapper.g.dart`); that is an unapproved serialization builder beyond drift (§7 table, §2 BANNED, §9).
Deviation: packages/locorda/pubspec.yaml - `build_runner: ^2.4.13` is a dev_dependency of the facade package although the package has no drift codegen of its own (§7 "build_runner + drift_dev — the exception" only where drift earns it).
Deviation: packages/locorda_annotations/ - the package exists solely to feed a code generator (`locorda_generator` is referenced by melos.yaml/CLAUDE.md but is absent from the tree), building the ecosystem around generated merge logic that §7 says to avoid (§7).
Deviation: packages/locorda_drift/pubspec.yaml - drift + drift_dev + build_runner are the only sanctioned builder, which conforms; the deviation is that the generated `sync_database.g.dart` (2039 lines) plus the example's generated drift code are checked in unformatted (§2 Formatter, §7 hygiene).
Deviation: packages/locorda_solid_auth/lib/l10n/*.dart - three generated localisation blobs are committed under `lib/` (and the build breaks with 13 analyzer errors when they are missing); generated code should be produced by the build, not carried in `lib/` (§7, §2 code placement).

### §8 Flutter ring

Deviation: packages/locorda/example/pubspec.yaml - `flutter_riverpod` is absent from the workspace (grep: 0 hits) and no providers exist; state is threaded through constructor-injected mutable service objects, against the settled "Riverpod with plain providers" doctrine (§8 State management, §11).
Deviation: packages/locorda/example/lib/main.dart:105 + screens - `go_router` is absent and navigation is imperative `Navigator.push`/`pop` inside screens (§8, §11 nav decision).
Deviation: packages/locorda/example/lib/screens/{notes_list_screen,categories_screen,note_editor_screen}.dart - widgets import and construct entities (`../models/note.dart`, `../models/category.dart`, `../models/note_index_entry.dart`, `../models/note_group_key.dart`) and receive services by constructor, so widgets see models and services rather than use cases through providers (§8 "The only seam: use cases").
Deviation: packages/locorda_solid_ui/lib/src/sync/sync_status_widget.dart - the widget takes a core `SyncEngine` and calls `widget.syncEngine.syncAll()` itself (line 128) and renders `error.toString()` (line 136); a widget reaching an engine is not the use-case seam, and the error text is derived from an exception rather than a failure type (§8, §4 "domain failures never carry display strings").
Deviation: packages/locorda_solid_auth/lib/src/ui/login_page.dart:58-67 - `catch (e, stackTrace)` converts a raw exception into widget state (§8 exceptions may be caught here, but the state must be derived from typed failures).
Deviation: packages/locorda_solid_auth/lib/src/ui/solid_status_widget.dart:73-122 - three `try/catch (e, stackTrace)` blocks turn exceptions into UI strings and a status flag (§8, §4).
Deviation: packages/locorda_solid_auth/lib/src/providers/solid_provider_service.dart - an "auth provider service" in the UI package stands in for the missing use-case/providers layer (§8).
Deviation: packages/locorda/example/lib/services/{notes_service,categories_service}.dart - the app's application layer is a pair of mutable services built on `rxdart` `BehaviorSubject`/`switchMap`, not use cases + plain Riverpod providers; `rxdart` is also outside the pinned stack (§8, §2 STACK).
Deviation: packages/locorda_solid_auth/pubspec.yaml and packages/locorda/example/pubspec.yaml - `solid_auth` is consumed as a git dependency pinned to a personal feature branch (`kkalass/solid_auth#feat/migrate-to-bdaya-oidc-security-fix`) plus a `dependency_overrides` git ref for `oidc_core` inside pubspec files; §3 sanctions a git dep only for the cross-repo core→UI split (§3, §2).
Deviation: packages/locorda/example/analysis_options.yaml - the app's analyzer config only includes `flutter_lints` and a handful of style rules; no `public_member_api_docs`, no `todo: error`, and it disables severity hygiene by excluding platform dirs (§2 analyzer gate).

### §9 / §10 conformance

Deviation: (workspace) - bootstrap checklist steps 2, 3, 6, 7, 8, 9 and 10 are unmet: no root `workspace:` list, no `*_domain`/`*_usecases`/`*_datasource_*` packages, no second adapter or contract suite, no `dart_arch_test` boundary test, no root lint config with `public_member_api_docs` + `todo: error`, no documented use case with a both-sides test, and no fatal-flag analyze gate in CI (§9).
Deviation: (workspace) - the review checklist fails on at least these rows: inward dependencies unverified, `throw`/`try`/`catch` inside core, entities not immutable/equatable, no ≥2 adapters + contract suite, failures not typed/mapped, error style not declared, unapproved codegen (`rdf_mapper_generator`), tests not shouldly/no GWT names/hand-rolled mocks, `///` file headers present, params not business-shaped, barrel naming wrong, no UnitOfWork, example code in READMEs, rules restated in READMEs/AGENTS, `melos.yaml` present, format not clean, TODOs in code, `ignore_for_file` used (§10).

## Environment note (Windows lane, not a repo bug)

### E1 — pub invocations must see a native `PUB_CACHE` path

With `PUB_CACHE` set to an MSYS path (`/c/Users/.../Pub/Cache`), pub builds the
git-mirror target as `/c/Users/.../Cache\_temp\dirXXXX` and the `oidc_core` git
override clone fails:

```
fatal: destination path ... already exists and is not an empty directory
```

`flutter pub get` then exits 69. Unsetting `PUB_CACHE` (native default
`C:\Users\<user>\AppData\Local\Pub\Cache`) makes every `pub get` exit 0. All
baseline results above were captured with `PUB_CACHE` unset.

## Verified clean in this pass

Recorded so the open items above are not over-read as a broken build:

- With the committed generated l10n files present, `pub get` is exit 0 for the
  root, all 7 packages and the example — no broken dependency.
- `analyze` is exit 0 with "No issues found!" in 7 of 8 packages and the example;
  the single finding is B1.
- All test suites pass: `locorda` 69, `locorda_core` 148, `locorda_drift` 32,
  `example` 11 — 260 tests, 0 failures, 0 skips.
- No build error and no broken dependency was found on the Windows lane.
- `dart pub publish --dry-run` exit 65 concerns `locorda_annotations` packaging
  metadata (B3–B9), not a compile or test failure.
- The deviation list is about standards conformance: the monorepo compiles,
  analyzes and passes its full suite at the audited commit.

## Open decisions for the maintainer

1. Is `locorda_annotations` published (B1–B9) or workspace-only?
2. Native pub workspaces instead of `melos.yaml` + `path:` overrides + committed
   `pubspec_overrides.yaml` (§2 deviations, B8, B14) — yes/no, and when?
3. Where does the repo record its own conventions: add `AGENTS.md` (B12)?
4. Root `CHANGELOG.md` is hand-written here, but `melos.yaml` has
   `workspaceChangelog: true`; decide whether melos owns the root changelog or
   this hand-written one does (see `CHANGELOG.md`).
