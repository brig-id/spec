# ADR-017 — Typography: Instrument Serif as the signature, Geist for text, embedded

**Status:** Proposed (2026-11-13). Recommendation: the pair the public site already uses.

## Context

The app declares system fonts and tries to load Inter from a public font host, which its
Content Security Policy blocks (app#40). Fonts must be embedded
([ADR-012](012-self-contained-app.md)).

The question asked: is there a **signature typeface** for brig·id (free, embeddable,
legible, accessible; for titles at least), or else Inter; with system fonts as the fallback.

The public site has in fact answered it. Its headline, "One private self. *Many
identities.*", is set in **Geist** with the emphasised words in **Instrument Serif italic**;
labels and code are in **Geist Mono**. All three are embedded from npm packages.

## Options

All under the SIL Open Font License 1.1, which allows embedding and redistribution. File
sizes are the Latin subset in WOFF2.

**Signature (titles, the voice of the product)**

| | Character | Sizes | Remarks |
| --- | --- | --- | --- |
| **Instrument Serif** | A condensed display serif, elegant; its italic is the site's accent | 21 KB regular, 22 KB italic | One weight. For display sizes only (24 px and above) |
| Fraunces | A soft, slightly eccentric variable serif | 36 KB (weight axis only) to 118 KB (all axes) | Distinctive, heavier; a new voice to establish |
| Bricolage Grotesque | An expressive variable grotesque | 40 KB (weight axis only) | Signature without a serif; trendy, may date |
| Newsreader / Source Serif 4 | Text serifs | 57 KB / 50 KB (weight axis only) | Legible, not distinctive |

**Text (everything else)**

| | Character | Size | Remarks |
| --- | --- | --- | --- |
| **Geist** | A neutral, compact grotesque | 29 KB variable | Already the site's text face |
| Inter | The reference interface face | 47 KB variable | Has alternates that tell `I`, `l` and `1` apart; ubiquitous, so it says nothing |
| Atkinson Hyperlegible Next | Designed with the Braille Institute for low vision: every letter shape is distinct | 33 KB variable | The most legible; a more marked personality |

## Recommendation

**Instrument Serif (signature) + Geist (text) + Geist Mono (codes and server names).**

- **A brand is one voice.** The site set it, and its owner approved it. A different pair in
  the product would make two products.
- **Weight**: about 95 KB for everything (Instrument Serif regular and italic, Geist
  variable, Geist Mono variable), Latin subset; the Latin Extended subset adds about 50 KB
  for the languages that need it.
- **Inter brings nothing Geist does not**, except its alternates, which matter in one place
  only (below).

Rules of use:

- Instrument Serif at 24 px and above only: page titles, the sign-in headline, moments where
  the product speaks. Functional headings, labels and buttons are Geist, semibold. A
  high-contrast serif at small sizes on a dark background is hard to read.
- **Recovery codes, server names and anything a person compares character by character are
  set in Geist Mono.** Telling `l`, `I` and `1`, or `O` and `0`, apart in a server name is a
  phishing matter, not a matter of taste.
- Text is never below 16 px for body copy; line length and spacing follow WCAG 1.4.12.
- Fallback: `system-ui` and the platform's fonts, with metric overrides so the page does not
  shift when the embedded font arrives. `font-display: swap`. The sign-in page preloads the
  text face only.
- A white-label theme ([ADR-016](016-design-system.md)) may replace the three families.

If accessibility is to weigh more than brand continuity, the alternative is **Atkinson
Hyperlegible Next** for text, keeping Instrument Serif. The site should then follow.

## Consequences

- `app`: the three families embedded from their npm packages (subsets, WOFF2), the font host
  removed from the Content Security Policy; app#40 is closed by this.
- The fonts' licence texts ship with the app (the OFL asks for it), listed with the image
  credits.

## References

- Instrument Serif: <https://github.com/Instrument/instrument-serif> (OFL 1.1)
- Geist and Geist Mono: <https://github.com/vercel/geist-font> (OFL 1.1)
- Inter: <https://rsms.me/inter/>; Atkinson Hyperlegible Next:
  <https://www.brailleinstitute.org/freefont/>
- SIL Open Font License 1.1: <https://openfontlicense.org>
- [ADR-012](012-self-contained-app.md), [ADR-016](016-design-system.md)
