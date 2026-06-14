# FitSense 🧵 — AI Personal Stylist on Telegram

**Paste any product link. Get told exactly how it fits *you* — size, colour, fabric, value, and an honest Buy/Maybe/Skip verdict.**

A working end-to-end MVP built on a **₹0 stack** (local LLM, self-hosted automation, free messaging) — designed so the production upgrade is a config change, not a rebuild.

> 🎥 **Demo video:** [ADD LOOM/YOUTUBE LINK]
>
> 📸 **The personalisation proof:** same product link, different users, different size advice:
>
> [ADD SIDE-BY-SIDE SCREENSHOT — e.g. snug-fit user told M, relaxed-fit user told L]

> Built as a working implementation of the [FitSense product case study by Asmit Birthare](https://github.com/asmit1624/fitsense_product_case_study). The product concept and case study are his; this repo is the engineering MVP that makes it run.

---

## The problem

Indian D2C fashion has a fit problem. Size charts exist on every product page — but a chart knows the garment, not the shopper. Every brand cuts differently (a Snitch M ≠ a Libas M), nobody measures themselves before ordering, and ~30% of fashion returns are fit-related.

**The chart is an input. FitSense is the judgement layered on top of it** — one assistant that knows *your* body, taste, and preferences, and applies them across every brand.

## What a user experiences

1. **Message the bot** → 7 quick onboarding questions (sizes, fit preference, chest range, skin tone, style, budget) → a personal **StyleDNA** profile, created once.
2. **Paste any product link** → in under a minute:

```
Sand Linen Co-ord Set - Rs 1299

Fit Score: 85/100
Great value and comfy fit - a perfect summer look.

Size: In Snitch, your usual M is cut for about a 101cm chest. Snitch runs
slim, so L (~106cm) sits more relaxed - pick M for fitted, L for comfort.
Colour: The sand colour will complement your warm medium skin tone nicely.
Fabric: The linen-cotton blend is breathable and suits casual western wear.
Value: At Rs 1299, this offers good value within your budget band.
Watch out: Snitch runs slim; size up if you prefer a relaxed fit.

Verdict: Buy
```

3. **Come back any day** → it remembers you. `reset` rebuilds your profile anytime.

---

## Architecture

One Telegram bot, one front door, three lanes:

```
Telegram message
      │
      ▼
 Get Context (Postgres: is this user onboarded? mid-questionnaire?)
      │
      ▼
 Router Brain (code) ──► new/reset/mid-questionnaire ──► ONBOARDING lane
      │                                                  (state machine, saves StyleDNA)
      ├──► onboarded + product link ──► SCORING lane
      │     (fetch product + THIS user's profile + brand size chart
      │      → deterministic size math → local LLM for judgement
      │      → reply → log recommendation)
      │
      └──► onboarded + anything else ──► welcome-back greeting
```

**Stack (and the deliberate swaps):**

| Layer | Case-study plan | This MVP (₹0) | Production path |
|---|---|---|---|
| Channel | WhatsApp Business API | **Telegram bot** | WhatsApp BSP (config swap) |
| AI | Claude API | **Ollama, qwen2.5:3b local** (1.5b fallback) | Hosted LLM API — one HTTP node change |
| Data | Supabase + Pinecone | **Postgres 16 in Docker**, keyed size-guide table | Managed Postgres; vectors only if needed |
| Catalog | Live scraping | **Seeded mock catalog** (15 products, 7 brand charts) | Feed/scraper ingestion |
| Hosting | Cloud | **Laptop + Cloudflare tunnel** | ~₹500/mo VPS, fixed domain |
| Orchestration | – | **n8n (self-hosted)** | Same, on the VPS |

---

## The two-brain design (the core engineering decision)

**Code decides everything that has a right answer. The LLM only writes judgement.**

- **Deterministic (code):** product facts, user profile, and the **size decision** — real centimetre math against each brand's actual chart:
  - With a chest range from onboarding: midpoint + ease by fit preference (snug +0 / true +3 / loose +6 cm, +2 if the brand runs slim) → first size that clears the target. *Same chest range → M for a snug-fit user, L for a relaxed-fit user.*
  - Without it: anchors on the user's usual size, reads what that size means in cm *in this brand*, applies the brand's fit note → fitted-vs-relaxed options.
  - Every recommendation carries an honest **confidence level** (high / good / moderate / low) based on input quality.
- **LLM (judgement only):** colour vs skin tone, fabric vs style, value opinion, verdict, one-liner — returned as strict JSON, with a repair guardrail for malformed output. The prompt explicitly forbids the model from recomputing size.

## How the Fit Score works today — honestly

The 0–100 **Fit Score is currently the LLM's holistic judgement**, not a formula. Consequences I've observed and can explain:

- It **clusters at round numbers** (85, 75) — language models gravitate to "nice" scores.
- It's **repetitive for identical inputs** — same profile + same product = same prompt = same groove.
- It's *grounded* (facts and the size decision are injected, not invented) but **not explainable** — you can't ask "why 85 and not 78?"

**This is the top item on the roadmap:** replace the guessed number with a **computed, weighted score in code** — size match + colour-tone match + style match + budget fit — with the LLM only writing the explanation. Same philosophy as sizing: move the score from the "AI vibes" side of the line to the "code with receipts" side. Example target output: `Size 30/30 · Colour 22/25 · Style 18/25 · Budget 15/20 → 85/100`.

---

## What's been tested

- **Multi-user, multi-device:** full journeys from four separate devices/Telegram accounts — four isolated profiles, correct per-user personalisation, **100% onboarding completion** in testing.
- **Bug found and fixed from real usage:** a reset-flow routing bug (an already-onboarded user redoing the questionnaire was greeted instead of asked Question 2) — diagnosed from n8n execution logs, root-caused to routing only checking the permanent `onboarding_done` flag, fixed by also detecting an in-progress session. Shipped within the hour.
- Edge cases: invalid answers re-ask the same question; a product link pasted mid-questionnaire escapes to scoring; unknown products get a graceful "not in catalog" reply.

*(Honest scope note: testing was multi-device by the builder — not yet an organic-user pilot. That pilot is on the roadmap.)*

## Observability

- Every recommendation is logged to a `recommendations` table (user, product, score, verdict).
- **DBeaver** connected to the Postgres instance gives a live spreadsheet view of `users`, `onboarding_sessions`, and `recommendations` — used throughout development to verify state transitions and debug.
- n8n execution logs trace every message through the graph node-by-node (this is how the reset bug was caught).

## Performance (8GB laptop, CPU-only)

| Scenario | Reply time |
|---|---|
| Cold start (model loads) | ~1m 50s |
| Warm (kept alive 30 min) | ~58s |
| Lighter 1.5b model | roughly half |
| GPU/hosted model (production) | 1–2s |

`OLLAMA_KEEP_ALIVE=30m` keeps the model resident between messages; `qwen2.5:1.5b` is installed as a low-memory fallback (a one-word config change).

## Roadmap / how I'd optimise next

1. **Computed Fit Score** (explainable, consistent — replaces LLM-guessed number)
2. **Tappable Telegram buttons** for onboarding (designed; replaces numbered replies)
3. **Real-user pilot** (3–5 people for a week) + drop-off and accuracy metrics
4. **Accuracy eval set** — benchmark size recommendations against known outcomes
5. **ShopPulse** — weekly personalised digest + price-drop alerts (where the budget question earns its keep)
6. **Feedback loop** — 👍/👎 updates the profile; returns-aware sizing per brand per user (the moat a static size chart can never copy)
7. **Cloud deployment** — fixed domain, always-on, GPU inference
8. Real catalog ingestion (feeds/scraping with monitoring)

## Repo contents

```
docker-compose.yml                  # n8n + Postgres
db/01_schema.sql                    # tables: users, product_catalog, brand_size_guides,
db/02_seed.sql                      #   recommendations, onboarding_sessions, wishlists...
db/03_add_onboarding_columns.sql
workflows/FitSense_MainBot_Router.json     # the live bot (19 nodes, 3 lanes)
workflows/FitSense_Onboarding_Phase1.json  # standalone onboarding (superseded, kept as reference)
workflows/FitSense_FitScoring_Phase2.json  # standalone scoring (superseded, kept as reference)
docs/PRD.md                         # retrospective product spec
docs/BUILD_JOURNEY.md               # how it was actually built, decisions & war stories
```

## Credits

- **Product concept & case study:** [Asmit Birthare](https://github.com/asmit1624/fitsense_product_case_study)
- **Engineering MVP (this repo):** Mehul Sharma — built hands-on with AI pair-programming assistance, every architectural decision argued through and owned.

*Questions or feedback welcome — [ADD EMAIL / LINKEDIN].*
