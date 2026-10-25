# VibeCon 2026 — VenueTwin (working name)

We are 2 students competing at VibeCon 2026 (YC × Emergent), Oct 24–25. **Submission deadline: Sun Oct 25, 12:00 PM PT (hard).**

**Read `docs/BRIEF.md` first.** It is the single source of truth: rules, the chosen idea, scope, research, risks and team split. Update it when decisions change, and tell your teammate.

## Hard rules for Claude in this repo

- The product must be **built entirely inside Emergent** (app.emergent.sh) by Emergent's agent. **Never write app code for us to paste into Emergent.** Pasting outside code is against the rules and can disqualify us.
- Use Claude for:
  - research, with sources (mark vendor/SEO-only stats as WEAK; never invent numbers)
  - specs, data models, and plain-language Emergent prompts (behavior + data + UI, no code blocks)
  - QA checklists, bug reports, demo script, pitch copy, submission text
- Synthetic data only for any personal or financial records. Stripe test mode only.
- Credits are scarce (~2–3k total). Prefer small, specific, testable Emergent prompts.


## ⚡ Current plan (Sat ~8:15 PM): `docs/EMERGENT_LEAN_PLAN.md`
- **6 prompts (L1–L6)**, demo-first.
- **Two venues** import at L1: the scanned VibeCon Hall and a modeled 152-seat Studio Theater (added Sat ~10 PM).
- We provide the venue twin plus 3D models and media as **data and design assets**. The app built in Emergent imports them and does all the interactive work: the 3D view, the blueprint editor, the AI edit agent, the kits, and the landing and how-it-works pages.
- Be transparent about this in the video. Confirm the asset question with the Emergent desk.

## Status

- Idea chosen: 3D digital twin + auto-layout + quote for small event venues (see BRIEF §3).
- **Demo venue = the actual VibeCon venue.** The Emergent team suggested reconstructing the room we're in, with editable objects.
- Workflow: videos/photos → AI finds objects → **dedupe into unique object types** (one 3D asset per type; instances are placements) → editable 3D twin → catalog, layouts, simulations, quote.
- Docs:
  - `docs/BRIEF.md`: rules, idea, scope, research, team split
  - `docs/CAPTURE_PROTOCOL.md`: exactly what to film and photograph
  - `docs/PIPELINE.md`: pipeline design, build tiers, optimization ideas, where the GPU work happens
  - `docs/EMERGENT_PROMPTS.md`: how the APIs plug into Emergent + the step-by-step build prompts (P0–P13)
  - `docs/EMERGENT_LEAN_PLAN.md`: **the current build plan (L1–L6)**: the prompts we actually send
  - `docs/DESIGN.md` + `assets/design/*.png`: design system and UI mockups (landing, editor, how it works)
  - `assets/`: two importable scenes (VibeCon Hall scanned, Studio Theater modeled), catalog (49 types), kits, 49 GLB models, media
- Decision: proof of concept. Renders must look great but needn't match the room 1:1. Use the LITE capture.
- To verify first: track fit with the organizers (Property & rental?), permission to scan the venue, and which reconstruction / image-to-3D APIs are live (PIPELINE "Open questions").
