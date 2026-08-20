# Flows — a page for the standard lifecycle sequences

Date: 2026-08-20 · Branch: `cortex/mvp`

A new left-nav page holding the five standard lifecycle flows, modelled on the Custom engagement tab
that `main` still carries at `#engagements/scenarios`.

---

## What a flow is

Gorgias authors it and it ships on. The merchant does not build one.

> A flow fires on a trigger you can name, every time it happens. ACE works the moments no fixed
> trigger describes.

That sentence is the page, and it is what resolves the overlap between **Browse Abandoned** and
ACE's *"Someone cannot make up their mind"* / *"Someone keeps coming back to one thing"*. Browse
Abandoned catches the session that **ended**; ACE reads the one still running. Same signal, different
job. Cart Abandoned is distinct outright — there is a cart. Welcome, Post-purchase and Win-back do
not overlap ACE at all.

### Who owns which half

This is the load-bearing decision on the page.

| | Owner | Surfaced as |
| --- | --- | --- |
| The trigger — event + condition | **Gorgias.** It is what makes a Welcome a Welcome. | Stated, not offered. A `tag-purple` "We set this" and a read-only card. |
| The delay | **The merchant.** | A real `<select>`, saved on the record. |
| The skill | **The merchant.** | The existing studio editor, unchanged. |

## The five flows

Volumes are sized to the `Flows` member of `DIMS.source` — 380 reached a week, so ~1,600 a month
across all five. They are stated, not derived; the reporting engine has never read this table, and
did not read the `ENG` table it replaces either.

| Flow | Trigger | Only if | Delay | Sent 30d | CVR |
| --- | --- | --- | --- | --- | --- |
| Welcome | Opted in to marketing | first opt-in, never ordered | immediately | 640 | 6.2% |
| Cart Abandoned | Checkout abandoned | cart ≥ $40, no order since | 45 minutes | 410 | 11.4% |
| Browse Abandoned | Session ends without a cart | two or more product views | 3 hours | 380 | 3.8% |
| Post-purchase | Delivery confirmed | no return or ticket on the order | 2 days | 150 | 4.1% |
| Customer Win-back | On the shopper's own rhythm | no order in 120 days, 2+ past orders | 5 days | 90 | 2.9% |

Win-back is the only rhythm trigger, which is what gives the "Rhythm and signal triggers" filter
something to separate. Its delay is a concrete 5 days rather than "When the engine judges it best" —
that option exists in the list, but shipping it as a default would undercut the page's own claim
that a flow fires on a trigger you can name.

All five ship **on**, so the Paused chip reads 0 until someone toggles one.

### `WHEN_EVENTS` gains one entry

There was no signup trigger. `"Opted in to marketing"` is **appended** at index 12: `RHYTHM_EVENTS`
pins indices 10 and 11, so nothing may be inserted ahead of them. Appending also keeps it out of the
rhythm filter, which is correct — an opt-in is an event.

## Page

Same shape as the tab it descends from: header, explainer banner, the All/Running/Paused chips with
the trigger-type and sort selects, then the table.

Columns: **Flow · When — the trigger · Sent 30d · CVR · Status**. No skill column, matching the
Engagements table.

Nav item sits between Engagements and Campaigns. Its count reuses `#scnBadge` — the id the old tab
badge carried, so `renderEng` keeps writing to it — but now shows the **running** count rather than
the total, which is the rule `#navEngCount` already follows.

## Code

Everything needed was still in the file; the MVP commit deleted only the markup. `ENG`,
`WHEN_EVENTS`, `WHEN_DELAYS`, `whenLine`, `renderEng`, `studioForEng` and the studio all survived.

- `whenBlock()` gains a third mode. It had `"read"` (ACE — nothing to set) and `"edit"` (everything
  yours). `"delay"` sits between them: the event and condition render as a read-only card plus
  **hidden inputs**, so `whenReadback()` and the save path read them exactly as they read the
  editable fields. No branching in either.
- `studioSave` gains a `delay` branch that writes back `delay` and the skill text, and leaves
  `ev`/`cond` untouched.
- `studioForEng` switches to `mode:"delay"`, `kind:"Flows"`, and a Flow-worded subtitle.
- The stale `selectTab("scenarios")` on save is gone — that tab does not exist.
- `data-eng` is free again for `ENG` now that ACE's toggles use `data-aceeng`.

## Verification

1. Both script blocks parse; jsdom boot with zero uncaught errors.
2. Isolation probe — all 15 render functions `ok`.
3. Reporting sweep — 150 states, `errors: 0`, `totalsAgree: true`.
4. All 8 nav pages open with correct hash; `#flows` deep link lands.
5. Studio: trigger renders fixed with no event select offered, delay select carries 9 options,
   read-back updates live, save persists to the table row.
6. Toggling a flow moves the chip counts and the nav badge; toggling an ACE engagement moves only
   the ACE badge — the old shared-attribute collision stays dead.

## Still out of scope

Unchanged from the previous spec, and now one item worse: `docs/PRODUCT.md` and `CLAUDE.md` describe
custom engagements as merchant-built, which this page replaces. Overview's breakdown dimension is
still labelled `ACE theme`.
