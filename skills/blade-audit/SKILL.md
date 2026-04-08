---
name: lc:blade-audit
description: Detect hardcoded text without @text(), duplicate CSS classes, unused Blade components, and accessibility issues.
argument-hint: "[report | fix | fix --dry-run | file-path | fix file-path]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob Agent
---

## Subcommands

| Subcommand | Description |
|---|---|
| `report` | Scan all Blade files and report issues without making changes |
| `fix` | Scan all Blade files, report issues, and auto-fix what is safe |
| `fix --dry-run` | Show what fixes would be applied without writing changes |
| `<file-path>` | Scan a single file and report issues |
| `fix <file-path>` | Scan a single file and auto-fix issues |

## Overview

This skill audits Blade templates for localization issues, accessibility problems, Blade syntax errors, and code quality concerns. It enforces the project's strict requirement that ALL user-visible text must use `@text()` or `text()` helpers with English defaults.

## Instructions

### Step 1: Determine scope

Parse the argument:

- **No argument or `report`**: Scan all `.blade.php` files under `resources/views/`.
- **`fix` (no path)**: Same scope, apply safe fixes.
- **`fix --dry-run`**: Show proposed fixes without applying.
- **`<file-path>`**: Scan only the specified file.
- **`fix <file-path>`**: Scan and fix only the specified file.

Use Glob to find files:
```
resources/views/**/*.blade.php
```

### Step 2: Run each detection rule

#### Rule 1: Hardcoded user-visible text not wrapped in @text() or text()

This is the most critical rule. ALL user-visible text must be translated.

**What counts as user-visible text**:
- Text content between HTML tags: `<span>Some text</span>`
- Button labels: `<button>Submit</button>`
- Placeholder attributes: `placeholder="Enter name"`
- Title attributes: `title="Click here"`
- Label text: `<label>Name</label>`
- Alt text on images: `alt="Profile photo"`
- Heading content: `<h1>Dashboard</h1>`
- Link text: `<a href="...">Click here</a>`
- Option text: `<option>Select one</option>`
- Aria labels: `aria-label="Close"`

**What is NOT user-visible (ignore these)**:
- HTML attributes: `class="..."`, `id="..."`, `type="..."`, `name="..."`, `wire:model="..."`
- Blade directives: `@if`, `@foreach`, `@extends`, `@section`
- PHP code blocks: `<?php ... ?>`
- JavaScript code inside `<script>` tags
- CSS inside `<style>` tags
- Route names: `route('...')`
- Variable interpolation that's already dynamic: `{{ $user->name }}`
- Comments: `{{-- ... --}}`
- Wire/Alpine attributes: `wire:click="..."`, `@click="..."`, `x-on:click="..."`
- Data attributes: `data-*="..."`
- Single characters used as separators: `/`, `|`, `-`, `:`
- Numbers standing alone
- HTML entities: `&nbsp;`, `&copy;`

**Detection approach**:

1. Parse each line of the blade file
2. Skip lines that are entirely PHP, comments, Blade directives, script, or style blocks
3. For remaining lines, look for text content between HTML tags that is:
   - Longer than 1 character (ignore single chars)
   - Not already wrapped in `@text()`, `text()`, `__()`, or `{{ }}`
   - Not purely whitespace
   - Not an HTML entity
4. Also check `placeholder="..."`, `title="..."`, `alt="..."`, and `aria-label="..."` attributes for hardcoded strings

**Report format**:
```
[HARDCODED-TEXT] file.blade.php:45
  <span>Send message</span>
  Should be: <span>@text('send_message', 'Send message')</span>
```

**Fix**: Wrap text in `@text()` with an appropriate key following the naming conventions:
- 1-3 words: lowercase with underscores (`send_message`, `our_address`)
- Longer text: dot notation with section (`web.contact.description`)
- Strip punctuation from key name
- Default text in English

#### Rule 2: Non-English default text in @text() calls

The project requires ALL default text to be in English. Detect common Spanish words/patterns in @text() defaults.

**Pattern to detect**: `@text('key', 'Non-English text')` or `text('key', 'Non-English text')` where the default contains non-English words.

Common Spanish indicators to check:
- Articles: `el`, `la`, `los`, `las`, `un`, `una`, `unos`, `unas`, `del`
- Prepositions: `de`, `en`, `por`, `para`, `con`, `sin`, `sobre`, `entre`, `desde`, `hasta`
- Verbs: `es`, `son`, `esta`, `estan`, `tiene`, `hacer`, `crear`, `editar`, `eliminar`, `guardar`, `enviar`, `buscar`, `ver`, `agregar`, `actualizar`, `cerrar`, `abrir`, `seleccionar`, `confirmar`
- Common words: `nombre`, `direccion`, `telefono`, `correo`, `mensaje`, `precio`, `fecha`, `tipo`, `estado`, `nuevo`, `todos`, `ninguno`, `hola`
- Accented characters common in Spanish: words with `a`, `e`, `i`, `o`, `u` + accents in the context of otherwise Spanish text

Also detect other languages (French, German, Portuguese) by checking for non-ASCII characters in default text that suggest a non-English language.

**Report format**:
```
[NON-ENGLISH-DEFAULT] file.blade.php:23
  @text('send_message', 'Enviar mensaje')
  Default text appears to be Spanish. Should be: @text('send_message', 'Send message')
```

**Fix**: Replace the default text with an English translation. Use the key as a hint for what the English text should be.

#### Rule 3: Duplicate identical CSS class strings

**Pattern to detect**: Identical long class strings (10+ classes) that appear in 3+ files. These should be extracted to a Blade component or Tailwind @apply.

Build a map of class strings -> files. Report duplicates.

**Report format**:
```
[DUPLICATE-CLASSES] Found in 4 files
  Class string: "px-4 py-2 text-sm font-medium text-white bg-pink-600 rounded-lg hover:bg-pink-700 focus:ring-4 focus:ring-pink-300"
  Files:
    - modals/create-user.blade.php:45
    - modals/edit-user.blade.php:67
    - modals/create-property.blade.php:89
    - modals/edit-property.blade.php:34
  Consider: Extract to a Blade component or use @apply in CSS
```

**Fix**: No auto-fix (requires architectural decision). Report only.

#### Rule 4: Unused Blade components

**Pattern to detect**: Component files in `resources/views/components/` that are never referenced anywhere in the project.

1. List all component files under `resources/views/components/`
2. For each component, derive its usage tag:
   - `resources/views/components/modal.blade.php` -> `<x-modal` or `x-modal`
   - `resources/views/components/fields/text-input.blade.php` -> `<x-fields.text-input`
3. Search ALL blade files for that component tag
4. Report components with zero usages

**Report format**:
```
[UNUSED-COMPONENT] resources/views/components/old-button.blade.php
  Component <x-old-button> is not used anywhere in the project.
  Consider removing it.
```

**Fix**: No auto-fix (may be used dynamically or in packages). Report only.

#### Rule 5: Nested {{ }} inside {{ }}

**Pattern to detect**: Blade expressions containing nested `{{ }}` which is a syntax error.

Look for patterns like:
```blade
class="{{ $condition ? 'bg-{{ $color }}-600' : 'bg-gray-600' }}"
```

This is a known Blade limitation documented in the project's CLAUDE.md.

**Report format**:
```
[NESTED-INTERPOLATION] file.blade.php:67
  class="{{ $tab === 'home' ? 'bg-{{ $color }}-600' : 'bg-gray-600' }}"
  Fix: Define class strings in a @php block, then reference variables.
```

**Fix**: Extract the inner expressions to a `@php` block:
```blade
@php
    $activeClass = "bg-{$color}-600";
@endphp
<!-- then use -->
class="{{ $tab === 'home' ? $activeClass : 'bg-gray-600' }}"
```

#### Rule 6: Missing alt attributes on images

**Pattern to detect**: `<img>` tags without an `alt` attribute. This is an accessibility violation (WCAG 2.1 Level A).

Look for `<img` tags that do not contain `alt="` or `alt='` or `:alt="`.

**Report format**:
```
[MISSING-ALT] file.blade.php:112
  <img src="{{ $property->image }}" class="w-full h-48 object-cover">
  Missing alt attribute. Add: alt="@text('property_image', 'Property image')"
```

**Fix**: Add an `alt` attribute with `@text()`. Infer a sensible description from context (nearby text, variable names, component purpose).

#### Rule 7: Forms without CSRF protection

**Pattern to detect**: `<form` tags that do not contain `@csrf` anywhere inside the form body.

Exclude:
- Forms with `wire:submit` (Livewire handles CSRF automatically)
- Forms with `method="GET"` (CSRF not needed for GET)
- Forms with `action` pointing to external URLs

**Report format**:
```
[MISSING-CSRF] file.blade.php:34
  <form method="POST" action="{{ route('contact.store') }}">
  Missing @csrf directive. Add @csrf inside the form.
```

**Fix**: Add `@csrf` as the first child inside the `<form>` tag.

#### Rule 8: Buttons without type attribute

**Pattern to detect**: `<button>` tags without a `type` attribute. Buttons default to `type="submit"` in HTML, which can cause unintended form submissions.

Exclude:
- Buttons that already have `type="submit"`, `type="button"`, or `type="reset"`
- Buttons inside Livewire forms that intentionally submit

**Report format**:
```
[MISSING-BUTTON-TYPE] file.blade.php:78
  <button @click="$dispatch('close-modal')" class="...">Cancel</button>
  Missing type attribute. Defaults to "submit" which may cause unintended form submission.
  Add: type="button"
```

**Fix**: Add `type="button"` for non-submit buttons (those with `@click`, `wire:click`, `onclick`, or that appear to be cancel/close/toggle buttons). Add `type="submit"` for buttons that should submit forms.

#### Rule 9: Hardcoded colors in group context

**Pattern to detect**: Hardcoded pink/brand colors in files that operate within the group/organization context (files under `resources/views/group/`, files that access `$group` or `group_context`).

Look for:
- `bg-pink-600`, `text-pink-600`, `border-pink-500`, `hover:bg-pink-700`, etc.
- In files that should use dynamic `$color` from `$group->color_scheme`

**Report format**:
```
[HARDCODED-COLOR] resources/views/group/pages/contact.blade.php:45
  bg-pink-600 should be bg-{{ $color }}-600 in group context.
  Groups support custom color schemes.
```

**Fix**: Replace hardcoded color with dynamic `{{ $color }}` variable. Ensure the `$color` variable is set from `$group->color_scheme` in a `@php` block at the top of the file.

### Step 3: Generate report

Group findings by file, then by rule:

**Severity levels**:
- **ERROR**: Will cause runtime errors or broken functionality (nested interpolation, missing CSRF)
- **WARNING**: Violates project requirements (hardcoded text, non-English defaults, hardcoded colors)
- **INFO**: Best practices and accessibility (missing alt, button types, unused components, duplicates)

**Output format**:
```
## Blade Audit Report

### resources/views/livewire/properties/index.blade.php

- [WARNING] Line 45: Hardcoded text "Send message" not wrapped in @text()
- [WARNING] Line 23: Non-English default in @text('enviar', 'Enviar mensaje')
- [INFO] Line 112: <img> missing alt attribute
- [INFO] Line 78: <button> missing type attribute

### resources/views/group/pages/contact.blade.php

- [ERROR] Line 67: Nested {{ }} inside {{ }}
- [WARNING] Line 45: Hardcoded bg-pink-600 in group context

### Unused Components

- resources/views/components/old-alert.blade.php (x-old-alert)
- resources/views/components/deprecated-card.blade.php (x-deprecated-card)

### Duplicate Class Strings (3+ occurrences)

- "px-4 py-2 text-sm font-medium text-white bg-pink-600..." (found in 4 files)

---
**Summary**: 1 error, 3 warnings, 2 info across 2 files.
**Plus**: 2 unused components, 1 duplicate class string pattern.
```

### Step 4: Apply fixes (if fix mode)

**Safe auto-fixes** (applied automatically):
- Wrap hardcoded text in `@text()` with appropriate key and English default
- Replace non-English defaults with English equivalents
- Fix nested `{{ }}` by extracting to `@php` blocks
- Add `alt` attributes to images with `@text()` wrapped descriptions
- Add `@csrf` to forms missing it
- Add `type="button"` to non-submit buttons
- Replace hardcoded colors with `{{ $color }}` in group context

**Unsafe fixes** (report only):
- Duplicate class strings (requires component extraction decision)
- Unused components (may be used dynamically or planned for future use)

### Generating @text() keys

When wrapping text in `@text()`, follow these key naming rules:

**Short text (1-3 words)**:
- `Send` -> `@text('send', 'Send')`
- `Our Address` -> `@text('our_address', 'Our Address')`
- `Get in Touch` -> `@text('get_in_touch', 'Get in Touch')`

**Longer text (sentences/descriptions)**:
- Determine the section from the file path:
  - `resources/views/group/pages/contact.blade.php` -> `web.contact`
  - `resources/views/livewire/properties/index.blade.php` -> `properties.index`
  - `resources/views/components/layouts/marketing.blade.php` -> `web.layout`
- Build key: `section.page.element`
  - `@text('web.contact.heading', 'Contact Us')`
  - `@text('properties.index.no_results', 'No properties found matching your criteria')`

**Punctuation rule**: Keep punctuation OUTSIDE the translation:
- CORRECT: `@text('sending', 'Sending')...`
- WRONG: `@text('sending', 'Sending...')`
- CORRECT: `@text('message_sent', 'Message sent successfully')!`
- WRONG: `@text('message_sent', 'Message sent successfully!')`

**Before generating a key**: Search existing `@text()` calls in the project to avoid creating duplicate keys for the same concept. Reuse existing keys when the meaning matches exactly.
