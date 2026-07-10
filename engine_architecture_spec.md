# Architectural Specification: Core Evaluation Engine

## Overview
This document outlines the design for a platform-agnostic feature flag and staged-rollout evaluation engine. The engine guarantees deterministic exposure, strict evaluation hierarchies (including emergency kill-switches), optimized mobile bootstrapping, and robust debugging capabilities.

---

## 1. Deterministic Targeting Engine

### The Problem with Random Assignment
If we use a standard random number generator (e.g., `Math.random() < 0.2` for a 20% rollout) at evaluation time, a user might see a feature on one app launch but lose it on the next, or see it on their phone but not their tablet. This destroys user trust and corrupts telemetry data for A/B testing. We need a system where a user's assignment is stable across time and devices.

### Hashing + Modulo
To achieve determinism without storing a server-side assignment database, we use a cryptographic or non-cryptographic hash function (like SHA-256 or MurmurHash3) to convert a string payload into a uniformly distributed integer, then take the modulo 100.

**Avoiding "Sticky Bucketing"**
If we only hash the `userId`, a user who hashes to bucket `1` will *always* be in the first 1% rollout of *every single feature*. This creates "sticky cohorts" where the same users get all alpha features simultaneously, skewing metric correlations. 
To solve this, we concatenate the `userId` with the `featureKey`. This guarantees that the hash output is unique per feature for the same user. User A might be in bucket `5` for `feature_checkout`, but bucket `82` for `feature_promo`.

### Pseudocode Implementation
```typescript
import { createHash } from 'crypto';

function getRolloutBucket(userId: string, featureKey: string): number {
    // 1. Concatenate inputs to ensure distribution per-feature
    const payload = `${userId}:${featureKey}`;
    
    // 2. Generate a stable hash. SHA-256 is ubiquitous, MurmurHash3 is faster.
    const hashHex = createHash('sha256').update(payload).digest('hex');
    
    // 3. Take the first 8 characters (32 bits) to avoid integer overflow
    const hashInt = parseInt(hashHex.substring(0, 8), 16);
    
    // 4. Modulo 100 to get a bucket between 0 and 99
    return hashInt % 100;
}
```

---

## 2. The Kill-Switch vs. Rollout Hierarchy

Feature flags are evaluated top-down. Emergency constraints must override targeting, and targeting must override pure percentage rollouts.

### The Evaluation Hierarchy
1. **Kill-Switch (Global Override):** If the flag is hard-disabled globally, evaluation stops immediately. (Returns `false`).
2. **Targeting Rules (Exclusion/Inclusion):** If rules exist (e.g., `appVersion >= 2.0.0`), they are evaluated. If the user does not match the rules, they are excluded. (Returns `false`).
3. **Percentage Rollout:** The user's deterministic bucket is compared to the rollout percentage.
4. **Default/Fallback:** If the config is missing or malformed, return the hardcoded default.

### JSON Config Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "version": { "type": "string" },
    "features": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "properties": {
          "isKillSwitchActive": { 
            "type": "boolean", 
            "description": "If true, immediately disables the feature." 
          },
          "rolloutPercentage": { 
            "type": "integer", 
            "minimum": 0, 
            "maximum": 100 
          },
          "targeting": {
            "type": "object",
            "properties": {
              "minAppVersion": { "type": "string" },
              "allowedCountries": { "type": "array", "items": { "type": "string" } }
            }
          }
        },
        "required": ["isKillSwitchActive", "rolloutPercentage"]
      }
    }
  }
}
```

---

## 3. Thread Safety & Bootstrapping in Mobile

Mobile environments introduce constraints around network latency and app lifecycle. Waiting on a network request to render the UI is unacceptable, but "eventual consistency" can leave dangerous features running for an entire session. We must evaluate three potential strategies:

### Strategy 1: Blocking Fetch
* **Mechanism:** The app halts on the splash screen until the HTTP request for the config completes.
* **Pros:** The user always gets the freshest config. Zero layout shifting mid-session.
* **Cons:** Massive penalty to App Start Time (TTI). If the network is spotty, the app hangs indefinitely or times out, resulting in a terrible user experience. **Not recommended for consumer apps.**

### Strategy 2: Asynchronous Fetch with Cached Fallbacks (The Standard)
* **Mechanism:** 
  1. App boots instantly. The Engine initializes synchronously using a cached JSON payload from local disk (or a compiled-in default if no cache exists).
  2. In the background, the app fires an HTTP request to fetch the latest config.
  3. When the network returns, the JSON is saved to disk for the *next* cold start.
* **Pros:** Zero boot latency. Fast time-to-interactive.
* **Cons:** "Eventual Consistency". If a kill-switch was flipped 5 minutes ago, a user launching the app right now will boot using their cached config (where the feature is active) for the duration of this session.

### Strategy 3 (Chosen): The Hybrid Compromise (Live Reactivity)
To solve the "Eventual Consistency" issue for critical kill-switches while maintaining zero-latency boots, we separate the configuration streams by building upon Strategy 2:

1. **Synchronous Boot:** App boots instantly using the cached JSON payload from local disk (or a baked-in default). UI renders without hanging.
2. **Background Fetch:** App fires an HTTP request to fetch the latest config.
3. **Selective Reactivity (The Hybrid Hook):** 
   * When the network returns, standard rollout percentages and targeting rules are saved to disk but **frozen** for the current session to prevent unexpected UI layout shifts.
   * However, if a `isKillSwitchActive` flag transitions to `true` during the fetch, the Engine emits a reactive event, forcing the app to tear down the feature mid-session. 
   
This strategy provides the absolute best of both worlds: robust session stability for regular rollouts, and instant emergency response for kill-switches.

---

## 4. Explainability & Debugging

A critical requirement of a platform SDK is transparency. If a feature flag returns `false`, developers must know *why* without guessing. The evaluation engine must not just return a boolean; it must return a rich `EvaluationResult` that explains the decision tree.

### The Debug Utility
The SDK will provide an internal utility method (accessible via a Developer Menu) where a developer can input a `userId` and a `featureKey`. The SDK will output exactly why that user got that variant.

### Evaluation Engine Pseudocode (with Explainability)

```typescript
enum EvaluationReason {
    KILL_SWITCH = 'KILL_SWITCH',
    TARGETING_MISS = 'TARGETING_MISS',
    ROLLOUT_HIT = 'ROLLOUT_HIT',
    ROLLOUT_MISS = 'ROLLOUT_MISS',
    FALLBACK = 'FALLBACK'
}

interface EvaluationResult {
    isEnabled: boolean;
    reason: EvaluationReason;
    debugMessage: string;
}

function evaluate(featureKey: string, userContext: UserContext, config: RemoteConfig): EvaluationResult {
    const featureConfig = config.features[featureKey];
    
    // 0. Fallback
    if (!featureConfig) {
        return { 
            isEnabled: false, 
            reason: EvaluationReason.FALLBACK, 
            debugMessage: `Config missing or malformed for feature '${featureKey}'. Using default FALSE.` 
        };
    }
    
    // 1. Kill-Switch (Highest Priority)
    if (featureConfig.isKillSwitchActive) {
        return { 
            isEnabled: false, 
            reason: EvaluationReason.KILL_SWITCH, 
            debugMessage: `Feature disabled by global kill-switch override.` 
        };
    }
    
    // 2. Targeting Rules
    if (featureConfig.targeting) {
        if (!doesUserMatchRules(userContext, featureConfig.targeting)) {
            return { 
                isEnabled: false, 
                reason: EvaluationReason.TARGETING_MISS, 
                debugMessage: `User did not match targeting rules: ${JSON.stringify(featureConfig.targeting)}.` 
            };
        }
    }
    
    // 3. Rollout Percentage
    if (featureConfig.rolloutPercentage === 100) {
        return { isEnabled: true, reason: EvaluationReason.ROLLOUT_HIT, debugMessage: `Rollout is 100%.` };
    }
    if (featureConfig.rolloutPercentage === 0) {
        return { isEnabled: false, reason: EvaluationReason.ROLLOUT_MISS, debugMessage: `Rollout is 0%.` };
    }
    
    const bucket = getRolloutBucket(userContext.userId, featureKey);
    const isEnabled = bucket < featureConfig.rolloutPercentage;
    
    return {
        isEnabled,
        reason: isEnabled ? EvaluationReason.ROLLOUT_HIT : EvaluationReason.ROLLOUT_MISS,
        debugMessage: `User '${userContext.userId}' hashed to bucket ${bucket}. Feature '${featureKey}' rollout is ${featureConfig.rolloutPercentage}%. Result: ${isEnabled ? 'ENABLED' : 'DISABLED'}.`
    };
}
```
