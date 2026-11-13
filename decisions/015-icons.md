# ADR-015 — Icons: Phosphor, embedded, for the interface and for identity logos

**Status:** Proposed (2026-11-13). Recommendation: Phosphor.

## Context

The interface draws its icons with Web Awesome's icon component, which fetches Font Awesome
glyphs. Both go ([ADR-012](012-self-contained-app.md)).

Icons have a second role: an identity's **logo** is an icon the user picks in a library of 45
(`app/src/lib/icons.ts`). The server stores the reference as text, `icon:<style>:<name>`,
with Font Awesome's vocabulary (`icon:solid:briefcase`, `icon:brands:github`), and checks
only its shape ([ADR-002](002-accounts-and-identities.md)).

The public site already uses **Phosphor** icons (duotone weight), read from the Iconify JSON
at build time and inlined as SVG: no font, no request.

## Options

Figures from the npm registry and the Iconify collections on 2026-11-13.

| | Phosphor | Tabler | Lucide | Material Symbols | Font Awesome Free |
| --- | --- | --- | --- | --- | --- |
| Licence | MIT | MIT | ISC | Apache-2.0 | Icons CC BY 4.0 (attribution) |
| Icons | 1,527 × 6 weights (thin, light, regular, bold, fill, duotone) | about 6,500 (outline and filled) | about 2,000, outline only | about 3,900 × styles | about 1,900 with brands |
| Look | Soft, friendly; duotone gives depth | Neutral, precise | Neutral, the most common look on the web | Google's | Familiar, heavy |
| For identity logos | Fill or duotone reads well at 24–64 px; 78 brand logos | Many subjects; about 300 brands; one stroke weight | Outline only: weak as a logo; brand icons removed | Variable fill | Today's set, unchanged |
| Activity | Last release of the core set in March 2024 | Weekly releases | Weekly releases | Continuous | Continuous |
| Weight in the app | Only the icons used, inlined | Same | Same | Same | Same |
| Qwik | SVG inlined at build time from Iconify JSON (no runtime, no wrapper needed) | Same | Same | Same | Same |

Others looked at and set aside: Iconoir and Remix (no advantage over the three above),
Simple Icons (brands only, and brands are removed on their owners' request), keeping Font
Awesome Free (no migration at all, but attribution to carry, and the decision is to leave).

Integration is the same for every candidate: a build step reads the Iconify JSON of the set
and emits the SVG of the icons actually used. No icon font, no component library, no request
at run time. Wrapper packages for Qwik exist and are unmaintained; none is needed.

## Recommendation

**Phosphor.**

- One icon language for the site and the product; the site's code for it can be reused.
- Six weights from one set: *regular* for the interface, *fill* for small active states,
  *duotone* for identity logos, which need to look like more than a toolbar icon.
- MIT, no attribution to carry in the interface.

What it costs: the set is no longer growing fast (last core release March 2024), and it is
four times smaller than Tabler. For 45 logos and a few dozen interface icons that is enough,
and icons are static files that do not rot. If a subject is missing, a Tabler icon can be
added to the logo library individually (same licence family).

Second choice: **Tabler**, if breadth or upstream activity weighs more than the look shared
with the site. The site would then switch too.

## Stored logos: no data migration

New references use the same shape with the set's Iconify prefix: `icon:ph:<name>`. The
weight is a matter of display, not of data.

Existing references are **not rewritten**. The app keeps a permanent table from the 45
legacy references to their Phosphor equivalent and resolves them at display time; an
identity's reference is rewritten only when its owner edits the logo. The server accepts
both forms (it only checks the shape, which does not change), so `core` needs no migration
and no release is coupled to another.

Proposed equivalents (all present in Phosphor 2.1, regular and duotone):

| Legacy (`icon:solid:…`) | Phosphor | | Legacy | Phosphor |
| --- | --- | --- | --- | --- |
| `user` | `user` | | `palette` | `palette` |
| `user-tie` | `user-circle-gear` ≈ | | `camera` | `camera` |
| `user-astronaut` | `rocket-launch` ≈ | | `music` | `music-notes` |
| `user-ninja` | `sword` ≈ | | `gamepad` | `game-controller` |
| `user-secret` | `detective` | | `futbol` | `soccer-ball` |
| `masks-theater` | `mask-happy` | | `bicycle` | `bicycle` |
| `ghost` | `ghost` | | `mountain` | `mountains` |
| `people-roof` | `house-line` ≈ | | `plane` | `airplane-tilt` |
| `children` | `baby` ≈ | | `mug-hot` | `coffee` |
| `people-group` | `users-three` | | `chess-knight` | `horse` ≈ |
| `briefcase` | `briefcase` | | `house` | `house` |
| `building` | `buildings` | | `globe` | `globe-hemisphere-west` |
| `landmark` | `bank` | | `cart-shopping` | `shopping-cart` |
| `graduation-cap` | `graduation-cap` | | `heart` | `heart` |
| `book` | `book-open` | | `star` | `star` |
| `flask` | `flask` | | `bolt` | `lightning` |
| `code` | `code` | | `leaf` | `leaf` |
| `terminal` | `terminal-window` | | `feather` | `feather` |
| `laptop-code` | `laptop` | | `gem` | `diamond` |
| `server` | `hard-drives` | | `shield-halved` | `shield-check` |

Brands (`icon:brands:…`): `github`, `gitlab`, `discord`, `mastodon`, `linkedin` →
`github-logo`, `gitlab-logo`, `discord-logo`, `mastodon-logo`, `linkedin-logo`.

≈ marks an approximation: Phosphor has no direct equivalent, so the picture changes for
identities that use it.

## Consequences

- `app`: an icon component over inlined SVG, the legacy table, the picker rebuilt on
  Phosphor names (the library can grow: nature, sea and travel subjects are well covered).
- `core`: nothing. `protocol`-level documents keep describing `icon:<style>:<name>`.
- Brand logos remain trademarks of their owners whatever the set's licence; the picker keeps
  them few.

## References

- Phosphor: <https://phosphoricons.com>, <https://github.com/phosphor-icons/core> (MIT)
- Tabler Icons: <https://tabler.io/icons> (MIT); Lucide: <https://lucide.dev> (ISC)
- Iconify collections: <https://icon-sets.iconify.design>
- [ADR-002](002-accounts-and-identities.md), [ADR-012](012-self-contained-app.md)
