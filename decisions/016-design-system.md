# ADR-016 — Design system: our own tokens and components on native HTML, headless primitives where needed

**Status:** Proposed (2026-11-13). Recommendation: a small in-house system, no component
library.

## Context

Web Awesome Pro goes ([ADR-012](012-self-contained-app.md)). What replaces it must be free
software, self-hosted, accessible (WCAG 2.2 AA at least), **dark first** with a light theme,
usable on a phone, simple, and themable through design tokens (CSS custom properties),
because three things are on the roadmap:

1. a sign-in page in each server's colours;
2. white-label for `forest`;
3. (out of scope here) replacing the whole interface.

Two facts about the app today frame the choice:

- It uses **six** components of Web Awesome: page, card, input, button, callout, icon.
- Web components do not sit well in Qwik. The app loads each component's code when it
  becomes visible and wires events by hand (`app/src/lib/wa.ts`), which gives up what Qwik is
  for (HTML that works before any script runs).

**Qwik's own state**, checked on npm on 2026-11-13: Qwik 1 is the stable line
(`@builder.io/qwik` 1.20.4, 2026-11-09). Qwik 2 (`@qwik.dev/core`) is at release candidate
(2.0.0-rc.5, same day; first candidate in October 2026) and needs Vite 8. The headless
component libraries follow the split: `@qwik-ui/headless` 0.7.7 (June 2026) targets Qwik 1;
its successor `@qds.dev/ui` 0.15 (June 2026) targets Qwik 2 only. Both are pre-1.0.

## Options

| | Licence | Dark / tokens | Accessibility | With Qwik | Remarks |
| --- | --- | --- | --- | --- | --- |
| **A. In-house: tokens + CSS on native elements, a few Qwik components** | ours | Designed for it | Native semantics (`button`, `input`, `dialog`, `details`, popover); ours to test | Native: plain HTML, works without script | Six components to write; no dependency on the most sensitive pages |
| B. Web Awesome Free (`@awesome.me/webawesome` 3.14) | MIT | `--wa-*` tokens, dark class | Good, maintained | Web components: the friction above stays; `wa-page` is Pro; its icon component defaults to a remote kit | Smallest migration from today |
| C. Qwik UI headless / QDS | MIT | Unstyled: our tokens | Built on WAI-ARIA patterns | Native | Pre-1.0; one library per Qwik major; needed only for complex widgets |
| D. Tailwind-based kits (Qwik UI styled, Flowbite Qwik, daisyUI) | MIT | Themes through Tailwind | Varies | Good | Adds a build pipeline and a utility vocabulary for six components |
| E. Classless or light CSS (Pico, Open Props) | MIT | Tokens (Open Props) or a fixed look (Pico) | Native semantics | Native | Open Props is a token source, not a system; Pico is hard to rebrand deeply |
| F. Shoelace 2 | MIT | Tokens | Good | Web components, as B | In maintenance; Web Awesome is its successor |

## Recommendation

**A, with C for the few widgets the platform does not give.**

- **The sign-in page is the most sensitive page of the product.** Every dependency loaded
  there is code that can read the page. With A it needs no component library at all:
  a form, a button, a script for the passkey ceremony.
- **Six components do not justify a library**, and none of the libraries brings the theming
  model wanted (below): we would write the tokens anyway.
- **The platform now covers most of what libraries were for**: `dialog`, the Popover API,
  `details`, `:focus-visible`, `color-scheme`, `light-dark()`, container queries, `oklch()`.
  Native elements bring their keyboard and screen-reader behaviour with them.
- **Headless primitives only where a correct pattern is hard to write** (combobox, tabs,
  toast, menu), imported one at a time, never on the sign-in page: `@qwik-ui/headless` while
  on Qwik 1, `@qds.dev/ui` after the move to Qwik 2.

**Qwik version:** stay on Qwik 1 for v1. Move to Qwik 2 after its stable release, as its own
piece of work. A thin in-house layer is the cheapest thing to carry across that move; a
component library is the most expensive.

### Tokens

Three levels, all CSS custom properties:

1. **Primitive**: the palette (in `oklch`), spacing, radii, type scale, durations.
2. **Semantic**: `--surface`, `--surface-raised`, `--text`, `--text-quiet`, `--accent`,
   `--accent-text`, `--border`, `--danger`, `--success`, `--focus-ring`… Components use these
   only.
3. **Component**, sparingly, when a component needs its own handle.

Dark is the default; light is a second set of semantic values, selected by the system
preference (`prefers-color-scheme`) and overridable by the user. Both are designed, neither
is derived by inversion.

What each roadmap item overrides:

| | Overrides | How |
| --- | --- | --- |
| Sign-in page per server | a short allow-list: accent colour, logo, server name, background image | The administrator picks values; the server validates each as a typed value (a colour, a reference to an embedded or uploaded image) and serves them as a same-origin stylesheet. No free CSS: free CSS can exfiltrate what is typed in a page. Shades are derived from the one accent colour, and a contrast check refuses a colour that would break AA |
| White-label (`forest`) | the whole semantic level, the fonts, the product name | A theme file |
| Full replacement | — | Another build of the app; not a theming feature |

### Accessibility: how AA is held, not hoped for

- Automated checks (axe) run in the end-to-end tests on every page, in both themes; a
  violation fails CI.
- A manual pass per release with keyboard only and with one screen reader.
- WCAG 2.2 additions built into the components: visible focus not hidden by sticky elements
  (2.4.11), targets of 24 px at least (2.5.8), nothing to remember or transcribe to sign in
  (3.3.8, which passkeys satisfy by nature).
- `prefers-reduced-motion`, `prefers-contrast` and `forced-colors` honoured.
- The accessibility audit offered to NLnet-funded projects
  ([ADR-010](010-third-party-audit-funding.md)) is requested for this interface.

## Consequences

- `app`: tokens, base styles, six components, the two themes; Web Awesome and its loaders
  removed. This is the main piece of the new interface for v1.
- The site's visual language (dark, the prism, its type) is the starting point, so site and
  product look like one thing.
- A cost accepted: accessibility of our components is ours to test, with no upstream to
  lean on.

## References

- Qwik 2 status: <https://www.npmjs.com/package/@qwik.dev/core>; Qwik 1:
  <https://www.npmjs.com/package/@builder.io/qwik>
- Qwik UI: <https://qwikui.com>; Qwik Design System: <https://www.npmjs.com/package/@qds.dev/ui>
- Web Awesome (free): <https://www.npmjs.com/package/@awesome.me/webawesome>
- WCAG 2.2: <https://www.w3.org/TR/WCAG22/>; Open Props: <https://open-props.style>
- [ADR-012](012-self-contained-app.md), [ADR-017](017-typography.md),
  [ADR-018](018-app-structure.md)
