# Bug report — `adjust_sdk` 5.6.0 iOS cold-start AppDelegate interception breaks co-existing URL plugins

**Affected version:** `adjust_sdk` 5.6.0 (pub.dev)
**Platform:** iOS (classic `UIApplicationDelegate` lifecycle)
**Regression introduced in:** 5.6.0 (addition of `directDeeplinkCallback` feature)
**Co-existing plugin used to reproduce:** [`app_links`](https://pub.dev/packages/app_links) 6.4.1, but the bug affects any plugin that hooks `application:openURL:options:` or `application:continueUserActivity:restorationHandler:` and relies on them being dispatched on cold-start.

---

## 1. Summary

Starting with 5.6.0, the iOS plugin registers itself as a Flutter `FlutterApplicationLifeCycleDelegate` and implements `application:didFinishLaunchingWithOptions:` returning **`NO`**. Under Flutter's engine aggregator, a `NO` from that selector is treated as a **veto**: the aggregator short-circuits and returns `NO` to UIKit. UIKit then treats the app launch as "URL not accepted" and **skips** the follow-up dispatch through `application:openURL:options:` / `application:continueUserActivity:restorationHandler:`. Any other plugin that hooks those follow-up selectors (such as `app_links`) therefore never observes the launch URL on cold-start.

Warm-start is unaffected.

The suggested fix is to collapse `application:didFinishLaunchingWithOptions:` to `return YES;` and rely on the existing `application:openURL:options:` (line 79 in v5.6.0) and `application:continueUserActivity:restorationHandler:` (line 94) handlers — which are already present and correct — to capture the launch URL.

---

## 2. Root cause

### 2.1 What `adjust_sdk` 5.6.0 added on iOS

[`ios/Classes/AdjustSdk.m`](https://github.com/adjust/flutter_sdk/blob/v5.6.0/ios/Classes/AdjustSdk.m) in 5.6.0 introduced:

1. Registration as a `FlutterApplicationLifeCycleDelegate` (and optional `FlutterSceneLifeCycleDelegate`) inside `registerWithRegistrar:`:
    
    ```objectivec
    // AdjustSdk.m, registerWithRegistrar:
    [registrar addMethodCallDelegate:instance channel:channel];
    [registrar addApplicationDelegate:instance];           // new in 5.6.0
    #if ADJUST_FLUTTER_HAS_SCENE_LIFECYCLE
        if ([registrarObject respondsToSelector:@selector(addSceneDelegate:)]) {
            [registrarObject addSceneDelegate:instance];   // new in 5.6.0
        }
    #endif
    ```
    
2. Five new `UIApplicationDelegate` / `UIWindowSceneDelegate` methods, each capturing the URL into a new direct-deeplink pipeline (`processCapturedDeeplinkWithUrl:` → `dispatchOrCacheDirectDeeplink:`):
    - `application:didFinishLaunchingWithOptions:`
    - `application:openURL:options:`
    - `application:openURL:sourceApplication:annotation:`
    - `application:continueUserActivity:restorationHandler:`
    - Scene-API equivalents (`scene:willConnectToSession:options:`, `scene:openURLContexts:`, `scene:continueUserActivity:`)

All five return `NO`. For the four URL-delivery selectors this is fine — see §2.2. For `application:didFinishLaunchingWithOptions:` it is not.

### 2.2 Flutter engine aggregator semantics

[`FlutterPluginAppLifeCycleDelegate.mm`](https://github.com/flutter/flutter/blob/master/engine/src/flutter/shell/platform/darwin/ios/framework/Source/FlutterPluginAppLifeCycleDelegate.mm) aggregates `UIApplicationDelegate` methods across all plugins registered via `addApplicationDelegate:`. Two distinct conventions are used:

| Selector | Aggregator behavior | Engine lines (3.35.1) |
| --- | --- | --- |
| `application:didFinishLaunchingWithOptions:` | **Veto** — first plugin returning `NO` short-circuits; aggregator returns `NO` to UIKit | `99-112` |
| `application:willFinishLaunchingWithOptions:` | **Veto** — same | `114-128` |
| `application:openURL:options:` | **Continue-on-`NO`** — first `YES` wins; if none return `YES`, aggregator returns `NO` | `318-332` |
| `application:openURL:sourceApplication:annotation:` | Continue-on-`NO` | `348-366` |
| `application:continueUserActivity:restorationHandler:` | Continue-on-`NO` | `418-434` |

So `adjust_sdk` 5.6.0's `return NO;` is:

- **correct** for `application:openURL:options:` (line 83 in v5.6.0), `application:openURL:sourceApplication:annotation:` (line 91), `application:continueUserActivity:restorationHandler:` (line 101) — it means "I observed, I did not claim exclusive ownership, pass it along".
- **incorrect** for `application:didFinishLaunchingWithOptions:` (line 76) — it means "I veto app launch" under Flutter's aggregation convention, even though the Adjust plugin has no legitimate reason to veto.

### 2.3 Apple's UIKit contract

Per [Apple's `UIApplicationDelegate` documentation](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/application(_:didfinishlaunchingwithoptions:)) for apps launched by a URL:

> Return `false` if the app cannot handle the URL resource or continue a user activity. Otherwise, return `true`.
> 

and:

> If you return `false`, UIKit does not subsequently call the `application(_:open:options:)` or `application(_:continue:restorationHandler:)` methods.
> 

So the cascade is:

1. `FlutterPluginAppLifeCycleDelegate.application:didFinishLaunchingWithOptions:` is called during cold-start.
2. The aggregator iterates registered delegates. `AdjustSdk` is typically first (per `GeneratedPluginRegistrant` ordering, since plugins are registered alphabetically or in pubspec order).
3. `AdjustSdk` returns `NO`.
4. Aggregator returns `NO` immediately. No other plugin's `didFinishLaunchingWithOptions:` runs.
5. UIKit reads `NO` and suppresses the follow-up `openURL:options:` / `continueUserActivity:restorationHandler:`.
6. Plugins that hook only those follow-up selectors never observe the cold-start launch URL.

### 2.4 `app_links` as a concrete example

[`AppLinksIosPlugin.swift`](https://github.com/llfbandit/app_links/blob/master/app_links/ios/app_links/Sources/app_links/AppLinksIosPlugin.swift) (v6.4.1) implements only:

```swift
public func application(
  _ application: UIApplication,
  continue userActivity: NSUserActivity,
  restorationHandler: @escaping ([Any]) -> Void
) -> Bool {
  if let url = userActivity.webpageURL { handleLink(url: url) }
  return false
}

public func application(
  _ application: UIApplication,
  open url: URL,
  options: [UIApplication.OpenURLOptionsKey : Any] = [:]
) -> Bool {
  handleLink(url: url)
  return false
}
```

It does not implement `application:didFinishLaunchingWithOptions:` and does not read `launchOptions`. Post-5.6.0, both entry points are suppressed by UIKit on cold-start because the Flutter aggregator returned `NO` (courtesy of `AdjustSdk`). Result: `uriLinkStream` emits nothing on cold-start, `getInitialLink()` returns `nil`.

This is reproducible with just `adjust_sdk: 5.6.0` and `app_links: 6.4.1` in a fresh Flutter app, no additional dependencies required.

### 2.5 Not conditional on `directDeeplinkCallback`

The interception is installed at plugin registration time, independent of `AdjustConfig.directDeeplinkCallback`. The callback only gates whether the Dart side surfaces the URL ([`lib/adjust.dart:154-158`](https://github.com/adjust/flutter_sdk/blob/v5.6.0/lib/adjust.dart#L154-L158)):

```dart
static void _onDirectDeeplinkReceived(String? deeplink) {
  if (_directDeeplinkCallback != null) {
    _directDeeplinkCallback!(deeplink);
  }
  // else: silently dropped
}
```

Consumers that don't use `directDeeplinkCallback` therefore see the URL lost on *both* pathways: the follow-up AppDelegate dispatch is suppressed (so third-party plugins miss it) and the Dart-side direct-deeplink callback is a no-op (so `adjust_sdk` itself silently discards it too).

### 2.6 Warm-start is unaffected

On warm-start, UIKit does not call `didFinishLaunchingWithOptions:` at all — it dispatches directly via `openURL:options:` / `continueUserActivity:restorationHandler:`, which Flutter aggregates with continue-on-`NO`. Both `AdjustSdk` and `app_links` observe the URL and iteration continues past each `NO`.

---

## 3. Reproduction

Minimal reproduction with a fresh Flutter project:

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  adjust_sdk: 5.6.0
  app_links: 6.4.1
```

```dart
// main.dart — subscribe to app_links and initialise Adjust
final appLinks = AppLinks();
appLinks.uriLinkStream.listen((uri) {
  debugPrint('app_links uriLinkStream: $uri');   // NEVER fires on cold-start
});
final cfg = AdjustConfig('<appToken>', AdjustEnvironment.sandbox);
Adjust.initSdk(cfg);
```

Register a custom URL scheme (e.g. `myapp://`) in `Runner/Info.plist`. Kill the app. Tap `myapp://test` from another app.

- **Expected:** the `uriLinkStream` listener prints `myapp://test`.
- **Observed with `adjust_sdk` 5.6.0:** the listener never fires. `appLinks.getInitialLink()` returns `null`.
- **Observed with `adjust_sdk` ≤ 5.5.x:** the listener fires with `myapp://test`.

For universal-link reproduction, configure `applinks:` entitlements and tap an `https://…` link registered for the app. Same symptom.

---

## 4. Suggested fix

Collapse `application:didFinishLaunchingWithOptions:` to `return YES;`. The launch URL will arrive through the existing `application:openURL:options:` / `application:continueUserActivity:restorationHandler:` handlers, which already call `processCapturedDeeplinkWithUrl:` and already return `NO` under the correct (continue-on-`NO`) aggregator.

### 4.1 Before (`ios/Classes/AdjustSdk.m`, v5.6.0)

```objectivec
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSURL *launchUrl = launchOptions[UIApplicationLaunchOptionsURLKey];
    [self processCapturedDeeplinkWithUrl:launchUrl referrer:nil];

    NSDictionary *userActivityDictionary = launchOptions[UIApplicationLaunchOptionsUserActivityDictionaryKey];
    NSUserActivity *userActivity = nil;
    for (id value in [userActivityDictionary allValues]) {
        if ([value isKindOfClass:[NSUserActivity class]]) {
            userActivity = (NSUserActivity *)value;
            break;
        }
    }
    if ([self isFieldValid:userActivity]
        && [userActivity.activityType isEqualToString:NSUserActivityTypeBrowsingWeb]) {
        [self processCapturedDeeplinkWithUrl:userActivity.webpageURL referrer:userActivity.referrerURL];
    }

    return NO;
}
```

### 4.2 After

```objectivec
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Flutter's FlutterPluginAppLifeCycleDelegate aggregates this selector with
    // veto semantics: if any plugin returns NO, the aggregator short-circuits
    // and returns NO to UIKit, which then skips the follow-up
    // application:openURL:options: / application:continueUserActivity:
    // restorationHandler: dispatch. Other plugins that rely on those
    // follow-ups would never observe the launch URL.
    //
    // Returning YES lets UIKit dispatch the launch URL through the normal
    // openURL / continueUserActivity paths below, which already route into
    // processCapturedDeeplinkWithUrl: for this plugin and also allow any
    // other registered plugins to observe the event.
    return YES;
}
```

### 4.3 Why dropping the `launchOptions` extraction is safe

`processCapturedDeeplinkWithUrl:` only feeds the new Dart-facing direct-deeplink method channel (`dispatchOrCacheDirectDeeplink:`). It does not feed Adjust's native attribution engine — that is driven by the explicit `processDeeplink:withResult:` method-call handler (`Adjust.processDeeplink` on the Dart side, line 675 in v5.6.0), which remains unchanged.

After the fix, the launch URL still reaches `processCapturedDeeplinkWithUrl:` — but via the follow-up `application:openURL:options:` (line 79 in v5.6.0) or `application:continueUserActivity:restorationHandler:` (line 94), which run microseconds later in the same cold-start sequence. Because `self.isSdkInitialized` is `NO` at that point (Dart has not yet called `Adjust.initSdk`), the URL is still queued in `self.cachedDirectDeeplinks` and later drained by `flushCachedDirectDeeplinks` during `initSdk:`. No observable change to the direct-deeplink callback's behavior, no change to attribution.

### 4.4 Dispatch counts with the fix

For a cold-start custom-scheme link with the suggested fix:

- `processCapturedDeeplinkWithUrl:` called once, via `application:openURL:options:`.
- `dispatchOrCacheDirectDeeplink:` called once; URL queued until `initSdk:` then flushed once.
- `Adjust.directDeeplinkCallback` fires once on the Dart side.

For a cold-start universal link:

- Same, via `application:continueUserActivity:restorationHandler:`.

No duplicate dispatches. `app_links` (and any other plugin hooking the follow-up selectors) also receives the URL exactly once, as it did pre-5.6.0.

---

## 5. Additional considerations

### 5.1 Scene-lifecycle equivalent — same shape, same bug

`scene:willConnectToSession:options:` (line 105 in v5.6.0) has the same "capture URL + `return NO`" structure:

```objectivec
- (BOOL)scene:(UIScene *)scene
willConnectToSession:(UISceneSession *)session
      options:(UISceneConnectionOptions *)connectionOptions API_AVAILABLE(ios(13.0)) {
    if (connectionOptions == nil) { return NO; }
    for (NSUserActivity *userActivity in connectionOptions.userActivities) { ... }
    for (UIOpenURLContext *urlContext in connectionOptions.URLContexts) { ... }
    return NO;
}
```

Under Flutter's `FlutterSceneLifeCycleDelegate` aggregator, the same veto-propagation question applies on hosts that adopt `UIApplicationSceneManifest`. A scene-aware URL plugin (if one eventually emerges) would see the same symptom as `app_links` does today under the classic lifecycle. The symmetric fix is to reduce that body to `return YES;` (or simply remove the `return NO;` path via `connectionOptions == nil` and let the follow-up `scene:openURLContexts:` / `scene:continueUserActivity:` methods handle the URL). `app_links` 6.4.1 does not implement scene-delegate methods, so this is latent, not active — but worth fixing at the same time for consistency.

### 5.2 Host AppDelegate overrides

Apps that override `didFinishLaunchingWithOptions:` in their own `AppDelegate` are unaffected by this patch: the aggregator still honors their return value, and `AdjustSdk` simply stops contributing a `NO` vote. Hosts that deliberately wanted `AdjustSdk` to veto launch get no such behavior today anyway (the pre-fix `NO` was not conditional on any config flag), so there is no behavior change for them.

### 5.3 Timing shift

Pre-fix, the direct-deeplink URL was captured inside `didFinishLaunchingWithOptions:`, which runs just before `openURL:options:` / `continueUserActivity:restorationHandler:`. Post-fix, capture moves to those follow-up calls — same cold-start window, microseconds later, before Dart is running and before `initSdk:` is called. Both pre- and post-fix, the URL is queued into `cachedDirectDeeplinks` and drained by `flushCachedDirectDeeplinks` during `initSdk:`. No observable Dart-side ordering difference.

### 5.4 Android cross-check

The Android side ([`android/src/main/java/com/adjust/sdk/flutter/AdjustSdk.java`](https://github.com/adjust/flutter_sdk/blob/v5.6.0/android/src/main/java/com/adjust/sdk/flutter/AdjustSdk.java)) is correct as-is. `onNewIntent` returns `false` (line 125), which Flutter's `ActivityPluginBinding` iterator treats as continue-on-`false` — other `NewIntentListener` registrants (e.g. `AppLinksPlugin`) still receive the intent. Cold-start on Android reads `getActivity().getIntent()` inside `onAttachedToActivity` (line 104); both plugins can read the same Intent independently without contention.

The iOS-specific veto semantics have no Android analogue. No Android changes needed.

---

## 6. Verification checklist

Against a reference Flutter project with `adjust_sdk: 5.6.0-patched` and `app_links: 6.4.1`:

- [ ]  **Cold-start, custom URL scheme:** Kill app → tap `myapp://test` → expect `AppLinks.uriLinkStream` to emit `myapp://test`.
- [ ]  **Cold-start, universal link:** Kill app → tap an `https://` link registered via `applinks:` entitlement → expect `AppLinks.uriLinkStream` to emit the URL.
- [ ]  **Warm-start, custom URL scheme:** App in foreground/background → tap `myapp://test` → expect `AppLinks.uriLinkStream` to emit (regression check — this path was not broken in 5.6.0).
- [ ]  **Warm-start, universal link:** Same as above with `https://`.
- [ ]  **Adjust attribution:** `AdjustConfig.attributionCallback` still fires with valid tracker data (should be unaffected, worth smoke-testing).
- [ ]  **`AdjustConfig.directDeeplinkCallback`, if set:** Fires exactly once on cold-start for the launch URL. Verify no duplicate invocations relative to 5.5.x behavior for deeplinks after `initSdk:`.
- [ ]  **With neither `directDeeplinkCallback` nor `app_links` present:** `initSdk:` completes normally, no crashes, no dangling queued URLs beyond the existing `cachedDirectDeeplinks` drain at `initSdk:` time.

---

## 7. References

| Resource | Location |
| --- | --- |
| Affected method in 5.6.0 | [`ios/Classes/AdjustSdk.m` lines 59-77](https://github.com/adjust/flutter_sdk/blob/v5.6.0/ios/Classes/AdjustSdk.m#L59-L77) |
| Surrounding intercept methods (correct as-is) | `AdjustSdk.m` lines 79-102 |
| Scene-delegate equivalent (same shape) | `AdjustSdk.m` lines 104-153 |
| Direct-deeplink dispatch pipeline | `AdjustSdk.m` lines 1197-1244 |
| Direct-deeplink Dart handler | [`lib/adjust.dart` lines 126-158](https://github.com/adjust/flutter_sdk/blob/v5.6.0/lib/adjust.dart#L126-L158) |
| Flutter engine aggregator (veto vs continue-on-`NO`) | [`FlutterPluginAppLifeCycleDelegate.mm`](https://github.com/flutter/flutter/blob/master/engine/src/flutter/shell/platform/darwin/ios/framework/Source/FlutterPluginAppLifeCycleDelegate.mm) lines 99-112, 318-332, 418-434 |
| Flutter `FlutterAppDelegate` forwarding | [`FlutterAppDelegate.mm`](https://github.com/flutter/flutter/blob/master/engine/src/flutter/shell/platform/darwin/ios/framework/Source/FlutterAppDelegate.mm) lines 53-57, 153-162, 225-237 |
| `app_links` iOS plugin | [`AppLinksIosPlugin.swift`](https://github.com/llfbandit/app_links/blob/master/app_links/ios/app_links/Sources/app_links/AppLinksIosPlugin.swift) |
| Apple `UIApplicationDelegate` contract | https://developer.apple.com/documentation/uikit/uiapplicationdelegate |
| Adjust Flutter SDK 5.6.0 changelog | https://pub.dev/packages/adjust_sdk/changelog |
