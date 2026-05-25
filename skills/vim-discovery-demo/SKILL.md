---
name: vim-discovery-demo
description: Build a personalized interactive HTML discovery demo for a developer or potential Vim partner. The user provides ONLY a company name — the skill auto-discovers context from the most recent Fathom call with that company, their HubSpot company record and notes, and a web search of their site (for brand colors and positioning). Produces one HTML doc per use case discussed on the call — each with a one-pager overview followed by an interactive mock EHR that mimics the Vim Canvas sandbox. Use when the user says "create a demo for [company]", "build a Vim demo for [company]", "make a discovery demo for [company]", "Vim x [company] demo", or any phrasing that names a partner company and asks for a demo or interactive artifact.
type: workflow
---

# Vim Discovery Demo Generator

You build interactive HTML discovery demos that Vim's BD/partnerships team sends to developers and prospective partners after the first discovery call. Each demo has a **one-pager overview** that explains the use case, followed by a **"Launch Demo →" button** that opens an **interactive mock EHR** showing Vim Connect in action — mimicking the Vim sandbox experience.

A "Vim demo" is per-use-case, not per-partner. One discovery call usually surfaces 1–3 use cases. Produce **one HTML doc per use case** unless the user asks otherwise.

## Required inputs

**Only one input is required: the partner / company name.** The user invokes the skill with something like "create me a demo for Quest Diagnostics" or "build a Vim demo for Acme Health" — and that's it. Do not ask for Fathom or HubSpot links. Discover them automatically.

If the company name is ambiguous (e.g. "Acme" with no qualifier and multiple matches across sources), surface the candidates back to the user and ask which one. Otherwise proceed silently.

## Workflow

### Step 1 — Auto-discover context

Discovery runs in three phases. The Fathom path depends on HubSpot, so it can't all be parallelized — but within each phase, parallelize aggressively.

#### Phase 1A — HubSpot company lookup (and web research, in parallel)

In one message, fire these in parallel:

- **HubSpot company by domain or name** — try `search_crm_objects` first with `objectType: "companies"` and a **domain-based filter** (`{propertyName: "domain", operator: "CONTAINS_TOKEN", value: "<guessed-domain-token>"}`), since text search often returns subsidiary records ahead of the main one. If domain isn't obvious, fall back to `query: "[Company Name]"`. Pull a wide property set so you don't need a second call:
  ```
  ["name", "domain", "industry", "description", "lifecyclestage",
   "hs_lead_status", "city", "state", "country", "numberofemployees",
   "annualrevenue", "hs_revenue_range", "hubspot_owner_id",
   "createdate", "notes_last_updated", "hs_last_sales_activity_date",
   "hs_last_booked_meeting_date", "num_associated_contacts",
   "num_associated_deals", "num_contacted_notes",
   "partnership_type__c", "account_classification",
   "console_account_type", "console_account_activation_date",
   "tos_signed_version__console_", "vbc_tam_target_tier",
   "propensity_account_warmth", "funnel_counter", "linkedinbio",
   "hs_keywords"]
  ```
  Pick the right record (highest `num_contacted_notes` / `num_associated_contacts`, or the one with `partnership_type__c` set, beats sparse subsidiary records).
- **Web** — `WebSearch` for the company name + relevant clinical context. Then `WebFetch` their homepage with a prompt asking for: (a) what the company does in one sentence, (b) primary brand colors as hex codes (visible CSS or logo description), (c) target user (PCPs, specialists, payers, health systems, employers, life sciences), (d) mention of EHR integration, interoperability, FHIR, or clinical workflow tools.

#### Phase 1B — HubSpot contacts + deals (parallel)

Once you have the company `id`:

- **Contacts** — `search_crm_objects` `objectType: "contacts"`, filter `associatedWith` that company id, sort by `lastmodifieddate DESC`, limit 25. Properties: `firstname`, `lastname`, `email`, `jobtitle`, `hs_lead_status`, `lastmodifieddate`. Identify the **top 1–3 partner contacts** to use for Fathom lookup: prefer `hs_lead_status: "Lead Converted"` or `"CONNECTED"`, then most-recently-modified, then most senior title (Director, VP, SVP, Chief).
- **Deals** — `search_crm_objects` `objectType: "deals"`, filter `associatedWith` the company id. Properties: `dealname`, `dealstage`, `pipeline`, `amount`, `description`, `closedate`, `hs_lastmodifieddate`. Deal names often encode the use case (e.g. "Quest Diagnostics - Connect").

> **Note on HubSpot notes/engagements**: On HIPAA-covered portals (which Vim's is), `notes`, `emails`, `calls`, `meetings`, and `tasks` are blocked at the MCP layer with `AUTHORIZATION_ERROR: "Cannot access NOTE data on accounts with HIPAA-covered sensitive data."` Don't try to read them — use the expanded company properties + deal records + the Fathom transcript instead. That combination is usually enough.

#### Phase 1C — Fathom via contact name (parallel)

For each of the top 1–3 partner contacts identified above, fire `find_person` **in parallel**:

```
find_person({ name: "<First Last>", recorded_by: "anyone",
              created_after: "<~12 months ago ISO>" })
```

`find_person` searches the speaker index — much more reliable than `search_meetings` text-matching. Union all returned meetings, dedupe by `recording_id`, then rank:

1. Most recent first
2. Title containing "POC", "pilot", "discovery", "intro", "kickoff", "scoping", "planning" → boost
3. Title containing the partner company name → boost
4. Recorded by a Vim BD/partnership team member → boost

Pull `get_meeting_transcript` for the **top 1–2 ranked meetings only**. Don't pull more than 2 — they're large.

**Fallback if `find_person` returns no meetings for any contact:**
- Try `search_meetings` with `query: "[Company Name]"`, `recorded_by: "anyone"`, `created_after: "<~12 months ago>"`, `max_pages: 5`. If output is large (>50k chars), spawn an Explore subagent to filter it.
- If still nothing, ask the user to paste any Fathom URL (share URL works — resolve it via `get_recording_by_url`).

If a discovery phase returns nothing useful, continue with what you have. Don't block.

### Step 2 — Synthesize use cases

From the call + HubSpot notes, identify the **1–3 concrete use cases** discussed. For each, you need:

- **A short name** (e.g. "Health Alerts", "Draw Tool", "Test Recommendations")
- **The problem** in the partner's words / industry terms (what breaks today)
- **Where Vim fits** — which point in the provider workflow (ordering, charting, reviewing labs, referrals, etc.)
- **Who the EHR user is** for this use case (ordering provider, MA/nurse, phlebotomist, specialist, etc.) — this drives the avatar and role label in the demo
- **Read or write?** — which Vim SDK capabilities are exercised (read patient context, read orders, writeback to encounter/order/referral)
- **The "magic moment"** — the push notification + side panel content the user will see

### Step 3 — Confirm with user (one round, lightweight)

Before generating HTML, post a brief synthesis so the user can catch a miss before you burn compute on the wrong demo:

```
Partner: [Name]
Fathom call: [meeting title, date] — [N min]
HubSpot: [stage] — [last activity]
Brand: [primary color hex] + Vim light blue (#4FC3F7)
Use cases identified:
  1. [Name] — [one-line problem → solution]
  2. [Name] — [one-line problem → solution]
  3. [Name] — [one-line problem → solution]

Generating one HTML per use case. Reply "go" or redirect.
```

Wait for "go" / "yes" / equivalent. If they redirect, adjust and re-confirm. Keep this check tight — it's a sanity tap, not a planning session.

### Step 4 — Generate HTML(s)

For each use case, produce one self-contained HTML file (React + Babel via CDN, no external dependencies — file should open in a browser standalone). Use the **Sandbox visual fidelity** spec below as your authoritative reference for layout, colors, components, and animations.

**Concrete reference HTMLs** are colocated in this skill's `examples/` folder (alongside SKILL.md). They are real artifacts the Vim BD team sent to Quest Diagnostics — read the closest one for a concrete scaffold, then adapt:

- **`examples/quest-draw-tool.html`** — read-only, two-phase user pattern. Use when Vim reads context and surfaces guidance to a SECOND user (e.g. MA/nurse/phlebotomist) after a FIRST user (e.g. ordering provider) has acted. No EHR writeback — Vim is purely a guidance surface. Key elements: phase-bar component at the top of the EHR view; user-switch button at end of phase 1; separate tab sets per user role; notification badge that activates only in phase 2; side panel with structured guidance ending in `Acknowledged` CTA.
- **`examples/quest-health-alerts.html`** — writeback pattern with state machine. Use when Vim corrects or updates EHR data. Key elements: a `step` state machine (`ordering` → `submitted` → `correcting` → `writeback` → `done`); Submit button triggers the notification; per-flagged-item selector cards in the side panel showing current value (strikethrough) vs. supportive options; `Apply & write back to EHR` CTA flips written-back state and triggers visible updates in the relevant EHR tabs with a small "updated via Vim writeback (updateEncounter / updateOrder / updateReferral)" footer note.
- **`examples/quest-test-recommendations.html`** — multi-source chart read + recommendation pattern. Use when the partner's app reads multiple chart sections (vitals, medications, problem list, prior labs) and surfaces a targeted recommendation based on rules. Key elements: schedule view as starting state; opening the chart simulates Vim "reading" (1.5s delay) then surfaces a notification; expandable recommendation cards in the side panel with severity dot, title, subtitle, source guideline (italics), rationale-on-expand listing data triggers; `Draft order & write back to EHR` CTA.

The example HTMLs lean slightly older in styling than the current Vim sandbox — when they conflict with the **Sandbox visual fidelity** spec below, the spec wins.

Then customize for the partner:
- Partner name, partner color (replace `#006B3F` Quest green with the partner's primary brand color throughout)
- The "Q" logo SVG — replace with a partner-appropriate single-letter or shape mark in the partner color
- Patient demographics (realistic but invented — never use real PHI)
- The clinical data, codes, and labels to match the use case
- The problem/solution text on the one-pager — write in the partner's vocabulary (pulled from the call transcript and their website)
- The 4 "How It Works" steps — phrase in the partner's domain language
- The 3 "Why Vim Connect" tiles — keep the spirit (right person/right time, no manual launch, same experience any EHR), reword to fit
- The user role labels and the EHR tab structure to match the workflow

**Do not change:**
- Vim's light blue heart color: `#4FC3F7` (this is Vim's brand mark)
- The dark sidebar `linear-gradient(180deg, #0A3D5C 0%, #0C2340 100%)` and the "eHealth Sandbox EHR" label — this is the pseudo-EHR shell common to all demos
- The right-side Vim ribbon (38px wide) with heart + partner logo + gear icons
- The "Vim Hub classic size" small label in the top-left of the EHR view
- The two-view App pattern: `Overview` (Georgia serif, max-width 780) → "Launch Demo →" → `EHR` (system-ui, full viewport)
- The `← Use Case` and `Reset` buttons in the EHR top bar
- React + Babel via CDN (kept inline for portability — recipient just opens the HTML)

## Sandbox visual fidelity (matches the Vim Canvas sandbox shown in the partner Loom)

The EHR mock should mimic the Vim Canvas sandbox closely. The Loom video the BD team shows partners has these exact visual anchors — match them. Where the inlined reference HTMLs (Example A/B/C below) lean older, the spec here wins.

**Outer layout (full viewport):**
- 3-column flex: sidebar (~150px) | main EHR pane (flex 1) | Vim ribbon (38px). No outer chrome.
- Background of main pane: `#F8FAFC` (slate-50).

**Sidebar (left, ~150px wide, dark navy gradient):**
- Background: `linear-gradient(180deg, #0A3D5C 0%, #0C2340 100%)`
- Top: cyan `eHealth` wordmark, ~16px font-weight 800 in `#4FC3F7`, preceded by a small 7px cyan dot. Underneath, tiny "Sandbox EHR" tagline in `#64B5F6` opacity .7, letter-spacing 1.5, uppercase.
- Nav items (each ~12.5px white at opacity .5, padding ~9px 14px): Calendar, Patients, Encounters, Referrals, Orders, Claims. Active item is brighter white. Each item has a small leading icon (📅 👥 📊 🩺 📝 📃 work as emoji placeholders if SVGs aren't available).
- Bottom: `SDK Docs →` link (same muted white). Final row, separated by a thin top border `rgba(255,255,255,0.06)`: a gear icon + "Vim Settings" label.

**Top white bar (above patient header):**
- Padding `6px 18px`, bottom border `1px solid #E2E8F0`.
- Left: a small role indicator (emoji avatar like 🩺 + user name like "Dr. Alon" + faint role label).
- Right: a "Search patient in the EHR" rounded input (~340px, with magnifying-glass icon, placeholder text in slate-400), then the user pill (avatar + name like "Dr. Alon" + caret), then a small **outline "Change skin"** button on the far right.
- The two demo-control buttons (`← Use Case` and `Reset`) live just below the top bar or in the top-bar right cluster — small outline buttons in slate.

**Patient header bar (white, ~10px 18px padding):**
- 44px circle avatar (linear gradient pastel + bold initials in a dark version of the same hue).
- Bold patient name (~16px, slate-900) with edit pencil + delete trash icons inline next to the name.
- Below, all on one line in 10px slate-500 (values bolded in slate-700):
  `DoB: May 15, 1980, Gender: Female, Zip code: 56789, EHR Insurance: -, Member ID: -, MRN: 123456, Clinic ID: -`

**Encounter title + tabs row (white, padding 10px 18px 0):**
- Title link (bold underlined blue): `Encounters / [Date]` (e.g. `Encounters / Apr 14, 2026`).
- Tabs below (with a 2px slate-200 bottom border): `Personal info`, `Encounters`, `Referrals/Orders` (combined — NOT split), `Problems list`, `Claims`, `Medications`. Active tab: 600 weight in slate-900 with a 2.5px solid `#1565C0` underline at the bottom. Inactive tabs: 400 weight in slate-500.

**Referrals/Orders tab content (if shown):**
- Two-column layout: `Referrals` on the left with a dark navy `+ New referral` button (`#0C2340` bg, white text), `Orders` on the right with a `+ New order` button (same styling, partner color background).
- Each Referral item is a card: small ⇄ icon, then `Specialty: [name] | Refer to: [provider or -]`, then `Created Date: [date]`. Trash icon at right.
- Each Order item is a card: small leading icon (🧪 for Lab, ℞ for Rx), label like `Lab name: Lipid Panel | Refer to: Quest Diagnostics`, then `Created date: [date] | Results date: -`. Trash icon at right.

**Vim ribbon (right side, 38px wide):**
- Background `#F1F5F9`, left border `1px solid #E2E8F0`.
- Top: Vim heart SVG in `#4FC3F7` (~17px). When notifications exist, a small red badge (`#E65100` bg, white number) sits at top-right of the heart, pulsing.
- Below the heart, a stack of app tiles — each a small rounded rectangle ~30x30px in the partner's brand color, with a single letter mark or shape representing the app (e.g. green Q for Quest, navy V for Vera, "DEMO" text for a generic test app). Tiles dim to ~.4 opacity when inactive.
- Bottom: muted gear icon (Vim Settings).

**Push notification (slides in from top-right, ~265-285px wide):**
- Floating white card, position `top: ~60-90px, right: ~48px`, z-index above ribbon. Shadow `0 8px 30px rgba(0,0,0,.15)`, border `1px solid #E2E8F0`, radius 9px, padding ~12px 14px.
- Top row: partner logo (small circle in partner color) + bold partner-app title (e.g. "Quest Diagnostics - Draw Tool", "Quest Health Alert", "Vera Health — Evidence Alert"). Close × on the right.
- Body: 1-2 lines of context (e.g. "This patient has 2 active lab orders pending collection. Collection guidelines are available.").
- Primary CTA button at the bottom, full-width, in the **partner color** (or dark navy `#0C2340` for neutral guidance). Examples: "View collection guidance", "Review & correct codes", "Review recommendations".

**Side panel (slides in from the right edge, ~300-330px wide, full height):**
- Position absolute, right: 38px (sits to the left of the ribbon), z-index above main content, animated slide-in 0.3s.
- Header (~10px 14px padding, bottom border):
  - Tiny patient mini-line at top: `Emma Smith / May 15, 1980 / 45 yo` (slate-500, 11.5px). Close × on the right.
  - Below: small partner logo + uppercase partner-name tag (`QUEST DIAGNOSTICS` in 8.5px slate-500, letter-spacing .5).
  - **Partner-color band** (full-width across the panel, ~8-10px padding, partner brand color background): app title in white-on-color (e.g. "Blood Draw Guidance", "Coverage & Coding Alert", "Test Recommendations") plus a small summary on the right (e.g. "Tube Plan: 1 test → 1 tube").
- Body (scrollable, padding 10-14px): structured cards — tube plan, code suggestions, recommendations, evidence citations. Each card uses muted background tints with partner-color accents.
- Footer (border-top, padding 9-14px): a full-width primary CTA in the partner color ("Acknowledged", "Apply corrections & write back to EHR", "Draft order & write back to EHR"). Below the button, faint footer row: Vim heart icon + "Terms of Service | Privacy Policy" on the left, "Need help?" on the right.

**Animation conventions (CSS keyframes used in the references):**
- `slideIn` for the push notification: `from { opacity:0; transform: translateY(-10px) } to { opacity:1; transform: translateY(0) }`, .35s ease.
- `panelSlide` for the side panel: `from { transform: translateX(100%) } to { transform: translateX(0) }`, .3s ease.
- `pulse` for the notification badge: scale 1 ↔ 1.15, 1.5s ease infinite.

**The "Vim Hub classic size" tiny label** sits top-left of the EHR view (absolute positioned, ~11px, color `#2AA2CF`, opacity to suggest it's a sandbox-only annotation). Keep it.

### Step 5 — Publish

For each generated HTML, publish via `html-publisher` (`publish` tool). Use a descriptive title like `[Partner] × Vim Connect — [Use Case Name]`. Default to **internal** sharing unless the user specifies otherwise.

After each `publish` call returns, **follow the organization instruction**: call Notion MCP (`notion-create-pages`) using the `archive_suggestion.title`, `archive_suggestion.icon`, and `archive_suggestion.content` verbatim, with `archive="user"`. This applies to every publish — don't skip it.

Return the share links grouped by use case at the end.

## Vim platform facts to bake into every demo

These are the load-bearing concepts that should appear naturally in the one-pager copy and demo interactions. Don't drop them all in — pick what's relevant per use case.

**Vim Connect** — Vim's in-EHR connectivity layer. The platform that lets a partner's app appear inside the EHR without the partner doing direct EHR integration work.

**Vim Hub** — the user-facing widget overlay on the EHR. Renders as a docked ribbon (right side of the EHR window) with app icons. Supports activation states (DISABLED / LOADING / ENABLED), notification badges, push notifications (momentary toasts), microphone badges, and dynamic app size (CLASSIC / LARGE / EXTRA_LARGE). The side panel that slides in when the user clicks the app icon is part of the Hub.

**Smart Launch** — Vim's patient-context-aware app triggering. When a user navigates in the EHR (opens a patient chart, places an order), Vim detects the context and can auto-launch the relevant app, set a notification badge, or push a notification. This is what differentiates Vim from SMART on FHIR — the partner's app doesn't require the user to manually click "launch app"; Vim surfaces it at the right moment.

**Read APIs** — Vim's SDK exposes these EHR resources for reading:
- `Patient` — demographics, insurance, problem list, medications, allergies, lab results, vitals
- `Encounter` — subjective, objective, assessment, plan; chart retrieval requests
- `Orders` — labs, imaging, procedures, prescriptions
- `Referral` — target provider, conditions, procedures, auth info
- `Claim` — billing data including service lines, diagnoses, rendering provider
- `Workflow Events` — point-in-time signals (order created, encounter signed) that may not be in the visible EHR state

**Writeback APIs**:
- `updateEncounter()` — append subjective/objective/assessment/plan notes, add diagnosis or procedure codes
- `updateOrder()` — add notes, set or modify target provider
- `updateReferral()` — modify specialty, dates, priority, visits, reasons, auth codes, diagnoses, procedures, target provider

**Supported EHRs** (mention 3–4 by name in the "any EHR" pitch tile): Athenahealth, eClinicalWorks, Elation, NextGen, Practice Fusion, DrChrono, Office Ally, MDland, TouchWorks, Aprima. (In dev: Azalea, Kareo Tebra.)

**Scale stats** (use sparingly — only if relevant): 10,800 clinics, 50,780 connected users, 15M+ patients detected.

## Brand & visual rules

- Vim's brand: **light blue heart, `#4FC3F7`**. The heart icon SVG in the example files is the canonical mark — keep it.
- Partner brand: pull primary color from their website. If you can't determine it confidently, ask the user. Use it for:
  - The header logo mark (replacing the green Q)
  - The "Launch Demo →" button on the one-pager
  - The "Place Order" / "Submit" / primary CTA buttons in the EHR demo
  - The "Acknowledged" / accept button at the bottom of the side panel
  - Step number circles in "How It Works"
- Use the partner's name and the partner's industry terminology in the problem statement and solution narrative — do not generalize to "healthcare" when the partner says "specimen collection" or "specialty pharmacy" or "value-based care."
- Patient data is **invented**. Use realistic-sounding names, DoBs, MRNs, ICD-10/CPT codes that fit the use case. Never use real patient information from Fathom transcripts even if it appears there.

## Output format

For each use case, post:

```
[Use case name]
→ [published URL]
```

At the very end, one sentence: which one is your strongest case for the partner and why.

## What to avoid

- Don't write copy that sounds like a generic "Vim is a healthcare middleware platform" pitch. The whole point is that the partner sees their own workflow reflected back at them.
- Don't reuse the Quest green or the Quest "Q" mark for non-Quest partners.
- Don't invent SDK methods that don't exist. The writeback surface is `updateEncounter`, `updateOrder`, `updateReferral` — that's it. If the use case needs something else, describe it in plain language ("Vim writes the corrected code back to the EHR") and don't claim a specific method name.
- Don't ship a demo with placeholder lorem ipsum. Every text field should be partner-specific and use-case-specific.
- Don't omit the Notion archive step after publishing. It's required for every publish.

