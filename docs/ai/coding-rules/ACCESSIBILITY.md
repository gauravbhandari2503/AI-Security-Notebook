# Accessibility & Web Compliance

> **Audience:** You (the AI assistant). These are binding rules for generating HTML/UI architectures mapping exactly to Web Content Accessibility Guidelines (WCAG) standard compliance in this codebase.

## 1. HTML: Semantic Structure & Forms

- **PROHIBITED:** Never use `<div>` or `<span>` to create interactive elements like buttons or links. Do not skip heading levels (e.g., jumping from `<h1>` directly to `<h3>`).
- **INSTEAD:** Always use native semantic elements (`<nav>`, `<main>`, `<article>`, `<button>`, `<a>`, `<dialog>`). Ensure a logical document outline with sequential heading levels (`<h1>` to `<h6>`).
- **FORMS:** Always explicitly associate `<label>` elements with their inputs using the `for` and `id` attributes. Group related radio buttons or checkboxes within a `<fieldset>` and describe the group using a `<legend>`.
- **TABLES:** Always use `<th>` with appropriate `scope` (`row` or `col`) for data table headers, and include a `<caption>` to describe the table.
- **NAVIGATION:** Provide a "Skip to main content" link at the top of the page to allow keyboard users to bypass repetitive navigation blocks.

## 2. CSS & Visual Design (Color & Contrast)

- **PROHIBITED:** Never use color as the _sole_ means of conveying information, indicating an action, prompting a response, or distinguishing a visual element. Never use `display: none;` or `visibility: hidden;` for text intended for screen readers.
- **INSTEAD:** Ensure text has a contrast ratio of at least 4.5:1 against its background (WCAG AA). When visually hiding content that _should_ be read by screen readers, use a visually-hidden (or screen-reader-only) CSS utility class (clipping layout properties).
- **FOCUS RINGS:** Never force styling via UI frameworks that overrides default DOM focus rings entirely (`outline: none;`) without providing a prominent, custom focus indicator alternative.
- **USER PREFERENCES:** Respect user system preferences by utilizing media queries like `@media (prefers-reduced-motion: reduce)` to disable non-essential animations.

## 3. JavaScript & Dynamic Content

- **PROHIBITED:** Never create custom interactive widgets without implementing full keyboard support. Never dynamically update the page with important information without notifying assistive technologies.
- **INSTEAD:** Ensure all custom widgets can be activated with both `Enter` and `Space` keys (like native buttons).
- **FOCUS MANAGEMENT:** Manage focus logically. For example, when opening a modal `<dialog>`, trap keyboard focus inside it; when closing, explicitly return focus to the trigger element that opened it.
- **DYNAMIC UPDATES:** Use `aria-live` regions (e.g., `aria-live="polite"` or `aria-live="assertive"`) for dynamic content updates, temporary toasts, or form error messages so screen readers announce them.

## 4. WAI-ARIA (Accessible Rich Internet Applications)

- **PROHIBITED:** "No ARIA is better than bad ARIA." Never use ARIA attributes to reinvent native HTML semantics (e.g., do not use `<div role="button">` if you can use `<button>`).
- **INSTEAD:** Use ARIA attributes only when native HTML falls short or when building complex custom widgets.
- **STATES:** Accurately bind UI state to ARIA states like `aria-expanded="true/false"` for accordions/dropdowns, `aria-pressed="true/false"` for toggle buttons, and `aria-invalid="true"` for form errors.
- **DESCRIPTIONS:** Use `aria-describedby` to link inputs to their specific error messages or helper text blocks.

## 5. Screen Reader Annotations & Multimedia

- **PROHIBITED:** Never omit alternative representations for images. Never upload or embed video/audio content without text alternatives.
- **INSTEAD:** Always provide accurate, descriptive `alt` text for informational images. For purely decorative images, use an empty alt attribute (`alt=""`) so screen readers ignore them.
- **ICONS:** For icon-only buttons (or non-semantic custom SVGs), always include visually hidden text or an `aria-label` describing the action (e.g., `aria-label="Edit"`).
- **MULTIMEDIA:** Provide transcripts for audio-only content. Use native HTML5 `<track>` tags (`kind="captions"`) for `<video>` elements to provide closed captions or audio descriptions. If building custom controls for video/audio, ensure they are fully keyboard and screen-reader accessible.

## 6. Mobile & Touch Accessibility

- **PROHIBITED:** Never disable pinch-to-zoom (e.g., `user-scalable=no` or `maximum-scale=1.0` in the viewport meta tag). Never require horizontal scrolling on standard mobile viewports.
- **INSTEAD:** Design responsively using relative units. Ensure interactive touch targets (buttons, links) are at least 44x44 CSS pixels in size with sufficient spacing between them to prevent accidental taps (Touch Target Size constraints).
- **RESPONSIVENESS:** Ensure applications remain fully functional and readable when zoomed up to 200%. Let layouts reflow into a single column intelligently on small screens.
