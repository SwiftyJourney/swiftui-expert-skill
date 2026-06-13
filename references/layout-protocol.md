# SwiftUI Layout & the Layout Protocol

How SwiftUI sizes and positions views — the negotiation model, why `.frame()` proposes rather than constrains, and building custom containers with the `Layout` protocol.

## Table of Contents

- [The Layout Negotiation Model](#the-layout-negotiation-model)
- [`.frame()` Is a View, Not a Constraint](#frame-is-a-view-not-a-constraint)
- [How Stacks Divide Space](#how-stacks-divide-space)
- [The Layout Protocol](#the-layout-protocol)
- [Caching Measurements](#caching-measurements)
- [Optional Layout Members](#optional-layout-members)
- [Custom Alignment Guides](#custom-alignment-guides)
- [Switching Layouts at Runtime](#switching-layouts-at-runtime)
- [visionOS: Depth Alignment](#visionos-depth-alignment)
- [Summary Checklist](#summary-checklist)

---

## The Layout Negotiation Model

SwiftUI layout is a negotiation, not a command. Three steps repeat all the way down the view tree:

1. The **parent proposes** a size to the child (a `ProposedViewSize`, which may carry `nil`, `.zero`, or `.infinity` for either dimension).
2. The **child chooses** its own size and is free to ignore the proposal entirely.
3. The **parent positions** the child — but must respect the size the child chose.

The parent never forces a size onto a child. Understanding this is the key to every "why isn't this view the size I asked for?" bug.

**Rule:** Reason about layout as proposal → choice → placement, top-down.

**Why:** A view that ignores its proposal (intrinsic content like `Text`) behaves completely differently from one that fills it (`Color`), and only the negotiation model explains the difference.

## `.frame()` Is a View, Not a Constraint

`.frame()` does not clamp a view's size. It inserts an invisible container view that proposes a size to its child, then sizes *itself* to the child's choice. Flexible children (`Color`, `Rectangle`, `Shape`) accept the proposal and fill it. Intrinsic children (a non-`resizable()` `Image`, `Text`) report their own size and can overflow the frame.

```swift
// BAD — the image keeps its native pixel size and overflows the frame
Image("hero")
    .frame(width: 120, height: 120)   // frame proposes 120×120; image ignores it

// GOOD — make the image flexible FIRST, then propose a frame it will honor
Image("hero")
    .resizable()
    .scaledToFit()
    .frame(width: 120, height: 120)
```

**Rule:** Fix an overflowing image with `.resizable().scaledToFit()` *before* `.frame()`, not by enlarging the frame. Modifier order is significant — `.frame()` only affects the view beneath it in the chain.

**Why:** `.frame()` proposes; it never constrains. An intrinsic view that refuses the proposal overflows no matter how large you make the frame.

## How Stacks Divide Space

`HStack` and `VStack` do **not** split available space evenly. They serve the **least flexible** child first, subtract what it took, then divide the *remainder* among the more flexible ones (in increasing order of flexibility). A child pinned with a fixed `.frame` is the least flexible, so it is served before any flexible sibling.

```swift
// "Why does the spacer/color look squeezed?"
HStack {
    Color.red                 // flexible — gets the LEFTOVER space
    Color.blue
        .frame(width: 250)    // fixed → served first, claims 250pt up front
}
```

The blue view takes its 250pt before red is measured; red receives only what is left. Raise a view's `.layoutPriority(_:)` to have it served earlier in this ordering.

**Rule:** When one fixed-size sibling appears to "steal" space, remember the stack served it first and divided the remainder — not the whole — among the rest.

**Why:** The even-split mental model is wrong and leads to fighting the stack with nested frames instead of adjusting flexibility or priority.

## The Layout Protocol

When stacks and grids can't express a design — wrapping chips, a radial menu, a diagonal cascade — conform to `Layout` (iOS 16+). You implement two **required** methods and never touch the child views directly: you operate on `LayoutSubview` proxies, measuring each with `subview.sizeThatFits(_:)` and positioning each with `subview.place(at:anchor:proposal:)`.

```swift
// A minimal flow layout: lay subviews left-to-right, wrap to a new row when full.
struct FlowLayout: Layout {
    var spacing: CGFloat = 8

    func sizeThatFits(
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) -> CGSize {
        let maxWidth = proposal.width ?? .infinity
        var x: CGFloat = 0, y: CGFloat = 0, rowHeight: CGFloat = 0
        for subview in subviews {
            let size = subview.sizeThatFits(.unspecified)
            if x + size.width > maxWidth {     // wrap
                x = 0
                y += rowHeight + spacing
                rowHeight = 0
            }
            x += size.width + spacing
            rowHeight = max(rowHeight, size.height)
        }
        return CGSize(width: maxWidth, height: y + rowHeight)
    }

    func placeSubviews(
        in bounds: CGRect,
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) {
        var x = bounds.minX, y = bounds.minY, rowHeight: CGFloat = 0
        for subview in subviews {
            let size = subview.sizeThatFits(.unspecified)
            if x + size.width > bounds.maxX {  // wrap
                x = bounds.minX
                y += rowHeight + spacing
                rowHeight = 0
            }
            subview.place(at: CGPoint(x: x, y: y), anchor: .topLeading, proposal: .unspecified)
            x += size.width + spacing
            rowHeight = max(rowHeight, size.height)
        }
    }
}

// Use it exactly like a built-in container, via its @ViewBuilder closure:
FlowLayout(spacing: 8) {
    ForEach(tags, id: \.self) { Text($0).padding(6).background(.quaternary, in: Capsule()) }
}
```

The exact required signatures are:

```swift
func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout Cache) -> CGSize
func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout Cache)
```

**Rule:** Conform to `Layout` only when stacks/grids genuinely can't express the geometry. Measure via `subview.sizeThatFits(_:)`, place via `subview.place(at:anchor:proposal:)`, and call `place` for *every* subview in `placeSubviews` — an unplaced subview renders at the origin.

**Why:** A custom container that nests stacks for a flow/radial design fights SwiftUI's geometry; `Layout` lets you compute positions directly while staying a first-class, composable container.

### `Cache` Defaults to `Void`

`associatedtype Cache = Void`. If you don't need a cache, the type is `()` — write `cache: inout ()` in both method signatures (as above). Don't declare a `typealias Cache` or implement `makeCache` unless you actually memoize something.

## Caching Measurements

`sizeThatFits` and `placeSubviews` can each be called **multiple times in a single layout pass**. Measuring every subview on every call is the classic performance trap. Implement `makeCache(subviews:)` to compute expensive values once, and `updateCache(_:subviews:)` to refresh them when the subviews change.

```swift
// BAD — re-measures every subview on every (possibly repeated) call
func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
    let sizes = subviews.map { $0.sizeThatFits(.unspecified) }  // recomputed each call
    // ...
}

// GOOD — measure once into a cache, reuse across calls
struct CachedFlow: Layout {
    struct Cache { var sizes: [CGSize] }

    func makeCache(subviews: Subviews) -> Cache {
        Cache(sizes: subviews.map { $0.sizeThatFits(.unspecified) })
    }
    func updateCache(_ cache: inout Cache, subviews: Subviews) {
        cache.sizes = subviews.map { $0.sizeThatFits(.unspecified) }
    }
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout Cache) -> CGSize {
        // read cache.sizes instead of re-measuring
        cache.sizes.reduce(.zero) { CGSize(width: max($0.width, $1.width), height: $0.height + $1.height) }
    }
    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout Cache) { /* ... */ }
}
```

**Rule:** Cache only genuinely expensive measurements, and recompute in `updateCache` so a changed subview set doesn't serve stale sizes.

**Why:** The layout engine may call your sizing and placement methods repeatedly per pass; recomputing measurements each time turns an O(n) layout into O(n × calls).

## Optional Layout Members

Beyond caching, `Layout` exposes optional members with default implementations:

- `func spacing(subviews: Subviews, cache: inout Cache) -> ViewSpacing` — the preferred spacing your container wants around itself relative to neighbors.
- `func explicitAlignment(of:in:proposal:subviews:cache:)` — expose explicit horizontal/vertical alignment guides so the container participates in a parent's alignment.
- `static var layoutProperties: LayoutProperties` — declare a stack-like orientation axis so SwiftUI can treat your container accordingly.

**Rule:** Implement these only when your container needs to advertise spacing, alignment, or an axis to its surroundings; otherwise the defaults are correct.

**Why:** Adding them blindly invites subtle interaction bugs with parent containers; the defaults already behave like an opaque, axis-less box.

## Custom Alignment Guides

To align views that are **not** in the same stack — e.g. a label in one row with a value in another — define a custom alignment. Conform a type to `AlignmentID` (implementing `defaultValue(in: ViewDimensions)`), wrap it in a `HorizontalAlignment`/`VerticalAlignment` static constant, then set `.alignmentGuide(_:computeValue:)` on the participating children.

```swift
private struct LabelAlignmentID: AlignmentID {
    static func defaultValue(in context: ViewDimensions) -> CGFloat {
        context[.leading]
    }
}

extension HorizontalAlignment {
    static let labelEdge = HorizontalAlignment(LabelAlignmentID.self)
}

// Children opt in by reporting their guide value:
VStack(alignment: .labelEdge) {
    HStack { Text("Name").alignmentGuide(.labelEdge) { $0[.trailing] }; Text(name) }
    HStack { Text("Email").alignmentGuide(.labelEdge) { $0[.trailing] }; Text(email) }
}
```

**Rule:** Use a custom `AlignmentID` to align elements across separate containers; the `computeValue` closure receives `ViewDimensions`, so derive the guide from the view's own edges/baselines, not a hardcoded number.

**Why:** Built-in guides only align siblings within one stack; a custom alignment ID is the supported way to thread one alignment through multiple containers.

## Switching Layouts at Runtime

To swap container *type* (e.g. `HStack` ↔ `VStack`) while preserving subview identity and animating the transition, use `AnyLayout` — covered in `performance-patterns.md`. Don't branch with `if/else` over two different containers, which destroys identity.

**Rule:** Reach for `AnyLayout` (see `performance-patterns.md`) when the container type itself is state-dependent.

## visionOS: Depth Alignment

On visionOS, `Layout` gains a depth (z) axis. `SpatialContainer` arranges subviews in 3D, and `DepthAlignment` (with `DepthAlignmentID`) aligns them along z, mirroring the 2D alignment-guide model. Gate any such code behind `#available` / `@available`.

```swift
if #available(visionOS 1.0, *) {
    SpatialContainer(alignment: .center) { /* depth-aware content */ }
}
```

**Rule:** Treat depth alignment as the z-axis analogue of custom alignment guides, and always gate it behind availability.

## Summary Checklist

- [ ] Reason about layout as parent **proposes** → child **chooses** → parent **places**
- [ ] Remember `.frame()` proposes a size; it does not constrain — flexible children fill it, intrinsic ones can overflow
- [ ] Fix overflowing images with `.resizable().scaledToFit()` *before* `.frame()`, respecting modifier order
- [ ] Know stacks serve the least-flexible child first, then divide the *remainder*; adjust with `.layoutPriority(_:)`
- [ ] Conform to `Layout` only when stacks/grids can't express the geometry
- [ ] Implement the two required methods exactly: `sizeThatFits(proposal:subviews:cache:)` and `placeSubviews(in:proposal:subviews:cache:)`
- [ ] Measure via `subview.sizeThatFits(_:)`, place via `subview.place(at:anchor:proposal:)`, and place *every* subview
- [ ] Write `cache: inout ()` when you don't need a cache (`Cache = Void`)
- [ ] Use `makeCache`/`updateCache` to memoize measurements — the methods run multiple times per pass
- [ ] Add `spacing`/`explicitAlignment`/`layoutProperties` only when the container must advertise them
- [ ] Use a custom `AlignmentID` to align views across separate containers
- [ ] Use `AnyLayout` (see `performance-patterns.md`) to switch container type while preserving identity
- [ ] Gate `SpatialContainer`/`DepthAlignment` behind availability on visionOS
