# SwiftUI Sheet and Navigation Patterns Reference

## Table of Contents

- [Container Choice: Tabs vs Stacks vs Split Views](#container-choice-tabs-vs-stacks-vs-split-views)
  - [Tab-Based Navigation (iOS 18 Tab API)](#tab-based-navigation-ios-18-tab-api)
  - [NavigationSplitView (Adaptive Two-Column)](#navigationsplitview-adaptive-two-column)
- [Sheet Patterns](#sheet-patterns)
  - [Item-Driven Sheets (Preferred)](#item-driven-sheets-preferred)
  - [Sheets Own Their Actions](#sheets-own-their-actions)
- [Navigation Patterns](#navigation-patterns)
  - [Type-Safe Navigation with NavigationStack](#type-safe-navigation-with-navigationstack)
  - [Inline vs Value-Based NavigationLink](#inline-vs-value-based-navigationlink)
  - [Programmatic Navigation](#programmatic-navigation)
  - [Typed-Array Path vs NavigationPath](#typed-array-path-vs-navigationpath)
  - [navigationDestination Placement Gotchas](#navigationdestination-placement-gotchas)
  - [Navigation with State Restoration](#navigation-with-state-restoration)
- [Presentation Modifiers](#presentation-modifiers)
  - [Sheet Detents](#sheet-detents)
  - [Full Screen Cover](#full-screen-cover)
  - [Popover](#popover)
  - [Stacking Modals](#stacking-modals)
  - [Alert with Actions](#alert-with-actions)
  - [Confirmation Dialog](#confirmation-dialog)
- [Summary Checklist](#summary-checklist)

## Container Choice: Tabs vs Stacks vs Split Views

**Rule:** Pick the navigation container by structure, not habit. Peer top-level sections → `TabView` with the `Tab` API. A drill-down within one section → `NavigationStack`. A list-plus-detail that should expand on iPad/Mac → `NavigationSplitView`. These compose: each tab typically owns its own `NavigationStack`.

### Tab-Based Navigation (iOS 18 Tab API)

**Rule:** Declare tabs with the `Tab` initializer inside `TabView`, not by tagging child views with `.tabItem { Label(...) }`.

```swift
// GOOD - Tab API: label and content in one declaration
enum AppTab: Hashable {
    case home, search, profile
}

struct RootView: View {
    @SceneStorage("selectedTab") private var selection: AppTab = .home

    var body: some View {
        TabView(selection: $selection) {
            Tab("Home", systemImage: "house", value: AppTab.home) {
                NavigationStack { HomeView() }
            }
            Tab("Search", systemImage: "magnifyingglass", value: AppTab.search) {
                NavigationStack { SearchView() }
            }
            Tab("Profile", systemImage: "person", value: AppTab.profile) {
                NavigationStack { ProfileView() }
            }
        }
    }
}

// BAD - legacy .tabItem on child views
TabView(selection: $selection) {
    HomeView()
        .tabItem { Label("Home", systemImage: "house") }
        .tag(AppTab.home)
    // ...
}
```

**Why:** The `Tab` API keeps the tab's identity, label, and content together, and is the foundation the adaptable-sidebar styles build on. For programmatic selection, give each `Tab` a `value:` whose type matches `TabView(selection:)`; without matching values, selection bindings silently do nothing. `@SceneStorage` persists the selection per-scene so the active tab survives backgrounding and state restoration.

**Adaptable sidebar (iPad / Mac):** `.tabViewStyle(.sidebarAdaptable)` lets the same `TabView` render as a bottom tab bar in compact width and a sidebar in regular width. Group related tabs under `TabSection("Title") { Tab(...) ... }` to create labeled sidebar sections.

```swift
TabView(selection: $selection) {
    Tab("Home", systemImage: "house", value: AppTab.home) { HomeView() }

    TabSection("Library") {
        Tab("Recents", systemImage: "clock", value: AppTab.recents) { RecentsView() }
        Tab("Favorites", systemImage: "star", value: AppTab.favorites) { FavoritesView() }
    }
}
.tabViewStyle(.sidebarAdaptable)
```

> Note: `TabSection` is a grouping container; to let users reorder/hide sidebar tabs, drive it with `TabViewCustomization` via `@AppStorage` and `.customizationID(...)` per tab — verify the exact customization API for your deployment target before adopting it.

### NavigationSplitView (Adaptive Two-Column)

**Rule:** For a list-detail screen that should be side-by-side on iPad/Mac but a push-stack on iPhone, use `NavigationSplitView` with a selection-bound `List` — one declaration adapts to both.

```swift
// GOOD - one declaration adapts iPhone <-> iPad
struct LibraryView: View {
    @State private var selection: Item.ID?
    let items: [Item]

    var body: some View {
        NavigationSplitView {
            List(items, selection: $selection) { item in
                Text(item.name).tag(item.id)
            }
            .navigationTitle("Library")
        } detail: {
            if let selection, let item = items.first(where: { $0.id == selection }) {
                ItemDetailView(item: item)
            } else {
                ContentUnavailableView("Select an Item", systemImage: "square.dashed")
            }
        }
    }
}
```

**Why:** In regular width (iPad landscape, Mac) the sidebar and detail show side-by-side; in compact width (iPhone) the split view auto-collapses into a `NavigationStack` and the selection drives a push. Always provide a no-selection placeholder for the detail column — otherwise the regular-width layout shows an empty pane on launch.

## Sheet Patterns

### Item-Driven Sheets (Preferred)

**Use `.sheet(item:)` instead of `.sheet(isPresented:)` when presenting model-based content.**

```swift
// Good - item-driven
@State private var selectedItem: Item?

var body: some View {
    List(items) { item in
        Button(item.name) {
            selectedItem = item
        }
    }
    .sheet(item: $selectedItem) { item in
        ItemDetailSheet(item: item)
    }
}

// Avoid - boolean flag requires separate state
@State private var showSheet = false
@State private var selectedItem: Item?

var body: some View {
    List(items) { item in
        Button(item.name) {
            selectedItem = item
            showSheet = true
        }
    }
    .sheet(isPresented: $showSheet) {
        if let selectedItem {
            ItemDetailSheet(item: selectedItem)
        }
    }
}
```

**Why**: `.sheet(item:)` automatically handles presentation state and avoids optional unwrapping in the sheet body.

### Sheets Own Their Actions

**Sheets should handle their own dismiss and actions internally.**

```swift
// Good - sheet owns its actions
struct EditItemSheet: View {
    @Environment(\.dismiss) private var dismiss
    @Environment(DataStore.self) private var store
    
    let item: Item
    @State private var name: String
    @State private var isSaving = false
    
    init(item: Item) {
        self.item = item
        _name = State(initialValue: item.name)
    }
    
    var body: some View {
        NavigationStack {
            Form {
                TextField("Name", text: $name)
            }
            .navigationTitle("Edit Item")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") {
                        dismiss()
                    }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button(isSaving ? "Saving..." : "Save") {
                        Task { await save() }
                    }
                    .disabled(isSaving || name.isEmpty)
                }
            }
        }
    }
    
    private func save() async {
        isSaving = true
        await store.updateItem(item, name: name)
        dismiss()
    }
}

// Avoid - parent manages sheet actions via closures
struct ParentView: View {
    @State private var selectedItem: Item?
    
    var body: some View {
        List(items) { item in
            Button(item.name) {
                selectedItem = item
            }
        }
        .sheet(item: $selectedItem) { item in
            EditItemSheet(
                item: item,
                onSave: { newName in
                    // Parent handles save
                },
                onCancel: {
                    selectedItem = nil
                }
            )
        }
    }
}
```

**Why**: Sheets that own their actions are more reusable and don't require callback prop-drilling.

## Navigation Patterns

### Type-Safe Navigation with NavigationStack

```swift
struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("Profile", value: Route.profile)
                NavigationLink("Settings", value: Route.settings)
            }
            .navigationDestination(for: Route.self) { route in
                switch route {
                case .profile:
                    ProfileView()
                case .settings:
                    SettingsView()
                }
            }
        }
    }
}

enum Route: Hashable {
    case profile
    case settings
}
```

### Inline vs Value-Based NavigationLink

**Rule:** Inside a `NavigationStack` you intend to drive programmatically or restore, use the value-based `NavigationLink(_, value:)` paired with a `.navigationDestination(for:)` — never the inline-closure form.

```swift
// GOOD - value-based: participates in the bound path
NavigationLink("Profile", value: Route.profile)
// ... resolved elsewhere by:
.navigationDestination(for: Route.self) { route in destination(for: route) }

// BAD (when a path is involved) - inline destination
NavigationLink("Profile") {
    ProfileView()
}
```

**Why:** The two forms look identical at runtime, but the inline form builds its destination directly and is invisible to the stack's `path` binding. That means you cannot push it programmatically and it is dropped during state restoration. The value-based form appends to the bound path, so deep links, "pop to root," and restoration all work. (Inline links are fine only in a stack with no `path` binding and no restoration needs.)

### Programmatic Navigation

```swift
struct ContentView: View {
    @State private var navigationPath = NavigationPath()
    
    var body: some View {
        NavigationStack(path: $navigationPath) {
            List {
                Button("Go to Detail") {
                    navigationPath.append(DetailRoute.item(id: 1))
                }
            }
            .navigationDestination(for: DetailRoute.self) { route in
                switch route {
                case .item(let id):
                    ItemDetailView(id: id)
                }
            }
        }
    }
}

enum DetailRoute: Hashable {
    case item(id: Int)
}
```

### Typed-Array Path vs NavigationPath

**Rule:** When every destination in a stack is the same type, bind the path to a typed array `[T]` instead of `NavigationPath`. Reserve type-erased `NavigationPath` for stacks that push **mixed** types.

```swift
// GOOD - homogeneous stack: typed array gives append, indexing, and pop-to-root
@State private var path: [Route] = []

NavigationStack(path: $path) {
    RootView()
        .navigationDestination(for: Route.self) { destination(for: $0) }
}

// push:        path.append(.profile)
// pop to root: path.removeAll()
// pop one:     path.removeLast()

// Use NavigationPath only when destinations are different types:
@State private var path = NavigationPath()   // can hold Route, Int, User, …
```

**Why**: a typed array is directly inspectable and mutable (`removeAll()` for pop-to-root, indexing, `count`), while `NavigationPath` erases the element types and only supports append/removeLast. Reach for the array unless you genuinely mix destination types.

### navigationDestination Placement Gotchas

**Rule:** Put `.navigationDestination(for:)` **high** in the hierarchy (on the stack's root content), not on a row inside a lazy `List`/`ForEach`. Declare exactly **one** destination per data type per stack.

```swift
// BAD - destination attached to a lazy row: deep links into an
// off-screen (not-yet-realized) row silently fail to resolve.
List(items) { item in
    NavigationLink(item.name, value: item)
        .navigationDestination(for: Item.self) { ItemView($0) }  // wrong place
}

// GOOD - one destination, on the stack root
List(items) { item in
    NavigationLink(item.name, value: item)
}
.navigationDestination(for: Item.self) { ItemView($0) }
```

**Why**: SwiftUI can't resolve a destination whose modifier lives on a view that hasn't been created yet (lazy containers only realize visible rows), so programmatic/deep-link pushes into unrendered rows do nothing. And if you declare two `navigationDestination`s for the same type in one stack, only the one **closest to the root** is used — the other is ignored.

### Navigation with State Restoration

```swift
struct ContentView: View {
    @State private var navigationPath = NavigationPath()
    
    var body: some View {
        NavigationStack(path: $navigationPath) {
            RootView()
                .navigationDestination(for: Route.self) { route in
                    destinationView(for: route)
                }
        }
    }
    
    @ViewBuilder
    private func destinationView(for route: Route) -> some View {
        switch route {
        case .profile:
            ProfileView()
        case .settings:
            SettingsView()
        }
    }
}
```

## Presentation Modifiers

### Sheet Detents

**Rule:** Control how much of the screen a sheet occupies with `.presentationDetents([...])`. Multiple detents make the sheet user-draggable between rest positions.

```swift
.sheet(item: $selected) { item in
    DetailSheet(item: item)
        .presentationDetents([.fraction(0.25), .medium, .large])  // also .height(_:)
        .presentationDragIndicator(.visible)
}
```

**Why**: a single detent pins the sheet height; supplying several lets the user drag between them (e.g. a peek at 25% that expands to full). Prefer `.fraction`/`.height` for custom sizes, `.medium`/`.large` for the system defaults.

### Full Screen Cover

**Rule:** A `fullScreenCover` has **no built-in dismissal** — unlike a sheet or popover it does not dismiss on swipe-down or an outside tap. You must provide an explicit exit.

```swift
.fullScreenCover(isPresented: $showFullScreen) {
    NavigationStack {
        FullScreenView()
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    if #available(iOS 26.0, *) {
                        Button(role: .close) { showFullScreen = false }   // iOS 26 close role
                    } else {
                        Button("Close") { showFullScreen = false }
                    }
                }
            }
    }
}
```

**Why**: full-screen covers are modal by design and trap interaction, so a forgotten close control strands the user. The iOS 26 `Button(role: .close)` renders the standard close affordance; gate it with `#available` and fall back to a labeled button.

### Popover

```swift
struct ContentView: View {
    @State private var showPopover = false
    
    var body: some View {
        Button("Show Popover") {
            showPopover = true
        }
        .popover(isPresented: $showPopover, arrowEdge: .top) {
            PopoverContentView()
                .presentationCompactAdaptation(.popover)  // Don't adapt to sheet on iPhone
        }
    }
}
```

**Rule:** Attach `.popover` directly to the triggering view so SwiftUI can compute the anchor; pass `arrowEdge:` to hint where the arrow points. In compact width a popover adapts to a sheet by default — `.presentationCompactAdaptation(.popover)` forces true popover behavior.

### Stacking Modals

**Rule:** To present a second modal on top of a first, the second `.sheet`/`.fullScreenCover` modifier must be **nested inside the first modal's content** — not declared as a sibling.

```swift
// GOOD - second sheet lives inside the first sheet's content
.sheet(isPresented: $showFirst) {
    FirstSheet()
        .sheet(isPresented: $showSecond) { SecondSheet() }
}

// BAD - sibling modifiers: tapping to show the second does nothing
// until the first is dismissed (a container presents one at a time).
.sheet(isPresented: $showFirst) { FirstSheet() }
.sheet(isPresented: $showSecond) { SecondSheet() }
```

**Why**: each presentation context manages a single active presentation. The second modifier has to belong to the already-presented content's context to stack on top.

### Alert with Actions

```swift
struct ContentView: View {
    @State private var showAlert = false
    
    var body: some View {
        Button("Show Alert") {
            showAlert = true
        }
        .alert("Delete Item?", isPresented: $showAlert) {
            Button("Delete", role: .destructive) {
                deleteItem()
            }
            Button("Cancel", role: .cancel) { }
        } message: {
            Text("This action cannot be undone.")
        }
    }
}
```

### Confirmation Dialog

```swift
struct ContentView: View {
    @State private var showDialog = false
    
    var body: some View {
        Button("Show Options") {
            showDialog = true
        }
        .confirmationDialog("Choose an option", isPresented: $showDialog) {
            Button("Option 1") { handleOption1() }
            Button("Option 2") { handleOption2() }
            Button("Cancel", role: .cancel) { }
        }
    }
}
```

## Summary Checklist

- [ ] Use `.sheet(item:)` for model-based sheets
- [ ] Sheets own their actions and dismiss internally
- [ ] Use `NavigationStack` with `navigationDestination(for:)` for type-safe navigation
- [ ] Use `NavigationPath` for programmatic navigation
- [ ] Use appropriate presentation modifiers (sheet, fullScreenCover, popover)
- [ ] Alerts and confirmation dialogs use modern API with actions
- [ ] Avoid passing dismiss/save callbacks to sheets
- [ ] Navigation state can be saved/restored when needed
