# Careem Design Companion

An AI design assistant prototype, built for the Careem Senior Product Designer challenge. One self-contained `index.html` — HTML, CSS and vanilla JS, no build step, no dependencies.

It helps a product designer with three chores:

| Tab | What it does |
|---|---|
| **1 · Feedback Synthesizer** | Paste raw usability feedback or app reviews. Claude clusters them into themes, rates each **Critical / Major / Minor**, names the most likely UX cause and proposes one concrete fix. The source lines stay visible under each theme, and the header flags any item no theme claimed, so the clustering can be audited. |
| **2 · UI Copy Generator** | Describe a screen, pick a tone (Friendly / Neutral / Urgent) and a language flavor (English / English with Arabic-market sensibility). You get heading, body, primary CTA, secondary CTA and an error-state variant, in two alternative versions, each previewed inside a phone mock. Every string has a copy button. |
| **3 · Layout Brainstormer** | Describe a feature. Claude returns three deliberately different layout concepts, each with a name, an above-the-fold hierarchy, a component list, one honest trade-off, and a low-fidelity wireframe drawn from a fixed block vocabulary as pure CSS boxes. |

Prototype by **Rijad Haliti**, built with Claude for the Careem design challenge. Not affiliated with Careem; all sample data is fictional.

## Running it

Nothing to install or build.

- **Locally:** open `index.html` in a browser. Demo mode works straight from disk. For live API calls, serve the folder over http (`python3 -m http.server`) or use the hosted link — some browsers block cross-origin requests from `file://`.
- **Netlify Drop:** drag the folder onto <https://app.netlify.com/drop>.
- **GitHub Pages:** push the folder, then *Settings → Pages → Deploy from branch → main / root*.

## Demo mode vs. live mode

The **Demo mode** toggle in *Settings* is **on by default**, so the whole prototype is clickable with no API key.

- **Demo ON** returns hand-written sample outputs shaped exactly like the JSON the prompts request, after a short delay so the loading state is visible. The copy tab honours all six tone × flavor combinations. A banner above the results always says when you are looking at a sample.
- **Demo OFF** makes each run a live call to the Anthropic Messages API (model `claude-sonnet-4-6`) using the key you paste in *Settings*.

**The API key** is held in a JavaScript variable only — never `localStorage`, cookies or a server — and is gone on refresh. The request carries `anthropic-dangerous-direct-browser-access: true`, which Anthropic requires for browser-side calls. That is acceptable for a personal prototype and wrong for production, where the call belongs behind a proxy so the key never reaches the client.

Errors render inline: invalid key, rate limit, overloaded API, 90-second timeout, malformed JSON. Everything the model returns is HTML-escaped before it reaches the DOM.

**Deep links.** `#copy` opens a tab directly, `?run=1` pre-runs the sample in demo mode, and `?tone=Urgent&flavor=Arabic-market` preselects the copy controls — e.g. `index.html?run=1#layout`.

## How the app talks to Claude

One request per run: a `system` prompt for role and voice, a `user` prompt for task, rules and output schema. Every prompt ends with an instruction to reply **only** with JSON matching that schema. The app strips stray code fences, parses, normalises (severity spelling, id ranges, unknown block types, missing fields) and renders styled cards. Demo mode feeds hand-written JSON of the same shape into the same renderers, so both paths exercise identical UI code.

## The three prompt templates

Exactly as they appear in the `PROMPTS` object in `index.html`. `${…}` marks values inserted at runtime.

### 1 · Feedback Synthesizer

**System** — *You are a senior UX researcher embedded in the design team of Careem, a ride-hailing and delivery super-app in the Middle East. You turn messy qualitative feedback into precise, prioritised design insight. You never pad, never moralise, and you always respond with a single valid JSON object and nothing else.*

**User**

```text
Here are ${n} raw feedback items from users of a ride-hailing app, one per line and numbered:

1. ${line 1}
…

Cluster these into 3-6 themes. Rules:
- Every item belongs to exactly one theme; reference items by their number in "feedback_ids".
- "severity": "Critical" blocks a core task, loses money or trust, or is a crash; "Major" is
  significant friction on a frequent task; "Minor" is cosmetic, rare, or has an easy workaround.
- "ux_cause": the most likely underlying design cause, one specific sentence (think information
  architecture, affordance, system feedback, error prevention, layout and thumb ergonomics,
  state persistence, perceived performance).
- "fix": one concrete, shippable design change in one sentence, written as an instruction a
  designer could act on tomorrow.
- Order themes from most to least severe.
- "summary": one sentence describing the overall pattern across all items.

Reply with ONLY this JSON, no markdown fences, no commentary:
{"summary": string, "themes": [{"title": string (max 6 words), "severity": "Critical" | "Major" |
"Minor", "feedback_ids": number[], "ux_cause": string, "fix": string}]}
```

**Why it is shaped this way.** Severity gets operational definitions, because without them the model drifts between runs and everything becomes "Major". Items are referenced by number rather than re-quoted, which keeps the output small, stops the model paraphrasing users, and lets the UI show the real lines under each theme. Cause and fix are separate fields because a designer needs to agree with the diagnosis before accepting the prescription.

### 2 · UI Copy Generator

**System** — *You are a senior UX writer at Careem. Voice: clear, warm, human, never robotic or salesy. Sentence case everywhere. CTAs are short verbs that describe the outcome ("Cancel ride", "Keep ride"), never "OK", "Yes" or "No". Drivers are Captains. Body copy states the consequence before the choice. You always respond with a single valid JSON object and nothing else.*

**User**

```text
Screen: "${screen}"
Tone: ${tone} - ${tone guidance}
Language flavor: ${flavor} - ${flavor guidance}

Write two distinct alternative versions of the copy for this screen. A and B must take different
angles (for example A reassures and B is direct, or A leads with the consequence and B leads with
the choice), not just word swaps.

Constraints for each version:
- heading: max 6 words
- body: max 25 words, states what happens if the user proceeds
- primary_cta: max 3 words, the action the screen exists for
- secondary_cta: max 3 words, the safe way out
- error_state: the copy shown when the primary action fails: {"heading": max 6 words,
  "body": max 20 words, must say what happened and what to do next}
- rationale: one sentence explaining the angle of this version, written for the design team

Reply with ONLY this JSON, no markdown fences, no commentary:
{"screen": string, "versions": [{"label": "A", "heading": string, "body": string,
"primary_cta": string, "secondary_cta": string, "error_state": {"heading": string,
"body": string}, "rationale": string}, {"label": "B", … same shape …}]}
```

**Tone guidance** inserted at runtime:

| Tone | Guidance |
|---|---|
| Friendly | warm and conversational, light touches of personality, contractions are fine, never cute for its own sake |
| Neutral | plain, efficient and informational; no personality flourishes, no persuasion either way |
| Urgent | time-sensitive and action-first; make the consequence and the deadline concrete without alarmism, no ALL CAPS, no exclamation marks |

**Language flavor guidance** inserted at runtime:

| Flavor | Guidance |
|---|---|
| English | standard international English |
| Arabic-market | English written for a Gulf/MENA audience that will also be translated into Arabic: warm and respectful register, hospitality-minded, no Western idioms, puns or slang, keep every string roughly 25% shorter than usual so it survives Arabic expansion and RTL layouts, prefer "please" over playful phrasing, be mindful of cultural and religious context (prayer times, Ramadan, family), and always call drivers Captains |

**Why it is shaped this way.** Demanding two *different angles* rather than synonyms is what makes the pair worth comparing. Word limits are per field so the copy fits real components. The error state is part of every version because the failure path is where copy quality is most often neglected. The Arabic-market flavor encodes localisation realities — string expansion, RTL, register — rather than translating, so an English-first team gets copy that survives the Arabic pass.

### 3 · Layout Brainstormer

**System** — *You are a senior product designer at Careem exploring layout directions before any high-fidelity work. You think in hierarchy, jobs-to-be-done and thumb reach, and you are honest about trade-offs. You always respond with a single valid JSON object and nothing else.*

**User**

```text
Feature: "${feature}"
Platform: mobile app screen, portrait, 390 x 844 pt.

Propose exactly 3 genuinely different layout concepts. They must differ in what dominates above
the fold and in how the hierarchy is organised, not three variations of one idea.

For each concept:
- name: max 4 words
- hierarchy: 2-3 sentences describing what the user sees above the fold, top to bottom, and why
  that order
- components: 5-8 named UI components
- tradeoff: one sentence on what this layout gives up
- wireframe: 5-9 blocks, top to bottom, that a renderer will draw as low-fidelity boxes. Each
  block is {"type": one of header | map | image | hero | text | card | list | chips | stepper |
  progress | input | cta | buttons | avatar | divider | sheet | tabs | nav, "label": max 3 words,
  "size": "sm" | "md" | "lg"}. Use "nav" only as the last block and at most one "map".

Reply with ONLY this JSON, no markdown fences, no commentary:
{"feature": string, "concepts": [{"name": string, "hierarchy": string, "components": string[],
"tradeoff": string, "wireframe": [{"type": string, "label": string, "size": "sm"|"md"|"lg"}]}]}
```

**Why it is shaped this way.** The model does not draw the wireframe, it specifies one: it picks from a fixed vocabulary of 18 block types and three sizes, and the app renders each block as a CSS box. Output stays deterministic and cheap, fidelity stays honest, and the same JSON could later drive a Figma plugin instead of CSS. Blocks shrink to fit the phone frame, so a nine-block concept renders in full rather than clipping. Forcing a trade-off per concept is what stops the model proposing three safe variants of one layout.

## Design system

Colour, type, radii and shadows follow Careem's visual language: neon green `#00EB79` for actions with **dark** text on it (white on that green is 1.6:1 and fails), forest `#00493E` for headings, Inter for UI with Outfit standing in for CareemSans, radius 8 on controls and 16 on cards, and the blue / mint / purple service-card colours to separate the three layout concepts. Every colour in the components reads from a CSS custom property in the `:root` block; there is no hardcoded hex below it. Severity and error states use desaturated reds and ambers so they stay rare against the brand palette. The header wordmark and favicon are Careem trademarks, used to make the prototype feel native.

Result grids use container queries, so cards reflow to the width they actually have. Keyboard: arrow keys move between tabs, ⌘/Ctrl+Enter runs the active tool. Reduced motion is respected.

## Files

```
careem-design-companion/
├── index.html    tokens + CSS, markup, prompts, demo data, API client, renderers
└── README.md     this file
```

## Limits and next steps

- Browser-side API calls expose the key to whoever is using the page. Beyond a personal prototype, add a proxy function and drop the browser-access header.
- Demo mode ships one sample brief per tab, plus six tone/flavor variants for copy. Live mode has no such limit.
- Natural extensions: export a theme board to Markdown or FigJam, drive real Figma frames from the wireframe JSON, and add a fourth tab that reviews a screenshot for accessibility.
