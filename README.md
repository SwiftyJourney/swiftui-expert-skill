# SwiftUI Expert Agent Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Agent Skills](https://img.shields.io/badge/Agent-Skills-blue)](https://agentskills.io)

A Claude Code skill that guides writing, reviewing, and improving SwiftUI code — correct state management, modern APIs, view composition, performance, animations, accessibility, and iOS 26+ Liquid Glass styling.

## Who is this for?

iOS developers who want AI-assisted guidance on:

- Choosing the right property wrapper (`@State`, `@Binding`, `@Observable`, `@Bindable`)
- Extracting and composing views following best practices
- Replacing deprecated APIs with modern equivalents (iOS 18/26)
- Optimizing performance and list identity
- Loading data correctly (`.task`/`.task(id:)`, offloading heavy work with `@concurrent`)
- Applying correct animation and gesture patterns
- Building custom layouts with the `Layout` protocol
- Structuring app lifecycle and scenes, and persisting state (`@AppStorage`/`@SceneStorage`/SwiftData)
- Adding accessibility and adaptive layout
- Adopting Liquid Glass (iOS 26+) following HIG

## Installation

Install via [skills.sh](https://skills.sh):

```bash
npx skills add SwiftyJourney/swiftui-expert-skill
```

## Compatible Agents

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Cursor](https://cursor.sh)
- [GitHub Copilot](https://github.com/features/copilot)
- [Windsurf](https://codeium.com/windsurf)
- Any agent supporting the `skills/` convention

## Quick Start

Try these prompts:

```
Review this SwiftUI view for state management issues.
```

```
This view body is 200 lines. Help me break it down.
```

```
Update this code to use modern SwiftUI APIs.
```

The agent will follow the skill's workflow decision tree, reference the correct guide, provide corrected code, and explain *why* each pattern is preferred.

## Skill File Structure

```
SKILL.md                            # Hub: behavior contract, diagnostic table, reference router
references/
├── state-management.md             # Property wrapper selection guide
├── data-persistence.md             # @AppStorage, @SceneStorage, SwiftData, PreferenceKey
├── data-loading-and-tasks.md       # .task/.task(id:) entry points, @concurrent, cancellation
├── view-composition.md             # View extraction rules and patterns
├── layout-protocol.md              # Layout negotiation, custom Layout, alignment guides
├── modern-apis.md                  # iOS 18/26 API replacements
├── performance-patterns.md         # Optimization techniques and anti-patterns
├── list-patterns.md                # Stable identity, ForEach, LazyVStack
├── navigation-patterns.md          # NavigationStack, programmatic navigation
├── app-lifecycle-and-scenes.md     # App protocol, scenes, WindowGroup, scenePhase
├── animation-basics.md             # withAnimation, implicit/explicit
├── animation-transitions.md        # AnyTransition, matched geometry
├── animation-advanced.md           # PhaseAnimator, KeyframeAnimator
├── gestures.md                     # Gesture types, @GestureState, composition & precedence
├── scroll-patterns.md              # scrollPosition, scrollTargetBehavior
├── text-formatting.md              # AttributedString, formatting, localization
├── image-optimization.md           # AsyncImage, caching strategies
├── accessibility-patterns.md       # Semantic styling, @ScaledMetric, VoiceOver
└── liquid-glass.md                 # iOS 26 Liquid Glass API and HIG guidance
```

## Related Skills

- [iOS Architecture Expert](https://github.com/SwiftyJourney/ios-architecture-expert-skill) — Clean architecture, composition root, protocol-oriented design
- [Requirements Engineering](https://github.com/SwiftyJourney/requirements-engineering-skill) — BDD stories and use cases

## Credits

- **Natalia Panferova (Nil Coalescing)** — "SwiftUI Fundamentals" and "The SwiftUI Way", which informed the patterns, performance, layout, gesture, and lifecycle guidance
- **Matteo Manferdini** — SwiftUI view composition methodology
- **Apple** — SwiftUI documentation, WWDC sessions, and Human Interface Guidelines

## License

MIT — see [LICENSE](LICENSE).
