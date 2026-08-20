# cortex/mvp — repair the boot chain, flatten Engagements

Date: 2026-08-20 · Branch: `cortex/mvp`

The MVP commit (`f25dee2`) deleted the Opportunities, Intelligence and Loyalty markup and replaced
the two-layer ACE model with a flat list, but left the JavaScript for the deleted pages in place and
bolted a `<script>` block onto the bottom of the file to override `renderAce`. The result is a page
that boots less than two thirds of the way through its own script.

This spec covers the repair and four product changes on top of it.

---

## 1. The boot failure

`index.html` has one `<script>` block carrying the whole application (lines 1828–5847) and a second
block at the end that monkey-patches it. Block one throws on line 3982:

```js
$("#intelList").addEventListener("click",e=>{ … });   // #intelList was deleted
```

`$` is `querySelector`, so this is `TypeError: Cannot read properties of null`. Every top-level
statement after it never executes. Confirmed by booting the file in jsdom, not by reading.

What that costs, in file order:

| Line | Never registers |
| --- | --- |
| 4612–4617 | mobile hamburger, sidebar backdrop |
| 4620–4627 | Opportunities controls (already dead — their nodes are gone) |
| 4640 | tab handler |
| 4673 | **the main delegated click handler** — `data-dom`, `data-dominstr`, `data-play`, `data-learn`, `data-excl`, conversation approve / vote / feedback, `data-closepanel` |
| 4809–4812 | status filter, Expand all |
| 4843 | **`data-acemode`** — the reported "level of autonomy is not clickable" |
| 5825–5847 | the entire boot chain and `landOnHash()` |

Function *declarations* hoist, so `renderConv`, `renderLoyalty` and friends exist — they are simply
never called. The pages look populated only because the last-rendered HTML is baked into the file as
a static snapshot. The one screen that does update is Engagements, because the second `<script>`
block runs after the first has already died and calls its own `renderAce`.

Two further throws are queued behind the first: `$("#oppType")` at 4620, and `ACE_DOMAINS` — renamed
to `ACE_ENGAGEMENTS` by the MVP commit but still referenced at eight sites, including the original
`renderAce`, `studioForDomain`, `studioForPlay` and the toggle handler.

### Fix

Guard at the source. No blanket `try/catch` around the boot chain: that would swallow the *next*
real error, which is precisely the failure mode `docs/WORKFLOW.md` exists to catch. Early-returns
still fail loudly when a node that should exist goes missing.

- Wrap the nine unguarded top-level registrations for deleted nodes in an existence check:
  `#intelList` (3982), the `#oppType … #oppVoice` block (4620–4627), `#statusFilter` (4809).
- Add an early-return on the missing host node to `renderOpportunities` (`#navOppCount`),
  `renderBench` (`#benchWrap`), `renderEng` (`#statusFilter`) and `renderLoyalty` (`#loyTiles`).
- Early-return the original `renderAce` — the bottom block replaces it wholesale, so its body is
  dead weight that only exists to throw on `ACE_DOMAINS`.
- `renderRules`, `renderVariants`, `renderExclusions` and `renderProductOpportunities` already guard
  themselves. Leave them.

`#aceExpandAll` and its listener are deleted outright under §4 rather than guarded.

### The `data-eng` collision

The override block registers a **capture-phase** listener on `data-eng` to toggle
`ACE_ENGAGEMENTS[i]`. The original handler already uses `data-eng` for the custom-engagement list
(`ENG[i].st`). Both fire on one click, against two unrelated arrays. Rename the new attribute to
`data-aceeng`.

### Scope decision

The owner chose the minimal repair: guard the dead references, leave the override block and the
dead code for removed pages in place. Folding the override back into the main script and deleting
the orphaned code is a larger change and is not part of this work.

---

## 2. Overview: rename `Custom` to `Flows`

`DIMS.source` is the single primitive behind the legend, the line chart and the revenue table, so
one rename propagates to all three. Renaming everywhere — the owner's choice, for one vocabulary —
also means:

- the `SRC` map key (it is keyed by display name),
- the `src:[…]` values on conversation records,
- the Conversations source `<option>`,
- any baked static markup, for consistency before the first re-render.

The CSS token `--dv-custom` keeps its name. It is not user-visible and renaming it touches the
palette bindings for no gain.

---

## 3. Engagements: two modes, no tab bar

- Drop `manual` from `ACE_MODES`. Off / Automatic only.
- `grid g-3` → `grid g-2`.
- Rewrite the copy that assumes a manual stage. Automatic currently reads *"You can drop back to
  manual at any time"* and *"Pick this once you've read a few dozen and you trust what it's doing"*;
  both describe a path that no longer exists.
- Delete the `.tabList` block. With one tab there is nothing to switch between.
- Delete `#aceTabBadge` with it. The selected autonomy card already carries a `Current` tag, so the
  status is not lost. The override block already null-checks the badge.

---

## 4. Engagements: one layer, no Skill column

The page describes two layers — a theme, and the plays under it — that the MVP branch collapsed into
a single flat list of engagements. Copy has to follow the model:

| Now | Becomes |
| --- | --- |
| `Themes and the plays underneath them` | `Engagements` |
| the two-layer paragraph | a flat-list paragraph: what ACE is allowed to act on, each switchable on its own, with the ones switched off still priced |
| `<th>Theme and plays</th>` | `<th>Engagement</th>` |
| `Skill` column | removed, from both the header and the row template |
| `Expand all` | removed — nothing expands |

Final columns: **Engagement · Shoppers reached · GMV · Conversion · Opt-out · Activated**.

Removing the Skill column removes the only route from this page into the Instruction Studio. That is
accepted: in the MVP the skill does not surface on Engagements at all.

---

## Verification

Per `docs/WORKFLOW.md`, plus one addition. jsdom stands in for the browser here because Playwright
cannot launch in this sandbox.

1. `node -e …` — the JS parses.
2. Boot in jsdom — **zero** uncaught errors. This is the check that would have caught the original
   regression.
3. The isolation probe — every render function `ok`.
4. The reporting sweep — `errors: 0`, `totalsAgree: true`.
5. `document.documentElement.scrollWidth === window.innerWidth` at 390px.
6. Click through: autonomy cards, engagement toggles, Learn more, every nav item, hash deep links.

Counts for deleted pages are legitimately zero now, so the `_dom` assertions in WORKFLOW.md no
longer all hold on this branch. Noted rather than rewritten — the runbook belongs to `main`.

---

## Out of scope

Two real inconsistencies this work deliberately leaves alone:

- Overview's breakdown dimension is still labelled `ACE theme` and still lists the eight old themes
  as its members. Renaming the label without rebuilding the member data would read worse than
  leaving it, and rebuilding it changes the reporting dataset.
- `docs/PRODUCT.md` and `CLAUDE.md` still describe the two-layer model, manual acceptance, and
  custom engagements.

Both are follow-ups.
