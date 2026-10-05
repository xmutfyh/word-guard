---
name: word-guard
description: "Edit existing Word (.docx) files in place while preserving document structure. Use for AI-assisted manuscript/report edits, run-aware text replacement, structural scans, scope/package checks, academic table formatting, and native equation layout. After edits, default formatting converts non-layout tables to three-line tables by clearing the old table style first and centering cell content horizontally/vertically; numbered OMML equations keep the equation centered with the equation number at the far right on the same line; likely running body-text paragraphs are justified while headings, captions, equations, and explicit center/right alignment are preserved."
---

# Word Guard

Use this skill to modify an existing `.docx` rather than rebuild it from scratch. Let the AI decide what content needs changing; use the bundled code to perform deterministic Word edits and formatting.

## Core workflow

1. Scan the uploaded DOCX when structural context is needed.
2. Let the AI identify the desired content changes.
3. Apply only the requested replacements to the original DOCX.
4. Automatically apply the default academic table, equation, and body-paragraph layout described below.
5. Validate the resulting DOCX package before returning it.

Do not use code-based heuristics as the authority for content correctness. Content errors, wording problems, factual issues, and duplicated/incorrect headings are primarily AI review tasks. The code's job is to modify the Word file reliably.

## Default formatting after edits

Apply these defaults automatically after `edit` and when `format-defaults` is used.

### Tables

For every non-layout table:

1. Clear the existing table visual style first, including the table style reference, style look, borders, and shading that can conflict with the target format.
2. Preserve table text, widths, merges, row heights, fonts, and emphasis unless the user explicitly asks to change them.
3. Apply a three-line table:
   - top rule on the first row;
   - header separator below the first row;
   - bottom rule on the last row;
   - no left/right/inside grid lines.
4. Center all cell content horizontally.
5. Center all cell content vertically.

Skip obvious layout/figure-container tables. Unknown non-layout tables are formatted by default.

### Body paragraphs

For likely running body text outside tables:

- Use justified alignment by default for likely running body prose, including narrative paragraphs stored inside layout-table templates.
- Preserve headings, captions, equations, reference sections, and paragraphs that are already explicitly centered or right-aligned.
- Keep short form labels/fields and template instruction prompts untouched rather than forcing them into justified alignment.
- Do not use this formatter to decide whether the wording is correct; content judgment remains an AI task.

### Equations

For native OMML display equations:

- If the equation has a trailing number such as `(7)` or `(4-14a)`, keep the equation body centered and place the equation number at the far right text margin on the same line using center/right tab stops.
- If the equation is unnumbered, center the equation paragraph.
- Do not rebuild or convert the OMML equation itself.

## Commands

```text
scan <file.docx>
edit <file.docx> <reps.json> [out.docx] [mode] [scope.json]
scope-check <before.docx> <after.docx> [scope.json]
package-validate <file.docx>
format-tables <file.docx> [out.docx] [baseline.docx]
format-equations <file.docx> [out.docx]
format-paragraphs <file.docx> [out.docx]
format-defaults <file.docx> [out.docx]
fingerprint <file.docx>
equation-audit <file.docx> [baseline.docx]
audit <file.docx>
```

Run commands with:

```bash
python scripts/word_guard/cli.py <command> ...
```

## Text editing

Represent AI-proposed changes as exact replacement pairs:

```json
[
  {"old": "exact text from the source paragraph", "new": "replacement text"}
]
```

Example:

```bash
python scripts/word_guard/cli.py edit paper.docx reps.json paper-edited.docx text_only '{"heading":"3 Methods"}'
```

`edit` applies the replacements and then applies the table/equation/body-paragraph defaults automatically.

Supported scope forms include:

```json
{"heading": "3 Methods"}
{"startHeading": "1 Introduction", "endHeading": "3 Methods"}
{"startParagraph": 10, "endParagraph": 25}
```

If scope is omitted, the full document is eligible for text matching.

## Formatting-only operations

Use these when the user wants no text changes:

```bash
# Apply all requested defaults
python scripts/word_guard/cli.py format-defaults input.docx output.docx

# Tables only: clear old styles, then three-line + center-center cells
python scripts/word_guard/cli.py format-tables input.docx output.docx

# Equations only: centered equation + right-edge number on same line
python scripts/word_guard/cli.py format-equations input.docx output.docx

# Body paragraphs only: justify likely running body text
python scripts/word_guard/cli.py format-paragraphs input.docx output.docx
```

## Files

```text
word-guard/
├── SKILL.md
├── agents/openai.yaml
├── manifest.yaml
└── scripts/word_guard/
    ├── cli.py
    ├── scanner.py
    ├── editor.py
    ├── scope.py
    ├── protected.py
    ├── manifest.py
    ├── integrity.py
    ├── package_guard.py
    ├── fingerprint.py
    ├── equation_audit.py
    ├── equation_format.py
    ├── paragraph_format.py
    └── table_format.py
```

## Delivery checks

Before returning an edited file:

1. Confirm `package-validate` reports `valid: true`.
2. Confirm the intended replacement/formatting action is present in the command result.
3. Return the edited DOCX rather than reconstructing the document from extracted text.
