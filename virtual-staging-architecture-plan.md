# Virtual Staging Platform — Product Deep Dive, MVP & Architecture Build Plan

**Client:** Wedgewood Homes (wedgewoodhomes.com) · **Prepared:** August 2026
**Sizing basis:** 300–500 finished images/month · small team (1–3 devs) · hosted model APIs

---

## 1. Executive Summary

Wedgewood Homes pays ~$300K/year to an offshore vendor for virtual staging and just fired them over quality. At their scale (~2,500–3,000 homes sold/year via the flip arm of Wedgewood Inc., with in-house design by Maverick Design), that spend implies roughly $25–40 per staged image. The same volume run through 2026-era image-model APIs costs **under $300/month in raw model spend** — a >99% cost reduction. The entire product problem is therefore not cost, it is **quality control and style fidelity**: matching Maverick Design's signature style (warm transitional / organic-modern, responsive to each home's architecture), keeping furniture consistent across photos of the same room, never distorting architecture, and staying compliant with the new disclosure laws (California AB 723, effective Jan 1, 2026 — directly relevant since California is Wedgewood's largest market).

The recommendation: a **single-tenant-first, multi-tenant-ready web app** built by a small team on hosted APIs. Core engine is a **hybrid model pipeline** — Google's Nano Banana Pro / Nano Banana 2 (Gemini image models) for one-shot full-room staging with style-reference conditioning, FLUX.1 Fill + SAM segmentation for pixel-exact "re-roll just this sofa" edits, and a fidelity upscaler to MLS resolution — wrapped in a **human review queue** (every image approved by a designer before export) and a **style library** that automatically harvests furniture and style references from Maverick's existing staged photography. MVP in ~8 weeks.

---

## 2. Product Deep Dive: Virtual Staging in 2026

### 2.1 The market

Virtual staging is a ~$1.3B market (2026), growing 13–18% CAGR, with AI-based staging now over 60% of volume. Buyers are listing agents (largest segment), real estate photographers (who resell staging at ~74% margin), and — Wedgewood's category — **flippers/investors**, who use it to list faster and stage rooms physical staging doesn't cover.

**Human-done staging** (the category Wedgewood's fired vendor was in): BoxBrownie $24–32/image at 48h turnaround; PadStyler $79; Virtually Staging Properties $39–60 at 2 business days; roOomy $49–69; offshore providers like Styldod's human service at $16–23. Universal complaints: limited furniture libraries, inconsistent style when work is spread across pooled offshore editors, slow 12–24h revision loops, fake-looking results. This complaint list maps almost exactly onto why Wedgewood fired their vendor.

**AI staging products**: Virtual Staging AI (acquired by Zillow, Oct 2024 — now powers Zillow Showcase's buyer-facing staging) at $0.53–2.67/photo with ~10–15s renders; Collov AI ($23M Series A, Apr 2026) at ~$0.24/photo with a white-label API; REimagineHome/Styldod (distributed free through CRMLS — every California MLS user gets 30 free credits/month); plus API-first players like InstantDeco ($0.05–0.16/image at volume) and VirtualStaging.ai ($0.28/image with multi-angle reference support).

### 2.2 Why quality fails, and what "quality" means

Documented AI failure modes — the reasons agents still distrust these tools:

1. **Cross-photo inconsistency** — each photo staged independently, so the living room has a different sofa in every angle. This is the #1 complaint (Inman, Apr 2026) and the headline differentiator of the newest entrants (Edensign "Multi-View", Stageflow "True Multi-View").
2. **Geometry/scale errors** — warped door frames, floating furniture, eight-seat sectionals in small rooms, wrong shadows.
3. **Architecture hallucination** — AI adding windows or features that don't exist ("housefishing"); NY Dept. of State has warned this can violate deceptive-advertising law.
4. **Style genericism** — one-size "modern" presets that don't match a brand's design language or the home's architecture. Maverick stages a midcentury home as midcentury; generic tools can't.

**Compliance is now part of quality.** California AB 723 (effective Jan 1, 2026) requires: written disclosure of digitally altered listing images, a visible on-image label ("Virtually Staged"), and the original unaltered image presented adjacent to the staged one, everywhere the image appears. CRMLS enforces this today. NAR Article 12 requires virtually staged photos to be "clearly identified as such." MLS fines run $500–$5,000/listing. For a California-centric flipper, the tool must produce compliant output pairs by default — this is a feature competitors mostly bolt on, and a reason an in-house tool is safer than ad-hoc AI use by listing agents.

### 2.3 Wedgewood specifics

- **Scale:** ~15,000 properties sold 2021–2025 (~3,000/yr); operations in 20+ states, concentrated in California; typical property 3bd/2ba, 1,200–2,000 sq ft; hold time 80–240 days. Listings go to MLS via cooperating local agents plus their affiliate brokerage (Wedgewood Homes Realty).
- **Design identity:** Maverick Design (in-house studio since 2014, led by CCO Jessica Sommer) does physical staging with custom furniture sets per home. Style: "tastefully creative and invitingly livable" — warm neutrals, white base palettes, walnut/olive accents, organic-modern materials (fluted cabinetry, Zellige tile, wood-slat detail), with the style adapted to each home's architecture. There is no public style guide, but their portfolio, trend reports, and shoppable product lines are a de facto one — and the raw material for the style library described below.
- **The implied volume check:** 300–500 staged images/month ≈ 60–100 listings/month getting 5 staged shots each — consistent with a subset of their ~200–250 monthly sales. Fits your sizing input.

### 2.4 Cost model (why this is a no-brainer)

Per finished image, worst case with a quality-first pipeline: 3 staging candidates on Nano Banana Pro at 2K ($0.134 × 3 = $0.40) + two object-level re-rolls via FLUX Fill ($0.05 × 2 = $0.10) + auto-QC vision calls (~$0.01) + fidelity upscale (~$0.02) ≈ **$0.55/image**. At 500 images/month: **~$275/month model spend**, or ~$3,300/year — against $300,000/year today. Even adding a part-time internal reviewer's cost, the payback is measured in days. This also means you can afford to be extravagant with quality: generate 3–4 candidates per photo and let the reviewer pick, re-roll freely, run automated QC on everything.

---

## 3. MVP Definition

### 3.1 Product shape

A web app used by Maverick Design staff / listing coordinators (single tenant), architected so tenancy is a column, not a rewrite — org-scoped data model from day one, so selling it to other flippers later is a business decision, not an engineering project. Every image passes a human review gate before export (this is the direct answer to "quality is absolutely important").

### 3.2 MVP feature set

**In scope:**

1. **Listing workspace** — create a listing, batch-upload its photo set (20–40 photos), auto-classification of room type (bedroom, living, kitchen…) and staging suitability via a vision model call.
2. **Style profiles** — named profiles (e.g., "Maverick Organic Modern," "Maverick Midcentury") each defined by: 3–10 curated reference photos of Maverick's real staged rooms, a distilled style prompt (palette, materials, furniture language), and per-room-type overrides. Staging jobs condition on the profile's reference images (Nano Banana Pro supports dedicated style-reference inputs). This is the "match specific styles of home images that I provide" requirement.
3. **Staging generation** — per photo: optional defurnish/declutter pass, then 3 candidate renders using the style profile. Reviewer picks a candidate or re-rolls all.
4. **Object-level re-render ("lock and re-roll")** — click any furniture item; SAM segments it; regenerate only that object via masked inpainting (FLUX.1 Fill), with everything else pixel-locked. Also supports "swap in this specific item from the library." This is the "easy tooling to re-render specific furniture options" requirement — and it's a genuine differentiator: most competitors only offer full-image re-rolls or chat edits.
5. **Furniture & style library** — automatic harvesting: when Maverick's physical-staging photos (and approved AI renders) are ingested, a detection + segmentation pass extracts furniture/decor crops, tags them (type, style, palette) via a vision model, embeds them for search, and files them into the library. Library items can then be passed as object references into new stagings ("use this exact armchair"). This is the "library of options that it pulls from other images automatically" requirement.
6. **Same-room consistency** — once one angle of a room is approved, its render is attached as a reference to the other angles of that room so the same furniture set appears across the photo set.
7. **Review queue** — approve / reject / re-roll / edit per image; before/after slider; side-by-side candidates; comment threads for a second reviewer.
8. **Auto-QC guardrails** — (a) pixel-diff between original and render outside the furniture mask to catch architecture drift, with auto re-composite of original pixels where safe; (b) a VLM judge scoring each candidate on scale/perspective/style-match/artifacts, surfacing scores in the review UI and auto-flagging failures.
9. **Compliant export** — final images upscaled to source resolution (3000–4000px), "Virtually Staged" watermark applied per MLS convention, exported as an AB 723-ready pair (staged + original adjacent), packaged per listing (ZIP + shareable gallery link).

**Explicitly out of scope for MVP** (fast follows): multi-tenant billing/self-serve signup, MLS/API integrations for direct publishing, video/virtual-tour staging, exterior/landscaping edits, renovation visualization (pre-reno "what it will look like" marketing — a very natural v2 for a flipper), mobile app, Matterport/360 staging.

### 3.3 Success criteria for MVP pilot

- ≥85% of photos approved within 2 generation rounds (candidate pick or one re-roll).
- Maverick design lead blind-rates AI output ≥ fired vendor's output on 20-listing sample.
- Zero architecture-alteration incidents reaching export (auto-QC + human gate).
- Cost per finished image < $1.00; turnaround per listing < 1 hour including review.

---

## 4. Model Selection

### 4.1 The landscape (Aug 2026), abridged

| Model | Strengths | Weaknesses for staging | Cost/image |
|---|---|---|---|
| **Nano Banana Pro** (`gemini-3-pro-image`) | Best one-shot editing realism; 3 dedicated style refs + 6 object refs; native 1K/2K/4K output | No hard mask — regenerates whole frame; tone/crop drift on repeated edits; slower | $0.134 (2K) / $0.24 (4K), batch 50% off |
| **Nano Banana 2** (`gemini-3.1-flash-image`) | 10 object refs; 4K; cheaper/faster than Pro | Same no-mask limitation | $0.067 (1K) – $0.151 (4K) |
| **GPT Image 1.5 / 2** | Up to 16 refs; cheap | Mask is soft guidance only — pixels outside mask drift; endemic and unfixed per OpenAI's own forum. Disqualifying for architecture-locked edits | $0.03–0.13 |
| **FLUX.2 [pro] / Kontext** | Cheap, fast (<2–7s); multi-ref editing; LoRA fine-tuning for style lock | ~4MP max (needs upscale); realism a notch below NB Pro on hard scenes | $0.025–0.08 |
| **FLUX.1 Fill [pro]** | **True hard-mask inpainting** — pixel-exact outside mask | Local edits only, not full-scene staging | $0.05 |
| **Qwen-Image-Edit-2509** | Apache 2.0, depth/edge control built in | Self-host path — not for this MVP (hosted-API constraint) | ~$0.02–0.03 hosted |
| **Seedream 4.5** | 4MP, 10 refs, strong retention | ~1 min latency; no hard mask; ByteDance sourcing may be a bad look for a client that just fired a Chinese vendor | $0.04 |

### 4.2 Recommendation: hybrid pipeline, orchestrated per stage

- **Full-room staging: Nano Banana Pro** (primary), with **Nano Banana 2** as the cheap/fast candidate-generator and **FLUX.2 [pro]** as fallback/second opinion. Rationale: NB Pro's style-reference conditioning is exactly the style-matching mechanism the product needs, its native 4K output nearly eliminates the upscaling step for many shots, and independent reviews rank it the best editing model available. All Gemini outputs carry invisible SynthID watermarking — actually *helpful* for AB 723 auditability.
- **Object-level re-rolls & fixes: SAM (hosted on fal/Replicate) for segmentation + FLUX.1 Fill [pro] for masked inpainting.** This pair is what makes "re-render just the sofa, change nothing else" a hard guarantee rather than a hope. Where a full-frame model must be used, the pipeline re-composites original pixels outside the furniture mask.
- **Defurnish/declutter: NB Pro instruction edit**, QC'd by the pixel-diff guard (defurnishing is where surface-smearing artifacts happen).
- **Upscaling to MLS resolution: fidelity-first upscaler** (Topaz API or Clarity/Real-ESRGAN on fal at low creativity) only when output < source resolution. Never generative upscaling at high creativity — it hallucinates architecture.
- **Vision/QC/tagging: Gemini 3 (multimodal)** for room classification, furniture tagging, and the VLM quality judge.
- **Style lock, v2 option:** if profile-based prompting + style refs ever proves insufficient, train a **FLUX.2 LoRA on Maverick's staged portfolio** (hosted training on fal — still no GPU ops). Park this as a lever, don't start with it.

Every model call goes through an internal **provider-adapter layer** so models are config, not code. This market moves every 6 months; the November 2025 releases (NB Pro, FLUX.2) obsoleted the mid-2025 stack. The adapter (plus a golden-set eval harness, §6.5) is what keeps you current.

---

## 5. System Architecture

### 5.1 High-level diagram

```
┌────────────────────────────────────────────────────────────────────┐
│  FRONTEND — Next.js (Vercel)                                       │
│  Listing workspace · Review queue · Mask editor · Style library    │
│  Compare slider · Exports                                          │
└──────────────┬─────────────────────────────────────────────────────┘
               │ HTTPS (REST + realtime subscriptions)
┌──────────────▼─────────────────────────────────────────────────────┐
│  BACKEND — Supabase                                                │
│  Postgres (+pgvector) · Auth (SSO) · Storage (originals/renders)   │
│  Row-level security keyed on org_id (multi-tenant-ready)           │
│  Realtime: job status → UI                                         │
└──────────────┬─────────────────────────────────────────────────────┘
               │ job events
┌──────────────▼─────────────────────────────────────────────────────┐
│  PIPELINE ORCHESTRATOR — durable job runner (Trigger.dev/Inngest)  │
│  ingest → classify → (defurnish) → stage ×3 → auto-QC → review     │
│  gate → consistency pass → upscale → watermark → export            │
│  Retries, idempotency, per-provider rate limiting, cost metering   │
└───────┬───────────────┬───────────────┬────────────────────────────┘
        │               │               │
┌───────▼──────┐ ┌──────▼───────┐ ┌─────▼─────────┐
│ Gemini API   │ │ fal.ai       │ │ Upscaler      │
│ NB Pro / NB2 │ │ FLUX.2, FLUX │ │ Topaz/Clarity │
│ Gemini 3 QC  │ │ Fill, SAM    │ │               │
└──────────────┘ └──────────────┘ └───────────────┘
```

**Stack rationale (small team, hosted everything):** Next.js + Vercel and Supabase give auth, storage, database, and realtime with zero ops. The one piece a naive build gets wrong is the pipeline: image jobs are long-running (30s–3min), multi-step, and failure-prone (provider rate limits, content-filter refusals, timeouts). Don't run them in serverless request handlers — use a durable job orchestrator (Trigger.dev or Inngest; both are hosted, integrate with Vercel, and give retries/steps/observability out of the box). Total infra bill at this volume: low hundreds of dollars/month.

### 5.2 Frontend architecture

Next.js (App Router) + TypeScript + Tailwind/shadcn. Key surfaces, in build order:

1. **Listing workspace** — grid of a listing's photos with status chips (unstaged / generating / needs review / approved / exported); batch actions ("stage all bedrooms with profile X").
2. **Review screen** — the heart of the app. Large before/after slider (drag divider), 3-candidate filmstrip, auto-QC score badges, approve/reject/re-roll, keyboard-driven (reviewers process hundreds of images — J/K navigation, A to approve, R to re-roll).
3. **Mask editor** — click-to-segment: click a sofa, SAM returns the mask, user refines with brush if needed, then "re-roll this item" (prompt or library pick). Canvas layer (Konva or plain canvas) over the render.
4. **Style library** — profile manager (reference photos, style prompt, per-room overrides) and the furniture browser (search by text/similarity, filter by type/room/palette, "pin to profile").
5. **Export view** — watermark preview, AB 723 pair layout, per-listing download.

State: TanStack Query against the API, Supabase Realtime channel for job progress (photos flip from "generating" to "needs review" live). Images served via Supabase Storage CDN with signed URLs; renders displayed as web-size derivatives, full-res only at export.

### 5.3 Backend & data model

Postgres schema (all tables carry `org_id` — tenancy-ready from day one):

```
orgs(id, name, settings)
users(id, org_id, role: admin|designer|reviewer|viewer)
listings(id, org_id, address, market, mls_id, status)
photos(id, listing_id, storage_key_original, room_type, room_group_id,
       width, height, exif_stripped_at, stageable, status)
style_profiles(id, org_id, name, style_prompt, per_room_overrides jsonb)
style_refs(id, profile_id, storage_key, room_type, weight)
render_jobs(id, photo_id, profile_id, kind: defurnish|stage|reroll|
            object_edit|upscale, params jsonb, provider, model,
            status, cost_usd, error, idempotency_key)
renders(id, job_id, photo_id, storage_key, candidate_idx,
        qc jsonb {arch_diff, vlm_scores}, state: candidate|approved|
        rejected|exported, approved_by, parent_render_id)
library_items(id, org_id, kind: furniture|decor, source_photo_id,
              crop_key, mask_key, tags jsonb, embedding vector(768))
room_groups(id, listing_id, label)        -- photos of the same room
exports(id, listing_id, manifest jsonb, zip_key, watermark_spec)
audit_log(id, org_id, actor, action, subject, at)  -- AB 723 audit trail
```

Notes: `room_group_id` is what powers cross-photo consistency (all photos of "primary bedroom" share a group; the first approved render becomes the group's furniture reference). `renders.parent_render_id` gives full lineage — every object re-roll chains to its parent, so you can always reconstruct how an exported image was made (compliance + debugging). `library_items.embedding` (pgvector) powers similarity search — no separate vector DB needed at this scale. Originals are immutable and retained forever (AB 723 requires the unaltered original to remain available).

### 5.4 API design (internal REST, versioned — becomes the customer API later)

```
POST /v1/listings                          create listing
POST /v1/listings/:id/photos              batch upload (presigned)
POST /v1/photos/:id/stage                 {profile_id, candidates: 3, defurnish: bool}
POST /v1/renders/:id/reroll               full re-roll with feedback prompt
POST /v1/renders/:id/segment              {point|box} → SAM mask
POST /v1/renders/:id/object-edit          {mask_id, prompt | library_item_id}
POST /v1/renders/:id/approve | reject
POST /v1/room-groups/:id/propagate        stage remaining angles from approved ref
POST /v1/listings/:id/export              {watermark: true, ab723_pairs: true}
GET  /v1/library/items?query=&room=&similar_to=
POST /v1/library/harvest                  {photo_ids[]} → extraction jobs
```

All generation endpoints enqueue orchestrator runs and return job IDs; the UI listens on realtime channels. Idempotency keys on every job-creating call.

### 5.5 The image pipeline (the core IP)

Per photo, the orchestrator runs:

```
1. INGEST      strip EXIF/GPS → store original → web derivative
2. CLASSIFY    Gemini vision: room type, occupied/empty, stageable?,
               suggest room_group by visual overlap
3. DEFURNISH   (occupied homes only) NB Pro edit → pixel-diff guard
4. STAGE ×3    NB Pro, inputs: photo + profile style refs (3) +
               room-type prompt + room_group furniture refs (if any
               angle already approved) → 3 candidates
               (candidate #3 optionally from FLUX.2 for diversity)
5. AUTO-QC     a) furniture mask via SAM on each candidate;
                  pixel-diff vs original OUTSIDE mask; if drift >
                  threshold → auto re-composite original pixels or
                  flag/discard candidate
               b) VLM judge: scale ✓ perspective ✓ shadows ✓ style-
                  match-to-refs score ✓ artifact scan → qc jsonb
6. REVIEW      human gate: pick/approve/re-roll/object-edit
   ↳ OBJECT-EDIT loop: SAM mask → FLUX.1 Fill inpaint → merge →
     back to review (architecture pixel-locked by construction)
7. CONSISTENCY on approval of a group's first render: harvest its
               furniture crops → attach as object refs → stage
               sibling angles → same review flow
8. FINALIZE    upscale to source res if needed (fidelity mode) →
               "Virtually Staged" watermark burn-in → AB 723 pair →
               export manifest + ZIP + gallery link
9. HARVEST     approved renders + any uploaded Maverick physical-
               staging photos → Grounded-SAM detect/segment furniture
               → Gemini tags → embed → library_items
```

The two guardrails in step 5 are the difference between this tool and the products that get agents fined: **(a)** makes "we never altered the architecture" a *verifiable, logged property* of every export, not a model behavior you hope for; **(b)** encodes Maverick's taste as a scored rubric so reviewers triage instead of inspecting from scratch.

### 5.6 Scale, reliability, cost

At 500 images/month (~25/business day, bursty), scale is trivial — the design constraints are provider rate limits (Gemini image tiers are strict at low spend tiers; batch API where latency allows for 50% cost savings), retry hygiene, and cost metering (every `render_jobs` row records actual provider cost; a dashboard sums by listing/month — the "we replaced $300K with $3K" chart writes itself). Availability target is business-hours; a stuck provider fails over to the secondary via the adapter layer. Monitoring: orchestrator-native observability + Sentry; alert on QC-failure-rate spikes (early warning that a provider silently changed model behavior).

---

## 6. Build Plan

### 6.1 Phase 0 — Style bench & model bake-off (Week 0–1) ⚠️ do this first

Before writing app code: collect 20–30 real Wedgewood listing photos (empty rooms, mix of room types + architecture styles) and 15–20 of Maverick's best physical-staging photos as style references. Build a throwaway eval harness (scripts, not app) that runs the same photo through NB Pro, NB2, and FLUX.2 with style refs, and produces a comparison grid. **Have Maverick's design lead score the grid.** This locks the primary model, the prompt templates, and the style-profile format with evidence — and produces the demo that sells the client. It also becomes the permanent regression suite (§6.5).

### 6.2 Phase 1 — Core pipeline + walking-skeleton UI (Weeks 1–3)

Supabase schema + auth, upload flow, orchestrator wired to Gemini + fal adapters, stage-×3 pipeline (steps 1–4), minimal review screen (candidates, approve, re-roll, before/after slider). *Exit: a coordinator can upload a listing and approve staged images end-to-end.*

### 6.3 Phase 2 — Quality tooling (Weeks 3–5)

Auto-QC (pixel-diff guard + VLM judge), SAM click-to-segment + FLUX Fill object re-roll with mask editor, style-profile manager, defurnish flow, keyboard-driven review queue. *Exit: reviewer can fix any single item without a full re-roll; QC scores visible.*

### 6.4 Phase 3 — Library, consistency, compliance (Weeks 5–6)

Furniture harvesting pipeline + library browser + "use this item" as object ref; room-group consistency propagation; watermarking, AB 723 export pairs, per-listing ZIP/gallery; audit log. *Exit: full feature set of §3.2.*

### 6.5 Phase 4 — Pilot & hardening (Weeks 6–8)

Run 15–20 real listings in parallel with whatever stopgap Wedgewood is using. Tune prompts/thresholds against reviewer behavior (every reject/re-roll is labeled training signal — log it). Stand up the golden-set regression harness as a CI job so any model/prompt change is scored before deploy. Measure the §3.3 success criteria. *Exit: Wedgewood cuts over.*

### 6.6 Team & running costs

Buildable by 2 engineers (1 full-stack, 1 pipeline/prompt-focused) + Maverick design lead as part-time taste authority; 1 strong full-stack can do it in ~10–12 weeks instead. Running costs at 500 img/mo: model APIs ~$300, Vercel + Supabase + orchestrator ~$150–300, total **well under $1K/month** vs $25K/month today.

### 6.7 Risks & mitigations

| Risk | Mitigation |
|---|---|
| Style match isn't good enough via refs+prompting | Escalation ladder: more/better curated refs → per-room prompt tuning → FLUX.2 LoRA trained on Maverick portfolio (hosted on fal, ~days of work) |
| Provider model churn/deprecation | Adapter layer + golden-set regression harness; treat model choice as config |
| Gemini rate limits at low spend tiers | Request tier upgrade early; batch API; FLUX.2 failover |
| Cross-photo consistency still imperfect | Reference-propagation covers most cases; keep worst cases to 1 hero shot per room (standard industry practice); Edensign-style 3D multi-view is a v2 research item |
| Compliance drift (new state laws) | AB 723 pattern (label + original pair + audit trail) is the strictest current standard; audit_log + immutable originals future-proof most variants |
| Reviewer bottleneck | Keyboard-first queue + QC pre-triage; measure approvals/hour in pilot; auto-approve-above-QC-threshold is a dial you can turn later, not a rewrite |

---

## 7. Beyond MVP (roadmap candidates)

Pre-renovation visualization ("this is what it will look like finished") — uniquely valuable to a flipper marketing homes mid-reno; renovation-adjacent edits (day-to-dusk, lawn repair — with compliance labeling); multi-tenant SaaS launch using the same org-scoped schema; direct MLS/photographer-workflow integrations via the already-versioned API; video walkthrough staging; a "Maverick style pack" as a marketable asset in itself.

---

## Appendix A — Key sources

Market/pricing: instantinteriorai.com 2026 stats; photoup.net company comparison; HousingWire virtual staging apps guide; Inman (Zillow–Virtual Staging AI acquisition, Oct 2024; multi-angle consistency, Apr 2026); TechCrunch (Virtual Staging AI profile); Zillow Showcase staging PR (Sep 2025); collov.ai/pricing & /API; virtualstagingai.app/pricing; instantdeco.ai/api.
Compliance: CRMLS AB 723 FAQ; CA leginfo AB 723; NAR Styled Staged Sold (Article 12); roomstage.ai MLS compliance guide.
Wedgewood: wedgewood-inc.com/about; wedgewoodhomes.com; maverickdesign.com; SFR Analytics Wedgewood overview; PRNewswire (Maverick Design debut).
Models: ai.google.dev Gemini image docs & pricing; docs.bfl.ml pricing (FLUX.2, Kontext, Fill); fal.ai model pages (FLUX, SAM, Seedream 4.5); OpenAI community forum (gpt-image mask limitations); huggingface.co/Qwen/Qwen-Image-Edit-2509; simonwillison.net (Nano Banana Pro review).
