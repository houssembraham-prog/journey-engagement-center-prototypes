# The product argument

This prototype exists to settle a handful of questions about how a merchant configures AI Journey.
Each section below is a decision, why it was made, and — where it matters — what it replaced. The
"replaced" part is the useful part: several of these were wrong on the first pass and the reason
they were wrong is the reason the current shape holds.

---

## 1. ACE is one switch, not a library of scenarios

**Decision.** ACE (Autonomous Customer Engagement) is a mode, not a set of things you turn on.
Behind the switch, a chain of agents watches your data, finds a moment worth acting on, decides
whether *this specific person* should hear from you, writes the message, gets checked by guardrails,
and learns from the outcome. The merchant configures none of that.

**What it replaced.** A first pass modelled ACE as a feed of "opportunities" you activate one by
one, and ACE-sourced rows sitting in the engagements table alongside the configured ones. Both were
wrong for the same reason: they turned an autonomous system back into a configuration surface. If
the merchant has to pick the use cases, the machine isn't deciding — they are.

**Consequence.** There is no "Opportunities for you" screen anywhere. Every ACE decision is
per-shopper and unique to that shopper, so it belongs in a log of conversations, not a backlog of
templates.

## 2. The three modes are Off / Manual acceptance / Automatic

**Decision.** The modes describe *who presses send*, and are rendered as three full cards with a
one-line brief and a **Learn more** that carries the real explanation — what it does, what you'll
see, when to pick it, and the agent chain that runs underneath in every mode.

| Mode | Meaning |
| --- | --- |
| **Off** | ACE stops looking. Nothing queued is held; it drops, so nothing goes out days late |
| **Manual acceptance** | It does every bit of the work, then waits. Nothing sends without you. Items expire after 48h rather than going out stale |
| **Automatic** | Same work, no waiting. Guardrails still run last. You review after instead of before |

**What it replaced.** "Dry run" — which described a testing posture rather than an operating one.
The distinction that matters to a merchant is not *is this real* but *do I press send*. Also
replaced a four-level ladder (Off / Suggest / AI-supported / AI-driven), which drew a distinction
between "suggest" and "supported" that nobody could name.

**Why the cards, not a segmented control.** This is the single highest-stakes choice in the product
and the one that needs the most education. It had a 28px-tall control; now it has a third of the
screen and a Learn more panel.

## 3. Themes are the only dial — and refusing one has a visible price

**Decision.** Eight themes bound what ACE may work on: replenishment, retention, cross-sell,
service recovery, subscriptions, loyalty moments, consideration, seasonal. Turn one off and ACE
stops looking there entirely, regardless of what any play underneath it is set to.

**Why they exist.** Enterprise control. A merchant who cannot bound an autonomous system will not
switch it on, and "all or nothing" is not a real choice at that size.

**The tension, and the resolution.** Themes give control but create a blind spot: you can't see
what you're declining. So a theme that is off does **not** show dashes. It shows its modelled
potential — shoppers reachable, GMV, conversion, opt-out — in purple italics, with a header tag
like `2 off · ≈$19,400 GMV left on the table`. The estimate is explicitly labelled as modelled on
comparable merchants rather than measured on you. Declining a theme stays a choice, but an
informed one.

**Metrics on every theme row:** shoppers reached, GMV, conversion rate, opt-out rate. Those four,
because they are the four a merchant actually argues about.

**Per-theme skill.** A theme can carry its own optional skill, for merchants who need
personalisation on one theme without touching the rest of the account. Same one-skill rule
as everywhere else.

**Renamed from Domain.** "Domain" tested as a technical word — merchants read it as data
governance, not as a lever they were meant to pull. "Theme" says the same thing in the language a
merchant already uses to talk about a campaign idea.

## 3b. Two layers: a theme, and the plays underneath it

**Decision.** A theme is not a black box. Expand one and you see the **plays** it runs —
*Consideration & browse* holds Cart abandonment, Session abandonment, Browse abandonment and
Welcome; *Subscriptions* holds renewal, card expiring, payment failed and the pause-save. Every
play shows its own reach, GMV, conversion and opt-out, the trigger the engine uses, its own
skill badged `Default` or `Customized`, and its own on/off switch.

**What it replaced.** A flat list of pre-built "standard" engagements sitting in the second tab
next to the merchant's own, each with its own on/off switch. That list was the scenario library
§1 says ACE isn't — it just wasn't labelled as one. A merchant switching *Cart abandonment* on and
*Browse abandonment* off was picking use cases by hand, which is exactly the thing the switch was
supposed to end. The first version of the two-layer table over-corrected for that by removing the
play's switch entirely; the switch came back once it was clear the objection was never "a play has
a dial", it was "the theme has to be the outermost one."

**The rule that makes the two layers hold: a play can never outrank its theme.** A play's own
switch only decides whether *that one play* runs; the theme's switch still decides whether *any*
play underneath it can run at all. Turning a play off narrows the theme without touching the
account-wide dial, and turning the theme off overrides every play's own setting, on or off. That
asymmetry — the outer dial always wins — is what keeps this from becoming the scenario library §1
rules out: a merchant can fine-tune inside a theme, but can't rebuild ACE one play at a time without
first deciding to allow the theme at all.

**A row expands by clicking anywhere on it**, not just a chevron — the two layers are the point of
the screen, so getting to the plays underneath a theme should take the smallest possible click.

**Why show them at all, then.** Because "trust the machine" is not a demo. A merchant asked to turn
on an autonomous system wants to know what it will actually *do*, and eight one-line theme
descriptions don't answer that. The plays are the answer, and they're also the honest place to
report a blocked integration: *Delivery exception* sits under Service recovery with a **Needs
setup** tag and a modelled figure, so the missing carrier feed has a price on it.

**Every theme number is derived from its plays.** `domTotals()` sums reach and GMV and re-derives
the rates from the reach they were measured on, filtering out any play that's off — its own switch,
its theme's, or ACE's. No hand-written totals, the same rule Reporting follows — so a theme row and
the rows underneath it can never disagree.

## 4. One skill per unit

**Decision.** Every engagement, every theme and every play carries exactly **one** skill — the
agent loads it when that thing fires. It has a badge: `Default` or `Customized`, and a
"Reset to default" when it has been edited.

**What it replaced.** A four-layer stack (brand → centre → mode → trigger) with a "resolved
instruction set" preview. Conceptually tidy, practically wrong: it invented a hierarchy the system
doesn't have, and it made every instruction feel like a negotiation with three invisible parents.
The word itself changed later too — "Instruction" tested as something you type into a form;
"Skill" says what it actually is, a thing the agent loads and runs, and matches how the studio
already labelled the technical identifier underneath it (`skill · cart-abandonment`).

**The line that survived, and matters:** *skills ask; guardrails decide.* A skill can request
anything; the guardrails run last and can refuse it. That distinction is what makes the model
defensible in a demo, and it is shown literally in the live test (§6).

## 5. Naming: AI engagement / Custom engagement

**Decision.** The two tabs inside Engagements are **AI engagement** and **Custom engagement**.
The second tab is now *only* what the merchant wrote — five things, not eighteen.

**What it replaced.** Twice. First "Scenarios you own", which implied the merchant owns only half
of this; they own all of it. Then a mixed list with a **Built by** column reading `Default` or
`Custom`. That column was the tell: if a table needs a column to say who authored each row, the
table is holding two different kinds of object. The pre-built ones went where they belonged —
inside a theme, as plays (§3b) — and the column disappeared with them.

**Consequence.** The tab has one nature again: things you wrote, that you can retime and switch
off. Nothing in it is "yours by default".

## 6. Opening an engagement opens a studio, not a summary

**Decision.** Clicking an engagement (or its Edit button, or a theme or play skill) opens a panel:
**when it fires** then **what it says**. A **live testable conversation** sits behind "Test this
skill" — closed by default, so the skill is the whole panel until you ask to test it, then it
widens into two columns. You pick a shopper, run it, then reply *as the shopper* and watch what
comes back — with the reasoning shown under the thread.

**When and What, in that order, and the When half is the tell.** For a custom engagement, the
trigger is a sentence first — "when someone buys a bike and has never bought a helmet, wait 5 days"
— the same "describe it, rules are the fallback" order §8 uses for audience. **Interpret** turns the
sentence into the event, the wait and an "only if" condition; a read-back underneath says the whole
thing in English and reminds you the timing guardrails still sit on top; "Edit as exact rules
instead" swaps to the raw fields for a merchant who wants to be precise rather than descriptive. For
a **play**, the same block is present but stated rather than editable — *Set by the engine* — with a
line pointing at the play's own switch, or the theme's, if you don't want it running. For a
**theme skill**, there is no trigger at all: "There's nothing to set. ACE watches for the
moment across this whole theme and picks the hour per shopper."

Three variants of one panel, and reading them in sequence is the fastest way to understand who owns
timing at each level. That is why the When block exists even where it has nothing to edit.

**What it replaced.** "What it produced last time" — a static transcript. It told you nothing you
could act on and couldn't be interrogated. The panel also used to open at full two-column width by
default; that gave testing the same visual weight as the skill on every single open, most of which
are a quick edit with no intention to test. Testing earns its width when you ask for it, not before.

**Why it earns the space.** It is where the guardrail model stops being a claim:

> *"what size should I get?"* → "Medium is the one — it sits under the tights without bunching…"
> **Note:** Answered the sizing question before selling. No offer used.

> *"any chance of a discount?"* → "I can't drop the price on this one, but shipping's on us this
> week — and you're 60 points from a free-shipping reward anyway."
> **Note:** Discount withheld: guardrail caps the AI at 10% and the loyalty lever was cheaper.

The second one demonstrates §4, §9 and §10 in a single exchange.

## 7. Two tabs, not three — and the switch is never inside a tab

**Decision.** Engagements has two tabs, with a sticky strip above them carrying ACE's state and,
in Manual acceptance, a "Review N waiting" action.

**Why three tabs were wrong.** Default / Custom / ACE split *the same kind of object* into
arbitrary buckets. That's a filter wearing a tab's clothing — and it's why Custom ended up looking
different from Default for no reason.

**Why two are right.** They separate two natures of work: a switch you trust, and a list you
maintain. That is a real boundary — and it only became a clean one once the pre-built engagements
moved out of the list and into the switch (§3b), as themes and plays. Before that, the "list you
maintain" contained thirteen things nobody had written and four they had.

**The condition.** A mode this important cannot be hidden behind tab selection, so its state lives
in the strip and is readable from either tab.

## 8. Campaign creation is a three-step conversation

**Decision.** Who → What → How, as a stepper. Each step opens with a question the agent asks, you
answer in plain language, and the answer expands into a structured result you can edit. Audience
can be re-read **as rules** if you'd rather be exact. Skills are A/B tested: three approaches,
removable and addable, and a rollout rule — send 10%, wait an hour, send the winner to the rest,
unless nothing leads by more than 5%, in which case hold and say so.

**What it replaced.** One dense screen with everything on it.

**Order matters:** "Describe it" is the first and default audience mode; rules are the fallback.

## 9. Guardrails separate timing from people

**Decision.** Two distinct blocks.

- **Suppression** is about *timing*: don't message this person *right now* (open ticket, recent
  refund, live conversation, frequency, quiet hours).
- **Who we never contact** is about *people*: they are out of every audience, permanently or until
  cleared. Left a review of 3★ or less. Rated a support conversation 1 or 2 — AI **or** human.
  Opened a dispute. Asked not to be contacted in words, without a formal STOP. Returned more than
  half of what they bought. Wholesale, press, employees, test accounts. Flagged do-not-market.

Three suppression rules and two exclusion rules are locked and cannot be turned off.

## 9b. Channels is its own page, master/detail like Lifecycle

**Decision.** Seven channels — SMS, RCS, Email, Instagram, Messenger, TikTok, TikTok Shop — sit in
a list on a dedicated **Channels** page; clicking one opens its detail on the right. Same shape as
Lifecycle's stage list and Guardrails' category list, deliberately: a merchant who already knows
how to pick a lifecycle stage or a guardrail category doesn't have to learn a new pattern to pick a
channel. The list row carries the channel's name and a one-line status (`On · Verified`,
`Off · Not connected`) so the whole set is scannable without opening any of them. A channel's
detail holds:

- **A switch**, next to the heading. SMS is locked on — it's what RCS and every social DM channel
  falls back to, so turning it off isn't a real choice. Everything else is a real toggle.
- **"Sends as,"** a picker over the identities already connected for that channel (a short code or
  long code for SMS, a verified brand agent for RCS, a from-address for Email, a handle or page
  name for the social channels) — not a free-text field, because a merchant doesn't type a sender
  identity, they select one that's already been set up.
- **A status tag** — `Verified`, `Domain verification pending`, `Not connected` — the same "Needs
  setup" honesty Engagements uses for a play with no integration behind it (§3b), applied here to a
  channel with no connection yet.
- **Settings that only make sense for that one channel**, inside its own detail rather than a
  shared list. Today that's Email's two: exempt from Timing's quiet hours (off by default — an
  inbox doesn't buzz in someone's pocket the way a text does, but that's a merchant's call, not an
  assumption) and a locked, legally-required unsubscribe line, separate from the SMS/RCS opt-out
  line in Guardrails → Content. TikTok Shop carries a note instead of a toggle: its own platform
  policy restricts it to order and shipping updates, so it never carries marketing content
  regardless of the skill.

Above the list, a **General** card holds account-wide settings that apply across every channel
rather than to any one of them — today just one: **"Optimize for"** decides which channel goes
first when a shopper can receive more than one — *Best chance of success* (richest channel first),
*Lowest cost* (cheapest first), or *Fastest delivery* (SMS first). Each option states the order it
implies rather than hiding it behind a label, and it never overrides a channel a merchant has
switched off below. It comes first because it's the thing that's true regardless of which channel
a merchant is about to click into — reading it after the list would mean re-reading the list once
you find it.

**What it replaced.** Two passes before this one. The first put Channels inside Guardrails, as one
more master/detail category next to Frequency and Timing — wrong because a guardrail is a limit on
what an engagement may do, and a channel is where a message goes and who it's sent as; conflating
them made both harder to scan. The second gave Channels its own page but as a grid of cards, one
per channel, each holding its own switch, sender picker and settings. That felt native for three
channels; once social channels made it seven, cards of uneven height (Email's two extra settings
against everyone else's none) stopped reading as one coherent list. Master/detail — the same shape
Lifecycle already uses for seven stages — scales the way a card grid didn't, and it means a
merchant never has to learn a second pattern for "pick one thing, see its settings."

**What it deliberately doesn't do.** It doesn't re-litigate consent — "no valid consent on file" is
already a locked Suppression rule in Guardrails, and it applies the same way regardless of channel.
Channels governs the medium and the identity behind it, not who's allowed to be contacted at all.

## 10. Levers are configured objects, not labels

**Decision.** Each lifecycle stage carries an ordered list of levers, and each lever is an object
you configure. The discount lever has type (% / fixed / free shipping), value, code strategy
(unique per shopper / shared pool), expiry and minimum order. Points boost has a multiplier and a
duration. Early access has a window. Tier fast-track has a jump and a hold period.

Levers are badged `cheap` or `costs margin`, and the guardrails carry an explicit order of
preference: answer the objection → early access → points boost → free shipping → **discount last**.

**What it replaced.** Chips with lever names on them, which implied a choice the merchant couldn't
actually make.

## 11. Lifecycle is "who is this" plus "how do we treat them"

**Decision.** Master/detail over seven stages: Visitor, Lead, First-time buyer, Repeat, VIP,
At risk, Dormant. Each carries what puts someone there, and then how they're treated: objective
(a dropdown of specific outcomes, not free text), tone (a **prefilled textarea** per stage), what
we're willing to give, a message cap as a value plus a timeframe unit, the lever objects, and
which sources may touch the stage.

**What it replaced.** A "your primary goal this quarter" section — removed. There isn't one goal;
the goal is revenue. And free-text fields where a dropdown of known-good options is kinder.

**"See the N shoppers"** opens a searchable drawer; clicking a shopper opens their profile. In the
real product that is a new tab, and the prototype says so.

## 12. Conversations is a marketing inbox

**Decision.** Not a table of events — an inbox. List on the left with search and filters (Waiting
for you / Live / Closed / Stopped, and a source filter), thread on the right. Every thread opens
with:

1. **Why this conversation started**, in plain language, badged with source and theme
2. **What it was reasoning from** — lifecycle stage, loyalty state, timing chosen, lever picked
3. The conversation, or the exact guardrail that stopped it

Approvals live here. There is no separate approval queue; "Waiting for you" is a filter.

**The list is resizeable.** Drag the handle between the list and the thread; double-click it to
reset. A fixed-width list is a good default but not a good ceiling — a long shopper name or a
reasoning-heavy thread deserves more room sometimes, and the merchant should get to decide that,
not the prototype.

**"Why this conversation started" is a sentence plus the bullets behind it.** The one-line summary
stays — it's what you read first — but underneath it, 3–4 bullets break the decision into the
specific evidence the engine weighed: the timing signal, the behavioral history, the thing that
made *now* the right moment rather than last week or next. "Day 51 of her window" is a claim; "the
median for buyers like her is day 54, and nothing today covers days 48–58" is the kind of thing a
merchant can actually push back on. Depth here is what makes the reasoning inspectable rather than
asserted — the same argument as the live test in the studio (§6).

**A feedback loop sits directly under it.** Two buttons — *Useful* / *Not quite right* — plus an
optional note, so a merchant can flag a specific decision without leaving the thread or filing a
ticket. Sending requires a vote first; the note alone isn't a signal, a vote is. Once sent, the
card shows a one-line confirmation instead of asking again — this is a place to register a
correction, not a survey a merchant re-answers every visit. It's per-conversation state (`c.fb`),
so it survives switching to another conversation and back, the same as the unread flag and the
approve-and-send spinner. Like everything outside the custom-engagement studio, sending is
decorative — it doesn't change future conversations in the prototype — but the surface exists
because the reasoning it's attached to is now detailed enough to actually disagree with.

**Why loyalty appears in the reasoning.** It is the cheapest lever and the least obvious one. Seeing
*"Member · 900 points, enough for free shipping → points before discount, costs you $6.40 instead
of $12"* is what makes the loyalty integration legible.

## 13. Loyalty is inside the Engagement Center on purpose

**Decision.** Tiers are editable — entry conditions you compose (spend in 12 months, orders,
points balance, returns rate), earn rate, perks, and the lifecycle stage the tier maps to, so "VIP"
means one thing across the product. Each tier reports members, AOV, LTV, repeat rate and points
outstanding. Each reward shows **your** cost next to its points cost, and carries a toggle for
whether the AI may offer it unprompted.

**The argument.** A discount costs margin every time. Points, early access and a tier bump cost far
less, and on the at-risk cohort they converted better. Loyalty is what lets the engine be generous
without being expensive — which is why it sits here and not in a separate product area.

## 14. Every screen is a link

**Decision.** Each page carries its own `#hash` in the URL — Engagements, Loyalty, Conversations,
all of them — updated on every navigation and read back on load. Copy the address bar and send it;
the other person lands on the same screen instead of always on Reporting. The two tabbed pages go
one level deeper: `#engagements/scenarios` opens straight to Custom engagement, `#engagements/ace`
to AI engagement, `#loyalty/loy-config` to the tier editor — so a link can point at the exact
argument being made, not just the page it lives on.

**Why it matters here.** The prototype's job is alignment in meetings (see the top of this doc,
and `CLAUDE.md`). "Look at the Custom engagement tab in Engagements" is a worse instruction than a link that opens
straight there.

## 15. Engagements explains the operating model once, then gets out of the way

**Decision.** The first visit to Engagements opens a welcome modal that explains the system through
two connected ideas: an orbital view of the business data feeding Gorgias Engage, and an editorial
sequence of the three actions it takes — Find, Decide and Message. Skills and guardrails remain the
final safety layer. A short checklist captures its boundaries, learning inputs, and optimization
goals without adding another explanatory section. The composition reveals in sequence to guide the read without requiring
interaction. The modal does not reproduce the Intelligence Hub or teach every control. It answers
the smaller question a merchant has before using this page: what does Gorgias Engage decide, and what
do I still control?

**The orbit stays live without becoming a feed.** Short, rotating signals show shopper behavior,
store events, shopper context and timing. Only one appears at a time and moves
toward Gorgias Engage, so the animation communicates continuous input without competing with the
four persistent data sources or the explanation beside them. Three signals complete the loop: the
center reacts, then emits either a shopper message or a visible decision to wait. The rest remain
inputs, which keeps the animation from implying that every signal triggers outreach.

**The composition makes ownership explicit.** Gorgias Engage owns the moment, shopper, timing and
message; the merchant sets autonomy and allowed themes. Skills and guardrails remain the safety layer
before anything reaches a shopper. On narrow screens, the explanation comes first and the connected
data illustration follows.

**Why a modal.** This is orientation, not a daily task and not another permanent block above the
table. It appears once, can be closed immediately, and remains available from **How it works** in
the page header. The one-time state is stored locally; nothing is sent or persisted to a
service.

---

## Vocabulary

Use these consistently; several of them were chosen against a worse alternative.

| Term | Means | Not |
| --- | --- | --- |
| **Engagement** | Anything that reaches out to a shopper | "flow", "journey" |
| **AI engagement / ACE** | The autonomous mode | "AI-found engagements", a list of scenarios |
| **Custom engagement** | The engagements the merchant writes, and the only ones with a trigger they own | "scenarios you own", "standard engagement" |
| **Theme** | The bound on what ACE may work on — the only account-wide dial | "domain", "category", "use case" |
| **Play** | One thing ACE does inside a theme. Readable, instructable, has its own switch — but never overrides its theme's | "standard engagement", "sub-domain", "scenario" |
| **When / What** | The two halves of an engagement: the trigger, then the skill | "conditions and content" |
| **Skill** | The single thing attached to a unit that the agent loads and runs | "instruction", "prompt", "instruction stack" |
| **Guardrail** | A hard limit no skill can unlock | "setting" |
| **Suppression** | Don't message them *now* | exclusion |
| **Exclusion** | Never contact them at all | suppression |
| **Lever** | A configured thing the engine may offer | "incentive" |
| **Manual acceptance / Automatic** | Who presses send | "dry run", "AI-supported" |
