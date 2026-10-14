# 01 — Market Research & Competitor Matrix

Version 0.1 · 2026-10-14 · Status: Draft. Research was desk-based. No product trials or customer interviews have happened yet.

Evidence labels: **[V]** vendor-verified · **[S]** secondary source · **[I]** inferred · **[?]** not found · **[R]** our recommendation. See [README](README.md).

---

## 1. Executive summary

1. **The market splits into two camps, and few products serve both well.**
   - **Booking-first marketplaces** (Booksy, Fresha, theCut, SQUIRE) are built around future appointments. Each one also includes discovery, reviews, payments, deposits, and no-show protection.
   - **Queue-first tools** (Waitwhile, Qminder, Great Clips' in-house Online Check-In) are built around walk-ins and live wait times. They have no consumer discovery marketplace and are often priced for enterprises (Qminder starts at $869/month [V]).
   - Barbershops that take walk-ins *and* appointments on the same chairs are served poorly by both camps. This is our main opening. **[R]**
2. **The established norm for reviews is verified visits only.** Booksy [V] and Fresha [V] only accept reviews tied to a completed appointment. We should match this; it is not a differentiator.
3. **Monetization has converged on a cheap subscription plus a marketplace acquisition fee.**
   - Booksy: $29.99/month + $20 per extra staff member. Its optional Boost charges a one-time 30% fee on a new client's first visit, capped at $100 [V].
   - Fresha: per-member subscription plus a one-time, non-refundable new-client fee [V]. The rate is reported at 20% [S].
   - Owners complain that new-client fees get attributed to clients they already had [S, Trustpilot]. **Transparent attribution is a trust opportunity.**
4. **Wait-time estimation is solved simply in practice.**
   - Waitwhile's "classic" mode sums service durations (configured, or historical once there are ≥50 logged visits) and divides by serving capacity and resource availability. An ML mode is optional [V].
   - Great Clips bases estimates on current salon conditions. It assumes the next available stylist, and requesting a specific stylist can lengthen the wait [S].
   - **An MVP does not need ML.** It needs good data capture: actual start and end times per service per barber.
5. **Regional signals (MENA) — relevant if we launch there (decision D-1):**
   - Fresha lists barbers in Saudi Arabia [V].
   - A Saudi/Bahrain app, Barbero, positions itself around queueing [S].
   - At least one Egyptian shop argues that WhatsApp, not apps, is where business conversation happens [S, single source].
   - If the launch market is Egypt or the Gulf, **WhatsApp notifications, phone-number identity, Arabic/RTL, and cash-first payment are probably table stakes.** These are ASSUMPTION-1…4 and must be validated with interviews.

---

## 2. Competitors studied

| Product | Positioning | Primary market | Model |
|---|---|---|---|
| **Booksy** | Booking app + consumer marketplace for barbers/beauty | US, UK, PL, others | Subscription + optional marketplace "Boost" fee + payments |
| **Fresha** | Salon/spa software + marketplace | Global, incl. Gulf | Per-member subscription + marketplace new-client fee + payments + paid add-ons |
| **SQUIRE** | Barbershop operating system (booking, POS, payouts, rent) | US, UK, CA [S] | Subscription + payments take rate + customer booking fee [S] |
| **theCut** | Barber-centric consumer app & discovery | US | Barber subscription (details not researched) [?] |
| **Waitwhile** | Virtual waitlist + appointments, cross-industry | Global | Tiered SaaS by visit volume + locations |
| **Qminder** | Enterprise queue management (gov, retail, healthcare) | Global | High-tier SaaS |
| **Great Clips Online Check-In** | Proprietary chain-wide walk-in queue | US/CA | In-house (not sold) — benchmark for UX only |
| **Barbero / barber.com.sa** | Regional barber discovery/queue | KSA/Bahrain [S] | Unknown [?] |
| **Google Actions Center (Reserve with Google)** | Distribution channel, not a competitor | Global | Partner integration |

Products noted for later research, not studied here: Vagaro, GlossGenius, Square Appointments, Setmore, Vezeeta (Egypt/MENA doctor booking — relevant to the future clinic vertical).

---

## 3. Capability matrix (by product)

Legend inside cells: evidence label + short note. Sources are numbered in §7.

| # | Capability | Booksy | Fresha | SQUIRE | theCut | Waitwhile | Qminder | Great Clips |
|---|---|---|---|---|---|---|---|---|
| 1 | Consumer discovery & search | [V] Marketplace discovery [1] | [V] Marketplace; Riyadh/KSA listings [5][9] | [?] Not found; procurement marketplace only [S][11] | [V] Search by location, compare prices, portfolios [13] | [?] None (business tool) | [?] None | [S] Find nearby salon in own app [15] |
| 2 | Maps / location results | [I] Implied by "local" discovery | [V] City landing pages [9] | [?] | [V] Interactive map, sort by distance (web, 2026) [14] | [?] | [?] | [S] Nearby salons [15] |
| 3 | Business & staff profiles / portfolio | [I] | [I] | [S] Per-barber booking links [11] | [V] Portfolios [13] | [?] | [?] | [?] |
| 4 | Service catalog & pricing | [I] | [V] (core) | [S] | [V] Compare prices [13] | [V] Services API [7] | [S] Visit reasons [S] | [?] |
| 5 | Appointment scheduling | [V] | [V] | [S][11] | [V] One-time/recurring [13] | [V] Bookings module [6] | [V] Appointments **add-on** [8] | n/a |
| 6 | Walk-in / virtual queue | [?] Not found; waitlist = notify on open slot [1] | [?] Waitlist (slot-based) [5] | [S] Kiosk used for walk-ins; waitlist [10][11] | [?] None found [13][14] | [V] Virtual waitlist, kiosk [6] | [V] Name-based waitlist, remote check-in [8] | [S] Online check-in = queue [15] |
| 7 | Queue position & ETA | [?] | [?] | [?] | [?] | [V] Classic (durations ÷ capacity) + ML "WaitIQ" [6][16] | [V] Wait-time threshold alerts; no ETA listed [8] | [S] Estimate from live conditions; ReadyNext text at 15 min [15] |
| 8 | Staff/resource assignment | [I] | [V] Calendar columns per member [5] | [S] Barber schedules, clock-in [11] | [I] | [V] Resources, availability, "resource speed" [7][16] | [S] | [S] Specific stylist request ↑ wait [15] |
| 9 | Reviews & moderation | [V] Completed bookings only; owner reply; client edit ≤30 days [3] | [V] Completed appts only, 1 per appt, 7-day window, automated pre-moderation, appeal, owner reply, no review gating [4] | [?] | [V] Reviews shown [13] | [?] | [?] | [?] |
| 10 | Notifications | [V] Free reminders; 2,000 SMS/mo marketing [2] | [V] Text/WhatsApp/email; paid per message beyond allowance [5] | [S] Email/text reminders [11] | [V] Reminders, in-app messaging (app only) [14] | [V] SMS (credits) [6] | [V] Multilingual SMS + chat [8] | [S] SMS opt-in [15] |
| 11 | Payments / deposits / no-show | [V] Deposits, cancellation fees, trusted-client waiver; 2.49–2.69% + fee [1][2] | [V] Optional payments add-on; deposits/no-show via payments [5] | [S] POS 2.9% + $0.30; $1 customer online fee; no-show auto-charge [11] | [?] | [?] | [?] | [?] |
| 12 | Business analytics | [V] Reporting [2] | [V] Paid "Insights" add-on [5] | [S] Analytics reports [10] | [?] | [V] Single → multi-location → report builder [6] | [V] Staff/location/service reports [8] | n/a |
| 13 | Multi-branch | [?] | [V] "Multiple business locations" row [5] | [S] Multi-location management [10][11] | [?] | [V] Up to 10 / custom [6] | [V] Unlimited locations [8] | chain |
| 14 | Multi-tenant / multi-business | [I] Marketplace of many businesses | [I] Same | [I] | [I] | [I] SaaS | [I] SaaS | n/a |
| 15 | APIs & integrations | [V] Reserve with Google [2] | [V] Xero, "Data Connector" add-on [5] | [?] | [?] | [V] API + webhooks (Business plan+); visits/locations/resources/services/customers/audit [6][7] | [V] API + webhooks (Premium+) [8] | n/a |
| 16 | Monetization | [V] $29.99/mo + $20/staff; Boost 30% capped $100 [2] | [V] Per-member subscription; new-client fee non-refundable [5][12]; 20% rate [S] | [S] $30–$250/mo + payments [11] | [?] | [V] Free / $31 / $55 / custom [6] | [V] $869 / $1,149 / custom per month [8] | n/a |
| 17 | Mobile / web | [V] Business & client apps [1] | [I] | [S] iOS app [10] | [V] App + new web search [14] | [I] | [V] Apple TV display app [8] | [S] iOS/Android [15] |
| 18 | Security / access control | [?] | [V] Verified accounts reduce fraud bookings [S][4] | [?] | [?] | [V] SAML/SSO (Enterprise), audit entries API [6][7] | [V] SAML SSO (Premium+), data-retention config [8] | n/a |

**Key gaps observed (absence = "not found", not proven absent):**
- No studied **consumer marketplace** showed a **live walk-in queue/ETA across many independent shops**. Great Clips does this only inside its own chain. Barbero claims it regionally [S].
- No studied **queue tool** offers **consumer discovery** (Waitwhile and Qminder are B2B-only).

---

## 4. Feature decision table

Columns: Target user · Business value (H/M/L) · Complexity (H/M/L) · Build? · Phase · Evidence (who has it) · Limitations / open questions.

| Feature | User | Value | Cx | Build? | Phase | Who has it | Notes / open questions |
|---|---|---|---|---|---|---|---|
| Geo search (radius, distance sort, list + map) | Customer | H | M | Yes | **P0** | theCut [V], Fresha [V] | PostGIS. Manual location entry must work without device permission. |
| Live status in search results (open / accepting walk-ins / est. wait) | Customer | H | M | Yes | **P0** | Not found in marketplaces | **Differentiator.** Depends on staff keeping status accurate. |
| Business profile, photos, services, prices, staff | Customer | H | L | Yes | **P0** | All marketplaces | Photo moderation needed. |
| Barber portfolio / haircut styles | Customer | M | M | Yes | P1 | theCut [V] | Owned by barber or shop? (D-5) |
| Appointment booking with availability engine | Both | H | H | Yes | **P0** | All booking tools | Concurrency-safe (DB constraint). |
| Multi-service booking (e.g., cut + beard) | Customer | H | M | Yes | **P0** | [I] common | Sequential, same barber, summed duration. |
| Walk-in queue managed by staff | Staff | H | M | Yes | **P0** | Waitwhile, Qminder [V] | Core of hybrid model. |
| Remote virtual queue join | Customer | H | M | Yes | **P0** (per-shop toggle) | Waitwhile, Qminder, Great Clips | Abuse risk: no-shows from remote joins → arrival window, limits (D-3). |
| Hybrid scheduling: appointments + walk-ins on same barber timeline | Both | **H** | **H** | Yes | **P0** | Partial at best [?] | **Primary differentiator**; priority rules (D-4). |
| ETA calculation (rule-based from durations + live state) | Customer | H | M | Yes | **P0** | Waitwhile classic [V] | Show ranges, not false precision. |
| ML ETA | Customer | M | H | Later | P2 | Waitwhile WaitIQ [V] | Needs months of data first. |
| "You're next / X min left" notifications | Customer | H | M | Yes | **P0** | Great Clips 15-min text [S] | Channel cost (D-8). |
| Booking confirmations / reminders | Customer | H | L | Yes | **P0** | All | |
| Slot waitlist (notify when slot opens) | Customer | M | M | Yes | P1 | Booksy, Fresha, SQUIRE | |
| Verified reviews + owner reply + report | Both | H | M | Yes | **P0** | Booksy, Fresha [V] | Window and edit rules (D-10). |
| Review photos | Customer | L | M | Later | P1 | [?] | PII / moderation burden. |
| Deposits / no-show fees | Business | H | H | Later | P1 | Booksy, Fresha, SQUIRE | Requires a local PSP; cash-first markets (D-7). |
| Trusted-client fee waiver / client block | Business | M | L | Yes | P1 | Booksy [V] | Block list is a P0 candidate for abuse control. |
| In-shop kiosk / TV queue display | Business | M | M | Yes | P1 | Waitwhile [V], Qminder [V], SQUIRE [S] | Public display must hide PII (first name + initial). |
| POS, payouts, chair-rent collection | Business | M | H | Later | P2 | SQUIRE [S] | Financial/regulatory scope. |
| Memberships, loyalty, gift cards | Business | M | M | Later | P2 | Booksy [V], Fresha add-on [V] | |
| Analytics (visits, wait, service time, no-shows) | Business | H | M | Basic | **P0** basic / P1 adv. | All | |
| Multi-branch management | Business | M | M | Data model yes, UI basic | **P0** model / P1 UI | Fresha, SQUIRE, Waitwhile | Model it from day one; it is cheap to model and expensive to retrofit. |
| Reserve with Google | Business | M | H | Later | P2 | Booksy [V] | 4–8+ week partner integration; Google approves merchants [V][17]. |
| Public API + webhooks | Business | L (now) | M | Later | P2 | Waitwhile, Qminder [V] | Design internal APIs so this is possible. |
| SSO (SAML) | Enterprise | L | M | Later | P2 | Waitwhile, Qminder [V] | Not relevant to barbers. |
| Transparent marketplace attribution | Business | M | M | Yes | P1 | Gap vs Fresha complaints [S] | Only if we charge acquisition fees (D-9). |

---

## 5. Differentiation opportunities [R]

1. **One timeline per barber covering both appointments and walk-ins.** Appointments are fixed blocks. Walk-ins fill the gaps between them, and the ETA honestly accounts for upcoming appointments. Competitors are either booking-first or queue-first.
2. **Live "wait now" in discovery.** Search results show *open now*, *accepting walk-ins*, *about 25–40 min wait*, and *next bookable slot*. For walk-in-heavy markets this is the decision customers actually make.
3. **Built for the local market (if MENA, D-1):**
   - Arabic/RTL as a first-class language.
   - WhatsApp + SMS notifications.
   - Phone-number OTP identity.
   - Cash-first, with online deposits as an optional upgrade.
   - Prices shown tax-inclusive where that is local practice.
4. **Fair, honest economics.** Pricing is simple. Any acquisition fee shows its attribution openly ("this client came via search on <date>"), with a dispute path. This answers known complaints about competitor fees [S].
5. **Remote queue that resists abuse.**
   - An arrival-window check-in.
   - An optional distance limit when joining.
   - One active queue entry per customer.
   - Private reliability signals visible to businesses.
6. **Barber identity that moves with the barber (D-5).** Barbers move between shops, and customers follow barbers. A portable professional profile is barber-centric (like theCut) and works with a business-centric tenant model.

---

## 6. Risks surfaced by research

| Risk | Evidence | Mitigation |
|---|---|---|
| Shops don't adopt apps; WhatsApp is sufficient | Egyptian shop blog [S][18] | Staff-side value first (queue board works even if customers never install app); WhatsApp notifications; walk-in-only customers handled by staff entry. |
| Marketplace cold start (no shops → no customers) | Inherent | Launch city-by-city with dense supply; business tool must be valuable stand-alone. |
| Incumbent (Fresha) already in Gulf | [V][9] | Differentiate on hybrid queue + local fit; avoid head-on feature parity race. |
| Remote queue no-shows destroy shop trust | Great Clips removes late arrivals [S] | Arrival windows, grace period, auto-expiry, reliability limits. |
| Messaging cost (SMS/WhatsApp) erodes margin | Fresha charges per message beyond allowance [V]; WhatsApp per-message pricing varies by country [S][19] | Push first; WhatsApp/SMS for critical events only; quotas per plan. |

---

## 7. Sources

Accessed 2026-10-14. Prices change; re-verify before any business decision.

1. Booksy — No-Show Protection: https://biz.booksy.com/en-us/features/no-show-protection
2. Booksy — Pricing: https://biz.booksy.com/en-us/pricing
3. Booksy — How Booksy Reviews Work (2024-06-14): https://biz.booksy.com/en-us/blog/how-booksy-reviews-work
4. Fresha — Review Policy: https://terms.fresha.com/review-policy · Ratings overview: https://www.fresha.com/help-center/knowledge-base/clients/103752-ratings-and-reviews-overview
5. Fresha — Pricing (page served in EGP to our request): https://www.fresha.com/pricing
6. Waitwhile — Pricing: https://waitwhile.com/pricing/
7. Waitwhile — Developer docs: https://developers.waitwhile.com/
8. Qminder — Pricing: https://www.qminder.com/pricing/
9. Fresha — Barbers in Saudi Arabia: https://www.fresha.com/lp/en/bt/barbers/in/saudi-arabia
10. SQUIRE — App Store / GetApp: https://apps.apple.com/app/id1254375220 · https://www.getapp.com/barbershop-software/a/squire
11. Contrary Research — SQUIRE (last updated ~Jul 2023, secondary): https://research.contrary.com/company/squire
12. Fresha — Marketplace new client fees: https://www.fresha.com/help-center/knowledge-base/billing-and-fees/101357-marketplace-new-client-fees
13. theCut — Client features: https://thecut.co/client-features
14. theCut — Help: "Finding your next barber just got easier": https://help.thecut.co/en/articles/15544755-finding-your-next-barber-just-got-easier
15. Great Clips — Customer service FAQ / app listing: https://www.greatclips.com/customer-service · https://apps.apple.com/app/id414012602
16. Waitwhile — Wait time estimation: https://help.waitwhile.com/en/articles/6762645-how-wait-time-is-estimated · https://help.waitwhile.com/en/articles/10048839-custom-configuration-wait-time-estimation · https://help.waitwhile.com/en/articles/9980010-intelligent-wait-time-estimation
17. Google Actions Center — Integration overview: https://developers.google.com/actions-center/verticals/local-services/e2e/integration-steps/overview
18. The Barber (New Cairo) — blog (single-shop opinion): https://thebarber-eg.com/en/blog/online-booking-grooming-lounges-egypt
19. Meta — WhatsApp Business Platform pricing: https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing
20. Trustpilot — Fresha reviews (owner complaints on fees, anecdotal): https://www.trustpilot.com/review/fresha.com?page=2
21. Barbero (KSA/Bahrain): https://barbero.codiantdev.com/ · barber.com.sa: https://barber.com.sa/?lang=en

**Conflicts found:**
- SQUIRE pricing: SQUIRE's own blog says $50/month for unlimited barbers, while third parties list $30–$250/month.
- Fresha review window: a secondary source says 30 days, but Fresha's current policy says 7 days. We use the policy page.
- Fresha new-client fee: its pricing page table shows "Free" next to fee text that describes a charge.
