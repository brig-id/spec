# ADR-017 — Typography: Geist and Geist Mono, embedded

**Status:** Accepted (2026-11-13, review round 2 with the owner).

## Context

The app declares system fonts and tries to load Inter from a public font host, which its
Content Security Policy blocks (app#40). Fonts must be embedded
([ADR-012](012-self-contained-app.md)).

Two needs: a face for text, and a **fixed-width face for what a person compares character
by character**: recovery codes and server names. Telling `l`, `I` and `1`, or `O` and `0`,
apart in a server name is a phishing matter, not a matter of taste.

The public site sets its text in **Geist**, its labels and code in **Geist Mono**, and the
emphasised words of its headlines in Instrument Serif italic.

## Options

All under the SIL Open Font License 1.1, which allows embedding and redistribution. Sizes
are the Latin subset in WOFF2 (variable fonts).

| Text + fixed width | Character | Size | Remarks |
| --- | --- | --- | --- |
| **Geist + Geist Mono** | Neutral, compact; one family drawn together | 29 + 23 KB | Already the site's |
| Atkinson Hyperlegible Next + Mono | Designed for low vision: every letter shape is distinct | 33 KB + mono | The most legible; a more marked personality; the site would follow |
| IBM Plex Sans + Mono | A complete superfamily, with a technical character | larger | Different from the site |
| Inter + JetBrains Mono | The reference interface face, with a fixed-width face from another family | 47 + mono | Inter has no fixed-width companion; two unrelated designs |

A separate signature face for titles (Instrument Serif, as on the site) was considered and
not kept for the app.

## Decision

**Geist for everything, Geist Mono for codes and server names. No signature face in the
app.**

- One family, drawn to go together, and the one the site already uses: one voice.
- About 52 KB for both, Latin subset; the Latin Extended subset is added for French and the
  languages that follow.

Rules of use:

- Recovery codes, server names, identifiers and anything compared character by character:
  Geist Mono.
- Body text is never below 16 px; line length and spacing follow WCAG 1.4.12.
- Fallback: `system-ui` and the platform's fonts, with metric overrides so that the page does
  not shift when the embedded font arrives. `font-display: swap`. The sign-in page preloads
  the text face only.
- A white-label theme ([ADR-016](016-design-system.md)) may replace both families.

## Consequences

- `app`: the two families embedded from their npm packages (subsets, WOFF2), the font host
  removed from the Content Security Policy; app#40 is closed by this.
- The fonts' licence text ships with the app (the OFL asks for it), listed with the image
  credits.

## References

- Geist and Geist Mono: <https://github.com/vercel/geist-font> (OFL 1.1)
- Atkinson Hyperlegible Next: <https://www.brailleinstitute.org/freefont/>; Inter:
  <https://rsms.me/inter/>; IBM Plex: <https://github.com/IBM/plex>
- SIL Open Font License 1.1: <https://openfontlicense.org>
- [ADR-012](012-self-contained-app.md), [ADR-016](016-design-system.md)
