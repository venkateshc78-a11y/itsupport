# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single self-contained HTML file, [index.html](index.html): a training/demo mockup of an internal
"UOB Bank" IT Support Service Desk (sticky header/nav, hero, an IT support ticket form, an FAQ
accordion with live search, and a footer). It is explicitly a training demo, not a real UOB
product — the footer carries a disclaimer stating it is not affiliated with or endorsed by
United Overseas Bank Limited, and this framing must be preserved in any future edits.

There is no build step, package manager, dependency list, test suite, or server. There is nothing
to install, compile, lint, or run other than opening the file.

## Running it

Open `index.html` directly in a browser (double-click, or `open index.html` on macOS). All CSS
lives in one `<style>` block and all JS in one `<script>` block in the `<head>`/`<body>` of that
same file — there are no external requests, no CDN dependencies, and no other source files to
build or bundle.

## Architecture

Everything is one file, organized top-to-bottom as: sticky header/nav → hero banner → 3 quick-info
cards → ticket form section → FAQ section → footer, with matching `<!-- ===== SECTION ===== -->`
comments in the HTML and `/* ===== ... ===== */` / `/* JS: ... */` comments in the CSS/JS marking
each block. When editing, keep changes to the relevant block and its matching CSS/JS comment
section rather than introducing new files or a build pipeline.

Key behavioral details worth knowing before touching the JS:

- **Ticket form validation** runs per-field on `blur`/`change` and again on `submit`, driven by a
  `validators` array of `{ field, run }` pairs. Each `run` function returns an error string (or
  `''`) and writes it into an adjacent `<span class="error-message" id="<fieldId>-error">` while
  toggling `aria-invalid` on the field. On submit, the first field with a non-empty error receives
  focus.
- **Credential warning**: the description textarea is checked against `CREDENTIAL_PATTERN`, a
  regex intentionally scoped to require a value-like separator after a credential term (e.g.
  `password:`, `pwd=`, `otp is ...`) rather than the bare word alone — this avoids false-triggering
  on ordinary text like "reset my password". Keep that separator requirement if the pattern is
  extended, since password-related tickets are one of the most common real inputs to this form.
- **No persistence**: submitted tickets are pushed into an in-memory `tickets` array only. There is
  no `localStorage`/`sessionStorage`/network call, and the form deliberately never collects
  passwords, OTPs, or card numbers — do not add fields or storage that would change this.
- **Ticket reference format**: mock references are generated client-side as
  `UOB-ITSD-YYYYMMDD-####` (random 4 digits), with SLA text looked up from a
  `SLA_BY_PRIORITY` map (Critical 1h / High 4h / Medium 1 business day / Low 3 business days).
- **FAQ accordion** enforces single-open behavior via a `toggle` listener that closes sibling
  `<details>` elements, and the search box filters by matching `item.textContent` against the
  query, toggling a "No matching FAQs" message based on visible count.

## Design tokens

Palette and layout constants are CSS custom properties on `:root` (`--uob-blue`, `--uob-gold`,
`--text`, `--radius`, `--max-width: 1120px`, etc.) — reuse these variables rather than hardcoding
colors/spacing when adding new elements. The single responsive breakpoint is
`@media (max-width: 768px)`, which collapses the nav to a hamburger menu and stacks the
quick-info/footer grids to one column.
