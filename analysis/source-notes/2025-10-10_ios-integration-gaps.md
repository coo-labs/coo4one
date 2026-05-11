# iOS integration gaps

**User:** Anonymous  
**Created:** 10/10/2025 20:42:55  
**Updated:** 10/10/2025 20:54:04  
**Exported:** 5/11/2026 15:06:21  
**Link:** [https://chatgpt.com/c/68e953aa-e3d8-8330-afb0-b8c55b7ccf07](https://chatgpt.com/c/68e953aa-e3d8-8330-afb0-b8c55b7ccf07)  

## Prompt:
10/10/2025, 8:42:55 PM

I don’t know anything about ios development, but I’ve noticed a pattern and I’m curious to know what the reason might be for this. iOS and iPad platforms offer many apple-specific features for apps - AppIntents, surfacing app contents in search and spotlight, widgets, integration with Shortcuts, multiple windows and so on. There seems to be two types of apps - one type take advantage of many if not all of these system-level integrations; others take advantage of none of them. The former are usually apple-exclusive apps, and the latter are almost always cross-platform apps that happen to have an iOS version. Those latter ones also typically have a web-app version as well, and their desktop versions are often electron-based. On iOS they are not since iOS does not support electron, but Ikm wondering why is that these types of apps almost never use the system-level integrations

## Response:
10/10/2025, 8:42:57 PM

> **Considering iOS cross-platform limits**
>
> Cross-platform apps often skip native iOS features like AppIntents, widgets, and Siri integration due to factors like engineering cost, shared codebase architecture (React Native, Flutter), API complexity, privacy concerns, and Apple's guidelines. Technical blockers include platform-specific requirements (Swift, SwiftUI) and limitations in background execution and features for web-backed apps. Cross-platform team dynamics also contribute.
>
> Thought for 9s

Short version: it’s incentives and architecture.

Longer version—why cross-platform apps usually skip iOS-specific integrations:

1) Lowest-common-denominator economics  
Teams building for Web + Windows + Android + macOS aim for one product spec and one codebase. Anything iOS-only (AppIntents, Core Spotlight, Widgets, UIScene multi-window, Share extensions, etc.) explodes scope: separate designs, separate QA matrices, separate regression risk. Most PMs will trade “delight on one platform” for “feature parity everywhere.”

2) Codebase mismatch  
A lot of these apps are fundamentally web apps (React + Node, or Electron on desktop; React Native/Flutter/Kotlin Multiplatform on mobile). Deep iOS features require Swift/Obj-C and Apple frameworks (often SwiftUI for widgets/AppIntents). That means:
- Native modules/bridges that need to be written, versioned, and maintained.
- Two mental models: declarative iOS APIs vs the app’s cross-platform UI/runtime.
- Extra CI/CD, extra test harnesses, extra release choreography.

3) Maintenance and API churn tax  
Apple’s platform features evolve yearly (new Intents schema, WidgetKit changes, scene management tweaks, background/task limits, privacy prompts). If iOS is <30–40% of your base, it’s hard to justify a permanent “follow WWDC” squad.

4) Product and data model friction  
- Core Spotlight requires local, indexable entities with stable identifiers and on-device privacy semantics. Many SaaS apps center everything in the cloud; local caching is partial and unreliable for indexing.  
- Shortcuts/AppIntents work best with crisp verb–object actions and small, typed parameters; many apps have mushy, complex flows that don’t map cleanly.  
- Good widgets require fast, small snapshot models and thoughtful glanceable UX—not just shrinking a web view (which you can’t do anyway).

5) Background execution limits  
Lots of “it would be nice if this auto-updated” ideas die on iOS because background refresh, notifications, and timeline budgets are tightly controlled. If the feature can’t reliably update, teams skip it to avoid user disappointment and support load.

6) Analytics, growth, and support incentives  
Surface content in Spotlight/Shortcuts and users may bypass your in-app funnels. Many companies optimize for in-app engagement and cross-platform consistency, not OS-level affordances that splinter usage patterns and complicate metrics and support docs.

7) Skillset + hiring  
Plenty of web engineers; fewer senior iOS engineers who love wrangling Intents, Extensions, and Scene life cycles. If your org is web-heavy, native niceties are always “Phase 2” that never ships.

8) App Store/Apple risk  
Some teams avoid deep entanglement with proprietary APIs because of policy uncertainty, review friction, or future deprecations. Minimal integration = fewer surprises.

Why Apple-exclusive apps go all-in  
Their strategy is to sell “best on Apple.” They’re already writing Swift/SwiftUI, they live on the yearly SDK cycle, and those integrations are a differentiator and growth channel (Spotlight → re-engagement, Shortcuts → power-user love, widgets → home-screen presence).

What would it take for a cross-platform app to do it well?  
- A hybrid architecture: shared core logic + platform-native UI/“integration layer.”  
- A small native team owning: AppIntents, Widgets, Core Spotlight, Share/Action extensions, BackgroundTasks.  
- A product philosophy that accepts platform divergence when it creates real user value (and a QA budget to match).

Blunt truth: until more companies treat iOS as a first-class product (not just another client for the web stack), you’ll keep seeing the split you’ve noticed.

## Prompt:
10/10/2025, 8:46:12 PM

but arent the ios app versions supposed to dto be in swift anyway?

## Response:
10/10/2025, 8:46:14 PM

Excellent question — and the short answer is: **no, not necessarily.**  

Here’s what’s actually happening under the hood for most cross-platform apps on iOS:

---

### 1. Many iOS apps are *not* written in Swift or Objective-C
They often use **cross-platform runtimes** that merely *wrap* a web or JS/Flutter codebase inside a thin native shell. Examples:

| Framework | What runs inside | “iOS app” is really |
|------------|-----------------|--------------------|
| **React Native** | JavaScript (React components) | A native wrapper with a bridge to JS |
| **Flutter** | Dart code | A single Skia-rendered canvas inside a UIView |
| **Unity** | C# + custom engine | An OpenGL/Metal scene |
| **Capacitor / Cordova** | HTML/JS | A fullscreen webview |
| **Qt / Avalonia / MAUI** | C++/C#/XAML | Their own UI toolkit drawn into a UIView |

These frameworks produce a `.ipa` that *looks* like an iOS app but contains very little Swift code—often just the glue to boot the runtime.

So when Apple releases something like **AppIntents** or **Widgets**, those frameworks can’t use them without writing **Swift-native bridge modules**—extra code that talks from JS/Dart/etc. to Swift.

---

### 2. The “wrapper” code is intentionally minimal  
For cross-platform teams, the iOS target is mostly a loader:
```swift
@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: ...
    ) -> Bool {
        CrossPlatformEngine.shared.start()
        return true
    }
}
```
That’s it.  
No iOS-specific logic, no Scene management, no WidgetKit target, no AppIntents definitions.  

---

### 3. Writing native integrations is costly  
To add an AppIntent, you need to:
- Add a new **Intents extension target**.
- Write Swift structs conforming to `AppIntent`.
- Define entities, schema, and parameter types.
- Localize, sign, and test it.
None of that can be done from JS/Dart.

Every such feature multiplies build complexity, and every API update (which Apple does *every year*) risks breakage. So cross-platform teams simply avoid it.

---

### 4. Why Apple’s own and indie-native apps feel richer  
They’re written **entirely in Swift/SwiftUI/UIKit**, using the full SDK stack—so system-level integrations come “for free” and evolve naturally with the platform.  
Cross-platform apps, in contrast, exist in a sealed box—functionally portable but culturally alien to iOS.

---

So yes, an app *published* for iOS must be compiled with Apple’s toolchain and signed—but that doesn’t mean it’s *written* in Swift or even *structured* like a native app. Most cross-platform apps are foreign guests living inside a Swift-made lobby.

## Prompt:
10/10/2025, 8:50:01 PM

got it that makes sense. I get almost all of it, but what does having multiple windows have to do with the rest? That doesn’t seem like the same type of feature, but those apps are usualy the only ones that do not allow to open multiple windows

## Response:
10/10/2025, 8:50:02 PM

Excellent observation — and you’re right that **multi-window support** seems conceptually different from things like Shortcuts or AppIntents.  
But it fits the same underlying pattern once you look at **how** it’s implemented.

Let’s unpack it:

---

### 1. What “multiple windows” means on iPadOS
When you open two instances of an app side by side (say, two Notes documents or two Safari tabs in split view), iPadOS is not just duplicating the UI.  
Under the hood, each window corresponds to a distinct **UIScene** managed by the system.

Every scene has:
- Its own lifecycle (connected, active, backgrounded, destroyed),
- Its own `UIViewController` hierarchy,
- Its own state restoration and delegate objects.

Implementing that properly requires:
- A Swift/Objective-C `SceneDelegate` setup,
- Code to create and persist per-window state,
- Handling drag-and-drop between scenes,
- UI logic that assumes multiple simultaneous instances.

That’s pure **UIKit/Swift** plumbing — not something generic frameworks expose.

---

### 2. Why cross-platform frameworks usually can’t handle it
Cross-platform toolkits generally assume a **single root view** managed by their own runtime:

| Framework | Limitation |
|------------|------------|
| React Native | One JS bridge and one root `RCTRootView`. Multiple scenes would need multiple JS runtimes. |
| Flutter | One `FlutterEngine` per window; multi-window is experimental and memory-heavy. |
| Capacitor / Cordova | One webview; the OS can’t just spin up a second without a separate process. |

Supporting iPad multi-window means running **two separate runtimes** (two JS or Dart VMs) simultaneously, each with its own memory and event loop — which most frameworks weren’t built for.

---

### 3. Product-level friction
Even when technically possible, app logic often assumes one active “session”:  
one logged-in user context, one API client, one navigation stack.  
Duplicating that safely across windows multiplies complexity and QA cost, for relatively little user demand outside power-user circles.

---

### 4. Why native-first apps embrace it
If your app already uses SwiftUI or UIKit natively (e.g., Notes, Mail, Files, Craft, OmniFocus), adopting multi-scene support is mostly a matter of flipping on scene activation and managing state restoration.  
It fits their existing architecture cleanly, so they get a huge UX win with modest effort.

---

**In short:**  
Multi-window looks like a “UI convenience,” but architecturally it’s a deep system feature tied to iOS scene management.  
Cross-platform frameworks abstract that away — which makes them portable, but also single-windowed by design.

## Prompt:
10/10/2025, 8:53:44 PM

Got it. I’ve grown to realize that these cross-platform apps are always dramatically less user-friendly, especially when I’ve grown to expect certain workflows, and I wish more were built native

## Response:
10/10/2025, 8:53:44 PM

Yes — that’s a sharp and very common realization once you start noticing it.  

The truth is that **cross-platform frameworks are optimized for developers, not for users**. They exist to *reduce engineering cost*, not to *maximize platform quality*. What you feel as “less user-friendly” is exactly the cost of that trade-off surfacing on your side of the screen.

A few ways this shows up in practice:

1. **UI feel and interaction latency** – Native apps inherit system physics, scroll inertia, input delays, accessibility, haptics, and keyboard behaviors. Cross-platform apps re-implement them, often imperfectly. That’s why scrolling or text selection in a Flutter or React Native app *feels off* compared to Mail or Notes.

2. **Integration and continuity** – Native apps live in the same ecosystem fabric: Spotlight, Share Sheets, Shortcuts, widgets, Files app, drag-and-drop, handoff, iCloud. Cross-platform apps live in a sandboxed island—fine on their own, but isolated from your broader workflow.

3. **Performance and memory footprint** – Each cross-platform runtime (JS, Dart, Chromium) drags along its own engine and garbage collector. They can’t match the power efficiency or responsiveness of compiled Swift code using native UI components.

4. **Accessibility and localization** – Apple’s APIs make it easy to get VoiceOver, Dynamic Type, and right-to-left layout “for free.” Non-native toolkits have to replicate all of that manually—and usually don’t.

5. **Platform evolution** – When Apple adds a new paradigm (widgets, Stage Manager, hover tooltips, visionOS input), native apps can adopt it within months. Cross-platform stacks often lag years, if ever.

To be fair, the developer perspective isn’t irrational: maintaining multiple fully-native codebases is brutally expensive, especially for smaller teams or B2B tools. But from a *craft and user-experience* standpoint, native apps are simply in another league.

That’s why Apple’s own apps, and a few independent ones that embrace the platform (Things, Craft, Bear, OmniFocus, Agenda, Drafts, etc.), feel like they “belong” on iOS—they *speak its language* rather than being translated into it.



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)