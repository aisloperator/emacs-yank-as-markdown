# yank-as-markdown.el

An Emacs command that yanks the head of the kill ring (the same text `yank`
would insert, which normally reflects the system clipboard) converted to
Markdown.

Rather than a single generic HTML-to-Markdown pass, the text is run through
an ordered chain of heuristics, and the first one that recognizes the text
wins:

1. **Thunderbird forwarded message** — text beginning with
   `-------- Forwarded Message --------`. The email headers are wrapped in a
   fenced ` ``` ` code block and every body line is quoted with the Markdown
   `> ` prefix.

2. **HTML** — obvious headings, paragraphs, links, bold, and italics are
   converted to their Markdown equivalents; most other markup is stripped,
   and a handful of common character entities are decoded.

3. **Markdown** — text that already looks like Markdown is lightly
   normalized: a blank line is guaranteed between consecutive bulleted or
   numbered list items, and `-` bullet markers are rewritten as `*`.

4. **Plain text (fallback)** — minimal conversion: a hard line break that is
   *not* the end of a paragraph (i.e. a newline whose next line is
   non-blank) gets an explicit `<br>` appended so the break survives
   Markdown rendering.

## Installation

Copy `yank-as-markdown.el` somewhere on your `load-path`, then:

```elisp
(require 'yank-as-markdown)
```

## Usage

```
M-x yank-as-markdown
```

or bind it to a key, e.g.:

```elisp
(global-set-key (kbd "C-c C-y") #'yank-as-markdown)
```

Mark is pushed at the insertion point, as with `yank`, so
`\[exchange-point-and-mark]` jumps back to the start of the inserted text.

## Customization

- `yank-as-markdown-announce` (default `t`) — if non-nil, echoes which
  heuristic was applied after inserting the text.

## Extending

The dispatch table lives in `yank-as-markdown-heuristics`, an alist of
`(PREDICATE . CONVERTER)` pairs tried in order; the first predicate that
matches the raw kill-ring text wins. New heuristics can be added over time by
pushing additional pairs onto that list — more specific detectors should be
placed before more general ones, since order determines precedence.

## Requirements

Emacs 26.1 or later.
