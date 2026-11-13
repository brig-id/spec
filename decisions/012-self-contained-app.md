# ADR-012 — The app is self-contained: nothing external at run time or at build time

**Status:** Accepted (2026-11-13)

## Context

The web interface (`app`) depends on third parties in four ways:

- **Web Awesome Pro**, a paid component library installed from a private registry: a token
  is needed to build, so nobody outside the project can build the interface from a clone.
- **Font Awesome** icons, through Web Awesome's icon component.
- **A font host**: the Inter font is loaded from a public CDN, and the Content Security
  Policy has to allow it.
- **Unsplash**: the sign-in page asks Unsplash's API for a random photo, from the browser, on
  each first visit of the day. Unsplash learns the address of everyone who opens a sign-in
  page, the server ships an API key, and the picture shown is not one anybody chose.

`forest` is meant for networks that may have no way out. And a sign-in page that calls
third parties contradicts what brig·id says about itself.

## Decision

### 1. The rule

At run time the interface loads **nothing from any origin but the server that serves it**:
no CDN, font host, icon kit, image API, analytics, captcha or map. This holds for the three
editions. A link the user chooses to follow is not a dependency.

At build time the interface builds from a clean clone with the public npm registry only: no
private registry, no token, no paid licence.

### 2. What replaces what

| Today | Replaced by |
| --- | --- |
| Web Awesome Pro | [ADR-016](016-design-system.md) |
| Font Awesome | [ADR-015](015-icons.md) |
| Inter from a CDN | [ADR-017](017-typography.md), fonts embedded |
| Unsplash API | a few free images embedded in the app (§3) |

### 3. Embedded background images

Rules: **public domain or CC0 only** (redistributable inside any licence, no attribution
duty, no share-alike); credited anyway, in a `CREDITS.md` and on an "About" page; decoration
only (`alt=""`, the page reads the same without them); the sign-in card keeps AA contrast
over any of them; no image when the browser asks to save data.

The set, chosen on 2026-11-13 from Wikimedia Commons:

| # | Image (Commons file) | Author | Licence | Use |
| --- | --- | --- | --- | --- |
| 1 | [Prism tribeam 1.jpg](https://commons.wikimedia.org/wiki/File:Prism_tribeam_1.jpg) | Kelvinsong | CC0 1.0 | Default, dark: a prism splitting one beam, the project's own figure |
| 2 | [Magic light (Unsplash).jpg](https://commons.wikimedia.org/wiki/File:Magic_light_%28Unsplash%29.jpg) | Ricardo Gomez Angel | CC0 1.0 | Dark, landscape: light through a forest |
| 3 | [Forest-ray-of-sunshine-sunbeams (23957956609).jpg](https://commons.wikimedia.org/wiki/File:Forest-ray-of-sunshine-sunbeams_%2823957956609%29.jpg) | Pixel.la Free Stock Photos | CC0 1.0 | Dark, portrait: phones |
| 4 | [Aurora borealis over Eielson Air Force Base, Alaska.jpg](https://commons.wikimedia.org/wiki/File:Aurora_borealis_over_Eielson_Air_Force_Base,_Alaska.jpg) | U.S. Air Force, Senior Airman Joshua Strang | Public domain (U.S. federal government work) | Dark, landscape |
| 5 | [Dingle cliff (Unsplash).jpg](https://commons.wikimedia.org/wiki/File:Dingle_cliff_%28Unsplash%29.jpg) | Ian Schneider | CC0 1.0 | Light theme: the Irish coast |

- Images 2 and 5 were published on Unsplash in March 2017 and October 2015, while its licence
  was CC0 (it changed in June 2017), and imported to Commons with that licence, which cannot
  be withdrawn. Each file page is checked again when the file is added to the repository.
- Weight: AVIF with a JPEG fallback, two widths (960 and 1920 px), at most 150 KB per image
  at the larger width, **1 MB for the whole set**.
- Which image is shown is decided without any network call: the default one, or the one the
  server's administrator picked ([ADR-018](018-app-structure.md)), or none.

### 4. Enforcement

- The interface's Content Security Policy becomes `default-src 'self'` plus the inline
  hashes: the font host and Unsplash leave `style-src`, `font-src`, `img-src` and
  `connect-src` ([`security-model.md`](../security-model.md) §4.6).
- A test in the server asserts that the policy names no external origin.
- A check in `app`'s CI fails when the built files reference an origin that is not on a
  short allow-list of documentation links.

## Consequences

- `app`: the four replacements; `src/lib/unsplash.ts`, the `UNSPLASH_ACCESS_KEY` build
  argument and the private registry line of `.npmrc` go.
- `server-leaf`: the Dockerfile loses the registry token secret and the Unsplash argument.
- The interface can be built by a packager, a contributor or an auditor without asking
  anyone for anything.

## References

- [ADR-015](015-icons.md), [ADR-016](016-design-system.md), [ADR-017](017-typography.md)
- CC0 1.0: <https://creativecommons.org/publicdomain/zero/1.0/>
- Unsplash licence history on Commons: <https://commons.wikimedia.org/wiki/Template:Unsplash>
