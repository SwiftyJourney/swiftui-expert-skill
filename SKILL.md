---
name: swiftui-expert
version: 2.0.0
description: Write, review, or improve SwiftUI code following best practices
  for state management, view composition, performance, accessibility, and modern APIs.
  Use when building SwiftUI features, refactoring views, reviewing code quality,
  or adopting modern SwiftUI patterns.
author: SwiftyJourney
tags:
  - swiftui
  - ios
  - apple
  - state-management
  - view-composition
  - performance
  - animations
  - accessibility
  - liquid-glass
  - swift
---

# SwiftUI Expert Skill

## Agent Behavior Contract

When this skill is active, follow these rules **strictly**:

1. **Focus on facts and best practices, not architecture** — defer architectural decisions (MVVM, MVC, VIPER, Clean Architecture, Coordinator) to `ios-architecture-expert`.
2. **Always prefer `@Observable` over `ObservableObject`** for new code. Mark `@Observable` classes with `@MainActor` unless using default actor isolation.
3. **Never use `applyIf` or generic conditional modifier helpers** — they break view identity.
4. **Prefer modifiers over conditional views** for state changes (preserves structural identity).
5. **Use modern APIs over deprecated equivalents** — see `references/modern-apis.md`.
6. **Lead with corrected/improved code** when reviewing; explain *why* a pattern is better (diffing, thread safety, API correctness).
7. **Use "prefer"/"avoid"** for recommendations; **"always"/"never"** only for API correctness.
8. **Respond in the same language the user writes in** — default to English when uncertain.
9. **Adopt Liquid Glass only when explicitly requested** — glass is for controls and navigation only (HIG).
10. **Gate iOS 26+ features** with `#available` and provide fallbacks.

---

## Diagnostic Table

| Symptom | First check | Smallest fix | Deep dive |
|---|---|---|---|
| Wrong property wrapper chosen | Ownership direction | Match wrapper to data flow | `references/state-management.md` |
| View re-renders too often | Dependency graph | Narrow state dependencies | `references/performance-patterns.md` |
| Deprecated API warning | API replacement table | Swap to modern equivalent | `references/modern-apis.md` |
| View body is 200+ lines | View extraction rules | Extract struct subviews | `references/view-composition.md` |
| ForEach flickers or reorders | Identity stability | Use stable Identifiable ID | `references/list-patterns.md` |
| Animation is janky or missing | Implicit vs explicit | Match animation to state change site | `references/animation-basics.md` |
| Glass effect on wrong layer | HIG compliance | Glass on controls only, not content | `references/liquid-glass.md` |
| Custom control lacks accessibility | Semantic components | Use Button/Label or `accessibilityRepresentation` | `references/accessibility-patterns.md` |

---

## Quick Summary

| Area | Key Principle | Reference |
|------|--------------|-----------|
| State Management | `@Observable` + `@MainActor`; `@State` for owned, `@Binding` for child-modifies-parent | `references/state-management.md` |
| View Composition | Small single-responsibility views; extract structs, not `@ViewBuilder` functions | `references/view-composition.md` |
| Modern APIs | `foregroundStyle`, `clipShape`, `NavigationStack`, `Tab`, `Button` over `onTapGesture` | `references/modern-apis.md` |
| Performance | Cheap inits, pure body, narrow dependencies, stable ForEach identity | `references/performance-patterns.md` |
| Animations | `withAnimation` for events, `.animation(_:value:)` for reactive, transforms over layout | `references/animation-basics.md` |
| Accessibility | Semantic fonts/styles, `@ScaledMetric`, `accessibilityRepresentation` for custom controls | `references/accessibility-patterns.md` |
| Liquid Glass | Controls/navigation only (HIG); `Glass.regular` default; `GlassEffectContainer` for groups | `references/liquid-glass.md` |

---

## Quick Reference

### Property Wrapper Selection

| Wrapper | Use When |
|---------|----------|
| `@State` | Internal view state (must be `private`), or owned `@Observable` class |
| `@Binding` | Child modifies parent's state |
| `@Bindable` | Injected `@Observable` needing bindings |
| `let` | Read-only value from parent |
| `var` | Read-only value watched via `.onChange()` |

> Full guide including legacy wrappers: [state-management.md](references/state-management.md)

### Modern API Replacements

| Deprecated | Modern Alternative |
|------------|-------------------|
| `foregroundColor()` | `foregroundStyle()` |
| `cornerRadius()` | `clipShape(.rect(cornerRadius:))` |
| `tabItem()` | `Tab` API (iOS 18+) |
| `onTapGesture()` | `Button` (unless need location/count) |
| `NavigationView` | `NavigationStack` |
| `onChange(of:) { value in }` | `onChange(of:) { old, new in }` or `onChange(of:) { }` |
| `GeometryReader` | `containerRelativeFrame()` or `visualEffect()` |

> Full table: [modern-apis.md](references/modern-apis.md)

---

## Workflow Decision Tree

### 1) Review existing SwiftUI code
- Check property wrapper usage -> `references/state-management.md`
- Verify modern API usage -> `references/modern-apis.md`
- Verify view composition and extraction -> `references/view-composition.md`
- Check performance patterns -> `references/performance-patterns.md`
- Inspect accessibility and Liquid Glass -> `references/accessibility-patterns.md`, `references/liquid-glass.md`

### 2) Improve existing SwiftUI code
- Audit state management (prefer `@Observable` over `ObservableObject`) -> `references/state-management.md`
- Replace deprecated APIs -> `references/modern-apis.md`
- Extract complex views into subviews -> `references/view-composition.md`
- Optimize hot paths and list identity -> `references/performance-patterns.md`, `references/list-patterns.md`
- Improve animations and accessibility -> `references/animation-basics.md`, `references/accessibility-patterns.md`

### 3) Implement new SwiftUI feature
- Design data flow first: owned vs injected state -> `references/state-management.md`
- Use modern APIs and `@Observable` -> `references/modern-apis.md`
- Structure views for optimal diffing -> `references/view-composition.md`
- Apply correct animation patterns -> `references/animation-basics.md`, `references/animation-advanced.md`
- Add accessibility and glass effects (if requested) -> `references/accessibility-patterns.md`, `references/liquid-glass.md`

---

## Guardrails

- Do not enforce specific architectures (MVVM, MVC, VIPER) — defer to `ios-architecture-expert`
- Do not use `applyIf` or conditional modifier helpers that break view identity
- Do not declare passed values as `@State` or `@StateObject` — they only accept initial values
- Do not create objects or start Tasks in view initializers or `body`
- Do not use `.indices` for ForEach identity with dynamic content
- Do not apply glass effects to content layer views — glass is for controls and navigation only (HIG)
- Do not use `AnyView` in list rows
- Do not use `GeometryReader` when `containerRelativeFrame()`, `onGeometryChange`, or `visualEffect` suffice

---

## Verification Checklist

When implementing or reviewing SwiftUI code:

1. Property wrappers match ownership direction (`@State` private, `@Binding` for modification, `@Observable` for shared)
2. No deprecated APIs (`foregroundColor`, `cornerRadius`, `NavigationView`, `tabItem`)
3. View body is pure — no side effects, no Task creation, no object allocation
4. Complex views extracted into struct subviews (not `@ViewBuilder` functions)
5. ForEach uses stable `Identifiable` identity (never `.indices`)
6. Animations use value parameter or `withAnimation` (no deprecated parameterless `.animation()`)
7. Accessibility: semantic fonts, `@ScaledMetric`, `Button` over `onTapGesture`
8. Liquid Glass (if present): `#available` gated, controls-only, `GlassEffectContainer` for groups
9. No `applyIf`, no conditional modifier helpers, no if/else for style changes
10. Views work in any context (no hardcoded sizes, no `UIScreen.main.bounds`)

---

## Reference Router

Open the smallest reference that matches the question:

- **State & Data Flow**
  - [state-management.md](references/state-management.md) — property wrappers, data flow, when to use a ViewModel
- **View Structure**
  - [view-composition.md](references/view-composition.md) — view extraction, styles, specialized views, reusability
  - [list-patterns.md](references/list-patterns.md) — ForEach identity, stability, list best practices
- **Modern APIs & Navigation**
  - [modern-apis.md](references/modern-apis.md) — modern API usage and deprecated replacements
  - [navigation-patterns.md](references/navigation-patterns.md) — NavigationStack, deep linking, programmatic navigation
  - [scroll-patterns.md](references/scroll-patterns.md) — ScrollView patterns and programmatic scrolling
  - [text-formatting.md](references/text-formatting.md) — modern text formatting and string operations
- **Performance & Optimization**
  - [performance-patterns.md](references/performance-patterns.md) — optimization techniques and anti-patterns
  - [image-optimization.md](references/image-optimization.md) — AsyncImage, image downsampling
- **Animation**
  - [animation-basics.md](references/animation-basics.md) — core concepts, implicit/explicit animations
  - [animation-transitions.md](references/animation-transitions.md) — transitions, custom transitions, Animatable
  - [animation-advanced.md](references/animation-advanced.md) — transactions, phase/keyframe, completion handlers
- **Accessibility & Platform**
  - [accessibility-patterns.md](references/accessibility-patterns.md) — semantic styling, adaptive spacing, VoiceOver
  - [liquid-glass.md](references/liquid-glass.md) — iOS 26+ Liquid Glass API and HIG guidance
- **Navigation index**
  - [references/_index.md](references/_index.md)
