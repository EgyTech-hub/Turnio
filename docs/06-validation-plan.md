# 06 — Validation Plan

Version 0.2 · 2026-10-14 · Launch market: Egypt (D-1). Candidate areas are TBD (D-1b).

> **Changelog v0.2** (audit round 2; see [05 §9](05-design-review.md)):
> - Customer interviews are adults-only.
> - The owner question about advance booking no longer assumes customers book ahead.
> - The concierge test (A4) is gated on initial legal advice.
> - Raw research notes stay outside Git.
> - Evidence added for D-30…D-38.
> - Hosting (D-23), privacy law (D-17), children's data (D-24), launch city (D-1b), maps (D-14), and notification provider (D-8) remain open. This plan makes no claim of legal compliance.

## 0. Principles

- **Primary question:** do Egyptian barbershops actually need *one* workflow that combines walk-ins, remote queue joining, and appointments? Or do they mostly need just one of these?
- **Agree thresholds before fieldwork.** The accept/reject criteria below are proposals. The product owner signs them off *before* the first interview, so results cannot be reinterpreted afterwards.
- **Ask about past behavior, not opinions about our idea.** Don't pitch the product before the discovery part of an interview. Ask "tell me about the last time…", not "would you use…".
- **Observe as well as ask.** People misreport frequency and waiting times. Site visits and the concierge test supply behavioral evidence.
- **Privacy in research too.**
  - Get verbal consent before recording.
  - Don't photograph customers.
  - Store notes without customer names.
  - Delete raw recordings after synthesis, within 60 days.
  - **Keep raw notes, recordings, and participant contact details outside Git**, in access-controlled team storage. Only anonymized synthesis goes into the repository (`docs/research/`).
  - Optionally add a `.gitignore` entry for a local raw-notes folder as a safety net. This is recommended but not yet applied; the `.gitignore` change awaits approval.
  - Research activities follow the same PDPL caution as the product. Nothing in this plan asserts that the research or the platform is legally compliant.

## 1. Hypotheses

| ID | Hypothesis | Decisions affected |
|---|---|---|
| **H1** | A meaningful share of target shops serve **both walk-ins and pre-arranged customers on the same chairs on the same day**. Today they coordinate this informally (WhatsApp, calls, memory). | Core MVP scope, phase order (03 §1.3), D-4 |
| **H2** | During peak hours customers **wait in the shop long enough to care** and would rather wait elsewhere. | Remote queue value, D-3 |
| **H3** | Customers who join remotely **arrive within a window** when they get a notification. | D-3, BR-6, remote-join default on/off |
| **H4** | Staff will **record call / start / complete** on a device during busy periods. | ETA feasibility (ASSUMPTION-8), staff UX, D-16, D-29 |
| **H5** | Prices or durations for the same service **differ by barber** within a shop. | D-12 UI exposure, D-26 |
| **H6** | WhatsApp is the expected channel for shop ↔ customer updates; customers will accept an app or web link for live status. | D-8, D-15 |
| **H7** | Most payments are cash at the shop; online deposits are not expected by shops or customers now. | D-7 |
| **H8** | Shops will pay a monthly subscription for this after a free pilot. | D-9 |
| **H9** | Many shops open past midnight and/or change hours seasonally (e.g., Ramadan). | D-22, schedule model |
| **H10** | Customers choose a barbershop mainly by proximity, the barber they know, and expected wait, rather than by price or ratings. | Discovery ranking, FR-DIS |

## 2. Activities

### A1. Owner / manager interviews (priority 1)

| | |
|---|---|
| **Tests** | H1, H4, H5, H7, H8, H9; also D-27, D-28, D-29 |
| **Participants** | 15–20 owners or managers in 2–3 candidate areas. Mix by size: 5× 1–2 chairs, 8× 3–6 chairs, 3× multi-branch. Exclude friends and family. Include at least 3 shops that already use some booking tool. |
| **Format** | 30–40 min, in person at the shop during a quiet period. Arabic. Two people: one asks, one takes notes. |
| **Evidence** | Notes; counts (walk-ins vs pre-arranged per day); photos of any paper or WhatsApp process (with permission, no customer data); tools used today; prices of services by barber. |

**Questions** (non-leading, past behavior):
1. Walk me through yesterday, from opening to closing. Who came in, and how did they get a chair?
2. Does anyone arrange a time with you before coming, or do people just show up? *(Accept "no one does" as a valid answer; don't follow up as if bookings exist.)*
   - If yes: tell me about the last time someone did. How did they reach you? What happened when a walk-in was waiting at that time?
   - If no: how do regulars decide when to come?
3. Tell me about the last time two customers disagreed about who was next. What happened?
4. When a customer phones to ask "how long is the wait?", what do you tell them? How do you work it out?
5. What did the last customer who booked ahead and didn't show up cost you? What did you do?
6. Does every barber here charge the same for a haircut? Does every barber take the same time? *(Ask, don't suggest.)*
7. How do customers usually pay? Has anyone asked to pay before arriving?
8. What time did you close last Thursday? During Ramadan? On Eid?
9. What tools or apps do you use to run the shop today? What did you stop using, and why?
10. Who in the shop holds a phone during work? Is there a shared device?
11. If you could change one thing about how a busy evening goes, what would it be? *(Open; don't mention queues.)*
12. *(At the end, after discovery)* Show a 3-screen prototype of the staff board. Ask what they'd do on each screen; don't ask whether they like it.
13. *(Only after 12)* What do you pay monthly for software or phone credit for the shop today?

**Effect on decisions**
- H1 decides whether the hybrid lane is P0 or whether the MVP focuses on queue-only or booking-only.
- H5 decides whether the per-barber pricing UI ships.
- H9 confirms the D-22 business-day defaults.

**Accept / reject criteria (proposed; sign off before fieldwork)**

| Hypothesis | Accept if | Reject if |
|---|---|---|
| H1 | ≥ 50% of shops report **both** ≥ 1 pre-arranged customer and ≥ 5 walk-ins on a typical peak day, **and** ≥ 3 shops describe a concrete conflict between them in the last month | < 30% handle both. Then the MVP centres on whichever model dominates, and the hybrid lane moves to P1. |
| H4 | ≥ 60% of shops have a device staff can reach while working **and** owners can name who would operate it | < 40%. Then staff UX is redesigned around a single front-desk operator, or ETA accuracy is downgraded. |
| H5 | ≥ 30% of multi-barber shops charge different prices for the same service | Otherwise the per-barber UI stays hidden (the model is kept) |
| H7 | ≥ 80% of revenue is reported as cash | — (informs D-7 only) |
| H8 | ≥ 40% name a current software or communications spend, or say they would replace a paid tool | < 20%. Then rethink the business model before the public launch. |
| H9 | Any shop open past 00:00 confirms D-22 is needed | — |

### A2. Peak-hour shop observation (priority 1)

| | |
|---|---|
| **Tests** | H1, H2, H4, D-27, D-28 (behavior, not opinion) |
| **Participants** | 4–5 shops from A1 that agree. Each gets 2 evenings, 18:00–23:00 or until close, including one Thursday/Friday. |
| **Evidence** (timestamp log on paper or a sheet, with no customer names) | For each arrival: time, walk-in or pre-arranged, party size, waited inside (Y/N), wait minutes, service(s), barber, service minutes, left without service (Y/N). Also: how "next" is decided; device use; interruptions (phone and WhatsApp messages answered per hour). |
| **Effect** | Real duration distributions (seeds ETA defaults and slot granularity); group frequency (D-21); one-at-a-time validation (D-28); load assumptions (ASSUMPTION-11) |
| **Accept / reject** | **H2 accept** if the median in-shop wait at peak is ≥ 20 min and ≥ 10% of arrivals leave or come back later. **Reject** if the median is < 10 min (remote queue becomes low priority). H1 is cross-checked against A1 claims; if observed pre-arranged customers are < ½ of what was claimed, use the observed figure. |

### A3. Customer interviews (priority 2)

| | |
|---|---|
| **Tests** | H2, H3, H6, H10 |
| **Participants** | 20–25 **adults (18+)** who had a haircut in the last 30 days. Recruit outside shops (with the owner's permission) and through personal networks beyond the team's friends. Include ≥ 5 parents who take children; ask the parents, never the children. **Nobody under 18 is interviewed in this initial round.** Confirm age verbally before starting. |
| **Format** | 15–20 min, in person or by phone, in Arabic. |

**Questions**
1. Tell me about your last haircut: how did you decide where and when to go?
2. What happened when you arrived? How long did you wait? What did you do while waiting?
3. Tell me about a time you left a shop without a cut, or went back later. Why?
4. How did you last contact a barbershop before going? (Call, WhatsApp, just walk in.)
5. Have you ever booked a time? What happened when you arrived?
6. What do you usually pay with?
7. *(Parents)* How does it work when you bring your kids?
8. What apps do you use to find local places (food, clinics)? When did you last use one?
9. *(End, after discovery)* Show the "join remotely, get notified" flow. Ask them to narrate what they'd do and what time they'd leave home.

**Evidence:** answers; current channel; any downloaded booking app; reaction to the ETA as a range vs a single number (D-26).

**Accept / reject**
- **H2:** ≥ 50% describe a wait they disliked in the last 3 months.
- **H6:** ≥ 60% name WhatsApp as their channel to shops. If < 30%, D-8 defaults to SMS/push.
- **H10:** the top 3 decision factors are proximity, a known barber, and wait. If price or ratings dominate, discovery ranking changes.
- **H3** is not validated here; stated intentions are weak evidence. See A4.

### A4. Concierge remote-queue test (priority 1 — strongest test of H1+H3)

| | |
|---|---|
| **Tests** | H3, H1, H4, and real ETA accuracy, **without writing product code**. Also gathers evidence for D-34 (does `arrive_by` need to move?), D-36 (do customers announce their own arrival, and does staff confirmation keep up?), and D-38 (converted late appointments). |
| **Gate (must pass before starting)** | **Initial written legal advice (A6)** on collecting and handling real customer phone numbers and messages for the test: who is controller (likely the shop, which runs its own WhatsApp number), the consent wording, the retention period, and whether team members may see the numbers. Without that advice, A4 does not start; substitute a staff-only observation run. |
| **Setup** | 2 shops for 7 days. The shop's existing WhatsApp number advertises "send 'دور' to get in line". A team member at the shop runs a simple spreadsheet queue, sends ETA updates by WhatsApp, and records arrivals. Appointments are taken the same way. |
| **Participants** | Adult shop customers who opt in. A consent message (wording per counsel) on first contact. Phone numbers stay on the shop's device and number. The team's sheet records ticket numbers and timings only, never phone numbers or names. No data is kept after the test beyond anonymized timings. |
| **Evidence** | Remote joins per day; share arriving within window; no-shows; average lateness; ETA error (predicted vs actual call time); owner/staff feedback at the end; conflicts between remote joiners and appointments. |
| **Effect** | D-3 defaults (window size, cooldown); remote join default on/off; ETA method validated (ASSUMPTION-9) |

**Accept / reject**
- **H3 accept:** ≥ 75% of remote joiners arrive within the window. Remote join stays on by default.
- **H3 reject:** < 50%. Remote join becomes opt-in per shop with a mandatory short window, or moves to P1.
- **ETA:** ≥ 70% of predictions within ±10 min (ASSUMPTION-9).
- **Owners:** both owners say they want to continue (behavioral signal: they keep it running after day 7 without our staff).

### A5. Provider and cost research (priority 2)

| Item | Question | Evidence | Decision |
|---|---|---|---|
| WhatsApp Business Platform (via Meta directly or a BSP) | Utility and authentication template prices for Egypt; template approval rules; sender verification time | Official rate card + 2 BSP quotes | D-8 |
| SMS (local aggregators + international) | Price per SMS to Egyptian mobiles; OTP delivery rate; sender ID registration | Quotes; a 100-message delivery test across the major Egyptian mobile networks | D-8, FR-IAM-1 |
| Maps/geocoding | Accuracy on 50 real shop addresses in candidate areas; cost per 1,000 map loads and geocodes | Side-by-side test: Google, Mapbox, an OSM-based provider | D-14 |
| Hosting | In-country hosting options vs regional cloud + PDPL transfer licence (cost, time) | Quotes; counsel input | D-23, ADR-0005 |
| Payments (P1 only) | Local PSP fees, onboarding of small merchants, wallets | Desk research | D-7 (later) |

**Accept:** for each item, one option with a known per-unit cost that fits the D-9 pricing hypothesis at pilot volume.

### A6. Legal consultation (priority 1, parallel)

**Scope:**
- **PDPL obligations for controllers and processors:**
  - licensing/permits and fees;
  - DPO;
  - records of processing;
  - breach procedure (72 h to the regulator; data subjects within 3 working days, as reported);
  - the cross-border transfer licence;
  - marketing licence;
  - minors (Art. 15).
- Consumer Protection Law 181/2018 (price display, cancellation terms).
- Review liability and takedown obligations.
- Terms of service, privacy policy, and the data processing agreement between the platform and businesses: who is controller and who is processor for customer data held by shops (D-5c).

**Evidence:** written memo.

**Decisions:** D-17, D-18, D-20, D-23, D-24, D-25, D-10.

## 3. Sequencing (4–6 weeks, parallel with Phase 0 engineering on synthetic data)

| Week | Activities |
|---|---|
| 1 | Sign off thresholds; recruit; legal brief; provider quote requests |
| 2–3 | A1 interviews; A2 observations; A3 customer interviews |
| 3–4 | A4 concierge test (2 shops × 7 days) **only if the A6 initial legal advice has been received**, otherwise a staff-only observation substitute; A5 results |
| 5 | Synthesis; update 04 decision log; adjust P0 scope; re-sign ADRs |

## 4. Outputs

- Synthesis report: `docs/research/2026-xx-validation-findings.md`, with anonymized data only. Raw notes and recordings never enter Git.
- Updated decision log and assumption statuses (validated / invalidated / still open).
- Duration and arrival distributions to seed ETA defaults and test fixtures.
