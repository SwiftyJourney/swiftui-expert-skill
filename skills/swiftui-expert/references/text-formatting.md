# SwiftUI Text Formatting Reference

## Modern Text Formatting

**Never use C-style `String(format:)` with Text. Always use format parameters.**

## Number Formatting

### Basic Number Formatting

```swift
let value = 42.12345

// Modern (Correct)
Text(value, format: .number.precision(.fractionLength(2)))
// Output: "42.12"

Text(abs(value), format: .number.precision(.fractionLength(2)))
// Output: "42.12" (absolute value)

// Legacy (Avoid)
Text(String(format: "%.2f", abs(value)))
```

### Integer Formatting

```swift
let count = 1234567

// With grouping separator
Text(count, format: .number)
// Output: "1,234,567" (locale-dependent)

// Without grouping
Text(count, format: .number.grouping(.never))
// Output: "1234567"
```

### Decimal Precision

```swift
let price = 19.99

// Fixed decimal places
Text(price, format: .number.precision(.fractionLength(2)))
// Output: "19.99"

// Significant digits
Text(price, format: .number.precision(.significantDigits(3)))
// Output: "20.0"

// Integer-only
Text(price, format: .number.precision(.integerLength(1...)))
// Output: "19"
```

## Currency Formatting

```swift
let price = 19.99

// Correct - with currency code
Text(price, format: .currency(code: "USD"))
// Output: "$19.99"

// With locale
Text(price, format: .currency(code: "EUR").locale(Locale(identifier: "de_DE")))
// Output: "19,99 €"

// Avoid - manual formatting
Text(String(format: "$%.2f", price))
```

## Percentage Formatting

```swift
let percentage = 0.856

// Correct - with precision
Text(percentage, format: .percent.precision(.fractionLength(1)))
// Output: "85.6%"

// Without decimal places
Text(percentage, format: .percent.precision(.fractionLength(0)))
// Output: "86%"

// Avoid - manual calculation
Text(String(format: "%.1f%%", percentage * 100))
```

## Date and Time Formatting

### Date Formatting

```swift
let date = Date()

// Date only
Text(date, format: .dateTime.day().month().year())
// Output: "Jan 23, 2026"

// Full date
Text(date, format: .dateTime.day().month(.wide).year())
// Output: "January 23, 2026"

// Short date
Text(date, style: .date)
// Output: "1/23/26"
```

### Time Formatting

```swift
let date = Date()

// Time only
Text(date, format: .dateTime.hour().minute())
// Output: "2:30 PM"

// With seconds
Text(date, format: .dateTime.hour().minute().second())
// Output: "2:30:45 PM"

// 24-hour format
Text(date, format: .dateTime.hour(.defaultDigits(amPM: .omitted)).minute())
// Output: "14:30"
```

### Relative Date Formatting

```swift
let futureDate = Date().addingTimeInterval(3600)

// Relative formatting
Text(futureDate, style: .relative)
// Output: "in 1 hour"

Text(futureDate, style: .timer)
// Output: "59:59" (counts down)
```

### Stabilizing Live Timers

**Rule:** Apply `.monospacedDigit()` to any `Text` using a self-updating date style (`.relative`, `.timer`, `.offset`).

```swift
// BAD - digits change width as numbers change, so the label wobbles
Text(deadline, style: .timer)

// GOOD - fixed-width digits keep the countdown from jittering
Text(deadline, style: .timer)
    .monospacedDigit()
```

**Why:** `.relative`, `.timer`, and `.offset` update in place every second. In most fonts, proportional digits have different widths (a `1` is narrower than an `8`), so the text reflows and shifts horizontally on each tick. `.monospacedDigit()` forces equal-width digits, holding the layout steady.

## String Searching and Comparison

### Localized String Comparison

**Use `localizedStandardContains()` for user-input filtering, not `contains()`.**

```swift
let searchText = "café"
let items = ["Café Latte", "Coffee", "Tea"]

// Correct - handles diacritics and case
let filtered = items.filter { $0.localizedStandardContains(searchText) }
// Matches "Café Latte"

// Wrong - exact match only
let filtered = items.filter { $0.contains(searchText) }
// Might not match "Café Latte" depending on normalization
```

**Why**: `localizedStandardContains()` handles case-insensitive, diacritic-insensitive matching appropriate for user-facing search.

### Case-Insensitive Comparison

```swift
let text = "Hello World"
let search = "hello"

// Correct - case-insensitive
if text.localizedCaseInsensitiveContains(search) {
    // Match found
}

// Also correct - for exact comparison
if text.lowercased() == search.lowercased() {
    // Equal
}
```

### Localized Sorting

```swift
let names = ["Zoë", "Zara", "Åsa"]

// Correct - locale-aware sorting
let sorted = names.sorted { $0.localizedStandardCompare($1) == .orderedAscending }
// Output: ["Åsa", "Zara", "Zoë"]

// Wrong - byte-wise sorting
let sorted = names.sorted()
// Output may not be correct for all locales
```

## Attributed Strings

### Basic Attributed Text

```swift
// Using Text concatenation
Text("Hello ")
    .foregroundStyle(.primary)
+ Text("World")
    .foregroundStyle(.blue)
    .bold()

// Using AttributedString
var attributedString = AttributedString("Hello World")
attributedString.foregroundColor = .primary
if let range = attributedString.range(of: "World") {
    attributedString[range].foregroundColor = .blue
    attributedString[range].font = .body.bold()
}
Text(attributedString)
```

### Styling Part of a String with Text-Returning Modifiers

**Rule:** Modifiers applied directly to a `Text` return another `Text`, so a styled fragment can be embedded inside a string interpolation.

```swift
// GOOD - style one word without building an AttributedString
Text("See you on \(Text("Friday").bold())")

let due = Text("overdue").foregroundStyle(.red).italic()
Text("Status: \(due)")

// BAD - this won't compile; view modifiers return `some View`, not Text
Text("See you on \(Text("Friday").padding())")
```

**Why:** `Text` has its own overloads of `.bold()`, `.italic()`, `.foregroundStyle(_:)`, `.font(_:)`, `.strikethrough()`, etc. that return `Text` rather than `some View`. Because the result is still a `Text`, it can be interpolated into another `Text` literal — letting you style only part of a string without constructing an `AttributedString`. View modifiers like `.padding()` or `.background()` return `some View` and cannot be re-interpolated.

### SwiftUI AttributedString Scope

**Rule:** When a `Text` renders an `AttributedString`, only attributes in SwiftUI's attribute scope take effect; attributes from other scopes are silently dropped.

```swift
var attributed = AttributedString("Sale")
attributed.foregroundColor = .red          // honored (SwiftUI scope)
attributed.font = .headline                // honored
attributed.kern = 1.5                      // honored
// attributed.uiKit / .appKit attributes -> ignored when Text renders it
Text(attributed)
```

**Why:** SwiftUI's scope supports a fixed subset: `foregroundColor`, `backgroundColor`, `font`, `kern`, `tracking`, `underlineStyle`, `strikethroughStyle`, `baselineOffset`, plus Foundation attributes like links and inline presentation intent, and the accessibility attributes. Attributes you set from the UIKit or AppKit scopes have no SwiftUI equivalent in this list, so `Text` ignores them rather than warning. Build attributed text using only SwiftUI/Foundation-scope attributes to avoid styling that silently disappears.

### Markdown in Text

**Rule:** `Text` renders only INLINE Markdown. Block-level syntax (headers, bullet/numbered lists, fenced code blocks, blockquotes) is NOT rendered — it appears as literal characters.

```swift
// GOOD - inline styling renders
Text("This is **bold**, *italic*, ~~struck~~, and `code`")
Text("Visit [Apple](https://apple.com) for more info")

// BAD - block syntax is shown verbatim, not rendered
Text("""
# Title
- Item 1
- Item 2
""")
// Renders the literal text: "# Title", "- Item 1", "- Item 2"
```

**Supported inline syntax:** `**bold**`, `*italic*`, `~~strikethrough~~`, inline `` `code` `` spans, and `[link](url)`.

**Why:** SwiftUI parses a `Text` literal as an inline `AttributedString`. There is no block layout engine behind `Text`, so anything that would require a new paragraph, list item, or code block stays as plain characters. For block-level Markdown, render to an `AttributedString` yourself or compose multiple views.

## Localization

### Literal vs Variable Initializer Trap

**Rule:** A string LITERAL passed to `Text` binds a different initializer than a `String` VARIABLE. They are not interchangeable.

```swift
// GOOD - literal binds the LocalizedStringKey initializer
Text("Welcome **back**")
// -> auto-localized (looked up in the string catalog),
//    parses inline Markdown, supports rich interpolation

// BAD (often unintended) - variable binds the StringProtocol initializer
let message = "Welcome **back**"
Text(message)
// -> rendered verbatim: no localization, no Markdown ("**back**" shows literally)
```

**Why:** `Text(_:)` is overloaded. A literal resolves to `init(_ key: LocalizedStringKey)`; a `String`/`StringProtocol` value resolves to `init(_ content: some StringProtocol)`, which renders exactly as-is. Refactoring an inline literal into a `let` silently disables localization and Markdown — and the compiler emits no warning. To keep localization on a variable, type it as `LocalizedStringKey` or wrap with `Text(verbatim:)` only when you explicitly want raw output.

### Localizing Strings Outside Views

**Rule:** Use `String(localized:comment:)` for user-facing strings in models and helpers so they still enter the string catalog.

```swift
// GOOD - in a model/helper, outside any View
let title = String(localized: "Order summary", comment: "Cart screen header")

// In a View, give translators context and route to a catalog/bundle
Text("Order summary", comment: "Cart screen header")
Text("Order summary", tableName: "Checkout")        // separate .strings catalog
Text("Order summary", bundle: .module)              // another module/framework's translations
```

**Why:** Only `LocalizedStringKey`-bound `Text` and `String(localized:)` are picked up by the catalog extractor. A bare `String("...")` in a model never gets translated. `comment:` gives translators disambiguation, `tableName:` splits large apps into multiple catalogs, and `bundle:` looks up a string defined in another module or framework.

## Pluralization and Grammar Agreement

### Automatic Inflection

**Rule:** Use inline automatic grammar agreement to pluralize from an interpolated count — no `.stringsdict` required.

```swift
// GOOD - inflects the noun to agree with the count
Text("^[\(count) item](inflect: true)")
// count == 1 -> "1 item"
// count == 5 -> "5 items"
```

**Why:** The `^[\(n) noun](inflect: true)` markup tells Foundation to apply automatic grammar agreement, choosing the correct plural form for the current language. This handles the common count case without authoring an explicit plural-rule catalog, and works in languages with more than two plural categories.

## List and Measurement Formatting

### Grammatical Lists

```swift
let items = ["apples", "oranges", "pears"]

Text("\(items, format: .list(type: .and))")
// Output: "apples, oranges, and pears" (locale-aware separators and conjunction)
```

### Measurements and Units

```swift
let distance = Measurement(value: 5, unit: UnitLength.kilometers)

Text("\(distance, format: .measurement(width: .wide))")
// Output: "5 kilometers" (en_US) / "5 Kilometer" (de_DE), localized unit + value
```

**Why:** `.list(type:)` and `.measurement(width:)` are locale-aware `FormatStyle`s, like `.number`/`.currency`/`.dateTime`. They pick the right separators, conjunctions, unit names, and even unit conversions for the user's locale instead of hardcoding English punctuation or units.

## Text Measurement

### Measuring Text Height

```swift
// Wrong (Legacy) - GeometryReader trick
struct MeasuredText: View {
    let text: String
    @State private var textHeight: CGFloat = 0
    
    var body: some View {
        Text(text)
            .background(
                GeometryReader { geometry in
                    Color.clear
                        .onAppear {
                            textWidth = geometry.size.height
                        }
                }
            )
    }
}

// Modern (correct)
struct MeasuredText: View {
    let text: String
    @State private var textHeight: CGFloat = 0
    
    var body: some View {
        Text(text)
            .onGeometryChange(for: CGFloat.self) { geometry in
                geometry.size.height
            } action: { newValue in
                textHeight = newValue
            }
    }
}
```

## Summary Checklist

- [ ] Use `.format` parameters with Text instead of `String(format:)`
- [ ] Use `.currency(code:)` for currency formatting
- [ ] Use `.percent` for percentage formatting
- [ ] Use `.dateTime` for date/time formatting
- [ ] Use `localizedStandardContains()` for user-input search
- [ ] Use `localizedStandardCompare()` for locale-aware sorting
- [ ] Use Text concatenation or AttributedString for styled text
- [ ] Use INLINE markdown only (`**bold**`, `*italic*`, `` `code` ``, links); block syntax is not rendered
- [ ] Keep literals (not `String` variables) for auto-localization and Markdown
- [ ] Use `String(localized:comment:)` for user-facing strings outside views
- [ ] Use `^[\(count) item](inflect: true)` for count-based pluralization
- [ ] Use `.list(type:)` and `.measurement(width:)` for locale-aware lists and units
- [ ] Add `.monospacedDigit()` to live `.relative`/`.timer`/`.offset` text
- [ ] Use Text-returning modifiers to style fragments inside interpolation
- [ ] All formatting respects user's locale and preferences

**Why**: Modern format parameters are type-safe, localization-aware, and integrate better with SwiftUI's text rendering.
