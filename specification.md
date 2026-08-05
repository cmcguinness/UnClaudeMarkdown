# Un-Claude Markdown

## Problem statement

If I copy a markdown block from claude code's terminal, it tends to have a lot of extraneous cruft.  Consider this:

```
  Here's the plan for the caching layer. The short version is that we should cache at the  
  repository boundary rather than in the HTTP handlers, because the handlers already have  
  three different entry points and we'd end up with three caches that quietly drift apart.  
  
 What this changes: every read goes through one place, so invalidation has exactly one owner.
                                                                                            
  Why it's worth doing now:                                                                 
                                                                                            
  1. The dashboard query fans out to six tables and runs on every page load. It is the      
  single slowest thing we have, and it is read-mostly.                                      
  2. We are about to add a second consumer of the same data, so the duplication problem is  
  about to get worse rather than better.                                                    
  
  What we'd need:
  - a cache key derived from the tenant id and the query shape, so two tenants can never    
  collide                                                                                   
  - an explicit TTL per query type rather than one global number                            
  - a way to bust a single tenant's entries without flushing everything                     
  
  Rough shape of the interface:
  
  cached(key, ttl, () => expensiveRead())
  
  Happy to adjust any of this — the tenant-scoped key is the part I'd argue for keeping.
```

There's a bunch of unwanted leading space on each line.  Paragraphs have hard returns in them which causes many markdown formatters to insert a <BR> that's not wanted.  Often times there's trailing space on the lines too for no reason.

## Objective

A simple web page that accepts a paste of text that's been copied from Claude Code and cleans up the formatting to produce clean markdown.

Then, it displays the result and has a COPY TO CLIPBOARD button that allows the user to extract the result.

## Considerations

Definitely KISS.  No complexity, no sessions, just a one-and-done in and out.

Use simple, clean, modern styling for the page.

**Scope guardrail:** This is *not* a markdown linter or pretty-printer. It does not
normalize heading styles, reflow to a column width, renumber lists, or canonicalize
bullet characters. Its only job is to remove the terminal-copy cruft that would
otherwise destroy the formatting when the text is pasted into a markdown renderer.
When in doubt, leave the text alone.

## Architecture

- **All transformation runs client-side in JavaScript.** No server round-trip, no
  network call, no data leaves the browser.
- **Delivered as a single self-contained HTML file** with inline CSS and JS. It can be
  opened by double-clicking the file, or dropped on any static host. No Python, no
  build step, no external dependencies (no CDN links).
- The existing Flask scaffolding (`app.py`, `templates/`, `static/`) is not needed and
  should be removed.

## Cleanup rules

Applied in order to the pasted text.

**Code-fence tracking is shared.** Steps 1, 3, 4 and 5 all need to know which lines sit
inside a fence. One function does the scan — toggling in/out state on each fence line,
with an unclosed fence leaving the state on through the end of the text — and every step
calls it, rather than tracking fences independently. Both ``` and ~~~ open a fence.

**Copy behavior.** `navigator.clipboard.writeText` is tried first; if it is unavailable
or rejects (which happens on `file://` pages in some browsers) the page falls back to
selecting the output textarea and using `document.execCommand('copy')`. The failure
toast appears only if both routes fail.

### 0. Normalize characters

Before anything else:

- **CRLF → LF.** All line endings become `\n`; output uses LF throughout.
- **Non-breaking spaces (U+00A0) → regular spaces**, so terminal padding collapses like
  any other whitespace instead of surviving as invisible cruft.
- **Zero-width characters removed** (U+200B, U+FEFF, and similar invisibles) — they ride
  along in terminal copies and confuse markdown parsers.
- **Leading tabs expanded to spaces** (tab stop = 4) so the dedent measurement below is
  correct even with mixed tab/space indentation.

### 1. Dedent — remove the common leading indent

"Common" means the **most frequent** leading-indent width across non-blank lines, not
the smallest. Strip that many leading spaces from every line; lines indented *less* than
that are simply flushed left. Relative indentation is preserved, so a nested list item
indented 4 spaces under a 2-space parent keeps its extra 2 spaces of nesting.

Using the *minimum* rather than the mode was the original rule, and the sample paste in
the Problem Statement broke it: one line (`What this changes: every read…`) is indented 1
space where every other line is indented 2, so the minimum is 1 and a stray leading space
survives on every other line in the output. Markdown ignores up to 3 leading spaces, so
this still rendered correctly — but a stray indent on every line is exactly the cruft
this tool exists to remove.

Code-block interiors are excluded from the vote, so a long indented listing cannot
outvote the surrounding prose. They are still dedented by the winning amount, so nesting
inside them survives. Ties go to the smaller indent.

### 2. Strip trailing whitespace

Trailing spaces are removed from every line.

Note this also removes markdown's "two trailing spaces = hard line break" idiom. That is
intended: in a Claude Code terminal copy, trailing spaces are essentially always padding
cruft rather than a deliberate hard break, and removing them is the whole point. The rare
false positive is accepted.

### 3. Handle Claude Code UI glyphs

A terminal copy may drag along Claude Code's own chrome: message bullets (`⏺` `●`),
the tool-result elbow (`⎿`), and box/tree rules (`│` `├` `└` `─` `→`). Any such glyph
appearing at the *start* of a line (after its indent) is handled as follows:

- **If it looks like a bullet list, it becomes `- `.** The test is purely structural,
  not glyph-specific: if **two or more lines at the same indent level start with the
  same glyph**, that glyph is replaced with `- ` on each of those lines.
- **What breaks a run:** a blank line, or a line at a *shallower* indent than the glyph
  lines. Intervening lines that start with no glyph and sit at the same-or-deeper indent
  are treated as wrapped continuations of the item above and do **not** break the run.
  (This check happens before the join step, so the implementation must look ahead across
  those continuation lines.)
- **Otherwise the glyph is stripped**, along with any spaces following it *and the indent
  that positioned it* — that indent existed only to place the glyph, so the line is
  flushed left. A glyph on a single isolated line is always stripped, never converted.
- **A stripped line never joins its neighbours.** It is flagged so the join step (5)
  keeps it on a line of its own. Without this, a pasted transcript's `⏺ Running a tool`
  and `⎿ Done` become adjacent plain lines and get glued into `Running a tool Done`.

Example — converted, because there are two siblings:

```
● First point
● Second point
```
→
```
- First point
- Second point
```

Example — stripped, because each glyph is isolated:

```
⏺ Running a tool
  ⎿ Done
```
→
```
Running a tool
Done
```

Glyphs appearing mid-line are left untouched. Fenced code blocks are exempt from this
rule entirely.

### 4. Collapse blank lines

A line containing only whitespace counts as blank and is emitted as a truly empty line.
Any run of two or more consecutive blank lines collapses to exactly one. Blank lines at
the very start and very end of the text are dropped entirely, so the result begins and
ends with real content.

### 5. Join hard-wrapped lines within paragraphs

Consecutive non-blank lines are joined into a single line with a single space between
them, with these protections:

- **Fenced code blocks are never joined.** Content inside ``` fences passes through
  untouched. A paste can start or end mid-code-block, leaving an unclosed fence; an
  **unclosed fence is treated as open through the end of the text**, so everything after
  it passes through untouched. No fence is ever added or removed.
- **Indented code blocks are never joined either.** After the dedent step, a line
  indented 4 or more spaces that follows a blank line starts an indented code block; it
  and the subsequent 4+ space lines pass through unjoined, exactly like a fenced block.
  Without this, the join step would silently mangle unfenced code, which Claude Code
  output does contain.
- A line that begins a new block **starts a new output line** rather than being joined
  onto the previous one. Block starts are: list markers (`-`, `*`, `+`, `1.`, or `1)`),
  headings (`#`), blockquotes (`>`), table rows (`|`), and thematic breaks (`---`,
  `***`, `___`). Thematic breaks are included because a horizontal rule glued onto the
  paragraph above it is precisely the destroyed-formatting failure this tool prevents.
- A continuation line of a wrapped list item **is** joined onto that item.
- A line flagged `noJoin` by step 3 always stands alone.

Example:

```
  1. The share picker — Ted needs to
  pick Bob from somewhere.
  2. The receive allowlist — the
  transport requires an allowlist.
```

becomes:

```
1. The share picker — Ted needs to pick Bob from somewhere.
2. The receive allowlist — the transport requires an allowlist.
```

## User interface

### Layout

Two panes side by side, under a page title. Input textarea on the left, cleaned output
on the right, with a **Clean** control between them and a **Copy to clipboard** button
on the output pane. On narrow windows the panes stack vertically.

```
┌────────────────────────────────┐
│  Un-Claude Markdown            │
├──────────────┬─────────────────┤
│ Paste here   │ Cleaned  [Copy] │
│              │                 │
│ █            │                 │
└──────────────┴─────────────────┘
```

### Behavior

- **Cleanup runs when the Clean button is pressed**, not live as you type. The input can
  be edited freely without the output churning underneath it.
- **The output pane is read-only**, showing the cleaned markdown *source* in a monospace
  font — not a rendered preview. Read-only makes it unambiguous that Copy gives you
  exactly what the cleaner produced.
- **Copy to clipboard** puts the full cleaned source on the clipboard, and confirms with
  a small transient toast message. If the clipboard write fails, the toast says so
  rather than silently doing nothing.
- **Clear** empties both panes and returns focus to the input textarea, ready for the
  next paste.
- **Buttons disable when there is nothing to act on.** Clean is disabled while the input
  is empty; Copy is disabled while the output is empty. Their state updates as the input
  changes, so no error messaging is needed for the empty case.
- **No keyboard shortcuts.** The controls are mouse/click driven only.
- **No change summary or diff.** The page shows the result and nothing about what it
  did; the two panes sit side by side if the user wants to compare.

### Styling

Light and neutral. White/off-white background, dark text, one restrained accent color
for the buttons. System UI font for page chrome, monospace for both text panes.
Generous whitespace, subtle borders, rounded corners. No dark-mode variant.

All CSS is inline in the single HTML file — no external stylesheet, no web font, no CDN.

## Deliverable

A single file, `index.html`, at the repo root — inline CSS and JS, no external
dependencies, opens by double-clicking. The existing Flask scaffolding (`app.py`,
`templates/`, `static/`) is deleted; nothing Python remains in the project.

## Hosting

Published via GitHub Pages from `main` at the repo root. The `index.html` name gives a
clean URL with no filename in the path.

Serving over https matters for one concrete reason: `navigator.clipboard` is only
available in a secure context. Opened as a local `file://` page — or served over plain
http from a non-localhost origin — the page falls back to `document.execCommand('copy')`,
which still works but is the deprecated path. On Pages the modern API is used.

Nothing the user pastes ever leaves the browser, so hosting the page publicly exposes no
data; the page is pure client-side code with no keys or backend.

## Acceptance

The long sample paste in the Problem Statement above is the acceptance test. Implement,
open `unclaude.html` in Chrome, paste the sample in, press Clean, and confirm the output
renders correctly as markdown — no stray `<BR>` breaks inside paragraphs, list structure
intact, no leading indent. Claude Code performs this browser check before handing the
work over. No test framework, no fixture files.
