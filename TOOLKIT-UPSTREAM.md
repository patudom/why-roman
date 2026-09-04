# vue-toolkit upstream fixes

Five defects in `@cosmicds/vue-toolkit` that *why-roman* currently works around in its
own source. Each one is a local patch we can delete the day the toolkit is fixed.
Ordered by how much consumer code they force.

| | Package | Version examined | Found in | Compiled |
| --- | --- | --- | --- | --- |
| | `@cosmicds/vue-toolkit` | 0.7.0 | why-roman | 2026-09-04 |

## At a glance

| ID | Defect | Severity | Costs us |
| --- | --- | --- | --- |
| TK-1 | Vuetify bundled instead of peered, so its CSS ships twice | High | 4 bugs, id-qualified selectors |
| TK-2 | Global `.v-card-actions` rule with an undefined variable | High | 2 override rules |
| TK-3 | `@wwtelescope/engine-pinia` inlined, duplicating the engine store | High | Components kept as local copies |
| TK-4 | `--focus-color` referenced but never defined | Medium | No focus ring on icon buttons |
| TK-5 | `IconButton` tooltip has no external model | Medium | Parallel `v-tooltip` per button |

A sixth item turned up while checking these: two of our own overrides are aimed at
toolkit rules that **no longer exist** in 0.7.0. That one is ours to delete, not theirs
to fix — it is written up last.

---

## TK-1 — Vuetify is a bundled dependency, so its CSS is emitted twice in every consumer build

**Severity:** High

### What ships

The toolkit lists `vuetify`, `vue` and `pinia` as ordinary dependencies and declares
*no* peer dependencies at all. Its bundle does not externalise Vuetify, so Vuetify's
compiled CSS is inlined into `index.common.js` — a 10.0 MB file.

### Evidence

```
peerDependencies: {}
dependencies:     { "vuetify": "^3.3.3", "vue": "^3.4", "pinia": "~2.1.7", … }

require('vuetify')   in bundle …… false   (not externalised)
'.v-btn{'            in bundle …… 4 occurrences
'.v-card-actions{'   in bundle …… 3 occurrences
'.v-overlay__scrim{' in bundle …… 2 occurrences
```

### What it breaks

The app imports Vuetify's CSS too, so the production bundle carries two copies — and
the toolkit's copy lands *after* the app's component `<style>` blocks. A component rule
at the same specificity therefore wins in `yarn dev` and silently loses once built.
Nothing in the source hints at it: the bug only appears on the deployed app and reads
as "my CSS isn't applying."

Four separate bugs in why-roman traced to this:

- The intro slides' Next button losing its border
- `.v-card-actions` losing `display: flex`, so a `v-spacer` had nothing to push
- The intro card losing its background, border and height
- The dialog scrim flipping colour between dev and deployed

### Our workaround

Every affected rule is qualified with an id so it outranks the late copy, rather than
reaching for `!important`. For example, in `src/components/IntroSlides.vue`:

```less
#intro-slides.intro-slides-dialog .v-overlay__scrim { … }
.intro-slides-container .intro-slide-button { … }
```

### Upstream fix

Move `vue`, `vuetify` and `pinia` to `peerDependencies` and mark them external in the
build config, so the toolkit consumes the host app's copies rather than shipping its
own. Stop emitting Vuetify's CSS from the bundle entirely — the consumer already
imports it.

---

## TK-2 — An unscoped `.v-card-actions` rule breaks card footers in every consuming app

**Severity:** High

### What ships

The bundle contains a global, unscoped rule setting `.v-card-actions`'s display from a
custom property — sitting among the rating dialog's styles, which is evidently what it
was written for.

### Evidence

```
… .rating-notification.error{background-color:#b30000}
   .comments-box{width:75%}
   .v-card-actions{display:var(--footer-visible)}      ← unscoped, global
   .close-button{display:inline} …

definitions of --footer-visible in the toolkit …… 0
definitions of --footer-visible in why-roman  …… 0
```

### What it breaks

`--footer-visible` is never defined *anywhere* — not by the toolkit, not by Vuetify,
and there is nothing in the API telling a consumer to define it. The `var()` therefore
resolves to nothing, the declaration is invalid, and it takes Vuetify's own
`display: flex` down with it. Every `v-card-actions` in every consuming app loses its
flex layout, which is why a `v-spacer` inside one stops pushing anything.

This one is a good citizenship issue as much as a bug: a component library shipping an
unscoped rule for a bare Vuetify class restyles parts of the host app it was never
pointed at.

### Our workaround

```less
/* src/components/IntroSlides.vue */
.intro-slides-container > .v-card-actions { display: flex; }

/* src/RomanFov.vue */
#privacy-popup-dialog .v-card-actions { display: flex; }
```

### Upstream fix

Two changes, either of which alone would have prevented this: scope the rule to the
rating dialog's own root class so it cannot reach a consumer's cards, and give the
`var()` a fallback — `display: var(--footer-visible, flex)` — so that an undefined
property degrades to Vuetify's default instead of voiding the declaration.

---

## TK-3 — The WWT engine store is inlined, so the toolkit and the app hold different `wwt-engine` stores

**Severity:** High

### What ships

The bundle externalises `pinia` correctly, but *not* `@wwtelescope/engine-pinia` — that
is compiled in, along with its `defineStore("wwt-engine", …)` call.

### Evidence

```
require('pinia')                     in bundle …… true   ✓ externalised
require('@wwtelescope/engine-pinia')  in bundle …… false  ✗ inlined

defineStore( calls in bundle …… 3
store ids          …… "wwt-engine"
```

### What it breaks

The app installs its own `@wwtelescope/engine-pinia` and registers its own
`wwt-engine` store. The toolkit registers a second, separate definition under the same
id. A toolkit component that reaches for the engine store therefore reads a different
store than the one the app is driving.

The practical consequence is that any toolkit component needing a layer or the engine
store cannot be imported — so several shared components live in `src/components/` as
local copies instead, and each is a maintenance fork that has to be re-synced by hand.

> **Correction for our own notes:** `CLAUDE.md` records this as "the published bundle
> inlines its own pinia." That is not quite right — pinia itself *is* external. It is
> `engine-pinia`, and therefore the engine store, that gets duplicated. Worth fixing in
> the repo note so the next person looks in the right place.

### Upstream fix

Externalise `@wwtelescope/engine-pinia` and add it (with `@wwtelescope/engine`) to
`peerDependencies`, the same treatment `pinia` already gets. Once the store is shared,
the local component copies can go back to being imports.

---

## TK-4 — `IconButton`'s focus indicator is styled with a property nothing defines

**Severity:** Medium (accessibility)

### What ships

```css
.icon-wrapper:focus-visible {
  color: var(--focus-color);
  border-color: var(--focus-color);
}
```

```
definitions of --focus-color in the toolkit …… 0
definitions of --focus-color in why-roman  …… 0
```

### What it breaks

The selector is right — `:focus-visible`, so it correctly skips mouse clicks — but both
declarations reference an undefined property, so both are invalid and the rule does
nothing. Out of the box, the toolkit's icon buttons show **no keyboard focus indicator
at all**. Nothing errors and nothing looks broken; the button simply never changes
appearance when tabbed to.

### Our workaround

why-roman draws its own focus ring on these buttons, gated on a
`body.keyboard-focus-only` class so it appears for keyboard users and not for mouse
users.

### Upstream fix

Give the property a real default on `.icon-wrapper`, or at minimum a fallback in each
reference — `var(--focus-color, currentColor)`. If consumers are meant to supply it,
that belongs in the README next to the other theming properties; today there is nothing
to tell them.

---

## TK-5 — `IconButton` keeps its tooltip's state internal, with no way in from outside

**Severity:** Medium

### What ships

The component's full public prop surface in 0.7.0:

```
border  color  modelValue  showTooltip  size  tooltipLocation  tooltipText
```

`modelValue` is the button's own active state, not the tooltip's. `showTooltip` enables
or disables the tooltip wholesale. Nothing exposes whether the tooltip is currently
open.

### What it breaks

- On touch devices a tap both opens the tooltip and activates the button, so the
  tooltip stays visible over the panel the button just opened, with no way to dismiss
  it programmatically.
- A tooltip cannot be shown deliberately — for onboarding, or to point at a control
  during a guided step.

### Our workaround

Anchor a separate `v-tooltip` to the button by the id the component generates from the
`id` prop, and drive that one instead:

```html
<v-tooltip activator="#<the-id>-button" …>
```

### Upstream fix

Expose the tooltip's open state as a normal model — either a second
`v-model:tooltip` or a plain `tooltipOpen` prop — so the parent can close it when the
button's action changes the view.

---

## LOCAL — Two of our `!important` overrides target toolkit rules that no longer exist

**Ours to delete — no upstream change needed.**

`src/RomanFov.vue` carries these, with a comment explaining that the toolkit's
icon-button ships "plain `:focus` rules" that need neutralising:

```less
.icon-wrapper:focus        { color: … !important; border-color: … !important; }
.icon-wrapper.active:focus { box-shadow: 0 0 10px 3px var(--active-shadow) !important; }
```

### Why they are dead

```
plain :focus rules on .icon-wrapper in 0.7.0 …… 0 matches
.icon-wrapper.active:focus          in 0.7.0 …… 0 matches
definitions of --active-shadow (toolkit / app) …… 0 / 0
```

The toolkit already uses `:focus-visible` and ships no plain `:focus` rule for this
class, so there is nothing to neutralise. The second rule's only declaration references
`--active-shadow`, which nothing defines, so it is invalid regardless. These were
presumably written against an older toolkit and outlived it.

### Action

Delete both rules and the comment above them. Worth doing alongside TK-4, since both
concern the same focus styling and the stale comment currently describes the toolkit
incorrectly.

---

## Reproducing these checks

Every claim above is a static read of the installed package, so any of it can be re-run
against a new release without a browser.

```bash
# declared dependency surface
node -p "require('./node_modules/@cosmicds/vue-toolkit/package.json').peerDependencies"

# what the bundle externalises vs. inlines
node -e "const s=require('fs').readFileSync(
  'node_modules/@cosmicds/vue-toolkit/dist/index.common.js','utf8');
  console.log(/require\(['\"]vuetify/.test(s), /require\(['\"]pinia['\"]\)/.test(s));"

# CSS custom properties referenced but never defined by the toolkit
node -e "const s=require('fs').readFileSync(
  'node_modules/@cosmicds/vue-toolkit/dist/index.common.js','utf8');
  const used=new Set([...s.matchAll(/var\((--[a-z0-9-]+)/g)].map(m=>m[1]));
  const def=new Set([...s.matchAll(/(--[a-z0-9-]+)\s*:/g)].map(m=>m[1]));
  console.log([...used].filter(v=>!def.has(v)).sort().join('\n'));"
```

That last command lists many properties that are *fine* — FontAwesome's `--fa-*` and
Vuetify's `--v-*` are defined by those libraries at runtime, and `--accent-color`,
`--color` and friends are the toolkit's intentional theming contract, supplied by the
consumer. The real defects are the ones with no owner anywhere: `--footer-visible` and
`--focus-color`.

---

TK-1, TK-2 and TK-4 are all the same underlying habit — shipping global CSS and unowned
custom properties from a component library — and fixing the packaging in TK-1 and TK-3
together would retire the largest share of the workarounds in this app.
