# STATUS — Un-Claude Markdown

_Last updated: 2026-08-27_

## Where things stand

Working and deployed at <https://cmcguinness.github.io/UnClaudeMarkdown/>.

The project's goal was restated on 2026-08-07 and is now broader than the
original commit: not just cruft removal, but **turning sloppy Claude output into
clean markdown that preserves the original formatting intent**. Cruft removal was
the first slice because it was the biggest annoyance; reconstructing what terminal
rendering flattened is the rest of the job.

The operating rule for that broader goal, chosen deliberately: **reconstruct
high-confidence patterns only.** Act on structurally unambiguous patterns, leave
everything else alone.

## Recent work

**Box tables were being destroyed (2026-08-27).** Reported from real use: tables
pasted into Slack came out with no grid. Three separate causes, all of them
pre-existing bugs in the markdown path as well, not new with Slackify:

- **Only light box-drawing was recognised.** `╭ ╮ ╰ ╯` (rounded — which Claude Code
  emits), `┏ ━ ┃` (heavy) and `╔ ═ ║` (double) all failed the `BOX_TOP` test, fell
  through to the glyph step, and had every `│` rewritten to `- ` as if the column
  were a bullet list. All four weights are now accepted.
- **An unrecognised box got dismantled.** A truncated paste (no top border) hit the
  same glyph damage, and a heavy box was additionally glued into a single line by
  the join step. Any line that opens *and* closes with a vertical, or is made
  entirely of box characters, is now held back from the glyph step and flagged
  `noJoin`. Requiring both bars keeps Claude Code's `│ continuation` chrome
  stripping exactly as before.
- **An unrecognised box was left unfenced.** Grid without monospace is not a grid.
  A run of 2+ box lines is now fenced verbatim by step 2.5 itself, so it applies to
  **both** targets — a markdown renderer would otherwise soft-wrap the box into a
  paragraph just as Slack renders it proportionally. A lone box line is still left
  alone. (This landed for Slack first and was ported to markdown the same day, at
  which point the Slack-specific copy of the logic was deleted rather than left to
  drift.)

Emphasis translation now skips box lines too — it was turning the `*/15` in a cron
expression into `_/15_`.

Verified: rounded, heavy and truncated variants of a real 3-column table from the
user all keep their grid on both targets, and the rounded one now converts to a
proper markdown table on Clean (it used to be destroyed). Markdown path still
byte-identical to the previous commit on the spec acceptance sample and the mixed
sample. Driven through the UI in Chrome with no console errors — a truncated box now
produces byte-identical output from Clean and Slackify, which is the right answer:
once a box is fenced there is nothing left for the mrkdwn step to translate.

Regression guards worth keeping in mind for the next change here: a complete box
still converts to a real markdown table rather than being fenced; a *lone* `│ … │`
line is still left unfenced; Claude Code's one-bar `│ continuation` chrome is still
stripped; and a directory tree is still caught by the tree rule, not the box rule.



**Slackify (2026-08-27).** A second output target alongside Clean, for pasting
into Slack rather than into a markdown renderer. Steps 0–5 are shared; the target
changes exactly two things:

- Step 2.5 skips the box-table conversion and fences every box verbatim. A
  markdown pipe table renders in Slack as literal pipes; the original box in a
  code block keeps its columns. A pipe table already present in the paste is
  fenced too, recognised strictly (a `|---|` separator row required).
- New step 6 translates markdown markers to mrkdwn: headings → bold lines,
  `**bold**` → `*bold*`, `*italic*` → `_italic_`, `~~strike~~` → `~strike~`,
  `[text](url)` → `<url|text>`, bullets → `•`, thematic breaks dropped and the
  surrounding blanks re-collapsed. Code spans and link URLs are lifted to NUL
  placeholders before emphasis runs so a URL containing `_` or `*` survives;
  bold is held as a placeholder too, so the italic pass can't see its stars.

The scope rule held: this translates markers that survived, it does not guess at
emphasis the terminal already destroyed.

**Known limit, recorded in the spec:** Slack's rich-text composer and the "format
messages with markup" preference don't treat pasted markers identically. mrkdwn is
right for the preference and close for the composer. Deliberately not offering two
modes for this.

Verified: the markdown path is byte-identical to `HEAD` on both the spec
acceptance sample and a mixed table/tree/list/link sample (diffed against the
pre-change `clean()` in the node harness). Driven through the real UI in Chrome —
both buttons disabled on empty input, Slackify sets the output title to
"Slack-ready" and Clean back to "Cleaned", Clear resets both panes and the title
and refocuses input, and a real click on Copy showed the "Copied to clipboard"
toast. (A *synthetic* click on Copy reports failure — no user activation — which
is expected and unrelated.)

**Resolved in real use (2026-08-27):** the composer-vs-markup-preference question is
settled. Charles tried it in an actual workspace: with Slack's **"Format messages
with markup"** preference *off*, backticks paste as literal characters and no code
block forms; turning it *on* makes the output work. So mrkdwn is the right target
and the preference is a genuine prerequisite, not a nicety — worth saying out loud
to anyone who tries Slackify and thinks it is broken.


**Box tables and trees (2026-08-07).** New pipeline step 2.5, running before the
glyph step so the box-drawing characters still exist when it runs:

- Box-drawn tables → real markdown tables. Cells wrapped across physical lines
  rejoin; columns pad to line up; `|` inside a cell is escaped.
- Directory trees → fenced blocks, with the root line pulled in when it's a bare
  path. Markdown would otherwise collapse a tree's line breaks.
- A box with fewer than two columns (boxed note, ASCII art) is fenced verbatim
  rather than converted or stripped.
- **Bug fixed:** `├ └ ─` were in the glyph set and were being stripped as chrome,
  turning `├── src/` into `── src/`. They're structure, not chrome, and are now
  out of that set entirely.
- An isolated branch line — too weak a signal to be a confident tree — is left
  alone but flagged `noJoin`, so the join step can't glue it into a paragraph.

Verified against seven cases (table, tree, transcript glyphs, hard-wrapped
paragraph, boxed note, fenced box, lone branch), plus a row-length assertion
confirming the emitted table columns actually align. The original spec acceptance
sample was re-run and does not regress. Driven through the real UI in Chrome:
Clean produced the right output and Clear reset both panes and both buttons.

**Not done in that browser pass:** the extension disconnected before I checked the
console for errors or captured a screenshot. The page executed the pipeline
repeatedly without throwing, so there's no parse-level problem, but a console
check is still owed.

**Earlier:** `b808698` — initial commit, the base pipeline. Two rules from it worth
remembering: dedent uses the *most frequent* leading indent rather than the
minimum, and a glyph-stripped line is flagged `noJoin`.

## Open threads

- **Console check owed** on the box-table change (see above).
- **Next candidates for the intent-preservation goal**, none started: detect and
  fence unfenced code blocks; recover headings that rendered as plain text;
  normalize list nesting. Roughly a week together, per the earlier sizing.
- **Inline emphasis is out of reach** and should stay out of scope. A plain-text
  terminal copy has already lost bold/italic — no markers, no ANSI escapes to
  recover from. Say so rather than accepting a task premised on it.
- **A table without a top border isn't recognised.** A paste truncated at the top
  falls through untouched. Deliberate under the high-confidence rule, but it's the
  most likely thing to come up as a real complaint.
- **`.venv/` is a leftover** from the abandoned Flask scaffolding. Untracked, so
  cosmetic — delete whenever.
- **Rules are still tuned to few samples.** Collect the failing paste before
  changing a rule.

## If you're picking this up

Read `specification.md` first — it's the source of truth and records *why* the
non-obvious rules are the way they are. `index.html` is ~730 lines and comments
each step back to its spec section. `CLAUDE.md` has the guardrails and the node
harness trick for iterating on rules without a browser round-trip.
