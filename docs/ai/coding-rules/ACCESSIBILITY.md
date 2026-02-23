# Accessibility & Web Compliance

> **Audience:** You (the AI assistant). These are binding rules for generating HTML/UI architectures mapping exactly to Web Content Accessibility Guidelines (WCAG) standard compliance in this codebase.

## 1. Semantic DOM HTML

- **PROHIBITED:** Never default structural elements or abstract containers (especially actionable/interactive features) to meaningless standard `<div onClick="...">` rendering without properties tracking their type.
- **INSTEAD:** Always generate semantic HTML (`<nav>`, `<main>`, `<article>`, `<button>`, `<a>`, `<dialog>`) that accurately conveys the structure/outline of UI code contexts inherently.

## 2. Screen Reader Annotations (Aria Labels)

- **PROHIBITED:** Never omit alternative representations for heavily illustrative image sources (`<img src="" />` without `alt`) or non-semantic, custom SVGs acting as buttons without descriptor properties.
- **INSTEAD:** Always include detailed alternative `alt` properties to imagery inherently. Use explicit `aria-label=""` tags for un-labeled vector/icon actions (e.g., an icon of a pencil for editing something should still have `aria-label="Edit"`).

## 3. Keyboard Visibility

- **PROHIBITED:** Never force styling via UI frameworks overriding default DOM focus rings entirely or rendering keyboard-navigability unusable natively via `tabindex="-1"` overrides globally.
- **INSTEAD:** Always ensure the DOM generates clear focus indicators logically, ordering interactive flows sequentially, preserving native global `.focus()` events inside complex nested subcomponents.
