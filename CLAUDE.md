# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a single-file Emacs Lisp package: `yank-as-markdown.el`. It provides the
interactive command `yank-as-markdown`, which inserts the head of the kill ring
(the same text `yank` would insert) after running it through a chain of
heuristics that convert it to Markdown.

There is no build system, package manifest, test suite, or README in this repo —
the `.el` file's header comments and docstrings are the only documentation.
When making changes, keep the `;;; Commentary:` block at the top of the file
(lines 11-51) in sync with the actual heuristics, since it is the closest thing
this project has to a spec.

## Development

There is no Makefile, Cask file, or test runner. To sanity-check changes:

- Byte-compile to catch syntax/warning issues:
  ```
  emacs -batch -f batch-byte-compile yank-as-markdown.el
  ```
- Load interactively and exercise `yank-as-markdown` by hand (put candidate text
  on the kill ring, e.g. via `kill-new`, then `M-x yank-as-markdown`):
  ```
  emacs -batch -l yank-as-markdown.el --eval "(kill-new \"...\") (yank-as-markdown)"
  ```
  or more practically, load the file in a running Emacs (`M-x eval-buffer`) and
  test against real clipboard content copied from a browser/email client.

There are no automated tests. If you add one, prefer plain `ert` tests exercising
the individual `yank-as-markdown--convert-*` and `yank-as-markdown--*-p` functions
rather than the top-level interactive command.

## Architecture

The whole package is a **predicate/converter dispatch table**:

- `yank-as-markdown-heuristics` is an alist of `(PREDICATE . CONVERTER)` pairs,
  tried **in order**. The first predicate that returns non-nil for the raw
  kill-ring text wins, and its converter transforms the text. If nothing
  matches, `yank-as-markdown--convert-plain` is used as the fallback (it is
  not in the alist).
- Order matters: more specific detectors must run before more general ones
  (e.g. the Thunderbird-forward check runs before the generic HTML check).
- To add a new heuristic, write a `PREDICATE` function and a `CONVERTER`
  function, then push a new pair onto `yank-as-markdown-heuristics` at the
  correct position (specific → general).

Current heuristics, in dispatch order:

1. **Thunderbird forwarded message** (`--thunderbird-p` / `--convert-thunderbird`) —
   triggers on a leading `-------- Forwarded Message --------` marker. The
   marker line itself is dropped; the header block that follows it (up to the
   first blank line) is wrapped in a fenced ` ``` ` code block, and every body
   line is prefixed with `> ` (blank body lines get a bare `>` so the
   blockquote isn't broken).
2. **HTML** (`--html-p` / `--convert-html`) — cheap tag-sniffing predicate, then a
   sequence of buffer-local regex passes (via `with-temp-buffer`) converts
   headings, links, bold/italic, `<li>`, paragraph/block boundaries, and `<br>`,
   strips everything else, and decodes a small fixed set of HTML entities via
   `yank-as-markdown--decode-entities`. Order of passes matters: inline elements
   (links, bold, italics) are converted before block-level tags are turned into
   blank lines, and generic tag-stripping runs last.
3. **Already Markdown** (`--markdown-p` / `--convert-markdown`) — detects ATX
   headings, fenced code, inline links, or list items already present. The
   converter normalizes `-` bullets to `*` and ensures a blank line between
   adjacent list items, while leaving the contents of fenced code blocks (```)
   untouched.
4. **Plain text (fallback)** (`--convert-plain`) — not part of the alist; used
   when no predicate matches. Its only job is appending `<br>` to a line when
   the next line is also non-blank (a hard break inside a paragraph), so the
   break survives Markdown rendering.

Shared helpers used across converters:
- `yank-as-markdown--collapse-blank-lines` — collapses 3+ newlines to exactly 2
  and trims the result; used at the end of the HTML converter, since its
  regex-based tag stripping is sloppy about blank lines by design.
- `yank-as-markdown--decode-entities` — decodes a small fixed list of HTML
  entities; note `&amp;` is decoded *last* to avoid double-unescaping (e.g.
  `&amp;lt;` must not become `<`).

The entry point `yank-as-markdown--convert` runs the dispatch and returns
`(NAME . CONVERTED-TEXT)`; the interactive command `yank-as-markdown` inserts
`CONVERTED-TEXT` and, if `yank-as-markdown-announce` is non-nil (the default),
echoes `NAME` via `message` so the user can see which heuristic fired.
