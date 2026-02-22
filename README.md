# SwiftUI Expert Agent Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Agent Skills](https://img.shields.io/badge/Agent-Skills-blue)](https://agentskills.io)

Write, review, or improve SwiftUI code following best practices for state management, view composition, performance, and modern APIs.

---

## Who This Is For

* **iOS Developers** building SwiftUI features from scratch
* **Swift Developers** refactoring UIKit or older SwiftUI code
* **Tech Leads** reviewing SwiftUI code quality on their team
* **Any Developer** adopting modern SwiftUI patterns (iOS 18/26)

---

## What This Skill Does

This skill guides AI agents through a structured workflow:

1. **Review existing code** → Check state, composition, APIs, performance, lists, animations, Liquid Glass
2. **Improve existing code** → Fix anti-patterns, modernize APIs, optimize layout, correct animations
3. **Implement new features** → Choose correct property wrappers, apply composition rules, use modern navigation

---

## Installation

### Option A: Using skills CLI (Recommended)

```bash
npx skills add SwiftyJourney/swiftui-expert-skill
```

This automatically installs the skill for your AI agent tools.

### Option B: Manual Installation

#### For Claude.ai

1. Download `swiftui-expert.skill` from [Releases](https://github.com/SwiftyJourney/swiftui-expert-skill/releases)
2. Upload to your Claude project

#### For Claude Code

```bash
git clone https://github.com/SwiftyJourney/swiftui-expert-skill.git
mkdir -p ~/.anthropic/skills
cp -r swiftui-expert-skill/swiftui-expert ~/.anthropic/skills/
```

#### For Cursor

```bash
git clone https://github.com/SwiftyJourney/swiftui-expert-skill.git
cp -r swiftui-expert-skill/swiftui-expert ~/Library/Application\ Support/Cursor/User/globalStorage/skills/
```

#### For Windsurf

```bash
git clone https://github.com/SwiftyJourney/swiftui-expert-skill.git
cp -r swiftui-expert-skill/swiftui-expert ~/.windsurf/skills/
```

See the [Installation Guide](docs/INSTALLATION.md) for detailed instructions.

---

## Quick Start

Try this prompt:

```plaintext
Review this SwiftUI view and tell me what's wrong with the state management.
```

The agent will:

* Check each property wrapper against the selection guide
* Identify `@StateObject` vs `@ObservedObject` mistakes
* Flag missing `@Observable` opportunities (iOS 17+)
* Suggest corrected code

---

## What's Included

```plaintext
swiftui-expert/
├── SKILL.md                        # Main skill (decision tree + workflow)
└── references/
    ├── state-management.md         # Property wrapper selection guide
    ├── view-composition.md         # Manferdini extraction rules
    ├── modern-apis.md              # iOS 18/26 API replacements
    ├── performance-patterns.md     # equatable, lazy loading, task(id:)
    ├── list-patterns.md            # Stable identity, ForEach, LazyVStack
    ├── navigation-patterns.md      # NavigationStack, programmatic nav
    ├── animation-basics.md         # withAnimation, implicit/explicit
    ├── animation-transitions.md    # AnyTransition, matched geometry
    ├── animation-advanced.md       # PhaseAnimator, KeyframeAnimator
    ├── scroll-patterns.md          # scrollPosition, scrollTargetBehavior
    ├── text-formatting.md          # AttributedString, textRenderer
    ├── image-optimization.md       # AsyncImage, caching strategies
    └── liquid-glass.md             # iOS 26 glassEffect, GlassEffectContainer
```

---

## Key Features

### State Management

* Decision tree for every property wrapper (`@State`, `@Binding`, `@StateObject`, `@ObservedObject`, `@Observable`, `@Environment`, `@EnvironmentObject`)
* Common anti-patterns and corrected alternatives
* iOS 17+ `@Observable` macro adoption guide

### View Composition

* Manferdini extraction rules — when to extract a sub-view vs use a computed property
* `@ViewBuilder` usage patterns
* Avoiding unnecessary view re-renders

### Modern APIs

* iOS 18/26 API replacements for deprecated calls
* `#Preview` macro, `navigationDestination`, `ContentUnavailableView`
* SwiftUI-native patterns over UIKit bridging

### Performance

* `equatable` modifier for preventing re-renders
* `task(id:)` for cancellable async work
* `GeometryReader` alternatives to avoid layout thrash

### Animations

* Implicit vs explicit animation patterns
* `matchedGeometryEffect` for hero transitions
* `PhaseAnimator` and `KeyframeAnimator` for complex sequences

### Liquid Glass (iOS 26)

* Correct `glassEffect` modifier usage
* `GlassEffectContainer` grouping rules
* When NOT to apply Liquid Glass

---

## Supported Agent Tools

This skill works with any tool supporting the [Agent Skills open format](https://agentskills.io):

* ✅ Claude.ai & Claude Code
* ✅ Cursor
* ✅ Windsurf
* ✅ Cline
* ✅ GitHub Copilot
* ✅ And many more

---

## Philosophy

> "The best SwiftUI code reads like a description of what the user sees, not like instructions to a layout engine."

This skill helps you:

* **Choose the right tool** — correct property wrapper for each situation
* **Keep views simple** — extract only when it adds clarity
* **Stay native** — prefer SwiftUI APIs over UIKit bridging
* **Think in data flow** — understand ownership before writing wrappers

---

## Documentation

* 📖 [Quick Start Guide](docs/QUICKSTART.md) — Example prompts and expected outputs
* 🔧 [Installation Guide](docs/INSTALLATION.md) — Detailed install for all tools

---

## Contributing

Contributions are welcome! Whether you want to:

* Add new SwiftUI patterns as they are released
* Improve existing reference documents
* Add more real-world examples
* Fix inaccuracies or outdated API usage

Open an issue or pull request on GitHub.

---

## License

This skill is open-source and available under the MIT License. See [LICENSE](LICENSE) for details.

---

## Author

Created by Juan Francisco Dorado Torres at SwiftyJourney.

---

## Related Skills

* [Architecture & Design Principles Skill](https://github.com/SwiftyJourney/architecture-design-skill) — SOLID, Clean Architecture, design patterns
* [Requirements Engineering Skill](https://github.com/SwiftyJourney/requirements-engineering-skill) — BDD stories and use cases
