# CLAUDE.md

Conventions and hard-won quirks for working in this repo. If an `AGENTS.md` is
present it goes deeper on architecture; this file is the orientation and the
house style.

## Where things live

| What | Where |
| --- | --- |
| The app shell — template, layout, all state | `src/RomanFov.vue` (large; nearly everything hangs off it) |
| Tour **step functions** — footprints, layers, camera per step | `src/RomanFov.vue`, one function per tour |
| Tour **content** — step titles and prose | `src/experiences/*.ts`, collected in `index.ts` as `tourExperiences` keyed by tour id |
| The tour outline / plan | `src/tour.md` |
| Footprint drawing | `src/footprint.ts` |
| Footprint geometry | `src/footprints/*.ts` — each exports `corners: Point[][]`, `[ra, dec]` in **degrees** |
| Composables | `src/composables/` — `useFootprint`, `useWtmlLoader`, `useLayerOrdering` |
| Engine monkey-patches | `src/wwt-hacks.ts` |
| Imagery collections | `public/*.wtml` — roughly one per place |
| Components | `src/components/` |

`src/experiences` is the source of truth for a tour's steps: the array length is
the step count and the titles feed the UI. The step functions in `RomanFov.vue`
must line up with it. Copy for a step goes in the experience file, never in a
component.

A tour can be entered directly with URL parameters — read them near the top of
`RomanFov.vue`'s script rather than guessing.

## Scope

Change only what the request names. No drive-by fixes, no opportunistic
refactors, no "while I was in here" cleanups, and no adding capability that
wasn't asked for. Report adjacent problems in prose and let the maintainer
decide.

This applies to the shape of a change too: if the ask is "replace these two
props with a local computed", do that and nothing else — don't also rewrite the
neighbouring lookups to route through it.

## Comments

**Comment sparsely.** A short trailing note on a branch is the house style:

```ts
if (n === 1) { // show a single hubble frame
```

Not paragraph-length preambles on every function and CSS rule. Write the code;
add a line only where the reason genuinely isn't visible. Everything you *would*
have put in a comment block belongs in your reply instead.

## Commented-out code is parked, not dead

Unused tour entries, the old options panel, the share button, whole steps,
individual CSS declarations — these are commented out deliberately and will come
back. Never tidy them away. When an interface changes, update the commented-out
code to match so it still works when uncommented.

Leave stale typos alone too; they aren't yours to fix.

## Prefer explicit over clever

- **Modes are set in one clear spot**, not derived through a chain of reactivity.
  A plain `ref(false)` flipped in three named functions beats a computed that
  infers the same thing from four other pieces of state.
- **Flat boolean refs drive the UI**, read directly by `v-if`. Transitions are
  small `handleXClose()` / `enterX()` functions. No state machine, no store.
- **Don't over-generalise.** Three fixed buttons should be three buttons, not an
  array and a `v-for`. A computed used once should be inlined where it's used.
  Data that really is data (a label↔object mapping) can stay a list.

## Components

Presentational: flat props in, events out. `RomanFov.vue` owns the layers,
camera, footprints and WWT store; children never touch them. Giving a child a
layer means giving it the engine store, which breaks: `@cosmicds/vue-toolkit`'s
published bundle inlines `@wwtelescope/engine-pinia` (pinia itself is correctly
external), so it registers its own `wwt-engine` store and anything importing the
store from there reads a different one than the app is driving. That is why
several shared components live here as local copies rather than imports. See
`TOOLKIT-UPSTREAM.md` for this and four sibling toolkit defects.

When a component needs to show something new, **reach for the smallest change
that gets there.** The failure mode to avoid is answering a modest request with
an architectural one — a new prop, plus the computeds to feed it, plus the state
to derive those from, plus threading it all down through a parent that already
had the content in hand. A slot, one extra prop, or moving two lines is usually
enough. If a change starts sprouting supporting machinery, stop and check
whether a simpler route exists.

## CSS

Style what needs styling — this is a visual project and the look matters. The
thing to avoid is *surplus*: rules added on spec, rules that duplicate a Vuetify
utility, and rules left behind after the markup they targeted has changed. Reach
for the utility classes (`d-flex flex-column align-end ga-2`) where they do the
job, write real CSS where they don't, and delete what you stop using. Colours
are often inline hex on the component — don't refactor those to variables
uninvited.

- **No container queries** — `container-type`, `@container`, `cq*` units. This
  one is a rule, not a preference: strip them from anything copied in.
- **Watch out: component `<style>` blocks are unscoped.** Rules have drifted
  between files, so a class defined in one component can be styling the whole
  app — `.info-box` lives in `TourSheet.vue`, for instance. Before adding a
  rule, check the name isn't already defined somewhere else, and expect a change
  here to show up in places you didn't open.
- Prefer `overflow-y: auto` to `scroll`, which paints a permanent gutter.
- The effective support baseline is the `:has()` selector, set by Vuetify. The
  `@vitejs/plugin-legacy` targets do not constrain CSS — that plugin transpiles
  JavaScript only.

### Layout quirks that keep biting

- **`#wwt-overlay` is `pointer-events: none`** so drags fall through to the sky.
  Any interactive panel must re-enable it on itself *and* be no taller than its
  content, or the invisible remainder swallows drags. This is why overlay panels
  are content-sized rather than full-height.
- **Percentage `max-height` silently resolves to `none`** on the overlay panels —
  the parent chain has no definite height. Cap with flex shrinking instead:
  `flex: 0 1 auto; min-height: 0` on the panel, `min-height: 0` on its column,
  and hand the scroll to the innermost element.
- **`#side-drawer` pushes, it doesn't cover.** It's a flex sibling of
  `#main-content`, so opening it shrinks the WWT view — a fraction of the width
  normally, a fraction of the *height* on small screens, where it comes up from
  the bottom.
- **`#app.app-is-small`** marks the small-screen layout, but **it cannot reach a
  dialog**: `v-dialog` teleports its content out of `#app`. Inside a dialog
  component, use `useDisplay()` and bind your own class.
- `<canvas id="shadow-footprint">` is left over from an older drawing approach
  and is unused — but it is *in flow* inside `#main-content`, which is why that
  element needs `flex: 1 1 auto; min-width: 0`. Removing one means checking the
  other.

### Vuetify quirks

- **Dialog dimensions must be props on the dialog.** `width` / `max-width` /
  `height` become inline styles on `.v-overlay__content`. A height set on the
  card inside is clamped, because the card is a flex child of a container that
  sizes to its content — even as an inline style.
- **`v-breadcrumbs divider=""` still renders the divider elements**, each with
  `padding: 0 8px`. With many items that padding, not your content, is what
  overflows the row.
- **`icon-button` (toolkit) keeps its tooltip's model internal** — no prop forces
  it open. To show a tooltip programmatically, anchor a separate `v-tooltip`
  with `activator="#<the-id>-button"`; the component generates that id from the
  `id` you pass it.

### Vue quirk worth knowing

A slot that renders **zero** nodes falls back to its default content. A `v-for`
over an empty list inside a slot therefore resurrects the fallback markup. Wrap
slot content in an element that always renders.

## WWT

- Footprint files store RA in **degrees**; the engine wants **hours** almost
  everywhere, hence the `/ 15` at the call sites. `Place.get_RA()` is hours too.
- Footprint caches are keyed on `options.id`, so **an id must be unique per
  geometry** — reusing an id with different corners keeps drawing the old shape.
- Everything in a WTML is keyed by **imageset index**, never by name: names are
  not unique within a file and can hold semicolon-separated aliases.
- Put `@keydown.stop` on overlay controls. WWT listens on the document, so
  without it typing in a field or nudging a slider also pans the view.
- Some places carry `ZoomLevel="0"` — treat that as "keep the current zoom".

## Copying from sibling projects

The CosmicDS mini-projects share code by hand-copying, not through a package.
Copy-paste with **minimal changes** rather than re-deriving — the same goes for
moving content between files here. This project is on Vuetify 3, so take
components from a sibling that is also on Vuetify 3 (`almagal`) rather than one
that has moved to Vuetify 4 (`roman`); build config is best taken from `roman`.

## Verifying

- `yarn lint`, `yarn type-check`, `yarn build` (note `build` is
  `run-p type-check build-only`, so it fails if either does).
- Don't drive a browser on your own initiative — ask, or wait to be asked. Static
  checks and reading the emitted bundle are cheaper and usually decisive. When
  the maintainer says they've checked something, take it as verified.
- When you *are* testing responsively, the target mobile view is **320×680**.
  Chrome clamps its own window at roughly 500px wide, so measure the narrow case
  by constraining the element under test rather than trusting a window resize.
- Close any tab you open; stop any dev server you start.

## Toolchain

- `.yarnrc.yml` (one line, `nodeLinker: node-modules`) and `yarn.lock` are both
  load-bearing and tracked. Without the lockfile Yarn refuses to run at all;
  without the `.yarnrc.yml` it falls back to the PnP linker, where several
  directly-imported packages stop resolving.
- Don't set `ignoreDeprecations` in the tsconfigs. The pinned TypeScript accepts
  only `"5.0"`, and a wrong value fails `type-check`, which takes `yarn build`
  down with it while `build-only` still passes.

## Editing

Use the editing tools rather than shell scripts that rewrite files. The diff is
reviewable as it happens, and it's what the approval prompt shows. A script is
fine for a genuinely mechanical change across many sites — say so when you use
one.
