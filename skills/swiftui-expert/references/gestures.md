# SwiftUI Gestures

Gesture precedence, modern gesture types, transient tracking with `@GestureState`, composition, and gesture-driven animation.

## Table of Contents
- [Gesture Precedence](#gesture-precedence)
- [Modern Gesture Types](#modern-gesture-types)
- [Transient Tracking with @GestureState](#transient-tracking-with-gesturestate)
- [Long Press: Perform vs Pressing](#long-press-perform-vs-pressing)
- [Tap Gestures and the Button Rule](#tap-gestures-and-the-button-rule)
- [Composing Gestures](#composing-gestures)
- [Gestures Driving Animation](#gestures-driving-animation)
- [Summary Checklist](#summary-checklist)

---

## Gesture Precedence

A plain `.gesture(_:)` attaches your gesture *below* a container's built-in gestures. Inside a `ScrollView` or `List`, the scroll gesture wins and your tap or drag may never fire. This is the single most common gesture bug.

| Modifier | Priority vs built-in gestures |
|----------|-------------------------------|
| `.gesture(_:)` | Lower — container gestures win |
| `.highPriorityGesture(_:)` | Higher — overrides the container |
| `.simultaneousGesture(_:)` | Parallel — both recognize at once |

```swift
// BAD - tap loses to the scroll gesture and never fires
ScrollView {
    Card().gesture(TapGesture().onEnded { open() })
}

// GOOD - run alongside scrolling (or use highPriorityGesture to override)
ScrollView {
    Card().simultaneousGesture(TapGesture().onEnded { open() })
}
```

Use `.gesture(_:including:)` with a `GestureMask` (`.all`, `.gesture`, `.subviews`, `.none`) to decide which gestures stay active in a subtree — e.g. `.gesture(drag, including: .gesture)` disables child gestures while yours is recognizing.

**Rule:** Inside a scrollable container, never rely on plain `.gesture(_:)`; reach for `.highPriorityGesture` (override) or `.simultaneousGesture` (coexist).

---

## Modern Gesture Types

The continuous gestures are `DragGesture`, `MagnifyGesture`, and `RotateGesture` (plus `RotateGesture3D` for spatial/visionOS). The older `MagnificationGesture` and `RotationGesture` are deprecated — flag them in reviews.

```swift
// BAD - deprecated names
.gesture(MagnificationGesture().onChanged { scale = $0 })
.gesture(RotationGesture().onChanged { angle = $0 })

// GOOD - current types
.gesture(MagnifyGesture().onChanged { scale = $0.magnification })
.gesture(RotateGesture().onChanged { angle = $0.rotation })
```

**Rule:** Use `MagnifyGesture`/`RotateGesture`; treat `MagnificationGesture`/`RotationGesture` as code smells to migrate.

---

## Transient Tracking with @GestureState

For values that exist only while a finger is down — a live drag offset, an in-progress scale — use `@GestureState` with `updating(_:body:)`. It snaps back to the initial value the instant the gesture ends, so the view returns to rest with zero cleanup code.

```swift
// BAD - @State persists after release; you must reset it by hand
@State private var offset: CGSize = .zero
DragGesture()
    .onChanged { offset = $0.translation }
    .onEnded { _ in offset = .zero }   // easy to forget

// GOOD - @GestureState auto-resets on end
@GestureState private var offset: CGSize = .zero
someView
    .offset(offset)
    .gesture(
        DragGesture().updating($offset) { value, state, _ in
            state = value.translation
        }
    )
```

**Why:** `updating` writes into a transient store SwiftUI owns; on gesture end it restores the initial value automatically, eliminating a whole class of "stuck after release" bugs. Keep `@State` only for values that must survive the gesture.

---

## Long Press: Perform vs Pressing

`onLongPressGesture` exposes two callbacks. `perform:` fires once, only after a *successful* long press. `onPressingChanged:` fires `true` when the press begins and `false` when it ends — use it for live feedback while the user is holding. Releasing before `minimumDuration` skips `perform:` entirely.

```swift
// GOOD - in-progress feedback plus a committed action
@State private var isPressing = false
someView
    .scaleEffect(isPressing ? 0.94 : 1)
    .onLongPressGesture(minimumDuration: 0.5) {
        commit()                       // only on success
    } onPressingChanged: { pressing in
        withAnimation { isPressing = pressing }   // begins/ends
    }
```

**Rule:** Drive hold-down visuals from `onPressingChanged`, not `perform:` — `perform:` never runs on an early release.

---

## Tap Gestures and the Button Rule

Prefer `Button` over `onTapGesture` for ordinary taps — `Button` gives you the accessibility trait, pressed state, and hit testing for free. The legitimate exceptions are multiple-tap counts and needing the tap location, which `Button` can't express.

```swift
// GOOD - stacked counts is a real reason to use onTapGesture
someView
    .onTapGesture(count: 2) { like() }     // most specific first
    .onTapGesture { select() }

// GOOD - location in a coordinate space (Button can't do this)
someView.onTapGesture(coordinateSpace: .local) { location in
    addPin(at: location)
}
```

**Rule:** Reach for `onTapGesture` only when you need `count:` or `coordinateSpace`/location; otherwise use `Button`.

---

## Composing Gestures

The `Gesture` protocol composes single gestures into one compound gesture: `simultaneously(with:)`, `sequenced(before:)`, `exclusively(before:)`, and `modifiers(_:)` (require a keyboard modifier). These differ from the view modifiers in [Gesture Precedence](#gesture-precedence): the operators build *one* gesture; the modifiers decide how a gesture ranks against a *container's* built-ins.

```swift
// GOOD - pinch and rotate together as a single recognized gesture
let transform = MagnifyGesture()
    .simultaneously(with: RotateGesture())

// GOOD - long press must succeed before the drag begins
let pickUpAndMove = LongPressGesture()
    .sequenced(before: DragGesture())
```

**Rule:** Use the `Gesture`-protocol operators to build a compound gesture; use the `.gesture`/`.highPriorityGesture`/`.simultaneousGesture` *view* modifiers to rank against the container.

---

## Gestures Driving Animation

Fan one `@GestureState` offset into several followers, each animating with an index-scaled delay. The leader tracks the finger 1:1 while followers lag behind for a trailing/elastic feel.

```swift
struct TrailingDots: View {
    @GestureState private var drag: CGSize = .zero

    var body: some View {
        ZStack {
            ForEach(0..<5) { i in
                Circle().frame(width: 24)
                    .offset(drag)
                    // later dots lag more; auto-resets to .zero on release
                    .animation(.linear.delay(Double(i) / 20), value: drag)
            }
        }
        .gesture(
            DragGesture().updating($drag) { value, state, _ in
                state = value.translation
            }
        )
    }
}
```

**Why:** A shared `@GestureState` keeps every follower reading the same live value, while the per-index `.animation(_:value:)` delay staggers when each one catches up — and the auto-reset makes the whole trail spring home on release.

---

## Summary Checklist

- [ ] Inside `ScrollView`/`List`, use `.highPriorityGesture` or `.simultaneousGesture`, not plain `.gesture`
- [ ] Use `GestureMask` via `.gesture(_:including:)` to scope which gestures stay active
- [ ] Use `MagnifyGesture`/`RotateGesture`; migrate off deprecated `MagnificationGesture`/`RotationGesture`
- [ ] Track transient drag/scale with `@GestureState` + `updating(_:body:)` for free auto-reset
- [ ] Keep `@State` only for values that must outlive the gesture
- [ ] Use `onLongPressGesture`'s `onPressingChanged:` for hold feedback, `perform:` for the committed action
- [ ] Prefer `Button`; use `onTapGesture` only for `count:` or location/`coordinateSpace`
- [ ] Compose with `simultaneously`/`sequenced`/`exclusively`/`modifiers` for one compound gesture
- [ ] Share a single `@GestureState` across followers with index-scaled `.animation` delays for trailing effects
