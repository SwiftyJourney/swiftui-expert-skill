# Quick Start Guide

Get started with the SwiftUI Expert Skill in 5 minutes.

---

## Installation

### Option 1: Skills CLI (Recommended)

```bash
npx skills add SwiftyJourney/swiftui-expert-skill
```

### Option 2: Manual

Download `swiftui-expert.skill` from [Releases](https://github.com/SwiftyJourney/swiftui-expert-skill/releases) and upload to your AI tool.

---

## Example Prompts

### 1. Review State Management

**Prompt:**
```
Review this SwiftUI view for state management issues:

struct ProfileView: View {
    @StateObject var viewModel: ProfileViewModel
    @ObservedObject var settings: AppSettings

    var body: some View {
        Text(viewModel.name)
    }
}
```

**What the skill does:**
1. Checks `@StateObject` vs `@ObservedObject` ownership rules
2. Flags if `AppSettings` should be `@EnvironmentObject`
3. Suggests `@Observable` if targeting iOS 17+
4. Returns corrected code with explanation

---

### 2. Improve View Composition

**Prompt:**
```
This view body is 200 lines. Help me break it down:

struct FeedView: View {
    var body: some View {
        // ... 200 lines of nested VStack/HStack/ForEach
    }
}
```

**What the skill does:**
1. Identifies which sub-trees merit extraction
2. Applies Manferdini's extraction rules
3. Distinguishes computed properties from sub-views
4. Returns refactored code with clear sub-view boundaries

---

### 3. Modernize Deprecated APIs

**Prompt:**
```
Update this code to use modern SwiftUI APIs:

NavigationView {
    List(items) { item in
        NavigationLink(destination: DetailView(item: item)) {
            ItemRow(item: item)
        }
    }
}
```

**What the skill does:**
1. Replaces `NavigationView` with `NavigationStack`
2. Moves destinations to `navigationDestination(for:)`
3. Removes deprecated `NavigationLink(destination:)` form
4. Returns iOS 16+ compliant code

---

### 4. Add Liquid Glass Styling (iOS 26)

**Prompt:**
```
I want to add Liquid Glass styling to this toolbar. Is this correct?

.toolbar {
    ToolbarItem {
        Button("Action") { }
            .glassEffect()
    }
}
```

**What the skill does:**
1. Checks if `glassEffect` is applied to the right container level
2. Identifies whether `GlassEffectContainer` is needed
3. Flags incorrect per-button vs container application
4. Returns corrected Liquid Glass implementation

---

### 5. Fix Animation Issues

**Prompt:**
```
My animation is janky. What's wrong here?

Button("Toggle") {
    isExpanded.toggle()
}
.animation(.spring(), value: isExpanded)

Rectangle()
    .frame(height: isExpanded ? 200 : 50)
```

**What the skill does:**
1. Identifies implicit vs explicit animation mismatch
2. Suggests `withAnimation` at the state mutation site
3. Recommends correct `.animation(_:value:)` placement
4. Returns corrected animation code

---

## Common Prompts

### State Management
```
"Which property wrapper should I use for this shared model?"
"When should I use @StateObject vs @ObservedObject?"
"How do I migrate from ObservableObject to @Observable?"
```

### View Composition
```
"Should I extract this sub-view or use a computed property?"
"How do I use @ViewBuilder correctly?"
"My view body is too complex — how do I simplify it?"
```

### Performance
```
"Why is my list re-rendering on every state change?"
"How do I cancel async work when a view disappears?"
"Should I use LazyVStack or List for this feed?"
```

### Navigation
```
"How do I implement deep linking with NavigationStack?"
"How do I navigate programmatically in SwiftUI?"
"What replaced NavigationLink(isActive:) in iOS 16?"
```

---

## What You Get

When using this skill, Claude will:

✅ Follow the workflow decision tree (review → improve → implement)
✅ Reference the correct guide for each topic
✅ Provide corrected SwiftUI code snippets
✅ Explain *why* a pattern is preferred
✅ Flag anti-patterns before they become bugs

---

## Next Steps

- Read [Installation Guide](INSTALLATION.md) for detailed setup
- See [README](../README.md) for complete feature list

---

**Ready to write better SwiftUI?** Start with a code review prompt!
