# Product Requirements Document: Remote-Config & Staged-Rollout Platform

## 1. Overview & Goals
The goal is to build a Flutter application that serves as a foundational platform for feature experimentation, remote configuration, and staged rollouts. Rather than focusing on a finished consumer product, the objective is to design a robust, scalable architecture that can support multiple teams. 

Key architectural goals:
* **Safe Experimentation:** Deterministic, stable rollouts with instant kill-switches.
* **Developer Ergonomics:** Adding a gated feature should be trivial (config + boolean check) without major code restructuring.
* **Separation of Concerns:** The evaluation engine must be isolated from the UI and business logic.

## 2. Core Concepts & Architecture (The Thought Process)

To make this a platform many apps and teams depend on, we need to think about the following pillars:

1. **Deterministic Evaluation Engine:**
   A percentage rollout (e.g., 20%) shouldn't randomly evaluate on every check. It must evaluate consistently based on the user's identity. The industry standard approach is to compute a hash of `(userId + featureFlagKey)`, modulo 100, and check if it falls within the rollout percentage. This guarantees that User A always sees the feature if they are in the bucket, and never sees it if they aren't, across sessions and devices.

2. **The Hierarchy of Resolution:**
   Feature flags resolve based on a strict priority hierarchy:
   1. **Kill-Switch:** Immediate override. If `killSwitch: true`, feature is OFF regardless of other rules.
   2. **Targeting Rules (Phase 2):** Explicit user attributes (e.g., country, app version).
   3. **Percentage Rollout:** If no specific target matches, fall back to the rollout bucket.
   4. **Default/Fallback:** If the remote config is unreachable, fall back to a baked-in default.

3. **Session Stability (Sync vs Async):**
   If the app starts with baked-in defaults while fetching the remote config asynchronously, applying the new config *immediately* upon receipt could cause the UI to shift unexpectedly while the user is interacting with it. A robust platform usually fetches in the background and either:
   * Applies the new config on the *next* app launch.
   * Emits a state change that the UI can gracefully listen to (e.g., showing a "refresh to apply new features" banner).

4. **Integration Ergonomics:**
   We will design a `FeatureService` that exposes a simple `isEnabled(FeatureKey)` method. UI widgets can listen to this service using an inherited widget or state management solution (e.g., Riverpod or Provider), ensuring that flipping a config reactively (or silently) flips the feature.

## 3. Feature Breakdown

### Must Have (Phase 1)
* **Remote-Config Object (Mocked):** A simulated network fetch of a JSON config.
* **Deterministic Percentage Rollout:** Hash-based evaluation guaranteeing stable feature exposure for a given user.
* **Instant Kill-Switch:** A high-priority boolean flag per feature to disable it immediately.
* **Multiple Config Profiles:** Support for simulating "User A" and "User B" receiving different configurations and thereby experiencing different feature sets.
* **Frictionless Feature Addition:** A clean pattern for adding new features (config schema + `FeatureKey` enum + `if(isEnabled)` check).
* **Sync vs Async Evaluation Handling:** The app boots instantly using a baked-in default config, fetches the mock remote config asynchronously, and handles the transition.

### Should Have (Phase 2)
* **Developer Menu:** An in-app debug screen to view current evaluated flags, override flags locally, and understand *why* a decision was made (explainability).
* **Targeting Complexity:** Support for rules based on user attributes (e.g., Country = "US", AppVersion > "1.2.0").
* **Telemetry / Exposure Logging (Mocked):** Logging when a feature flag is evaluated to measure experiment impact.

### Future Scope (Phase 3+)
* **A/B/n Multivariate Testing:** Support for multiple variants (e.g., string/JSON payloads) rather than just booleans.
* **Dynamic Configuration:** Updating themes, copy, and structural values directly from the config.
* **Backend Integration:** Connect to a real service like Firebase Remote Config, LaunchDarkly, or a custom backend.

## 4. Finalized Design Decisions & Implementation Strategy

Based on review, we have established the following strict architectural guidelines:

### 1. Session Stability Strategy: Hybrid Approach
* **Production Standard vs. Exercise Needs:** In a true production environment, UI layout shifting mid-session is a cardinal sin. We would typically fetch in the background, cache to local storage, and apply it on the next app cold start. However, to demonstrate the "instant kill-switch" capability requested, the UI must react to config changes in real-time.
* **Implementation:** We will use a reactive state management solution (`flutter_riverpod`) so the UI rebuilds when the config updates.
> [!WARNING]
> **Production Note:** In a real-world scenario, we would freeze the config state at launch for standard rollouts to prevent mid-session UI shifts, reserving real-time reactive updates *only* for emergency kill-switches.

### 2. Deterministic Hashing: Stable Cross-Session Hashes
* **The Trap:** Dart's built-in `String.hashCode` randomizes per execution/isolate for security. Using it would cause a user's bucket to change every time they restart the app, completely breaking determinism.
* **Implementation:** We must use a stable hashing algorithm. Finalized decision (see implementation plan, section 0): MurmurHash3 (x86 32-bit variant), vendored as a reference implementation with published test vectors - no third-party dependency. Bucket = `murmur3("userId:featureKey") % 100`. The colon separator prevents concatenation collisions, and this choice is permanent - changing it later would reshuffle every user's bucket. (An earlier draft proposed MD5 via the `crypto` package; it was superseded to keep the engine dependency-free and aligned with industry SDKs like LaunchDarkly and Unleash.)

### 3. Fallback Defaults: In-Code (Dart Map/Enum)
* **The Rationale:** Loading a JSON file from the `rootBundle` in Flutter is an asynchronous I/O operation. Relying on an asset for defaults means the app cannot render its first frame synchronously and would hang on a splash screen.
* **Implementation:** We will define fallback defaults as a `const` Dart Map or Class. This guarantees zero-latency, synchronous availability at frame 0.

### 4. Explainability Level: EvaluationResult Wrapper
* **The Rationale:** A simple boolean is impossible to debug at scale. Developers need to know *why* a flag returned false (e.g., kill-switch, out of bucket, network failure).
* **Implementation:** The core evaluation engine will return an `EvaluationResult` object containing the boolean value and a `Reason` enum.
* To maintain integration ergonomics, the `FeatureService` will expose two methods:
  * `bool isEnabled(key)`: Strips metadata for simple, frictionless UI `if` checks.
  * `EvaluationResult evaluate(key)`: Used by the Developer Menu and mock telemetry logger to explain the decision.
