# Codebase map

One file: `index.html`, ~2,550 lines, no build, no dependencies, no network calls. Three zones in
order: **tokens + CSS**, **markup for every screen**, **data + render functions**.

Line numbers below drift as soon as you edit. Grep for the marker instead — every section has one.

## Layout of the file

| Zone | Find it with | What's there |
| --- | --- | --- |
| Tokens | `AXIOM TOKENS` | Axiom core colours, semantic tokens, spacing scale, the validated dataviz palette |
| CSS | after `*{box-sizing` | Utilities, then components, each with a comment header |
| Markup | `<!-- ======== ` | One `<section class="page" id="page-*">` per screen |
| Data | `const [A-Z_]* =` | Every list the UI renders from |
| Render | `function render*` | One per screen or block |
| Interaction | `/* ACE — the switch` onwards | Delegated click handler, filters, tabs, nav |
| Boot | `/* boot */` | The render chain — **read the failure mode below** |

## Screens

Each is a `<section class="page">`; `goto(page)` unhides one, hides the rest, and syncs `location.hash`
to the page name — every screen has its own URL, so any of them can be copied out of the address bar
and shared. `KNOWN_PAGES` (derived from the `.page` ids) gates which hashes `goto` will act on.

Pages with tabs (Engagements' AI/Custom, Loyalty's Overview/Configuration) get a deeper
`#page/tab` hash, so a link can point at "Custom engagement" specifically, not just Engagements.
`syncHash()` reads the currently visible page and, if it has one, its currently active tab
(`currentTabFor(page)`, which just reads whichever `.tabItem` inside that page carries `.on`) and
writes the combined hash — called from `goto()` and from the `[data-tab]` click handler, so both a
plain page nav and a same-page tab switch keep the URL honest. `landOnHash()` is the read side: it
parses `page/tab` off `location.hash`, calls `goto(page,{fromHash:true})`, and only calls
`selectTab(tab)` if that tab actually exists on that page — a stray or mistyped tab segment is
ignored rather than landing on a blank tab area. A `hashchange` listener and a boot-time call to
`landOnHash()` make the URL work both ways — navigate and it updates, land on a link and it opens
there.

| id | Nav item | Rendered by |
| --- | --- | --- |
| `page-overview` | Reporting | `renderReport` → `renderTiles`, `renderRevTable`, `stackedBars`/`lines`/`hBars`; plus `renderBench` |
| `page-intelligence` | Intelligence Hub | static business briefing with `renderIntelligence()` panel selection; written once in the HTML |
| `page-intelligence` | Intelligence Hub | `renderIntelligence` — master/detail over `I_SECTIONS`; each section lazily draws (`drawIntelGraph`, `drawIntelDecay`, `drawMarkets`, `drawPlays`, `drawSegments`, `drawRevisions`, `startStream`, `drawBlind`) |
| `page-opportunities` | Opportunities | `renderOpportunities` → `renderOpportunityDetail`; actions route into campaigns, guardrails, ACE themes, lifecycle and loyalty |
| `page-engagements` | Engagements | `renderAce` (AI tab — themes **and** their plays), `renderEng` (Custom tab) |
| `page-campaigns` | Campaigns | `renderCampaigns` |
| `page-campaign-new` | (via Create) | `resetWizard` → `askTurn`/`campAnswer` drive a linear script (`WIZARD`), rendering `renderRules`/`renderVariants` inline as cards |
| `page-guardrails` | Guardrails | `renderGuardrails` (master/detail over `GR_CATS`) → `renderSuppression`, `renderExclusions` |
| `page-channels` | Channels | `renderChannels` (master/detail over `CHANNELS`, same `.lcWrap`/`.lcItem` shape as Lifecycle and Guardrails) |
| `page-lifecycle` | Lifecycle | `renderLifecycle` |
| `page-conversations` | Conversations | `renderConv`, `renderThread` |
| `page-loyalty` | Loyalty | `renderLoyalty` — overview tab plus a master/detail over tiers, earn rules and rewards |
| `page-soon` | — | placeholder for anything not built |

`NAV_OF` maps a page to the nav item that should look active — that's how `campaign-new` keeps
Campaigns highlighted.

### Engagements welcome

`#engagementsWelcome` is a centered dialog with its own backdrop rather than the shared side panel.
`goto("engagements")` calls `maybeOpenEngagementsWelcome()`, which opens it once per browser using
`localStorage`; the **How it works** `[data-engagements-help]` button in the page header always opens it again. Close buttons and
the backdrop use `[data-close-engagements-help]` plus the delegated document click handler. Escape
closes it and Tab stays inside its two buttons while it is open. `.engWelcomeBody` splits the modal
between an orbital connected-data illustration and an editorial explanation of Gorgias Engage's
Find, Decide and Message actions. Narrow screens place the explanation first and the illustration below it.
CSS animation delays reveal the sources, core, actions and safety layer in order; the reduced-motion
media query shows the complete composition immediately. Six decorative `.engWelcomeEvent` tags
cycle one at a time toward the core across shopper behavior, store events, shopper context and timing.
Three selected signals continue through the decision: the center glyph spins,
then an `.engWelcomeOutput` message or wait decision exits along the same path. Reduced motion hides
this secondary layer. Modal colors use the Gorgias brand palette:
Coral `#FF9780`, Rose `#FFD2C9`, Ginger `#FFEAE6`, Blush `#FEF4F3`, Black `#1A1E23`, Smoke
`#E8E3E1`, Fog `#F3F1F0` and White `#FAFAFA`. Body text uses Inter Tight; display text uses the
locally installed BB Modern Semi Condensed, with a Georgia fallback.

## Intelligence Hub and the shopper read

**`page-intelligence`** is deliberately static markup — an AI business briefing, not a driven
report. It uses the same master/detail shape as Lifecycle: `#intelList` holds five `.lcItem` buttons
and `renderIntelligence()` shows the matching `[data-intel-panel]` for `intelSel`. The sections are
Purchase path (Hardshell → Cirrus → accessories), Product roles (why Cirrus drives repeat and Trail
Gloves do not when bought first), Live signals, Engine response, and Markets. The brief begins with
the engine's specific current read: a Sample Hardshell owner without insulation is most likely to
make their second purchase on Cirrus within 54 days. The engine section explains the actual decision
chain — detect the ownership gap, find the moment, select the lowest-cost market-specific lever,
then measure against a holdout. **Live signals** is intentionally animated: CSS pulse/rising bars
show model inputs continuing to move (consideration, cold window, repeat window at risk), while the
copy is explicit that no signal alone sends a message. All numbers are fictional but internally
consistent. The page has no data render function and doesn't appear in the isolation probe — edit
the markup directly.

**`shopperProfile(s)`** builds the descriptive read inside `openShopper`'s panel — the "shopper MD
file". It's a mix of derived and written: cadence, engagement, discount sensitivity, the working
lever and the loyalty posture are all derived from the shopper's `orders`/`ltv`/`tier`, then folded
into one readable paragraph plus a row of tags and a written archetype ("The regular you build the
business on."). Same shopper, same read every time — it's deterministic, so a merchant sees a
consistent person, not a random sketch.

## Data objects

Change these, not the markup, when you want different content.

| Const | Drives |
| --- | --- |
| `ENG` | The Custom engagement table — **merchant-written only**. `when` is `{ev, delay, cond}` (indexes into `WHEN_EVENTS` / `WHEN_DELAYS`, plus a free-text condition), `ins` is `Default` or `Customized`, `st` is `on`/`off` |
| `WHEN_EVENTS` / `WHEN_DELAYS` | The trigger vocabulary the When block edits |
| `ACE_DOMAINS` | The two-layer theme table (variable name kept as "domain" in code — see below). A theme carries `on`, `ins` and a `plays` array; **it carries no metrics of its own** — see below |
| `ACE_DOMAINS[i].plays[j]` | One play. Real metrics (`reach`, `gmv`, `cvr`, `opt`) and/or the modelled ones (`potReach`, `potGmv`, `potCvr`, `potOpt`); `trg` is the trigger phrase, `ins` the badge, `on` the play's own switch (`undefined`/`true` = on, `false` = off), `st:"setup"` + `note` marks one blocked on an integration |
| `ACE_MODES` | The three mode cards. `sum` is the headline, `brief` the one-liner on the card, `det`/`see`/`who` fill the Learn more panel |
| `ACE_CHAIN` | The six agent steps, shown inside Learn more |
| `CONV` | The conversations inbox. `why` is the one-line reason, `whyBullets` (optional) the evidence behind it, `reason` the key/value reasoning rows, `thread` the messages, `stop` the guardrail that blocked it, `fb` the feedback-loop state (`{vote, note, sent}`, lazily created by `convFeedback()` the first time a thread renders) |
| `STAGES` | The seven lifecycle stages. `lever` holds indexes into `LEVERS`; `capN`/`capU` are the message cap value and unit |
| `LEVERS` / `LEVER_CFG` | Lever names, and per-lever configuration fields keyed by lever index |
| `TONE_TEXT` | The prefilled tone textarea, keyed by stage name |
| `OBJECTIVES`, `DISCOUNTS`, `CAP_UNITS` | The dropdown option lists on a lifecycle stage |
| `SUPPRESSION` / `EXCLUSIONS` | The two guardrail blocks — timing vs people. `locked: true` means it can't be toggled |
| `TIERS`, `EARN`, `REWARDS` | Loyalty. Tiers carry structured `cond` rows so the editor opens populated |
| `PRODUCT_OPPORTUNITIES` | Horizontal, account-pattern recommendations for Guardrails, Lifecycle and Loyalty; each opens an evidence and action panel |
| `OPPORTUNITIES` | The cross-product conversational opportunity inbox, including evidence, impact, proposal fields and action routing |
| `DIMS` | The reporting dimensions. Each member carries a weekly base, a growth rate and **its own rates** — every number on the page is derived from those, so no two figures can disagree. `source` is ACE / Custom / Campaign; ACE absorbed the old `Default` volume when the standard engagements became plays |
| `METRICS` | The ten outbound metrics, each with a kind (`count` / `money` / `rate`) and a direction |
| `GR_CATS` | The seven guardrail categories driving the master/detail. A category is built from optional pieces — `fields` (short value inputs), `toggles`, `select`, plus per-category special blocks (`levers`, `banned`, `list`, `kill`) — `renderGuardrails` checks each and appends the matching markup, so a new category is additive: reuse the pieces that fit, skip the rest |
| `CHANNELS` | The seven rows in the Channels master/detail — SMS through TikTok Shop. `sends` is the list of already-connected identities the "Sends as" picker offers; `extra` (optional) is settings that only apply to that one channel, e.g. Email's quiet-hours exemption; `chSel` (like `grSel`/`stg`) holds which row is open |
| `WEEKS`, `BENCH` | The 12-week axis and the benchmark rows |
| `CAMPAIGNS`, `CAMP_METRICS`, `VARIANTS`, `RULES`, `WIZ`, `WIZ_CHECKS` | Campaigns list, selectable performance columns and the wizard |
| `SIM_SHOPPERS` | The shoppers you can run a live test against |
| `PREV_*` | Word pools for `genPreview(n)`, the pre-launch campaign preview |
| `I_PRODUCTS` / `I_EDGES` | The commerce graph: products and the observed transitions between them. Every Intelligence figure derives from these |
| `MARKETS` | Six markets with a 12-month demand curve, window, lever, discount ceiling and register. The southern curve is the northern one rotated six months, so the inversion cannot drift |
| `PLAYS` | Marketing and e-commerce plays with measured lift — the winners and, deliberately, the losers |
| `SEGMENTS`, `REVISIONS`, `BELIEFS`, `FEEDS` | Shopper clusters, the belief-revision log, and the confidence/blind-spot tables |
| `ASK_Q` | The Ask-the-model shortcut. Each answer is a **function** that reads the live model objects, so it cannot contradict the page |

## The two-layer theme table

`ACE_DOMAINS` holds no metrics. Every figure on a theme row is folded from its plays, so the row
and the rows under it cannot drift apart:

- `playReal(p)` / `playPot(p)` — one play's measured and modelled numbers. `playPot` falls back to
  the real ones, so an active play only needs `potX` fields where they differ.
- `playLive(d,p,aceOn)` — is this play actually running? False if ACE is off, the theme is off, the
  play's own switch (`p.on`) is off, or the play is `st:"setup"`.
- `fold(rows)` — sums reach and GMV and **re-derives** cvr and opt from the reach they were measured
  on. Never an average of averages.
- `domTotals(d,aceOn)` — the live fold, or `null` when nothing under the theme is running.
- `domPotential(d)` — the fold of every play's modelled numbers, including the off and blocked ones.
  This is what the italics show, and what the "GMV left on the table" tag sums.

`aceOpen` is a `Set` of expanded theme indexes; it starts empty — the theme row already carries the
folded figures, so the plays only appear on click. `data-domopen` sits on the whole theme row (not just the chevron) so clicking anywhere
on it expands or collapses that theme; it must be checked **after** the row's own controls
(`data-dom`, `data-dominstr`) in the delegated click handler, or a click on either of those would
just toggle the row instead of doing what it says. `#aceExpandAll` toggles every theme at once.

**A play's own switch (`p.on`, via `data-play="i.j"`) never outranks its theme's.** `playLive`
ANDs both together, so turning a theme off still overrides every play under it regardless of the
play's own setting — the two-layer rule from §3b of PRODUCT.md holds; a play just narrows *within*
an allowed theme now, instead of having no dial at all.

`updateNavCount()` sets the Engagements nav badge from *both* layers: live plays plus custom
engagements that are on. It is called by `renderAce` and `renderEng`, so either one keeps it honest.

## The studio's When block

`openStudio(ctx)` renders `whenBlock(ctx)` above the skill. Three shapes, driven by `ctx.when`:

| `ctx.when` | Shape | Used by |
| --- | --- | --- |
| absent | "There's nothing to set" — the engine owns timing | `studioForDomain` |
| `{mode:"read", line, domain}` | The trigger, stated, tagged *Set by the engine* | `studioForPlay` |
| `{mode:"edit", ev, delay, cond}` | A trigger you set, described in a sentence first, exact rules as the fallback | `studioForEng`, `studioForNew` |

For `mode:"edit"`, the sentence (`#studioNl`) is the primary surface — same "describe it, rules
are the fallback" order as campaign audience (§8 of PRODUCT.md). `whenToSentence(w)` renders the
current `{ev,delay,cond}` back as English for an existing engagement; `studioForNew` seeds a worked
example instead, since there's no trigger yet to describe. `#whenModeToggle` swaps between
`#whenNlWrap` and `#whenRulesWrap` (the old two-select-plus-condition form) without touching either
one's underlying fields — they both write to the same hidden `#whenEv`/`#whenDelay`/`#whenCond`,
which is what `whenReadback()` and Save actually read. **Interpret** is decorative like the rest of
the studio: for a brand-new engagement it fills in a canned example; for an existing one it just
re-confirms the read-back, since the sentence shown was generated from that engagement's real
trigger in the first place.

`whenReadback()` rebuilds the English sentence on every change; it is wired in `openStudio` only when
the block is editable. `ctx.nl` adds the describe-and-interpret box on top, which is the old
`NEW_BODY` panel folded into the studio — the dead end CODEBASE.md used to warn about is gone.

## The reporting engine

`DIMS[dim].members[i]` holds a weekly base, a growth rate and per-member rates. `prim(m,i,w)`
expands that into six primitives for one week — reached, replies, clicks, orders, optouts, gmv.
Everything else is derived:

- `metricAt(key,m,i,w)` — one member, one week
- `metricTotal(key,m,i)` — one member over the period
- `fromTotals(key,totals)` — a metric from summed primitives

**Rates are always re-derived from period totals, never averaged across weeks.** That is why the
chart, the tiles and the revenue table can never disagree — there is one source of truth and three
projections of it. There is a test for this in WORKFLOW.md.

`repSeries()` takes the top four members and folds the rest into `Other`, re-deriving Other's
values from primitives so its rates stay honest.

**Colour is bound to the member, not to its rank.** Assignment happens once at load: the four
biggest members of each dimension take the four categorical hues, everything else is `--dv-other`.
Changing metric or view can never repaint a series.

## The Intelligence Core

`I_PRODUCTS` + `I_EDGES` + the decay curve + `MARKETS` are the only inputs. Everything stated on the
page is computed from them:

- `iDown(k)` / `iStake(k)` / `iStartValue(k)` — what follows a product, and what a shopper who starts
  there is worth. `HERO_MULT` is the ratio of two of those, not a typed number.
- `pAt(d)` is the second-order probability curve. `PEAK_D` finds its peak, and `WINDOW` is the day the
  probability has **halved** — searched forward from the peak, because the curve ramps up over the
  first few days and searching from day 1 lands on the ramp.
- The window headline, the chart's `WINDOW` marker, the scrubber readout and the revision-log entry
  all read `WINDOW` and `pAt()`. Change the half-life and every one of them moves together.

The southern-hemisphere market is `rot(NORTH,6)` — the northern curve rotated six months — so the
seasonality inversion is structural rather than two hand-maintained arrays.

**Ask the model** (`ASK_Q`, `buildAsk`, `askAnswer`) is a floating shortcut scoped to this page:
`goto()` shows the FAB on `intelligence` and hides it everywhere else. Each answer is a function
evaluated at call time against the same objects the page renders from.

## Mutable state

Mutable module-scope state includes `aceState`, `engFilter`, `stg` (selected lifecycle stage),
`convFilter`/`convSel`/`convQ`, `campColumns`, `wizStep`, `studio`, `chSel` (selected channel).
Every one of them is followed by a `render*` call — there is no reactive layer, so **if you mutate
state you must re-render**. A few pieces of state live on the data object itself instead of a
module-scope variable, because they need to survive being scrolled past and back: `c.unread`,
`c.sending`, and `c.fb` (the Conversations feedback loop) all live on the `CONV` row they belong
to, not in a variable that only knows about whichever conversation is currently open.

## Mobile

One breakpoint, `@media (max-width:880px)`, in a single CSS block near the end of `<style>` (search
`MOBILE`) so it always cascades after every other rule it needs to override. Below it:

- **The sidebar becomes an off-canvas drawer.** `.sidebar` is `position:fixed`, translated off-screen
  by default; `.sidebar.open` slides it in. `#mobileMenuBtn` (a hamburger, floating fixed top-left,
  `display:none` above the breakpoint) toggles it via `openSidebar()`/`closeSidebar()`;
  `#sidebarBackdrop` dims the page and closes it on click; clicking a nav item closes it too
  (`goto()` doesn't know about the drawer — it's viewport state, not page state — so the nav item's
  own click handler calls `closeSidebar()` after `goto()`). `body:has(#panel.on) .mobileMenuBtn`
  hides the hamburger while a side panel is open — both float in the same top-left corner, and the
  panel already has its own close button.
- **Master/detail lists go horizontal.** `.lcWrap` drops to one column; `.lcList` flips to a
  horizontally-scrolling row of pills instead of a vertical list, since a vertical list of 7-8 items
  each spanning the full width would push the detail below the fold. Covers Guardrails, Lifecycle,
  Channels and Loyalty's tier/program lists for free — they all already used `.lcList`/`.lcItem`.
- **The Conversations inbox and Opportunities workspace stack** — list on top with a capped
  `max-height` and its own scroll, detail below. `.inboxResize` (the desktop drag handle) is hidden;
  dragging a vertical stack apart isn't a meaningful action once the columns are gone.
- **Tables scroll horizontally instead of losing columns.** The global `table{width:100%}` rule is
  what makes columns compress illegibly on a narrow screen; `.card:has(table) table{width:max-content}`
  lets a table claim its natural width inside a `.card:has(table){overflow-x:auto}` wrapper, so the
  hidden columns are a swipe away instead of gone. The `!important` on `overflow-x` is load-bearing —
  several table cards set `overflow:hidden` inline for their rounded corners, and inline styles beat
  an external stylesheet rule for the same property regardless of selector specificity.
- **Label-left, control-right rows wrap.** `.row.between` (a guardrail toggle, a lifecycle field,
  Channels' "Optimize for") is the single most common settings-row shape, and none of the dozens of
  call sites were built expecting to wrap — several carry an inline `style="max-width:70%"` (or 76%)
  on the label sized for a wide desktop row. `.row.between{flex-wrap:wrap}` plus a blanket
  `[style*="max-width"]{max-width:100% !important}` override fixes all of them at once rather than
  editing each call site. `flex-wrap:wrap` alone never forces a row that already fits to break, so
  this is safe on rows that were never a problem.
- **Two headers build their own markup instead of using `.pageHeader`** — Opportunities
  (`.oppMasterHead`) and the studio panel's own header. Both got their own hamburger-clearance
  padding rather than inheriting `.pageHeader`'s; if a future page does the same, it needs the same
  treatment, `.pageHeader`'s fix won't reach it.

If you add a new two-column layout or a new label+control row, it should already work at 390px
without a special case — check it against this section before assuming it needs one.

## Panels

`openPanel(title, sub, bodyHTML, footHTML)` fills the single `#panel`. `closePanel()` hides it.

The studio can become a wide two-column variant, but starts single-column at the normal panel
width — testing is secondary to the skill itself, so it costs no width until asked for.
`openStudio` explicitly removes `wide` up front (so a stale state from a previous studio session
can't leak in), and the body's `#studioTestCol` starts `hidden` inside a `.studio` grid that
defaults to one column. Clicking "Test this skill" or "Run test" calls the shared `openTest()`
closure, which adds `wide` to the panel, `testOpen` to `#studioGrid` (switching the grid to
`3fr 2fr`), and un-hides `#studioTestCol`. `openPanel` also strips `wide` whenever the body isn't a
studio, and `closePanel` clears it after the transition — so a normal panel never inherits the wide
width either way.

Entry points: `studioForEng(i)` for an engagement, `studioForDomain(i)` for a theme,
`studioForPlay(i,j)` for a play, `openLearn(k)` for a mode, `openShoppers(stage)` → `openShopper(s)`,
`openTier(i)` (`null` for a new tier).

## The live test

`simOpener(ctx, shopper)` produces the first message. `simReply(text)` matches the merchant's typed
reply against keyword groups (sizing / price / problem / refusal / anything else) and returns
`{msg, note}` — the note is the reasoning line under the thread. `studioSay()` appends and schedules
the reply; `drawStudio()` repaints.

To add a behaviour to the simulation, add a branch to `simReply`. Keep the `note` honest about which
rule produced the answer — that note is the whole point of the feature.

## The resizeable conversation list

`.inbox` is a three-track grid: the list, a 6px `#convResize` handle, then the thread. The list's
width lives on `--convListW`, a CSS custom property on `.inbox` with `344px` as the CSS-side
default — so as long as nothing ever sets the property, the layout is exactly what it was before
this existed. Dragging updates `--convListW` directly via `inbox.style.setProperty`, clamped to
240–560px; double-click calls `removeProperty` to fall back to the CSS default. Because the width
lives on the grid container rather than in any state `renderConv()` touches, a re-render (new
filter, new search term) can never reset it — `renderConv()` only ever replaces
`#convItems`/`#convThread` innerHTML, not `.inbox` itself.

## Charts

Hand-rolled inline SVG, no library. The main time-series view always uses `lines`, regardless of
metric type: cardinal-spline curves, subtle per-series gradient fills, an animated draw, shared
crosshair and hover points. It **measures `host.clientWidth`** and generates at 1:1 so text doesn't
get downscaled; the report re-runs on a debounced resize. The same data also has ranked and table
views through the `data-view` segmented control.

**Palette.** `--dv-1 #7e55f6` (purple, ACE) · `--dv-2 #c35e4a` (coral) · `--dv-3 #149db8` (teal,
Default) · `--dv-4 #0e4ea7` (blue), plus `--dv-other #b3b8c1`. Axiom steps validated **on all
pairs**, not just adjacent ones — the earlier blue/purple pairing failed the normal-vision floor
(ΔE 11.5, under 15) and was genuinely hard to tell apart in the line chart. The set is on
`<body data-palette>`. **If you change a series colour, re-run the validator** (see WORKFLOW.md);
don't eyeball it.

**Stat tiles** match the Gorgias metric card: label, info icon and a `+`, a large tabular number,
a delta whose direction comes from a rotated arrow rather than from colour, and a full-bleed
`sparkline()` with a gradient fill. They are **read-only and account-wide** — deliberately
independent of the report below them.

Stacked bars put the 2px gap on the *top* of each segment except the topmost, so the bottom segment
stays anchored to the baseline. Only the topmost segment gets the 4px rounded end.

## Conventions

- No hex colours in new CSS — use the semantic tokens.
- Spacing only from the scale: `--xxxxs` 2 → `--xxl` 48.
- Type through the `.h-*` / `.t-*` classes; never a raw font-size.
- Chart text wears ink tokens, never the series colour. Identity comes from the legend, the swatch
  and the direct label — never colour alone.
- New interactive things use `data-*` attributes and the delegated document click handler, not
  per-node listeners, so they survive a re-render.
- Copy is plain English, no em-dash-free rule but no jargon either. It reads like a colleague
  explaining the product, because half the prototype's job is the copy.

## Ranked and table views

`hBars` is a single-measure chart, so it uses one hue — ranking is magnitude, not identity. The
table view is not a nicety: the dataviz rules require a table alternative wherever colour carries
meaning.

## The campaign wizard

`page-campaign-new` is a single scrolling conversation, not three static panels. `WIZARD` is a
linear script of turns; each has an `ask`, `hint`, `chips`, an `ack` (string or a function of the
answer), and an optional `card` (function returning HTML shown after the ack). `askTurn()` renders
the next question; `campAnswer()` echoes what the merchant typed, shows a typing indicator, then the
ack and card, then advances. The Who/What/How stepper at the top is a **passive status bar** — it is
not clickable and does not drive the flow; `wizIdx` (position in `WIZARD`) is the only source of
truth, and `wizStep` is derived from it for display.

`resetWizard()` runs on every `goto("campaign-new")`, clearing the thread and restoring `VARIANTS`
to its starting three — so opening a new campaign is always a fresh conversation, and demoing it
twice doesn't accumulate a stale variant list from the last run.

**The flow, turn by turn.** The script is longer than three steps because the stepper is passive —
the conversation can revisit a step's substance more than once:

1. **Opening pitch** — the Cirrus launch is argued, not just named: the gap (shell owners with no
   mid-layer), the moment (first sub-zero forecast), the fit (compatibility, not replacement), the
   size (≈$42k). `opportunityCard()` lays out the why.
2. **Audience, recommended first** — rather than asking the merchant to describe an audience from
   nothing, the wizard proposes one (`audienceCard()`), states why it's the recommendation, and
   invites adjustment in plain words. The answer re-renders the card with the adjustment applied
   (e.g. "only those active this winter, skip recent returners" adds the two chips). A merchant who
   wants their own says so and the next turn takes the description. Exact rules remain the fallback.
3. **The story becomes a skill in blocks** — the answer is drafted into `skillBlocksCard()`: four
   labelled blocks (Product / Angle & tone / Offer / Never say), because a real skill is a long,
   complex instruction and reviewing it means reacting to specific parts, not one wall of text. The
   next turn is the back-and-forth: the merchant comments ("don't mention price at all", "make it
   less salesy", "add the warranty") and the matching block is rewritten and the card re-shown.
4. **A/B test or not** — a plain question: "Do you want to A/B test this, or send one version to
   everyone?" `wizAB` records it. Testing calls `resetVariants()` for three approaches; one version
   collapses `VARIANTS` to a single `singleVariant()` and `variantCard()` reads "The message" /
   "One version, sent to everyone". Downstream, `WIZ_CHECKS()` (a function, not a constant, for
   this reason) swaps its Discount-ceiling line, and `launchCampaign` marks the campaign
   "Testing 3 skills" or "One skill".

**Every recommendation tag opens a Learn more.** Each tag on the opportunity, audience and
skill-block cards is a `button.tagBtn` carrying `data-wizlearn="key"`. `WIZ_LEARN` maps the key to a
title, a subtitle and the detailed instruction; `openWizLearn(k)` shows it in the side panel,
read-only, with a **"Use as my answer"** action (`data-useanswer`) that prefills the compose box
(`#campSay`) with the instruction and closes the panel — so the merchant sees the full text land,
can personalize it, and sends it back as their own reply. It is the same pattern at every stage:
the recommendation is always inspectable, and always editable before it's committed.

**The suggested-reply chips do the same.** A chip that has Learn more content (mapped by
`chipLearnKey()` from its text to a `WIZ_LEARN` key) renders as `.chipLearn` with a `?` affordance
and **opens the panel instead of answering** — the reply is only filled (into the same compose box)
when the merchant picks "Use as my answer". A chip with no detailed content (e.g. "I have another
idea") keeps the old behaviour: click fills the box and sends. So a suggestion is never committed
blind — you can always read what it actually means first.

**The preview is a question, not a default block.** The wizard's last turn asks whether the
merchant wants to see what would actually send; only a "yes" answer calls `genPreview(n)` and opens
a scrollable side panel (`openPreviewPanel`) of n generated conversations. `genPreview(n)` builds
plausible conversations from word pools, splitting the variants the way the wave would and marking
roughly one in nine as deferred by a collision — it is index-driven, so the same n always produces
the same set. `openPreviewConv(i)` opens one as a full thread with the skill that produced
it, with a "Back to the list" action. Nothing about it simulates the model; it exists so a merchant
can see the shape and tone of a send before committing.

## Known dead ends

`TEST_BODY` is a template string for the page-level Test buttons (`data-act^="test"`, currently only
the Lifecycle stage tester). It predates the studio and overlaps with it. `NEW_BODY` was the other
half of that pair and is gone — "New engagement" now opens `studioForNew()`, so creating and editing
a custom engagement are the same panel. `TEST_BODY` could go the same way.
