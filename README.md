# FarmGateIQ

**A cooperative farmgate price-intelligence & buyer-matching agent for a Nigerian maize cooperative — built as an n8n automation.**

A field officer types what a farmer told them in plain text; the system normalizes it,
summarizes recent maize prices (grounded in public market reports via RAG), matches the
lot to a buyer database, drafts an outreach message, and routes it to the cooperative
manager for approval before anything is sent. Every AI step is logged. All data is
synthetic.

**Live app:** https://farmgateiq-demo-console.vercel.app  ·  **Interactive walkthrough:** https://fisayo-rm.github.io/farm-gate-iq/demo/

---

## See it three ways

1. **Interactive walkthrough (no setup, always works)** — a click-through, narrated tour of the
   whole pipeline:
   **▶ https://fisayo-rm.github.io/farm-gate-iq/demo/**
2. **Live demo console (no setup)** — a guided web app: click one button per step, it runs the
   **real** workflow on our n8n instance, and the proof appears from the live workbook:
   **▶ https://farmgateiq-demo-console.vercel.app**
3. **Run the workflows directly** — use the n8n form links in *Live run* below to trigger each
   workflow yourself and watch the results land in the read-only workbook. No account needed.

**Demo workbook (view-only):** https://docs.google.com/spreadsheets/d/1q2y6vbxwNKHYlZrfSNN6qSmJjjB1Ea_eKhbsPv-d0Dg/edit?usp=sharing

---

## Tech stack

| Layer | Tool |
|---|---|
| Orchestration & forms | n8n Cloud |
| LLM | Claude (Anthropic) |
| RAG vector store | Pinecone + OpenAI embeddings |
| Database & dashboard | Google Sheets |
| Messaging | Gmail (email simulation) |

## What's in this repo

- **`workflows/`** — the n8n workflow exports (`WF-000` … `WF-009`, plus `WF-RESET`). Import the JSON into n8n.
- **`demo/index.html`** — the self-contained interactive walkthrough (open locally or via the Pages link above).

---

## Live run — runs on our instance, no setup

Each form below is hosted on our n8n instance; submitting it runs the real workflow and
writes to the demo workbook. **Open the view-only workbook side by side to watch the proof
appear.** Bag prices are per **100kg bag**.

**Step 1 — Submit a produce lot (WF-001).**
Open: **https://fisayo-dmg.app.n8n.cloud/form/52355cf2-4ab5-433c-97f8-b7a839858983** and paste:
> `I have 120 bags of dry maize in Zaria available from 15 June. Asking 48000 per bag.`

→ Claude extracts the structured fields. **See it:** a new normalized row in `produce_lots`,
and the event in `audit_log`.

**Step 2 — Price summary, RAG-grounded (WF-003).**
Open (or GET) **https://fisayo-dmg.app.n8n.cloud/webhook/wf003-price-summary** to run it (it
takes no input). It computes the cooperative's own min/max/median **in code**, then retrieves
public market context (FEWS NET, NBS, FAO) from Pinecone and has Claude write a **cited**
summary. **See it:** a new `farmer_update` row in `approvals` — the price range, a cited
market line, the flagged outliers, and a confidence level.

**Step 3 — Buyer matching (WF-005).**
Open: **https://fisayo-dmg.app.n8n.cloud/form/cc6fe384-b96c-4647-80ce-39ca1edbf691** and enter lot id `lot_001`.
A fixed rule-based formula scores buyers (the LLM explains, never decides); blocked buyers
are excluded, unverified ones get risk flags, and the draft references each buyer's stated
requirements. **See it:** up to 3 ranked rows in `buyer_matches` and a pending
`buyer_outreach` row in `approvals`.

**Step 4 — Nothing sent yet.** The match step notifies the cooperative manager by email
(that lands in our inbox), but **no buyer has been contacted.** **See it:** the
`buyer_outreach` row in `approvals` still shows `decision = pending`.

**Step 5 — Approve & send (WF-006).**
Open: **https://fisayo-dmg.app.n8n.cloud/form/1de970a9-daa1-47ab-873a-937d21609050**, paste the approval id from the `approvals` row,
and choose `approve`. **See it:** the `approvals` row flips to `approved` (the original draft
is kept beside the final version), and a new `deal_records` row appears as `contacted`. (The
simulated outreach email goes to our inbox, with the intended buyer shown in the body.)

**Step 6 — Buyer response (WF-007).**
Open: **https://fisayo-dmg.app.n8n.cloud/form/e8c6d9ef-d2a7-4f13-8f36-f1a54bb7103e**, enter buyer `buyer_001`, lot `lot_001`, and a
reply like *"Yes, interested — can we inspect on June 18 at Zaria?"* Claude classifies it.
**See it:** `deal_records` advances to `interested` with a recommended next action.
(Suspicious replies are flagged and do **not** advance.)

**Step 7 — Audit trail.** **See it:** `audit_log` shows the full chain — submission →
extraction → price summary → match → approval → message sent → classification — each with
actor, input/output, confidence, and timestamp.

**Step 8 — Failure case (WF-001).** Submit `50 bags corn, good quality, call me` on the
Step-1 form. The system extracts what it can, sets **low confidence**, flags `needs_review`,
and holds the lot out of matching. **See it:** the new `produce_lots` row with empty
location/date/price and `needs_review = TRUE`. Graceful triage, not a hard block.

> Submissions land in a shared workbook, so rows from earlier visitors may already be
> present — look for the newest row at the bottom of each tab.

---

## Notes

- **Synthetic data only** — 20 lots, 15 price entries, 10 buyers; no real personal or
  financial data.
- **Email is simulated** — outbound messages go to the demo inbox with the intended
  recipient shown in the body; in a real pilot that becomes the buyer's address.
- **Human-in-the-loop** — 100% of outbound messages pass through manager approval; the AI
  assists, the human decides.

---

## Reproduce it yourself (optional)

You don't need to — the live run above is on our instance — but to stand up your own copy:

1. Import each `workflows/WF-*.json` into an n8n instance.
2. Create the Google Sheets workbook with these tabs: `produce_lots`, `price_entries`,
   `buyers`, `buyer_matches`, `approvals`, `deal_records`, `audit_log`, `Overview`.
   (Seed data available on request.)
3. Set credentials: Claude (Anthropic), Google Sheets, Gmail, Pinecone, OpenAI embeddings,
   and point each Google Sheets node at your workbook.
4. Run **WF-009** once to index the market-intelligence corpus into Pinecone (the price
   summary retrieves from it).
5. Publish the form-triggered workflows to get their Production form URLs.
6. Use **WF-RESET** to restore the workbook to its starting state between runs.
