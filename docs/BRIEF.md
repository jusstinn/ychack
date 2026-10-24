# VenueTwin — Project Brief (working name)

> Single source of truth for the team (Justin + cofounder) and each of our Claude sessions.
> Last updated: Sat Oct 24, 2026, 12:20 PM PT. Update this file when decisions change.

---

## 1. The event (VibeCon 2026, YC × Emergent, SF, Oct 24–25)

Official rules: https://vibecon.com/rules (the version there is authoritative).

- **Theme:** "Build for Main Street": software for a small business in one of 6 tracks (healthcare, logistics, construction, finance & insurance, **property & rental**, education).
- **What judges want:** "Pick the **one job the business does every day** and make it work live on Emergent from start to finish. Prove the problem costs money… Make it obvious why an owner would want it on Monday morning… who pays, what it saves them, how you'd land your first 100."
- **Hard rules:**
  - All code is written after kickoff (Sat 9:30 AM).
  - **Built and deployed in Emergent**, at a public Emergent URL.
  - **No pasting code from outside Emergent.** No building in another coding agent and importing.
  - Allowed: Claude and other AI tools for **research, planning, copy**; the Claude API and other APIs **inside the app** (own or sponsor keys); design assets, UI kits, icons.
  - Synthetic data only for health, financial, student and tenant records. Payments in test mode only.
- **Team:** 1 Emergent project, owned by the team lead. About 2,000 credits for the whole team, plus one 1,000-credit refill at the credit desk.
- **Deadline: Sun Oct 25, 12:00 PM, hard.**
  - Submit: live URL (plus test login), Emergent job ID, a **≤2-minute demo video** (screen recording plus voiceover, no slides beyond a title card, Google Drive link with anyone-can-view), one sentence on the customer and how they handle the problem today, 2 sentences on what we built, and research with sources.
- **Judging (1–10 per criterion, weighted):**

  | Criterion | Weight |
  |---|---|
  | Execution & UI/UX (works live, data persists, a non-technical owner can use it, **handles a mistake**) | 25% |
  | Originality | 20% |
  | Problem research & data | 20% |
  | Value / market / pitch | 20% |
  | Use of Emergent (ambitious scope shipped end to end) | 15% |

  Pod round judges the video (Sun 1–3 PM). The Top 10 do a 5-minute live demo plus 2-minute Q&A (Sun 5 PM).
- **Prizes:** $10k plus a direct YC interview; $5k; $3k.

## 2. How we got here (so nobody re-litigates)

We explored about 40 ideas with AI expert panels. Note: the AI scoring was partly self-reinforcing, so treat scores as rough.

| Idea | Why it was dropped |
|---|---|
| Trucking detention | Generic: we suspect many teams are building it. |
| Dental PPO / insurance tools | No insurance expertise; not a daily job. |
| Dental 3D scan viewer | iTero and 3Shape already do it free; scans don't show most problems. |
| Theater stage-fit | Tiny market, about $5–30M. |

**Chosen: a 3D digital twin for event venues.** It plays to Justin's prior project ("DoBu": parcel → buildable envelope → **product catalog with dimensional constraints** → packed layouts → genetic-algorithm optimization → rendered plan). Emergent staff said the idea "sounds very very good."

## 3. The idea

**One-liner:** *Film your venue once and get a 3D twin. Every inquiry then gets an instant, accurate layout in your actual room, plus a quote and deposit link, in minutes instead of a site tour.*

**Customer:** small event venues that rent by the day: wedding barns, banquet halls, community halls, black-box theaters, church halls. Typically 1–3 staff, about 10–20 inquiries a week, using HoneyBook or Google Drive plus hand-drawn diagrams or Prismm.

**Daily job (the core loop):** answering inquiries like "will 150 guests with rounds, a dance floor and a buffet fit, and what will it cost?" Today this means back-and-forth email, PDFs of standard setups, and in-person site tours.

**Why now:**
- Cvent acquired Prismm (AllSeated) in April 2025. It went view-only on Oct 1, 2026 and **shuts down Dec 31, 2026**, so venues that used it must switch.
- New AI 3D reconstruction (OpenAI GPT-6 Astra, released Sept 3, 2026: photos → 3D models and walkable scenes) makes "film your room → 3D twin" possible without Matterport hardware.

### Product (two sides)

**Venue side (one-time setup, then occasional edits):**

1. **Capture.** Staff take photos or video of the room.
2. **Reconstruct.** A high-quality 3D model of the room.
3. **Calibrate scale.** The owner enters one known measurement (for example, door width = 36 in) so all dimensions are real. Required for fire-code checks.
4. **Mark fixed features:** doors and exits, columns, stage, bar, outlets, windows.
5. **Set capacity limits:** permitted occupancy per setup type, from the venue's fire permit. Never invent these.
6. **Inventory catalog (the DoBu-style catalog).** Each item has a footprint (W×D×H), clearance needs, seats or capacity, price, and quantity owned.
   - **Venue-owned items:** for example 20 × 60" rounds, 200 chairs, a 16×16 ft dance floor, a stage riser.
   - **Add-on / partner items with links:** for example chiavari chairs, lounge sets, draping or lighting from local rental companies, with a link and a price. This is an upsell for the venue and possibly affiliate revenue for us.
7. **Saved standard layouts:** "Ceremony 180," "Banquet 150," and so on.

**Client / inquiry side (the daily loop):**

1. The client enters date, guest count, event type, and must-haves (dance floor, stage, buffet, head table, lounge).
2. An **auto-layout optimizer** places the items in the real room. It respects exits and egress aisles, the occupancy limit, and owned inventory counts, and it suggests add-ons when owned inventory runs short.
3. The client views it in **3D in the real room**: walk-through, "view from the head table," and drag to move items (constraints re-checked live).
4. A **quote** is generated automatically: venue fee plus inventory plus add-ons. Then a **deposit** (Stripe test mode).
5. The venue gets a dashboard: inquiries, layouts, quotes, deposits.

**"Handles a mistake" moments for the demo:**
- The client asks for 220 guests in a room permitted for 180. The app flags it and offers "180 seated + cocktail standing," or a different setup.
- The client drags a table into the exit aisle. It turns red: "blocks egress (min 44 in)."
- Inventory shortfall: "You own 18 rounds and need 20. Add 2 from [partner] for $X?"
- Scale wasn't calibrated. The app prompts for one known measurement before showing capacity.

### Data model (concept)

- `Venue`
- `Room`: 3D model URL, scale factor, polygon, fixed features, occupancy limits per setup type
- `CatalogItem`: name, footprint W/D/H, clearance, seats, price, owned_qty, source (`owned` / `partner`), link, image
- `Layout`: room, placed items `[{item, x, y, rotation}]`, validation results
- `Inquiry`: date, guests, event type, requirements → layout → quote
- `Quote`: line items, deposit, status

### Optimizer approach (concept; re-implement inside Emergent, no copied code)

Mirror the DoBu pattern:
1. **Catalog** with dimensional minima.
2. **Hard constraints:** inside the room polygon, no overlap with fixed features, exit clear zones, egress aisle widths, occupancy cap, inventory counts.
3. **Soft goals:** sightlines to the stage or head table, dance floor centered, buffet near the kitchen door, even table spacing.
4. Start with greedy grid packing of rounds around the required zones, then local search (swap / nudge / rotate). A small genetic algorithm only if time allows.
5. Output placements plus a list of violations.
6. **Egress and occupancy numbers must be cited** (NFPA 101 / IBC occupant-load factors) or come from the venue's own permit. Don't invent fire-code numbers.

## 4. Market and competition (sourced; mark anything unverified)

**Market size:**
- **~35,800 US wedding venues** (IBISWorld via MMCG): https://www.mmcginvest.com/post/the-u-s-wedding-venue-market-a-investment-thesis-for-2026-2030
- **Average reception venue cost $12,900**, about 29% of the $34,200 average wedding (The Knot 2026 Real Weddings Study, n=10,474): https://www.theknotww.com/press-releases/the-knot-worldwide-unveils-2026-real-weddings-study and https://www.theknot.com/content/average-cost-reception-venue
- **Theaters that rent:** about 6–9k (Census CBP 2022 NAICS 711310: 4,178; 711110: 3,503; AACT: 6,000+ community theatres). Rental day rates $150–2,000.
- **Our own estimate** (labeled assumptions): ~45k venues × $50–150/mo gives about **$25–80M/yr US** as a subscription. Taking 1–2% of bookings (about $130–260 per wedding) makes it much bigger.
- Ignore the "venue management software = $1–7B" market reports. They come from low-quality research firms.

**Competition:**

| Category | Players |
|---|---|
| Layout / diagramming | **Cvent** (Event Diagramming / Social Tables; CventIQ AI prompts), **Tripleseat Floorplans**, Prismm alternatives chasing switchers: **floors.live** ($149), PlanSeats, eventdiagram, SemiChair, Seated With Love, SeatPlanning |
| 3D tours | **Matterport**, iGUIDE, CloudPano, Cvent 3D tours, Roomagen (AI 360 tours) |
| Venue CRM / booking | Tripleseat, Perfect Venue, HoneyBook, Planning Pod, VenueBot (AI inquiry replies) |
| Theater-specific | Setplot ($9–299), Propared, Theatre Manager |
| Schools | Facilitron |

**Our differentiation:**
1. **No manual floor plan drawing:** film the room and get a 3D twin.
2. **No manual table dragging:** an optimizer auto-lays out each inquiry, with egress and capacity checks.
3. **Inventory plus partner add-ons in the quote.**
4. Built for the 2-person venue, not the hotel ("Cvent-level tools for the wedding barn").

**Biggest threat:** Cvent, which has 3D, AI and diagramming. Answer: it's priced and built for hotels and big venues.

## 5. Known risks and open questions — verify FIRST

1. **Track fit.** Ask the organizer / Emergent desk explicitly: "Does an event venue that rents by the day count as Property & rental?" An app that misses its track can't advance.
2. **Reconstruction API: RESOLVED (Sat ~1 PM).**
   - **GPT-6 Astra's API outputs text only, with no 3D files.** Use it or Claude as the "brain."
   - Use **fal SAM 3 / SAM 3D Objects / TRELLIS.2** for masks and per-type 3D assets.
   - Use **World Labs or KIRI** for an optional photoreal splat.
   - Details: `docs/EMERGENT_PROMPTS.md` → "Verified stack."
   - Never paste generated *code* into Emergent.
3. **Scale accuracy** from photos. Require one known measurement; show "estimated" labels.
4. **Mobile 3D performance.** Keep models small (decimate, compressed GLB). Test on a real phone early.
5. **react-three-fiber version:** Emergent apps ship **React 19.2.x**, so use **r3f v9 + drei v10** (pinned). A mismatch causes credit-burning fix loops.
7. **Emergent limits:**
   - ~125 s request cutoff, so every AI step must be a background job with polling.
   - 5 GB storage, files can't be deleted, so keep files small and never store raw videos.
   - Container disk wiped on restart.
6. **Credits:** about 2–3k total. Small, specific prompts; don't rebuild from scratch.

## 6. Scope for 26 hours

**MUST:**
- 1 demo venue with a 3D model
- Scale calibration
- Fixed features plus occupancy limit
- Catalog: about 8 item types, owned plus 3–4 partner add-ons with links
- Inquiry form
- Auto-layout (greedy plus local search)
- 3D view plus drag-to-move with live constraint checks
- Quote plus Stripe-test deposit
- Venue dashboard
- Persisted data

**NICE:** "view from seat," multiple rooms, saved standard layouts, AI-drafted inquiry reply email.

**CUT:** CRM integrations (HoneyBook), multi-venue marketplace, real payments, live reconstruction during the demo (pre-bake it).

## 7. Demo video (≤2:00) beat sheet

| Time | Beat |
|---|---|
| 0:00–0:15 | Customer + pain + strongest stat ("Prismm shuts down Dec 31 and thousands of venues need a replacement," plus a venue owner's real inquiry volume or tour hours from our interviews). |
| 0:15–0:35 | Venue setup: a phone video of the room becomes the 3D twin, with one measurement to calibrate. Inventory loaded. |
| 0:35–1:30 | The daily loop live: inquiry "160 guests, dance floor, buffet" → auto-layout in the real room → the client drags a table into the exit and it turns red → fix → inventory shortfall → partner add-on → quote → deposit. |
| 1:30–1:50 | The venue dashboard shows the booking. |
| 1:50–2:00 | Why we win: no drawing, no dragging, quote in minutes, built for small venues. |

## 8. Research and traction plan (counts for 20% research + 20% value)

- **Saturday is wedding day**, so Sonoma and Napa barns won't answer until Sunday. Today, target **East Bay / SF community halls, banquet halls, black-box theaters and event spaces.**
- **Questions:**
  1. How many inquiries a week?
  2. How do you answer "will X guests fit?" today?
  3. How many site tours a week, and how long is each?
  4. What % of inquiries ghost?
  5. What do you use for diagrams (Prismm? by hand?)
  6. What would you pay?
  7. Can we use your room as our demo venue?
- **Evidence to collect:** owner quotes (with permission), inquiry and tour numbers, non-binding LOIs ("would pilot at $__/mo").
- **Don'ts:** never claim endorsements, never inflate numbers, don't spam groups.

## 9. Team split (suggested)

- **Builder (prompts the Emergent agent; one person at a time to avoid conflicting edits):** venue setup, catalog, 3D viewer, optimizer, quote and deposit.
- **Other teammate:**
  - Verify Astra / reconstruction API
  - Capture the demo venue
  - Owner interviews and research with sources
  - Synthetic test data
  - QA the live preview and write bug lists for the next prompt
  - Demo script and video
  - Submission form
- **Swap every few hours.**

## 10. Claude guardrails (for every teammate's Claude)

- Claude is for **research, planning, specs, plain-language Emergent prompts, copy, QA checklists and the demo script**.
- **Do NOT write app code to paste into Emergent.** All code must come from Emergent's agent.
- When drafting Emergent prompts: describe behavior, data and UI in plain language. No code blocks.
- Cite sources for every number; mark vendor-only stats as WEAK; never invent stats.
