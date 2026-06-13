# SwiftUI Data Loading & Tasks

Async load-on-appear, fetch-on-change, change observation, and offloading heavy work without hanging the main actor.

## Table of Contents
- [Entry Points: .task vs .onAppear](#entry-points-task-vs-onappear)
- [Always Provide an Initial/Loading State](#always-provide-an-initialloading-state)
- [Fetch-on-Change with .task(id:)](#fetch-on-change-with-taskid)
- [.onChange for Synchronous Reactions](#onchange-for-synchronous-reactions)
- [Keep Observed Identifiers Cheap](#keep-observed-identifiers-cheap)
- [Offloading Heavy Work with @concurrent](#offloading-heavy-work-with-concurrent)
- [Cooperative Cancellation](#cooperative-cancellation)
- [Summary Checklist](#summary-checklist)

---

## Entry Points: .task vs .onAppear

Reach for `.task {}` to kick off async loading when a view appears. It gives you an async context (you `await` directly, no nested `Task`) and SwiftUI binds the work's lifetime to the view's presence — when the view leaves the hierarchy, the task is cancelled for you.

```swift
// GOOD - async context + lifetime tied to the view; auto-cancels on disappear
.task {
    items = await loader.loadItems()
}

// BAD - unmanaged work; outlives the view and leaks on re-appear
.onAppear {
    Task { items = await loader.loadItems() }
}
```

The `.onAppear { Task { } }` pattern detaches the work from the view. Navigate away mid-flight and it keeps running; a view that re-appears spawns a second overlapping task, and so on. Reserve `.onAppear` for *synchronous* first-frame seeding, and guard it — it can fire more than once.

```swift
// GOOD - sync seed only, guarded against re-fire
.onAppear {
    guard items.isEmpty else { return }
    items = Self.placeholderRows
}
```

**Rule:** Use `.task {}` for async load-on-appear; use `.onAppear` only for guarded synchronous seeding — never wrap async work in `.onAppear { Task { } }`.

**Why:** `.task` is an async context whose lifetime SwiftUI manages and cancels on disappear; a manual `Task` in `.onAppear` is unmanaged and leaks/overlaps across navigation and re-appearance.

---

## Always Provide an Initial/Loading State

`.task` is not guaranteed to complete before the first frame is painted. If `body` renders nothing while data is `nil`/empty, the first frame is a broken empty state that flickers when results arrive.

```swift
// GOOD - first frame shows a placeholder, not a broken empty view
.overlay {
    if items.isEmpty { ProgressView() }
}
.task { items = await loader.loadItems() }

// BAD - first frame is empty/blank until the task happens to finish
.task { items = await loader.loadItems() }
```

**Rule:** Render an explicit loading/empty placeholder for the window before `.task` resolves.

**Why:** The task may not finish before first paint; without a placeholder the user sees a flash of empty content.

---

## Fetch-on-Change with .task(id:)

When the fetch depends on a value that changes (a selection, a query, an id), use `.task(id:)`. SwiftUI cancels the in-flight task and starts a fresh one whenever `id` changes — and still cancels on removal.

```swift
// GOOD - cancels old fetch, restarts on change, cleans up on disappear
.task(id: selection) {
    detail = await loader.loadDetail(for: selection)
}

// BAD - manual cancel/restart dance, easy to leak the old task
.onChange(of: selection) { _, new in
    currentTask?.cancel()
    currentTask = Task { detail = await loader.loadDetail(for: new) }
}
```

This replaces the manual "cancel the old `Task`, start a new one in `.onChange`" boilerplate. (See also `state-management.md`, which uses `task(id:)` for reactive model initialization.)

**Rule:** For fetches keyed to a changing value, use `.task(id:)` rather than hand-rolling cancellation in `.onChange`.

**Why:** `.task(id:)` automatically cancels the superseded task and restarts on change, eliminating a class of leak/race bugs.

---

## .onChange for Synchronous Reactions

`.onChange(of:)` is for *synchronous* side effects when a value changes — syncing derived state, not async loading. By default it skips the initial value; pass `initial: true` to also run once on first appearance.

```swift
// GOOD - sync derived state; runs on first appearance too
.onChange(of: query, initial: true) { oldValue, newValue in
    filtered = allItems.filter { $0.matches(newValue) }
}
```

The closure receives `(oldValue, newValue)`. For asynchronous work driven by a change, prefer `.task(id:)` instead — `.onChange` does not give you a managed async context or cancellation.

**Rule:** Use `.onChange(of:initial:)` for synchronous derived-state updates; pass `initial: true` when the derived value must be correct on the first frame.

**Why:** `.onChange` runs sync side effects and skips the initial value by default; `initial: true` seeds derived state once at appearance.

---

## Keep Observed Identifiers Cheap

SwiftUI evaluates the observed value on every update to decide whether it changed. If you observe a large array or a model with an expensive `Equatable`, that comparison runs on the main thread on each pass and drops frames.

```swift
// GOOD - observe a cheap, stable proxy
.task(id: items.count) { /* ... */ }
.onChange(of: model.version) { /* ... */ }

// BAD - comparing the whole collection on the main thread each update
.task(id: items) { /* ... */ }
.onChange(of: hugeModel) { /* ... */ }
```

Observe a stable id, a `count`, or a monotonically-incremented version counter — anything `O(1)` to compare — never the whole collection or an expensive-Equatable value.

**Rule:** Make `id:`/`of:` a cheap-to-compare proxy (id, count, version), not a large array or expensive-Equatable model.

**Why:** The equality check runs on the main thread every update; an expensive comparison there drops frames.

---

## Offloading Heavy Work with @concurrent

**Diagnostic:** "I wrapped my parsing in a `Task`, but the UI still hangs." Wrapping work in `Task {}` or `.task` does **not** move it off the main thread. A task created in a `@MainActor` context — a SwiftUI view, or an `@Observable @MainActor` model held in `@State` — inherits that isolation. Heavy synchronous compute (decode, parse, image processing) then runs on the main actor and freezes the UI.

System APIs like `URLSession` offload themselves; your own CPU-bound function will not. Under Swift 6.2's approachable-concurrency default (`NonisolatedNonsendingByDefault`, SE-0461), even a plain `nonisolated async` function runs on the *caller's* actor. To actually leave the main actor, mark the function `@concurrent` so it runs on the global concurrent executor; `await` it and assign the result back on the main actor.

```swift
// GOOD - heavy work runs off the main actor, result lands back on it
@concurrent
func decode(_ data: Data) async throws -> [Item] {
    try JSONDecoder().decode([Item].self, from: data)  // off main
}

// in the @MainActor model / view:
let parsed = try await decode(data)
self.items = parsed  // back on the main actor

// BAD - inherits @MainActor isolation; parse hangs the UI despite the Task
.task { self.items = try? heavyDecode(data) }  // runs ON main
```

Constraints: `@concurrent` applies only to `async` functions and implies `nonisolated`.

**Rule:** Mark your own CPU-bound async functions `@concurrent` to offload them; do not assume `Task`/`.task` leaves the main actor.

**Why:** Tasks inherit the enclosing actor's isolation, and under `NonisolatedNonsendingByDefault` even `nonisolated async` stays on the caller — only `@concurrent` (or a system API that offloads itself) reaches the global executor.

---

## Cooperative Cancellation

Cancellation in Swift is cooperative: cancelling a task only *sets a flag*. The work keeps running until it checks. `.task` and `.task(id:)` signal cancellation for you, but a long-running `@concurrent` function must check that flag itself to actually stop early.

```swift
// GOOD - bails out before each expensive step
@concurrent
func process(_ pages: [Page]) async throws -> Result {
    for page in pages {
        try Task.checkCancellation()   // throws CancellationError if cancelled
        accumulate(page)
    }
    return result
}

// BAD - ignores cancellation; keeps churning after the view is gone
@concurrent
func process(_ pages: [Page]) async -> Result {
    for page in pages { accumulate(page) }  // runs to completion regardless
    return result
}
```

Use `try Task.checkCancellation()` before expensive steps and inside loops; use the non-throwing `Task.isCancelled` when you need to clean up rather than throw.

**Rule:** In long `@concurrent` work, call `try Task.checkCancellation()` (or check `Task.isCancelled`) before each expensive step so cancellation actually takes effect.

**Why:** Cancellation only flips a flag; without an explicit check the work runs to completion even after SwiftUI has cancelled the task.

---

## Summary Checklist

- [ ] Async load-on-appear uses `.task {}`, never `.onAppear { Task { } }`
- [ ] `.onAppear` is reserved for synchronous seeding and guarded against re-fire
- [ ] An explicit loading/empty placeholder covers the window before `.task` resolves
- [ ] Fetch-on-change uses `.task(id:)`, not a manual cancel/restart in `.onChange`
- [ ] `.onChange(of:initial:)` is used only for synchronous side effects; `initial: true` when first-frame correctness matters
- [ ] Observed `id:`/`of:` values are cheap to compare (id, count, version) — never a large array
- [ ] CPU-bound custom async work is marked `@concurrent` to leave the main actor
- [ ] Results from `@concurrent` work are assigned back on the main actor
- [ ] Long `@concurrent` work calls `try Task.checkCancellation()` before expensive steps
