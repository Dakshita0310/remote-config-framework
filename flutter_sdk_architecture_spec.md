# Flutter SDK Architecture: Remote-Config Client

## Overview
This document specifies the client-side architecture for the Flutter Remote-Config SDK. The primary goal is to ensure that gating a feature feels like a simple configuration check, minimizing architectural surgery for product teams and preventing technical debt accumulation.

---

## 1. SDK Public Interface

The SDK exposes a clean, unified API. It completely hides the complexity of hashing, evaluating targeting rules, and parsing JSON from the product developer. 

To prevent string-typing errors (e.g., typos in flag names), the SDK mandates the use of an `Enum` for feature keys.

```dart
/// The central client for accessing feature flag evaluations.
abstract class RemoteConfigClient {
  
  /// Initializes the SDK. Loads the cached config from disk instantly, 
  /// then triggers a background fetch.
  Future<void> initialize(UserContext context);

  /// Syntactic sugar for standard UI conditional rendering.
  /// Automatically evaluates the hierarchy (Kill Switch -> Targeting -> Rollout -> Default).
  bool isFeatureEnabled(FeatureKey key);

  /// Returns the rich evaluation object, primarily used by the Developer Menu 
  /// and exposure logging systems to explain the decision.
  EvaluationResult evaluateFeature(FeatureKey key);
  
  /// Updates the current user context (e.g., upon login) and re-evaluates all flags.
  void updateUserContext(UserContext newContext);
}
```

---

## 2. State Management Integration (Riverpod)

To support the "Hybrid Compromise" (where kill-switches update in real-time but standard rollouts are frozen), the SDK must integrate tightly with state management. We will use **Riverpod** to broadcast configuration changes efficiently.

Instead of product developers manually listening to SDK streams, we provide a family of Providers.

```dart
// 1. Holds the core SDK client and triggers rebuilds when the internal state changes 
// (e.g., when a kill-switch is thrown during a background fetch).
final remoteConfigClientProvider = StateNotifierProvider<RemoteConfigNotifier, RemoteConfigClient>((ref) {
  return RemoteConfigNotifier();
});

// 2. The Provider that product developers actually use.
// It watches the core client and evaluates a specific feature key.
final featureFlagProvider = Provider.family<bool, FeatureKey>((ref, key) {
  final client = ref.watch(remoteConfigClientProvider);
  return client.isFeatureEnabled(key);
});
```

**Usage in UI:**
The integration becomes frictionless. The widget simply watches the specific flag. If the config updates in real-time with a kill-switch, Riverpod seamlessly re-evaluates this specific provider and re-renders only the dependent widgets.

```dart
class CheckoutScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // A single line of code to check the flag, fully reactive.
    final isNewCheckoutEnabled = ref.watch(featureFlagProvider(FeatureKey.newCheckout));
    
    return isNewCheckoutEnabled ? NewCheckoutWidget() : LegacyCheckoutWidget();
  }
}
```

---

## 3. The "Clean Up" Strategy (Technical Debt Mitigation)

Every feature flag is technical debt. When an experiment concludes and a feature is fully rolled out (or killed), cleaning it up shouldn't require untangling spaghetti code. 

**Anti-Pattern:** Sprinkling `if(isFeatureEnabled)` checks deep inside complex build methods.
**Solution:** The **Facade / Component Substitution Pattern**.

We enforce a rule: Feature flags must be checked at the **highest possible component boundary**, ideally swapping entire independent widgets or screen routes, rather than tweaking parameters inside a monolithic widget.

### Clean-Up Using Abstract Interfaces (For Complex Logic)
For business logic, use an abstract interface and provide two concrete implementations.

```dart
abstract class PaymentProcessor {
  Future<void> process();
}

class LegacyPaymentProcessor implements PaymentProcessor { ... }
class StripePaymentProcessor implements PaymentProcessor { ... }

// In the Riverpod Provider:
final paymentProcessorProvider = Provider<PaymentProcessor>((ref) {
  final useStripe = ref.watch(featureFlagProvider(FeatureKey.stripeIntegration));
  return useStripe ? StripePaymentProcessor() : LegacyPaymentProcessor();
});
```
**The Clean Up:** When the experiment is over, simply delete `LegacyPaymentProcessor`, delete the `featureFlagProvider` condition in `paymentProcessorProvider`, and return `StripePaymentProcessor` directly. Zero spaghetti.

### Clean-Up Using Pattern Matching (For UI routing)
Using Dart 3 Pattern Matching and sealed classes allows for exhaustive checks, ensuring that when you remove a flag, the compiler forces you to clean up dead code.

```dart
sealed class CheckoutFlow {}
class LegacyFlow extends CheckoutFlow {}
class ModernFlow extends CheckoutFlow {}

Widget buildCheckout(CheckoutFlow flow) {
  return switch (flow) {
    LegacyFlow() => const LegacyCheckoutScreen(),
    ModernFlow() => const ModernCheckoutScreen(),
  };
}
```

---

## 4. Two-Config Mock Setup

To demonstrate the real-time reactivity and kill-switch architecture, we will build a `MockConfigRepository`. This allows the developer to instantly swap between two simulated environments without needing a real backend.

```dart
class MockConfigRepository {
  // Config A: Feature X is at a 50% rollout. Kill-Switch is OFF.
  static const Map<String, dynamic> configA = {
    "version": "v1.0",
    "features": {
      "newCheckout": {
        "isKillSwitchActive": false,
        "rolloutPercentage": 50,
      }
    }
  };

  // Config B: Feature X is killed. (Simulating an emergency rollback).
  static const Map<String, dynamic> configB = {
    "version": "v1.1",
    "features": {
      "newCheckout": {
        "isKillSwitchActive": true,
        "rolloutPercentage": 50,
      }
    }
  };

  /// Simulates a network fetch with artificial latency.
  Future<Map<String, dynamic>> fetchConfig(String environmentId) async {
    await Future.delayed(const Duration(milliseconds: 800)); // Artificial latency
    return environmentId == 'env_A' ? configA : configB;
  }
}
```

**Testing the UI Reactivity:**
In the Developer Menu, we provide two buttons: "Load Config A" and "Load Config B". 
1. The app boots with Config A. The user's ID hashes to `bucket 20`. Because `20 < 50%`, the feature is **ENABLED**.
2. The developer taps "Load Config B".
3. The `RemoteConfigNotifier` fetches Config B, parses it, and updates its state.
4. Riverpod detects the state change. The `featureFlagProvider(FeatureKey.newCheckout)` re-evaluates.
5. Because `isKillSwitchActive == true` in Config B, the provider evaluates to **FALSE**.
6. The UI instantly tears down `NewCheckoutWidget` and replaces it with `LegacyCheckoutWidget` mid-session, proving the emergency kill-switch capability.
