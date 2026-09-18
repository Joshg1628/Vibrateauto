# Design system

The interface is built on a fixed palette, a fixed type scale and a 4 dp spacing grid, rather than values chosen per screen. The constraint that drove every decision was the target device: a small, inexpensive handset used by someone who is not especially technical and may not see small text well.

---

## The brief I designed against

- **320 × 640 dp is the design target**, not an edge case to degrade into. If a layout only works at 400 dp it is wrong for this app.
- Everything must survive the Android **1.3× font scale** with nothing clipped and no text truncated mid-word.
- Every touch target is at least **48 × 48 dp**.
- Nothing may depend on a gesture the user has to discover.
- Favour larger type and fewer elements over dense, clever layouts.

---

## Palette

Dark only. There is no light theme and none is needed.

| Token | Value | Used for |
|---|---|---|
| PageBackground | `#121212` | Page behind everything |
| Surface | `#1E1E1E` | Cards and sections |
| SurfaceSunken | `#161616` | Input wells inside a card |
| Accent | `#8126F7` | Primary purple |
| AccentMuted | `#BB86FC` | Icons, section labels |
| Tertiary | `#2B0B98` | Deep end of the button gradient |
| Magenta | `#D600AA` | Warm end of the badge gradient |
| TextPrimary | `#F5F5F5` | Headings and body |
| TextSecondary | `#B0B0B0` | Meta lines |
| TextMuted | `#808080` | Placeholders |
| Divider | `#333333` | Strokes |
| Success | `#3DBE6E` | Granted, active now |
| Danger | `#FF4444` | Destructive actions |
| WarningBg / Stroke / Text | `#3D2C00` / `#FFB74D` / `#FFE082` | Amber states |
| IconBg | `#3B185F` | Icon tiles |

Two gradients carry the brand: **Accent → Magenta** for badges and card strokes, **Accent → Tertiary** for filled buttons.

**Colour has meaning.** Amber means the app may not be doing its job, and is used for exactly three things: setup incomplete, automation paused, and a form that cannot be saved. Green means working. Red is destructive only.

---

## Type scale

| Role | Size | Weight |
|---|---|---|
| Page title | 21 | Bold |
| Card title, progress heading | 17–18 | Bold |
| Primary button | 17 | Bold |
| Secondary button, body | 15–16 | Semibold / regular |
| Meta lines, descriptions | 14 | Regular |
| Hints, helper text | 13 | Regular |
| Status pills | 12 | Bold, uppercase |

Nothing sits below 12, and 12 is reserved for uppercase pills. An earlier iteration dropped to 10 dp at narrow widths, which is unreadable for the intended user.

---

## Spacing and shape

A 4 dp grid throughout. Page padding is 16 at every width. Cards use 14 padding with 10 between them; sections sit 12 apart. Radii are 14 on cards and banners, 12 on inputs, and fully round on buttons and pills.

Strokes carry state. An active card gets the 1.5 dp gradient stroke; an inactive one drops to a flat 1 dp divider stroke at reduced opacity.

---

## Restraint with the glow

The identity is a dark purple gradient theme with glowing accents, and the easy failure is to glow everything. Each screen has exactly two glowing elements: the radial wash behind the header, and the single primary button. Card shadows were removed entirely. That restraint is what lets the amber banners and the primary action actually stand out.

---

## Decisions worth explaining

**Tap the card to edit, not swipe.** Edit and delete originally lived behind a left swipe. For this audience that is an invisible feature. The whole card is now the tap target, with a pencil tile making the affordance visible, and swipe kept only as a shortcut for people who already know it.

**Minimum heights, never fixed heights.** Every button and input was originally a fixed height, so text clipped at large font scales. They are all minimums now, with padding, so a control grows instead of cropping.

**Icons are drawn, not typed.** Every icon is vector path geometry defined in the style dictionary. The previous build used a Unicode gear character and an emoji pin, both of which can render as empty boxes on a stripped-down ROM.

**Paused is a banner, not a label.** Previously the only sign was the button changing from "Pause" to "Resume". A phone that silently stopped protecting you is this app's worst failure, so the paused state is now a full-width amber banner that says so in a sentence.

**One adaptive breakpoint, not two.** The base styles are the 320 dp design. A single state below 300 dp exists for foldable cover screens. Previously 320 dp fell into a layout written for 240 dp cover displays, which hid the address line and shrank text.

**Layout is mirror-safe.** No hard-coded left or right margins. Positions use Start and End so the interface mirrors correctly if a Hebrew localisation is added.

---

## Accessibility

Every interactive element carries a semantic description for screen readers. State is never conveyed by colour alone: the switch on the nearby shuls list is labelled On or Off in words, and the active-rule badge reads "Active" rather than being a coloured dot.
