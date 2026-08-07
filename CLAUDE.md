# Un-Claude Markdown — working notes

A single self-contained web page that turns text copied from the Claude Code
terminal into clean markdown preserving the original formatting intent. Live at
<https://cmcguinness.github.io/UnClaudeMarkdown/>.

**The goal is intent preservation, not just cruft removal.** Cruft removal was
the starting point because it was the biggest annoyance, but Claude's terminal
output is rendered for a terminal and often never meant to be re-used. Restoring
it means removing what the terminal added *and* rebuilding what it flattened —
box-drawn tables back into markdown tables, trees back into fenced blocks.

## Project shape

- `index.html` — **the entire application.** Inline CSS and JS.
- `specification.md` — the source of truth for behavior. Read it before changing
  any cleanup rule; it records the reasoning behind several non-obvious choices.
- `README.md`, `STATUS.md` — orientation and current state.

There is no build step, no package manager, no server, no test framework, and no
Python. Do not add any of these without asking. In particular:

- **Never add an external dependency, CDN link, web font, or stylesheet.** The
  single-file, zero-dependency, opens-by-double-clicking property is a
  requirement, not an accident.
- **Never move the transformation server-side.** Everything runs in the browser
  precisely so that nothing the user pastes ever leaves it. That property is why
  the page is safe to host publicly.
- `index.html` is ~570 lines. It is allowed to be one file — the usual
  "split large files" guidance doesn't apply here, since being one file is the
  point. Revisit only if it passes ~1500 lines.

## Scope guardrail

This is not a markdown linter or pretty-printer. It does not reflow to a column
width, renumber lists, canonicalize bullet characters, or normalize heading
styles. It only reverses what terminal rendering did to the text.

**The operating rule is: reconstruct high-confidence patterns only.** Act when
the pattern is structurally unambiguous; leave the text alone otherwise. A rule
that imposes structure the author didn't intend is worse than one that misses
some, because the user won't notice the damage. Keep recognition thresholds
strict — a complete top border for a table, two branch lines for a tree — and let
anything weaker fall through to the ordinary rules.

**Inline emphasis is unrecoverable.** A plain-text terminal copy has already lost
bold and italic — no markers, no escapes. Don't accept a task premised on
restoring them, and don't add heuristics that guess at emphasis. Block structure
is the achievable scope.

## Changing a cleanup rule

The pipeline is ordered steps (normalize → dedent → strip trailing → reconstruct
blocks → glyphs → collapse blanks → join wraps), each commented in `index.html`
back to its `specification.md` section. Block reconstruction runs *before* the
glyph step on purpose: the glyph step would otherwise eat the box-drawing
characters that carry the structure. When you change one:

1. Update `specification.md` in the same change. The spec and the code are meant
   to agree; a divergence between them is a bug in its own right.
2. Get the failing paste first. Every heuristic here was tuned against a real
   sample, and changing one on speculation tends to break another.
3. Check it doesn't regress the acceptance sample (below).

Rules that look wrong but are deliberate, with the reasoning in the spec:

- Dedent uses the **most frequent** leading indent, not the minimum.
- Trailing-whitespace stripping kills markdown's two-space hard-break idiom.
  Accepted.
- A line whose UI glyph was stripped is flagged `noJoin` so it can never be
  joined to its neighbours.
- An unclosed code fence stays open through the end of the text.
- Code-block interiors are excluded from the dedent vote but still dedented.
- Tree branches (`├` `└` `─`) are **not** in the glyph set. They're structure, not
  chrome; putting them back turns `├── src/` into `── src/`.
- A single-column box is fenced verbatim, not converted — it's a boxed note or
  ASCII art, and its border was deliberate.
- Reconstructed blocks carry a `raw` flag; every later step must pass them
  through untouched. If you add a step, honour `raw`.

## Testing

Manual, by design — the spec rules out a test framework and fixture files.

For fast iteration on pipeline rules, you can drive `clean()` from node without
adding anything to the repo: read `index.html`, take the `<script>` contents, cut
it at the `UI wiring` comment banner, and `new Function(pipeline + '; return
clean;')()`. Keep that harness in a scratch directory, not in the project.

The acceptance test is the long sample paste in `specification.md`'s Problem
Statement: open `index.html` in Chrome, paste it, press Clean, and confirm the
output renders as correct markdown (no stray `<br>` inside paragraphs, list
structure intact, no leading indent). **Do this in a real browser before handing
work back** — reasoning about the output is not sufficient.

Also spot-check the UI states after any wiring change: Clean disabled on empty
input, Copy disabled on empty output, Clear resets both and refocuses input, and
the copy toast appears (`navigator.clipboard` first, `execCommand` fallback on
`file://`).

## Git and deploy

This directory has a `.envrc` — run git/gh through `direnv exec .` (e.g.
`direnv exec . git push`), or you get the wrong gh account and commit identity.

Deploy is a push to `main`: GitHub Pages serves the repo root. There is nothing
else to run.
