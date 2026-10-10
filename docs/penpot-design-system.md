# Penpot screens and the design system: how the pieces fit together

The goal: the builder agent (the eventmodelers CLI's build loop, running inside a
`run-agent-task` VM) implements screens from their Penpot designs while strictly
following the Gastronak design system.

## The pieces

| Piece | Where | Role |
|---|---|---|
| `public/css/main.css` | gastronak repo | **The design system and the source of truth** for every design value |
| Penpot file "Components" | Penpot project "Gastronak v2" (file id `17193a1f-939a-80b5-8008-c2792aaa5301`) | Screens (page "Pages") and the design-system components (page "Design system"). Its tokens are a *rendering* of `main.css` |
| Event Modeling board | EM Studio (eventmodelers) | Slices; each SCREEN carries the exported Penpot image (for people) and a `penpot:` reference (for the agent) |
| `scripts/penpot/generate-tokens.mjs` | gastronak repo | CSS → Penpot tokens. **Run by you**, never by the agent |
| `scripts/penpot/drift-check.mjs` | gastronak repo | Penpot ↔ `main.css` consistency check. Run by you and by the agent |
| `scripts/penpot/screen.mjs` | gastronak repo | Prints one Penpot board as a spec to implement from. Run by the agent |
| `scripts/penpot/lib.mjs` | gastronak repo | Shared code: auth, fetching the file, finding boards, the naming rule |
| `.claude/skills/penpot-screen/SKILL.md` | gastronak repo | The agent's instructions for implementing a Penpot screen |
| `CLAUDE.md` | gastronak repo | Points every Claude session at the skill and the "never edit main.css" rule |
| `run-agent-task` | this repo | Runs the agent in a disposable smolvm VM |

All scripts are plain Node (>= 18) with no dependencies, so they run in the VM's
`node:24-slim` image. `generate-tokens.mjs` also needs Chromium, which is why it runs on
your machine.

## The naming rule

One fixed rule links Penpot token names to CSS, used by every script and by the skill:

```
color.fg-base       -> var(--color-fg-base)
step.-1             -> var(--step--1)
typography.heading  -> class="typography:heading"   (.typography\:heading in main.css)
```

Penpot shapes reference tokens **by name**, so token names never change in Penpot. When a
name has to change, rename it on the CSS side.

## Workflows

### Changing the design system (you)

1. Edit `main.css`.
2. `PENPOT_TOKEN_FILE=… node scripts/penpot/generate-tokens.mjs --dry-run` and read the
   CHANGED / NEW / REMOVED report. It refuses to write the file if a token used in Penpot
   would disappear (`--allow-removals` overrides).
3. Run it without `--dry-run`. It writes `design/penpot-tokens.json`; commit it, so token
   changes show up as readable diffs.
4. In Penpot: Tokens panel → Import that file. Try it on "Components (copy)" first.
5. `node scripts/penpot/drift-check.mjs` must exit 0.

Values in Penpot are what the browser renders at a 1280px-wide viewport in the light
scheme, measured in headless Chromium. Fluid values (`clamp()`, `vw`, `rem`) therefore
show their 1280px size. Penpot can't express fluid values; `main.css` stays the truth.

### Designing a screen (designer)

1. Build the screen as a top-level board on the "Pages" page, using the design-system
   components and applying tokens to every fill, text style, padding, gap, radius and
   shadow.
2. Put the exported image on the slice's SCREEN on the Event Modeling board, and add a
   line to the SCREEN's description:
   ```
   penpot: <board name | board id | Penpot link with board-id=…>
   ```
3. Optional: `node scripts/penpot/drift-check.mjs --board "<name>" --strict` shows raw
   values and untokenised text on that board.

### Building a screen (agent)

The builder runs `claude` with `--dangerously-skip-permissions` and no MCP restrictions,
from the repo root. So it picks up the repo's `.claude/skills` and `CLAUDE.md`. For a
SCREEN with a `penpot:` line, the skill makes it:

1. Run `drift-check.mjs --board "<ref>"`. If tokens are missing from `main.css` it stops and
   reports the slice as blocked; it never adds tokens itself.
2. Run `screen.mjs "<ref>"`: layout, spacing, sizing and colours as token names, exact
   text, component instances with variants, and gaps marked `(no token)`.
3. Map each Penpot component to its existing Hiccup markup (the skill has a table, e.g.
   Button → `input[type=submit]`, Card → `section.card`), and use only `main.css`
   variables, `typography:*` classes and utility classes.
4. Re-run the drift check and look for raw values in its diff.

Penpot is reached through its REST API with an access token, **not** through an MCP
server. The official Penpot MCP server only works through a plugin running in an open,
focused browser tab, which an unattended agent can't rely on.

## Running it in the sandbox

The agent needs a Penpot token and network access to Penpot. Nothing Penpot-specific is
built into `run-agent-task`; you set it per run:

```bash
# task.env -- one KEY=VALUE per line, never committed
PENPOT_ACCESS_TOKEN=…

export TASK_ENV_FILE=~/secrets/task.env
export ALLOW_HOSTS_EXTRA="design.penpot.app <eventmodelers API host>"
./run-agent-task <gastronak repo url> -- npx @eventmodelers/cli@1.0.148 run
```

Pin the eventmodelers CLI version: the agent setup depends on its internals, checked
against 1.0.148. That version's **builder** (`run`) starts `claude` with no MCP lockdown.
Its **modeling** agent (`run --modeling`) is locked down to the eventmodelers MCP server
only, by design; that doesn't matter here, because the Penpot work happens in the builder.

A Penpot access token acts as the account that created it. Use a dedicated Penpot account
that can only view the Gastronak team for the agent's token.

## Decisions

- **`main.css` is the source of truth**; Penpot mirrors it. Typography is classes
  (`.typography\:x`); renames happen on the CSS side.
- **`<html>` picks up `--step-0` on purpose** (the `*` rule), so the root font size is fluid
  too and `1rem` is 18.12px at 1280px. Penpot's current values were calculated with a
  16px rem, so they are about 11% smaller than what the app renders. The generated tokens
  correct that; importing them scales Penpot's spacing, sizes and type up to match the app.

## Open items

- **Light colours outside the sRGB gamut** (`color.dom-5/6`, `sub-5/6`, `acc-5/6`; to be
  fixed later in `main.css`). The browser clips them: button backgrounds render #FFC7D3,
  surfaces #FFF0E9, where Penpot shows near-white. Until fixed, `generate-tokens.mjs`
  reports them as changes, and importing writes the clipped colours into Penpot.
- **First token import** hasn't been done yet: do it on "Components (copy)", check, then on
  the real file.
- **Design gaps** the drift check warns about: a hard-coded `#ffffff` on the `menu` frame in
  "Meal plan - add recipe - menu open", and text layers without a typography token
  (`quote-mark`, `upcoming-label`, the "Note / …" board).
- **No PNG export for the agent.** Penpot's renderer needs a browser login session; an
  access token can't provide one. The structure dump is the spec.
- **`eventmodelers init`** asks before replacing the gastronak `CLAUDE.md`. Keep it, and add
  the router's lines to it.
- **`penpot-token`** in this repo is a live token; keep it out of git (add it to `.gitignore`).
