# Un-Claude Markdown

Text copied out of the Claude Code terminal doesn't paste cleanly into a markdown
renderer. It arrives with a leading indent on every line, trailing space padding,
hard line breaks inside paragraphs, and Claude Code's own UI glyphs (`⏺`, `⎿`,
box rules). Worse, anything the terminal *rendered* has lost its markdown: a
table is now box-drawing characters, a file listing is now a tree.

This is a single web page that turns that back into clean markdown — removing
what the terminal added, and rebuilding what it flattened.

**Live: <https://cmcguinness.github.io/UnClaudeMarkdown/>**

## Usage

Paste into the left pane, press **Clean**, copy the result out of the right pane.
That's the whole tool — no sessions, no accounts, no settings.

Press **Slackify** instead of Clean when the destination is a Slack message. Same
pipeline, but the output is Slack mrkdwn: headings become bold lines, tables stay
as fenced monospace boxes (Slack has no tables), `**bold**` becomes `*bold*`,
`*italic*` becomes `_italic_`, `[text](url)` becomes `<url|text>`, and bullets
become `•`.

**Turn on Slack’s “Format messages with markup” preference** (Preferences → Advanced)
before using this. With it off, Slack’s composer pastes the backticks as literal
characters and no code block forms — which makes tables in particular look broken.

You can also just double-click `index.html` and use it offline. On a `file://`
page `navigator.clipboard` may be unavailable, so Copy falls back to
`document.execCommand('copy')`; it still works.

## What it does

Applied in order (see [specification.md](specification.md) for the full rules):

0. **Normalize** — CRLF → LF, non-breaking spaces → spaces, zero-width
   characters removed, leading tabs expanded (tab stop 4).
1. **Dedent** — strips the *most frequent* leading indent, not the smallest, so
   one outlier line can't leave a stray space on every other line. Relative
   nesting is preserved.
2. **Strip trailing whitespace** — including markdown's "two trailing spaces =
   hard break" idiom, which in a terminal copy is essentially always padding.
2.5 **Reconstruct box-drawing blocks** — a box-drawn table becomes a real
   markdown table, with wrapped cells rejoined and columns padded to line up. A
   directory tree gets fenced so its structure survives. A box that isn't a
   table (a boxed note, ASCII art) is fenced verbatim rather than mangled. All
   four box weights are recognised — light, rounded, heavy, double. A box that
   *isn't* recognised — a truncated paste, an unusual border — is fenced
   verbatim rather than converted, so its grid survives instead of having its
   `│` stripped as a UI glyph and its lines soft-wrapped into a paragraph.
3. **Handle UI glyphs** — two or more lines at the same indent starting with the
   same glyph are a bullet list and become `- `; an isolated glyph is chrome and
   is stripped. A stripped line never joins its neighbours.
4. **Collapse blank lines** — runs of two or more become one; none at the start
   or end.
5. **Join hard-wrapped lines** — consecutive prose lines merge into one
   paragraph. Fenced code, 4-space indented code, list markers, headings,
   blockquotes, table rows, and thematic breaks are all protected.
6. **Translate to Slack mrkdwn** *(Slackify only)* — headings → bold lines,
   tables → fenced blocks, `**bold**` → `*bold*`, `*italic*` → `_italic_`,
   `~~strike~~` → `~strike~`, `[text](url)` → `<url|text>`, bullets → `•`,
   thematic breaks dropped. Code spans and link URLs are held aside so emphasis
   translation can't mangle a URL containing `_` or `*`. Fenced blocks pass
   through untouched.

It is **not** a markdown linter or pretty-printer. It won't reflow to a column
width, renumber lists, canonicalize bullet characters, or normalize heading
styles — it only reverses what terminal rendering did.

When a pattern is ambiguous it **leaves the text alone**. Reconstruction happens
only for structurally unambiguous cases: a box table needs a complete top border,
a tree needs two or more branch lines. Imposing structure the author didn't
intend is worse than missing some, because you won't notice it.

One thing is out of reach: a plain-text copy has already lost bold and italic
entirely, so inline emphasis is unrecoverable. This restores block structure.
Slackify translates the markers that survived; it never guesses at ones that
didn't.

One known limit on the Slack side: Slack's rich-text composer and its "format
messages with markup" preference don't treat pasted markers identically. mrkdwn
is correct for the markup preference and close for the composer; no single output
is exactly right for both.

## Design

One self-contained `index.html` — inline CSS and JS, no build step, no
dependencies, no CDN links, no Python. All transformation runs client-side, so
nothing you paste ever leaves the browser. That's why the page is safe to host
publicly and why there is no backend to speak of.

## Hosting

GitHub Pages, from `main` at the repo root. Pushing to `main` deploys.
`index.html` at the root gives a clean URL with no filename in the path.
