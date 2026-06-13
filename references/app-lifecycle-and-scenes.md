# SwiftUI App Lifecycle & Scenes

How the `@main` App entry point, scenes, windows, and `scenePhase` fit together — and the read-site and platform traps that surface in real apps.

## Table of Contents
- [The App Body Returns Scenes, Not Views](#the-app-body-returns-scenes-not-views)
- [The @main Entry Point](#the-main-entry-point)
- [WindowGroup Is a Template](#windowgroup-is-a-template)
- [Scene Modifiers vs View Modifiers](#scene-modifiers-vs-view-modifiers)
- [Platform-Specific Scenes](#platform-specific-scenes)
- [Menu-Bar-Only macOS Apps](#menu-bar-only-macos-apps)
- [Custom Reusable Scenes](#custom-reusable-scenes)
- [scenePhase Read-Site Semantics](#scenephase-read-site-semantics)
- [Delegate Adaptors](#delegate-adaptors)
- [Summary Checklist](#summary-checklist)

---

## The App Body Returns Scenes, Not Views

The body of an `App` is `some Scene`, not `some View`. Views live one level down, inside a scene's content closure. The boundary is easy to forget because everything else in SwiftUI hands you views.

```swift
// GOOD
@main
struct DemoApp: App {
    var body: some Scene {        // some Scene, never some View
        WindowGroup {
            ContentView()         // views start here
        }
    }
}

// BAD - VStack is a View, the app body expects a Scene
@main
struct DemoApp: App {
    var body: some View {         // wrong type, will not compile
        VStack { ContentView() }
    }
}
```

**Rule:** An `@main` App's `body` is `some Scene`; wrap your root view in a scene like `WindowGroup`.
**Why:** `App` and `Scene` are both `@MainActor`-isolated protocols with system-managed life cycles. A bare view has no life cycle for the system to drive, so it cannot stand in for a scene.

---

## The @main Entry Point

`@main` marks the struct as the program entry point — it replaces a hand-written `main.swift`. You never call an initializer on your App type; the runtime instantiates it and reads `body`.

```swift
// GOOD - the attribute is the entry point
@main
struct DemoApp: App {
    var body: some Scene { WindowGroup { ContentView() } }
}

// BAD - do not construct the app yourself
let app = DemoApp()   // pointless; the system already owns the instance
```

**Rule:** Annotate exactly one App type with `@main` and let the system own its lifetime.
**Why:** `@main` synthesizes the entry point. Adding a separate `main.swift`, or instantiating the App manually, conflicts with that synthesis.

---

## WindowGroup Is a Template

`WindowGroup`'s content closure is a *template*, not a single instance. On platforms that support multiple windows (iPad, macOS) the closure runs once per window, and each window owns its own `@State`, scroll position, and selection. On iPhone it backs a single window.

```swift
// GOOD - per-window state is expected; each window is independent
WindowGroup {
    ContentView()   // its @State is isolated to this window
}

// BAD - assuming one shared instance across every open window
WindowGroup {
    ContentView(shared: globalMutableStore)  // windows fight over one store
}
```

**Rule:** Treat each window as an independent instance of the content; do not assume `@State` is shared across windows.
**Why:** Multi-window platforms reinstantiate the template per window. State that must be shared belongs in a model passed through the environment, not in per-window `@State`.

---

## Scene Modifiers vs View Modifiers

Some modifiers apply to the *scene* and return `some Scene` — `windowStyle(_:)`, `windowResizability(_:)`, `menuBarExtraStyle(_:)`. They attach to `WindowGroup`/`Window`/`MenuBarExtra`, not to a view inside the closure.

```swift
// GOOD - scene modifier on the scene
WindowGroup {
    ContentView()
}
.windowResizability(.contentSize)

// BAD - scene modifier reached for inside a view body
struct ContentView: View {
    var body: some View {
        VStack { /* ... */ }
            .windowResizability(.contentSize)  // not a view modifier
    }
}
```

**Rule:** Apply window/menu-bar styling at the scene level, outside the content closure.
**Why:** Scene modifiers transform `Scene` values. Inside `body: some View` there is no scene to configure, so these modifiers are unavailable.

---

## Platform-Specific Scenes

Several scenes exist only on one platform: `Settings`, `Window`, and `MenuBarExtra` are macOS; `ImmersiveSpace` is visionOS. In a multiplatform target, gate them with `#if os(...)` *inside* the body — the scene builder accepts multiple sibling scenes, so they compose next to `WindowGroup`.

```swift
// GOOD - gate platform-only scenes; siblings compose
var body: some Scene {
    WindowGroup {
        ContentView()
    }
    #if os(macOS)
    Settings {
        SettingsView()
    }
    #endif
}

// BAD - referencing a macOS-only scene unconditionally on iOS
var body: some Scene {
    WindowGroup { ContentView() }
    Settings { SettingsView() }   // does not exist on iOS
}
```

**Rule:** Wrap platform-exclusive scenes in `#if os(...)` and let them sit as siblings of `WindowGroup`.
**Why:** The scene types are not declared on every platform. Unconditional use breaks the build on platforms where the symbol is absent.

---

## Menu-Bar-Only macOS Apps

`MenuBarExtra` can be the *only* scene in a macOS app — no main window at all. It also supports an `isInserted:` binding to control whether the item shows in the menu bar.

```swift
// GOOD - menu-bar-only app, presence is bindable
@main
struct StatusApp: App {
    @State private var shown = true
    var body: some Scene {
        MenuBarExtra("Status", systemImage: "gauge", isInserted: $shown) {
            StatusMenu()
        }
    }
}
```

**Rule:** A `MenuBarExtra` may stand alone as the app's sole scene; expose its visibility with `isInserted:` when users should be able to hide it.
**Why:** A menu-bar-only app has no window to keep it alive — if the user removes the extra, the system can terminate the app. Keep that in mind before shipping a hide toggle as the only way to dismiss it.

---

## Custom Reusable Scenes

Just as you extract reusable views, you can extract reusable *scenes*: conform a type to `Scene` whose `body` is `some Scene`, then compose it in the app body. This keeps a multi-scene app body readable.

```swift
// GOOD - scene-level composition mirrors view composition
struct MainWindow: Scene {
    var body: some Scene {
        WindowGroup { ContentView() }
            .windowResizability(.contentSize)
    }
}

@main
struct DemoApp: App {
    var body: some Scene {
        MainWindow()
        #if os(macOS)
        Settings { SettingsView() }
        #endif
    }
}
```

**Rule:** Extract a `struct X: Scene { var body: some Scene { … } }` to compose scenes, the same way you compose views.
**Why:** Scene composition is first-class. Factoring window/settings/menu-bar units into named scenes keeps the App body declarative instead of a sprawling closure.

---

## scenePhase Read-Site Semantics

`@Environment(\.scenePhase)` means *different things depending on where you read it*. Read in the App struct it is the **aggregate**: `.active` if any scene is active, `.background` only when none are. Read inside a View it reflects **that view's own scene**. Same key path, different meaning.

```swift
// GOOD - aggregate in the App; react with onChange + @unknown default
@main
struct DemoApp: App {
    @Environment(\.scenePhase) private var scenePhase
    var body: some Scene {
        WindowGroup { ContentView() }
            .onChange(of: scenePhase) { _, newPhase in
                switch newPhase {
                case .active:     resume()
                case .inactive:   pause()
                case .background: persist()
                @unknown default: break   // ScenePhase is non-frozen
                }
            }
    }
}

// BAD - reading in a deep view but assuming whole-app meaning
struct RowView: View {
    @Environment(\.scenePhase) private var scenePhase  // this view's scene only
    // treating .background here as "the entire app backgrounded" is wrong
    var body: some View { /* ... */ }
}
```

**Rule:** For whole-app transitions read `scenePhase` in the App and react with `.onChange(of:)`; include `@unknown default` in the switch.
**Why:** In the App the value aggregates across scenes; in a View it is scoped to that view's scene. `ScenePhase` is a non-frozen system enum, so an exhaustive switch without `@unknown default` will not compile cleanly against future cases.

---

## Delegate Adaptors

`@UIApplicationDelegateAdaptor` (and `NSApplicationDelegateAdaptor` on macOS) wires a UIKit/AppKit delegate into a SwiftUI app, giving you the classic launch callbacks that some SDKs need for swizzling or push registration. This is purely the SwiftUI↔UIKit seam.

```swift
// GOOD - just the wiring syntax for the SwiftUI <-> UIKit seam
@main
struct DemoApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate
    var body: some Scene {
        WindowGroup { ContentView() }
    }
}
```

**Rule:** Use the delegate adaptor only to expose UIKit/AppKit launch hooks; keep it to the wiring.
**Why:** The adaptor is the bridge, nothing more. *What* you do inside the delegate — app bootstrapping, dependency injection, building the object graph — is composition-root territory. Defer that design to the `ios-architecture-expert` skill; do not bootstrap or inject dependencies from here.

---

## Summary Checklist

### Do
- Return `some Scene` from the App body; wrap views in `WindowGroup`
- Use a single `@main` and let the system own the App instance
- Treat each window as an independent template instance
- Apply `windowStyle`/`windowResizability`/`menuBarExtraStyle` at the scene level
- Gate `Settings`/`Window`/`MenuBarExtra`/`ImmersiveSpace` behind `#if os(...)`
- Extract reusable `Scene` types for multi-scene app bodies
- Read `scenePhase` in the App for aggregate transitions; add `@unknown default`
- Limit delegate adaptors to UIKit/AppKit launch wiring

### Don't
- Write `body: some View` (or a bare `VStack`) in an `@main` App
- Instantiate the App type or add a competing `main.swift`
- Assume `@State` is shared across multiple windows
- Reach for scene modifiers inside a view's `body`
- Reference platform-only scenes unconditionally in a multiplatform target
- Treat a View's `scenePhase` as the whole app's state
- Omit `@unknown default` when switching on `ScenePhase`
- Put bootstrapping or dependency injection inside the delegate (see `ios-architecture-expert`)
