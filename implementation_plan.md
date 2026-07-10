# Phase 1 Implementation Plan

## 0. Finalized Architectural Decisions

These decisions were made after reviewing the four spec documents and supersede any conflicting statements in them:

1. **Bucketing hash:** MurmurHash3 (x86 32-bit variant), vendored as a reference implementation with published test vectors - no third-party dependency. Bucket = `murmur3("userId:featureKey") % 100`. The colon separator prevents concatenation collisions. This choice is permanent; changing it later reshuffles every user's bucket.
2. **Reactivity - selective freeze (engine spec Strategy 3):** Kill-switch transitions apply live mid-session and reactively tear down features. All other config changes (rollout percentages, targeting rules) are validated and persisted to disk but frozen for the current session, taking effect on next launch. This prevents mid-session UI layout shifts while preserving instant emergency response.
3. **Disk persistence is in Phase 1 scope:** Fetched configs are schema-validated, then saved as a Last-Known-Good (LKG) cache. Boot order: const in-code defaults -> hydrate from LKG cache -> background fetch.
4. **Config propagation - pull-based with push-triggered pulls:** Fetch on cold start and app foregrounding (via `WidgetsBindingObserver`). Instant kill-switches are modeled as a silent-push trigger that invokes the same `refresh()` path. Phase 1 simulates the silent push from the developer menu; real FCM/APNs ships later as an optional companion package so the core stays dependency-free.
5. **Distribution - two separate repositories:**
   - A pure Dart engine SDK package, published to pub.dev first.
   - A Flutter demo client app, consuming the published package as a normal pub dependency.
6. **Process - strict TDD, phased commits:** roughly 15 commits total (about 5 engine, 10 app) as a guideline, not a hard number - adjust as the work requires. Each commit is a completed red-green-refactor cycle with its tests included. Architectural docs are committed alongside the code.
7. **State management:** Riverpod with the modern `Notifier` API (not the deprecated `StateNotifierProvider` shown in the Flutter spec).
8. **Open-source posture:** minimal dependencies, ecosystem conventions, cross-language portability of the engine spec.

---

## 1. The Split: What Lives Where

### Repo 1 - Engine SDK (pure Dart, zero dependencies, published to pub.dev)

Everything platform-agnostic:

- Vendored MurmurHash3 and deterministic bucketing
- Config domain models with strict JSON validation
- The evaluation hierarchy (kill-switch > targeting > rollout > fallback) with `EvaluationResult` explainability
- A `ConfigSessionController` implementing selective freeze
- Two abstract interfaces with no implementations shipped:
  - `ConfigFetcher` - where config comes from
  - `ConfigStore` - where the LKG cache lives

Keeping implementations out means no Flutter or network dependencies. The package is usable server-side as well, which is the strongest open-source positioning.

### Repo 2 - Demo Client App (Flutter)

Consumes the published package like a real user would. Provides the platform bindings:

- `MockConfigRepository` implementing `ConfigFetcher` (Config A/B, artificial latency)
- SharedPreferences-backed `ConfigStore` (LKG cache)
- Refresh triggers: cold start, foregrounding, simulated silent push
- Riverpod glue: `featureFlagProvider` family, evaluation-result providers
- Gated demo feature via component substitution
- Developer menu with explainability

This boundary is what makes the pub.dev friction survivable: the engine is fully built and tested before publishing, so the app should not force engine changes. If a gap surfaces anyway, bump to 0.1.1 - pub.dev versions are immutable.

---

## 2. TDD Workflow (applies to every feature commit)

- Each commit is one or more completed red-green-refactor cycles: tests written first against the public API, implementation until green, then refactor.
- Every feature commit contains its tests; no commit lands with failing or missing tests.
- Both repos get a GitHub Actions workflow running `analyze` + `test` on every push, added in commit 1 so the discipline is enforced from the start.

---

## 3. Engine Repo - 5 Commits

1. **`chore: scaffold package`**
   Pubspec (SDK constraint only, no dependencies), strict `analysis_options`, MIT LICENSE, README stub, CI workflow, and `docs/` containing the engine architecture spec updated to reflect the finalized decisions (MurmurHash3, selective freeze).

2. **`feat: deterministic bucketing via vendored MurmurHash3`**
   TDD starts with published MurmurHash3 x86 32-bit test vectors, then `getRolloutBucket(userId, featureKey)`. Tests: stability across calls, the `userId:featureKey` separator, roughly uniform distribution across 100 buckets, and known user/bucket pairs pinned as regression tests (these exact values must never change).

3. **`feat: config models and strict validation`**
   `RemoteConfig`, `FeatureConfig`, `TargetingRules`, `UserContext`, `EvaluationResult` + `EvaluationReason`. TDD on JSON parsing: valid configs parse; truncated, malformed, or wrong-typed configs are rejected with typed errors and never partially applied (the corruption edge case from the comprehensive PRD).

4. **`feat: evaluation hierarchy with explainability`**
   Kill-switch > targeting (semver `minAppVersion`, `allowedCountries`) > percentage rollout > fallback. TDD with a full decision matrix: every reason code reachable, correct precedence when rules conflict, 0%/100% short-circuits, missing-feature fallback, debug messages verified.

5. **`feat: session controller (selective freeze) + v0.1.0 release`**
   `ConfigSessionController` over abstract `ConfigFetcher`/`ConfigStore`: boot hydration (defaults -> LKG cache), refresh with validate-then-persist, freezing non-kill-switch changes for the session, live kill-switch overlay stream, `updateUserContext` with memoization flush. Plus dartdoc on the public API, `example/`, CHANGELOG, and publish to pub.dev.

---

## 4. Client App Repo - 10 Commits

1. **`chore: scaffold Flutter app`**
   Depends on the published engine package + `flutter_riverpod` + `shared_preferences`, strict lints, CI, `docs/` with the PRD and Flutter SDK architecture spec (updated to the finalized decisions and modern Riverpod API).

2. **`feat: feature registry and baked-in defaults`**
   `FeatureKey` enum, `const` default config (synchronous frame-0 availability), the "add a feature in three lines" pattern documented. TDD: defaults resolve when nothing else is available.

3. **`feat: mock config repository`**
   Implements `ConfigFetcher`. Config A (newCheckout 50% rollout, kill-switch off) and Config B (kill-switch on), ~800ms artificial latency, switchable environment. TDD on fetch behavior and payloads.

4. **`feat: SharedPreferences LKG store`**
   Implements `ConfigStore`. TDD with mocked prefs: round-trip persistence, hydration on boot, corrupted cache falls back to defaults without crashing.

5. **`feat: Riverpod integration`**
   `Notifier`-based config state wrapping the session controller, `featureFlagProvider` family, `evaluationResultProvider` for the dev menu. Provider unit tests: watching widgets rebuild on kill-switch overlay, do not rebuild on frozen changes.

6. **`feat: refresh triggers and user context switching`**
   Trigger abstraction feeding one `refresh()` path: cold start, `WidgetsBindingObserver` foreground resume, simulated silent push. `updateUserContext` re-evaluation flow. TDD on each trigger firing exactly one refresh.

7. **`feat: demo UI with gated feature`**
   Checkout screen using component substitution (`NewCheckoutWidget` vs `LegacyCheckoutWidget`, one `ref.watch` line), User A / User B profile switcher showing different bucket outcomes. Widget tests.

8. **`feat: developer menu`**
   All flags with their `EvaluationResult` explanations (why each decision was made), Config A/B environment switch, "simulate silent push" button, current user bucket display. Widget tests.

9. **`test: end-to-end scenarios`**
   Integration-style widget tests proving the headline behaviors: boot on defaults -> hydrate -> fetch; kill-switch tears down the feature mid-session; rollout percentage changes freeze until "next launch"; determinism across simulated restarts; logout/login bucket change.

10. **`docs: finalize`**
    README with architecture overview, screenshots/GIF of the kill-switch teardown, usage guide, and any doc updates discovered during implementation.

---

## 5. Publishing Mechanics (before engine commit 5)

- **Package name:** must be unique on pub.dev and snake_case. Common names like `remote_config` are likely taken; pick something distinctive (e.g. `forge_flags`, `rollout_engine`, or a brand name). The name goes in the pubspec at commit 1, so it must be chosen before scaffolding. Check availability on pub.dev first.
- **Account:** publishing runs `dart pub publish`, which authenticates via a Google account in the browser on first use. This step is interactive and must be run by the maintainer.
- **Score hygiene:** pub.dev scores packages on documentation, example, analysis cleanliness, and platform support - engine commit 5 covers all of it.

---

## 6. Risks and Notes

- **Commit budget is a guideline:** the 5 + 10 split keeps commits meaningful and phase-shaped, but split or merge commits wherever quality demands it. The likeliest split is engine #5 (controller + release prep together) - prefer an extra commit over shipping a weak v0.1.0.
- **Version immutability:** pub.dev versions cannot be overwritten. Get the engine solid before publishing; patch releases (0.1.x) cover any gaps discovered during app development.
- **Hash permanence:** the pinned bucket regression tests in engine commit 2 are the guardrail - if they ever fail, determinism has been broken and the change must not ship.
