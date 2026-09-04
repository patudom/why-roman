# Decisions

Why things are the way they are. The code shows *what*; this file is the *why*, for
the decisions that took discussion and would otherwise get relitigated.

Written 2026-09-04, covering work from 2026-08-25 onward. Where this file and the code
disagree, **the code is right** — every claim below was checked against the source on
the date above, and two labels recorded in my working notes turned out to be wrong.

## Privacy notice

The largest cluster of decisions, most of them driven by the PI's objection to anything
that covers the sky view.

- **It is a small corner card, not a strip across the bottom.** An earlier version
  spanned the window and had to dodge every panel in the bottom band, which meant it
  kept climbing into the WWT view. The card covers a fixed ~304×71px patch instead. The
  general principle, worth keeping: a small thing that overlaps a little beats a big
  thing that has to avoid everything.
- **It appears only after the intro slides close**, not alongside them. The intro is a
  modal dialog and takes focus, so a notice opened underneath it is unreachable — a
  keyboard user could not answer it without tabbing the whole app first. It now takes
  focus itself when it opens, so it is the first thing reached.
- **It shows at most once per session** (`privacyNoticeShown`). Replaying the tour
  re-opens the intro, which would otherwise re-trigger the notice for anyone who let it
  time out. The lock icon still opens it on demand — that path deliberately ignores the
  flag.
- **In wide landscape the opacity slider yields to the card**, rather than the card
  moving. `#bottom-content` gets a left padding while `app-privacy-notice-open` is set,
  pushing the slider right; the card stays on the lock's baseline. This was the PI-
  friendly resolution: the furniture moves, not the sky coverage.
- **In portrait and small landscape the card lifts above the bottom band instead**
  (`--privacy-band-clearance`). There is no room to seat both: at 700×400 the band is
  388px wide, and reserving 304px for the card leaves 52px, so the row just overflows.
  **This lift was deliberately kept** after being removed once — a tour step can put an
  opacity slider in that band while the notice is still up, so the gap is insurance, not
  waste.
- **A 20s timeout dismisses it** (`PRIVACY_NOTICE_TIMEOUT_MS`) because the dialog is
  persistent and would otherwise hold the corner indefinitely.

## Moons

- **The moon size comparison is parked, not deleted**, at the PI's request — they did
  not like the visualization. `showMoons` returns `false` with the real condition
  commented directly above it. That one computed gates both the slider and the
  `drawMoon` call in the frame callback, so the single line takes out the whole feature.
  `moonsOpacity`, `MOON_POSITIONS`, the `#moons-control` markup and `src/moon.ts` are
  all left in place.
- The step 1 text was reverted to its pre-slider wording, *"It spans 3 degrees (or 6
  full Moons!) across the sky."* The moon **mention** stays — it is a size comparison,
  not a reference to the visualization. The revert was checked against history first:
  two commits had touched that line, and the one titled "Text edits to address feedback
  from SED colleagues" left it alone, so no colleague feedback was undone.

## Labels

Settled after the team disagreed. Current values in `src/RomanFov.vue`:

| Footprint | Label |
| --- | --- |
| Roman planned pointings | `Roman Planned Images` |
| Roman pointings, per-chip | `Roman Images (Chips)` |
| Hubble boundary | `Hubble Survey Area` |
| Hubble pointings | `Hubble Images` |

The rule that settled it: **telescope name always first**, and no jargon the public
would not recognise — which is why "footprint" and "pointings" do not appear in any
user-facing label. Rejected alternatives included "Proposed Roman Survey" and "Roman
Survey (Simplified)".

The control panel's section headers are `Compare Fields of View` and `Andromeda`.
(My notes recorded a decision to rename the second one "Mapping Andromeda" — that is
**not** in the code, so either it was reverted or it never landed. Treat the code as
current and re-decide if it matters.)

## Telescope info cards

- Each telescope in the Compare Fields of View controls has an info button opening a
  card, sourced from `src/telescopeInfo.ts`.
- The camera named in each entry is **the one whose footprint this app draws**, not the
  mission's most famous instrument — the numbers only mean something next to the shape
  on screen.
- **Open:** that file's header says the figures want a read-through from someone on the
  science side. As far as I know that has not happened.

## Layout and CSS

- `IntroSlides.vue` deliberately uses Vuetify's own card styling rather than overriding
  it. The overrides were losing to the duplicated Vuetify CSS once built, and the
  elevated card that fell out of that is the look we wanted — so dev and deployed now
  agree instead of fighting.
- Fixed `height: 60vh` was dropped from the intro card so it sizes to its content; that
  is what stopped the figure caption being clipped.
- The three top-left buttons appear after the user leaves the tour (`inExploreMode`),
  not when entering the last tour step.

## Open questions

Nobody has decided these. They are not bugs — they need a person.

1. **Data collection begins before consent is answered.** `createUserEntry()` runs in
   `onMounted` (`src/RomanFov.vue`), and its guard is
   `if (responseOptOut.value) return;` — for a first-time visitor `responseOptOut` is
   `null`, which is falsy, so it proceeds. A user UUID is also minted and written to
   `localStorage` at composable setup. Both happen before the notice is shown, let alone
   answered.
2. **An unanswered notice reads as "allow."** If the 20s timeout fires,
   `responseOptOut` stays `null`, which every guard treats as consent. If the team wants
   silence to mean *don't collect*, that is a real change.
   Items 1 and 2 are an IRB / team call, not an engineering one. The design is opt-out
   rather than opt-in, which is common for NASA/CfA outreach but is worth someone
   signing off on deliberately.
3. **`telescopeInfo.ts` figures need a science review** (see above).
4. **Firefox footprint line widths.** A thickness of 2 looked right in Chrome and came
   out garbled in Firefox, so it was reverted. It reproduces only when the browser zoom
   is not 100% — the canvas is sized in CSS pixels and resampled at a non-1
   `devicePixelRatio`, and the engine has no `devicePixelRatio` handling anywhere. Not
   fixed; the revert stands.
5. **`.claude/` is untracked** in this repo. Decide whether it holds settings the team
   should share, or belongs in `.gitignore`.

## Related documents

- `TOOLKIT-UPSTREAM.md` — five `@cosmicds/vue-toolkit` 0.7.0 defects this app works
  around, with evidence and upstream fixes. Also copied into `almagal`, which is
  affected by all five.
- `CLAUDE.md` — conventions and the layout/WWT quirks that keep biting.

## Archive of record

The full conversation behind this work — every message and tool call from 2026-08-25 to
2026-09-04, 9,509 entries — is at:

```
~/.claude/projects/-Users-Pat-Documents-GitHub-cosmicds-wwt-vue-why-roman/cd437514-0911-4c32-97f8-e23276af29fe.jsonl
```

**Caveat:** that directory name is the *encoded path of this repo*. Moving or renaming
the repo does not move the transcript, and nothing will point at it afterwards. If this
repo relocates and the history still matters, copy that file somewhere durable first.

## Branches not merged to `main`

As of 2026-09-04, in case something is stranded on one of them:

`claude-updates` (most recent), `remove-moons-slider-and-text`,
`privacy-policy-card-updates`, `fix-keyboard-accessibility-issues`,
`address-comments-from-roman-team`, `fix-issues-with-scrim-on-privacy-dialog`,
`add-info-popups-for-each-telescope`, `fix-text-and-positioning-of-privacy-popup`.
