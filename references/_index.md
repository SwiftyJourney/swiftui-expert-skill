# Reference Index

Quick navigation for the SwiftUI Expert skill.

## State & Data Flow

| File | Use it for |
|---|---|
| `state-management.md` | property wrappers, `@Observable`, `@Binding`, `@Bindable`, environment, when to use a ViewModel |

## View Structure

| File | Use it for |
|---|---|
| `view-composition.md` | subview extraction, `@ViewBuilder`, container views, custom styles, specialized components, reusability |
| `list-patterns.md` | stable ForEach identity, `Identifiable`, `LazyVStack` trade-offs |

## Modern APIs & Navigation

| File | Use it for |
|---|---|
| `modern-apis.md` | deprecated API replacements, iOS 18/26 modern equivalents |
| `navigation-patterns.md` | `NavigationStack`, `navigationDestination`, deep linking, programmatic navigation |
| `scroll-patterns.md` | `scrollPosition`, `scrollTargetBehavior`, programmatic scrolling |
| `text-formatting.md` | `AttributedString`, modern format parameters, `textRenderer` |

## Performance & Optimization

| File | Use it for |
|---|---|
| `performance-patterns.md` | view initialization costs, structural identity, dependency narrowing, geometry observation |
| `image-optimization.md` | `AsyncImage`, image downsampling, caching strategies |

## Animation

| File | Use it for |
|---|---|
| `animation-basics.md` | core concepts, implicit vs explicit animations, timing curves |
| `animation-transitions.md` | `AnyTransition`, combined transitions, `matchedGeometryEffect`, `Animatable` |
| `animation-advanced.md` | transactions, `PhaseAnimator`, `KeyframeAnimator`, completion handlers |

## Accessibility & Platform

| File | Use it for |
|---|---|
| `accessibility-patterns.md` | semantic styling, `@ScaledMetric`, `accessibilityRepresentation`, VoiceOver |
| `liquid-glass.md` | iOS 26+ Liquid Glass API, `GlassEffectContainer`, HIG compliance |

## Problem Router

- "I need to choose a property wrapper" -> `state-management.md`
- "My view is too large and complex" -> `view-composition.md`
- "I'm using a deprecated API" -> `modern-apis.md`
- "My list is flickering or reordering" -> `list-patterns.md`
- "I need to add animations" -> `animation-basics.md`
- "My animation is janky or missing" -> `animation-basics.md`
- "I need complex multi-step animations" -> `animation-advanced.md`
- "I need hero transitions" -> `animation-transitions.md`
- "I need Liquid Glass styling" -> `liquid-glass.md`
- "My view re-renders too much" -> `performance-patterns.md`
- "I need to fix accessibility" -> `accessibility-patterns.md`
- "I need help with navigation or sheets" -> `navigation-patterns.md`
- "I need to optimize images" -> `image-optimization.md`
- "I need programmatic scrolling" -> `scroll-patterns.md`
- "I need modern text formatting" -> `text-formatting.md`
- "I have an architecture question" -> `../SKILL.md` (diagnostic table) or `ios-architecture-expert` skill
