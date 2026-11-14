# Interface — brig·id

> **Audience:** auditors and contributors.
> The web interface (`app`): what it may load, what it is built with, how it is organised.
> Statuses are explained in the [README](README.md).

---

## 1. Nothing external

**Status:** built.

At run time the interface loads **nothing from any origin but the server that serves it**:
no CDN, font host, icon kit, image API, analytics, captcha or map. This holds for the three
editions; the largest is meant for networks that may have no way out, and a sign-in page
that calls third parties contradicts what brig·id says about itself. A link the person
chooses to follow is not a dependency.

At build time the interface builds from a clean clone with the public npm registry only: no
private registry, no token, no paid licence. A packager, a contributor or an auditor can
build it without asking anyone for anything.

Enforcement:

- The interface's Content Security Policy is `default-src 'self'` plus the hashes of its
  inline blocks ([`security-model.md`](security-model.md) §4.6). A test in the server asserts
  that the policy names no external origin.
- A check in the interface's CI fails when the built files reference an origin that is not
  on a short allow-list of documentation links.

---

## 2. What was replaced

**Status:** built.

| Until step 1 | Problem | Replaced by |
| --- | --- | --- |
| Web Awesome Pro, from a private registry | A token is needed to build | Our own components (§3) |
| Font Awesome icons | Fetched by the component library | Phosphor, inlined (§4) |
| Inter from a public font host | A third party on every page; the policy blocks it today | Geist, embedded (§5) |
| A random photo from Unsplash's API on the sign-in page | Unsplash learns the address of everyone who opens a sign-in page; the server ships an API key; nobody chose the picture | A few free images embedded in the app |

### Embedded background images

**Public domain or CC0 only**: redistributable inside any licence, no attribution duty, no
share-alike. They are credited anyway, in a `CREDITS.md` and on an "About" page. They are
decoration: `alt=""`, the page reads the same without them, the sign-in card keeps AA
contrast over any of them, and none is loaded when the browser asks to save data.

The set, chosen on 2026-11-13 from Wikimedia Commons:

| # | Image (Commons file) | Author | Licence | Use |
| --- | --- | --- | --- | --- |
| 1 | [Prism tribeam 1.jpg](https://commons.wikimedia.org/wiki/File:Prism_tribeam_1.jpg) | Kelvinsong | CC0 1.0 | Default, dark: a prism splitting one beam, the project's own figure |
| 2 | [Magic light (Unsplash).jpg](https://commons.wikimedia.org/wiki/File:Magic_light_%28Unsplash%29.jpg) | Ricardo Gomez Angel | CC0 1.0 | Dark, landscape: light through a forest |
| 3 | [Forest-ray-of-sunshine-sunbeams (23957956609).jpg](https://commons.wikimedia.org/wiki/File:Forest-ray-of-sunshine-sunbeams_%2823957956609%29.jpg) | Pixel.la Free Stock Photos | CC0 1.0 | Dark, portrait: phones |
| 4 | [Aurora borealis over Eielson Air Force Base, Alaska.jpg](https://commons.wikimedia.org/wiki/File:Aurora_borealis_over_Eielson_Air_Force_Base,_Alaska.jpg) | U.S. Air Force, Senior Airman Joshua Strang | Public domain (U.S. federal government work) | Dark, landscape |
| 5 | [Dingle cliff (Unsplash).jpg](https://commons.wikimedia.org/wiki/File:Dingle_cliff_%28Unsplash%29.jpg) | Ian Schneider | CC0 1.0 | Light theme (step 2 of v1): the Irish coast |

- Images 2 and 5 were published on Unsplash in March 2017 and October 2015, while its
  licence was CC0 (it changed in June 2017), and imported to Commons with that licence, which
  cannot be withdrawn. Each file page is checked again when the file is added.
- Weight: AVIF with a JPEG fallback, two widths (960 and 1920 px), at most 150 KB per image
  at the larger width, **1 MB for the whole set**.
- Which image is shown is decided without any network call. In v1 it is the default one,
  and **only that one is shipped** (8 KB in AVIF at 1920 px); the others join the build when
  something shows them.

---

## 3. Design system: our own, on native HTML

**Status:** built for the dark theme and the two languages; decided — step 2 for the light
theme.

**No component library.** The interface uses six components (page layout, card, input,
button, callout, icon). They are written on native elements, with our own styles.

- **The sign-in page is the most sensitive page of the product.** Every dependency loaded
  there is code that can read the page. It needs a form, a button and a script for the
  passkey ceremony.
- **The platform covers what libraries were for**: `dialog`, the Popover API, `details`,
  `:focus-visible`, `color-scheme`, container queries, `oklch()`. Native elements bring
  their keyboard and screen-reader behaviour with them.
- A widget the platform does not give (combobox, tabs, toast) is written when a page needs
  it, following its WAI-ARIA pattern, with its tests.
- Free design systems were compared (Web Awesome's free edition, daisyUI, Pico CSS, Flowbite,
  the headless Qwik libraries) and set aside: each adds a dependency or a build pipeline for
  six components, and none brings the theming wanted.

**Qwik 2.** The interface is rewritten on Qwik 2, a release candidate at the time of the
decision, whose build tool once broke this app's static generation. The first task of the
rewrite was therefore to prove one page end to end (static build, script-hash list, passkey
ceremony in the end-to-end tests) before anything else was written. It held.

### Tokens

All CSS custom properties, on three levels: **primitive** (the palette in `oklch`, spacing,
radii, type scale, durations), **semantic** (`--surface`, `--text`, `--accent`, `--border`,
`--danger`, `--focus-ring`…, the only ones components use), and **component**, sparingly.

**Dark first.** Step 1 ships the dark theme. The light theme is a second, designed set of
semantic values (not an inversion), selected by the system preference and overridable by the
person. Tokens are named for both from the start, so that adding it touches no component.

### Languages

**English and French**, chosen from the browser's languages, with a switch that is
remembered. Every text lives outside the components; a page is not done until both
languages are written, and a test fails on a text left in a component or written in one
language only.

The pages are static files, so the server cannot choose a language. Both languages are in
each page and the stylesheet shows the one named by the document's `lang`, which a script in
the head sets before the first paint. That script is fixed at build time and allowed by its
hash, like the framework's own.

### Accessibility

WCAG 2.2 AA, held by tests and not by intention:

- automated checks (axe) in the end-to-end tests on every page, in every theme and both
  languages; a violation fails CI (*step 2*);
- a manual pass per release with the keyboard only and with one screen reader;
- visible focus never hidden by a sticky element, targets of 24 px at least, nothing to
  remember or transcribe to sign in (which passkeys satisfy by nature);
- `prefers-reduced-motion`, `prefers-contrast` and `forced-colors` honoured.

---

## 4. Icons: Phosphor

**Status:** built.

Phosphor (MIT): 1,527 icons in six weights. It is the set of the public site, so the site
and the product share one icon language. *Regular* serves the interface, *duotone* the
logos of identities, which need to look like more than a toolbar icon. Tabler (larger, more
active) and Lucide (outline only) were the alternatives.

Icons are SVG inlined at build time from the set's Iconify data: only the icons used, no
icon font, no request at run time.

### Stored logos

An identity's logo is a reference stored as text ([`model.md`](model.md) §2), of the form
`icon:ph:<name>`: the name of an icon of the set. The interface offers a library of 45
of them, sorted in five groups; the server checks the form of a reference, not that the
library holds it, so that the library can grow without a change to the server.

Nothing was converted when the set changed: no server held identities to keep
([`operations.md`](operations.md) §6). A reference the interface does not know is drawn as
the plain person icon.

Brand logos (five of them) remain trademarks of their owners whatever the set's licence;
the library keeps them few.

---

## 5. Typography: Geist and Geist Mono

**Status:** built.

**Geist for everything, Geist Mono for what a person compares character by character**:
recovery codes, server names, identifiers. Telling `l`, `I` and `1`, or `O` and `0`, apart
in a server name is a phishing matter, not a matter of taste. One family, drawn to go
together, already the public site's. No separate face for titles.

- SIL Open Font License 1.1, embedded as WOFF2: about 52 KB for both in the Latin subset;
  the Latin Extended subset is added for French. The licence text ships with the app.
- Body text is never below 16 px; line length and spacing follow WCAG 1.4.12.
- Fallback: the platform's fonts, with metric overrides so that the page does not shift when
  the embedded font arrives. The sign-in page preloads the text face only.

---

## 6. Three parts

**Status:** sign-in and account are built; administration is decided —
step 2.

The interface has three parts, called what they are, in the code as in the documents:

| | Sign-in | Account | Administration |
| --- | --- | --- | --- |
| For | Someone arriving | Someone at home | The server's administrator |
| Pages | Sign in; choose another server; create an account; recover; the linking page ([`model.md`](model.md) §5); the first administrator's setup | Onboarding; identities; passkeys; recovery code; leaving | Accounts; administrators; applications; sign-up policy |
| Session | None, or being created | Account session | An administrator's session, with a recent passkey ceremony for any change |
| Routes | `/`, `/signup/`, `/recover/`, `/continue/`, `/setup/` | account pages | `/admin/…` |
| Script | The smallest: the page works as HTML, a script runs the passkey ceremony | Normal | Normal |

- **One application, one build, three route groups** with their own layouts. The framework
  already loads code per route: three builds would not make sign-in lighter.
- **Separation is enforced where it counts.** The server authorises every request; the
  interface hiding a page is not a control. Sign-in imports nothing from the two other
  groups, checked by a lint rule, so that its pages can be reviewed in one sitting.
- The interface is a static build. What differs per server (name, sign-up policy) is read
  from a same-origin endpoint.

---

## 7. After v1

- **A server's own colours** on the sign-in pages: an accent colour, a logo, the server's
  name, one of the embedded backgrounds. The administrator picks values, the server
  validates each as a typed value and serves them as a same-origin stylesheet. No free CSS:
  free CSS can exfiltrate what is typed in a page. Shades derive from the one accent colour,
  and a colour that would break AA contrast is refused.
- **White-label**: the whole semantic level, the fonts, the product name.
- **Replacing the whole interface**: another program talking to the server, not a theme.
