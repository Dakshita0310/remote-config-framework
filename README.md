# Remote-Config & Staged-Rollout Framework - Design Documents

This repository is the complete design and thought-process trail behind
[`feature_flag_kit`](https://pub.dev/packages/feature_flag_kit) (a
platform-agnostic feature flag and staged-rollout engine for Dart, published
on pub.dev) and
[`feature_flag_kit_demo`](https://github.com/Dakshita0310/feature_flag_kit_demo)
(the Flutter client that showcases it).

It intentionally preserves how the design evolved: the questions asked, the
alternatives weighed, the traps caught, and the decisions finalized - not just
the end state.

## Reading order

1. **[prd.md](prd.md)** - the product requirements: goals, evaluation
   hierarchy, phase-wise feature breakdown, and the first round of design
   decisions (session stability, stable hashing, in-code fallbacks,
   explainability).
2. **[comprehensive_platform_prd.md](comprehensive_platform_prd.md)** - the
   platform-scale view: user personas, non-functional requirements (P99 < 1ms
   evaluation, zero network dependency for first frame), edge cases (corrupted
   payloads, mid-session logout, clock drift), and the pull-vs-push
   kill-switch tradeoff matrix.
3. **[engine_architecture_spec.md](engine_architecture_spec.md)** - the core
   engine design: deterministic hashing + modulo bucketing, avoiding sticky
   cohorts, the kill-switch > targeting > rollout > fallback hierarchy, the
   three bootstrapping strategies, and `EvaluationResult` explainability.
4. **[flutter_sdk_architecture_spec.md](flutter_sdk_architecture_spec.md)** -
   the client-side design: SDK public interface, Riverpod integration, the
   flag-debt clean-up strategy (component substitution, sealed classes), and
   the two-config mock setup.
5. **[implementation_plan.md](implementation_plan.md)** - the finalized build
   plan. **Section 0 records the final architectural decisions and supersedes
   any conflicting statements in the earlier documents.**

## The key decisions and why

- **Deterministic bucketing, permanent by design.** A user's rollout bucket is
  `murmur3_x86_32("userId:featureKey") % 100` - stable across sessions and
  devices, independent per feature (no sticky cohorts where the same users get
  every alpha feature). MurmurHash3 was chosen over Dart's `String.hashCode`
  (randomized per isolate - would break determinism entirely) and over an
  earlier MD5-via-`crypto` draft (superseded to keep the engine
  dependency-free and aligned with LaunchDarkly, Unleash, and GrowthBook).
  Pinned regression vectors guard the mapping forever.
- **Selective freeze (the hybrid reactivity compromise).** The app boots
  instantly on baked-in defaults, hydrates from a Last-Known-Good cache, and
  fetches in the background. Non-emergency changes (rollout percentages,
  targeting) are persisted but frozen for the session - no mid-session UI
  shifts. Kill-switch activations apply live and tear the feature down
  immediately.
- **Pull-based delivery with push-triggered pulls.** A pure WebSocket push
  architecture is battery-hostile and expensive at scale; pure polling makes
  kill-switches eventual. The chosen hybrid: CDN-cacheable pulls on cold start
  and foregrounding, plus silent push (FCM/APNs) that wakes the app to run the
  same pull path when an emergency kill-switch fires.
- **Explainability as a first-class API.** Evaluation never returns a bare
  boolean; every decision carries an `EvaluationResult` with a reason code and
  debug message, powering the demo app's developer menu.
- **Engine/app split for open source.** A pure Dart engine with zero runtime
  dependencies (fetching and persistence are abstract interfaces) published to
  pub.dev, consumed by a separate Flutter demo app like any real user would.

## Where it landed

| Artifact | Link |
| :--- | :--- |
| Engine package (pub.dev) | https://pub.dev/packages/feature_flag_kit |
| Engine source | https://github.com/Dakshita0310/feature_flag_kit |
| Flutter demo client | https://github.com/Dakshita0310/feature_flag_kit_demo |

The specs in the shipped repositories
([engine spec](https://github.com/Dakshita0310/feature_flag_kit/blob/main/doc/engine_architecture_spec.md),
[app spec](https://github.com/Dakshita0310/feature_flag_kit_demo/blob/main/docs/flutter_app_architecture_spec.md))
are the updated, as-built versions; the copies here preserve the original
design-time reasoning.
