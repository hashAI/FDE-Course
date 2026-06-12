# FDE Story Bank: 6 Frameworks + Worked Examples

> **The rule that separates FDE candidates from everyone else:** End-to-end ownership only. "I contributed to" and "we did" are disqualifying. If you can't own the story completely — from problem to production — find a different story. Interviewers at FDE-caliber companies probe the edges of every story looking for the "we." If they find it, the story collapses.

---

## Why Stories Matter More Than You Think

FDE is a hybrid role. Half the interview is technical, half is behavioral. Companies hire FDEs who can do the technical work AND communicate with customers, navigate ambiguity, and own outcomes. Your stories are the evidence for the non-technical half.

**The failure mode most candidates show:** Generic stories with no specifics. "I led the migration of our legacy system" — OK, what was the legacy system, what specific decisions did you make, what broke along the way, what did you do when it broke? Interviewers are trained to probe for the specific. Vague stories fall apart under probing.

**What makes an FDE story excellent:**
- Technical specifics (not just "built a service" — what service, what tech, what trade-offs)
- Personal ownership (not "we" — what did YOU do)
- Customer impact (not "the system worked better" — the specific measurable outcome)
- A learning (not "it went well" — what would you do differently)

---

## The STAR Framework for FDE Stories

Every story follows STAR. But understand each component at FDE depth:

### Situation (2-3 sentences, no more)
Context that a stranger needs to understand the story. Company type, scale, your role at the time. Don't over-explain. The details that matter will come out in Task and Action.

BAD: "So I was working at a logistics startup in 2022 and our company was going through a lot of growth and we had a bunch of technical debt and we were using this old system that wasn't really working well and management had decided we needed to..."

GOOD: "At my last company, a 3PL serving 200 e-commerce brands, I was the sole backend engineer on the carrier integration team."

### Task (what you specifically owned — not "the team")
What were you personally responsible for? What decision or problem was yours to own? What would have happened if YOU hadn't done your job?

BAD: "We needed to build a new carrier integration."

GOOD: "I was responsible for designing and shipping the FedEx integration end-to-end — from reading their API docs to production deployment — with a hard deadline of 4 weeks because we'd already promised it to our largest customer."

### Action (specific steps YOU took, with technical specifics)
This is the longest part. Be specific. Use real names: AWS Lambda, not "serverless infrastructure." Redis, not "a cache." Walk through your actual decisions and trade-offs. This is where technical depth shows.

BAD: "I built the integration and handled the edge cases and got it deployed."

GOOD: "First I read the FedEx Ship API docs and found that their sandbox environment had three known differences from production that weren't documented anywhere — I had to find those by reading their forum posts. Then I built the label creation endpoint using httpx with exponential backoff because FedEx's API rate-limits inconsistently under load. The edge case that almost killed the deadline was that FedEx requires a specific residential vs. commercial address flag, and our customer's address data didn't have that field — I had to add a geocoding step to infer it, which added 3 days."

### Result (quantified outcomes + one learning)
What measurably changed? What did you learn?

BAD: "The integration shipped on time and the customer was happy."

GOOD: "Shipped on day 27 of 28. The integration now handles 4,000 shipments/day. Error rate under 0.1%. The customer renewed their contract 3 months later and specifically cited the reliability of the carrier integration. What I'd do differently: build the test harness against production API inconsistencies first, not last — I found two prod-only behaviors in the final week that almost pushed the deadline."

---

## Story 1: Ownership — Building Something Complex End-to-End Under Constraints

**What this story must show:**
- You owned the entire thing, start to finish
- You made real technical decisions and trade-offs
- You hit constraints and navigated them, not around them
- It went to production with real users

### Template

**Situation:** [Company type, your role, scale of what you were building]

**Task:** I was responsible for [specific thing], end-to-end, with [hard constraint: timeline, resources, technical limitation].

**Action:**
- How you started: what did you need to understand first?
- The key technical decisions you made and why
- The constraint you hit and how you navigated it (not around it — through it or with a trade-off)
- What almost broke the deadline and how you handled it
- How you got it to production

**Result:** [What shipped], [measurable outcome], [what you'd do differently]

---

### Worked Example: The ShipStation-to-SAP Integration

**Situation:**
At a mid-market logistics company (50 employees, $30M revenue), I was the only backend engineer supporting our operations team. We'd just signed a contract with a new enterprise customer who used SAP as their ERP and needed their orders from ShipStation — where their e-commerce orders landed — to flow automatically into SAP for warehouse processing. They'd given us 6 weeks.

**Task:**
I owned the entire integration: reading the ShipStation webhooks, transforming the data into SAP BAPI format, calling the SAP RFC connector, handling failures, and getting it to production. No prior art internally — we'd never integrated with SAP before.

**Action (detailed):**

Week 1: Discovery. I spent two days reading ShipStation webhook docs and another two days on SAP RFC documentation — which is notoriously bad. The critical discovery: SAP BAPI calls are synchronous and blocking, which means if SAP is slow (it often is), my webhook handler would time out. I decided early to decouple — webhook handler would ack immediately and push to an SQS queue, and a separate worker would process the queue against SAP at SAP's pace.

Week 2-3: The ShipStation side. Built the webhook handler in FastAPI — it validated the payload, extracted the order fields, pushed to SQS, and returned 200 immediately. Added idempotency: ShipStation retries failed webhooks, so I needed to dedup by `order_id`. Used a Redis SET — if the order_id was already in the set, skip processing but return 200 (so ShipStation stops retrying). This was the most important single decision in the whole integration.

Week 3-4: The SAP side. This is where everything got hard. SAP's Python RFC library (`pyrfc`) requires a native SAP NW RFC library installed on the machine — it's not just a pip install. This doesn't work in Lambda. I had three options: (1) run the SAP worker on EC2, (2) containerize with the NW RFC library in Docker on ECS, (3) use SAP's OData REST API instead (newer, no native library needed). The customer's SAP was version 7.5 — their OData API was too limited for what we needed. I chose ECS with a custom Docker image that included the NW RFC library. Added three hours to the setup, saved months of maintenance headaches.

Week 5: Integration hell. Two issues. First: the SAP BAPI expected address fields in German (Strasse, Ort, PLZ) but our data was English-format. Had to write a field mapping layer. Second: SAP returned success codes even when the record failed validation — it would return "success" with a warning message buried in the response. I had to parse the BAPI response warnings to detect silent failures. This is an undocumented SAP behavior that cost me 2 days.

Week 6: Testing and deployment. Built a replay tool so I could run any historical ShipStation order through the integration and verify the SAP output. Found two more edge cases (orders with multiple ship-to addresses, orders with backorder lines). Fixed both. Deployed to production on day 39 of 42.

**Result:**
Integration shipped 3 days ahead of the 6-week deadline. Processing 300 orders/day reliably. Error rate: 0.3% (mostly malformed address data from ShipStation — we added a validation alert for those). The customer went from 4-hour manual batch processing to sub-5-minute order routing. They've been on the integration for 14 months with no outages.

What I'd do differently: I'd have asked for a 2-week sandbox access period with real SAP data before committing to the 6-week timeline. The undocumented BAPI behaviors cost me 2 weeks I didn't have. Now I always ask for extended sandbox time before estimating any ERP integration.

---

### The Three Versions

**30-second version (elevator):**
"I built an end-to-end ShipStation-to-SAP integration in 6 weeks as the sole engineer. The key challenge was that SAP's Python library requires a native binary that doesn't run serverless, so I containerized the whole worker with the native library bundled in Docker on ECS. Also had to handle SAP's silent failure mode — it returns 'success' even when a record fails validation, so I had to parse the warning messages. Shipped 3 days early, 300 orders/day, still running."

**2-minute version (interview):**
Use the full Situation + condensed Action (hit the 3 key decisions: decouple with SQS, idempotency with Redis, ECS for native library) + full Result.

**5-minute version (deep-dive):**
Full story above. Add: questions you'd ask if you were doing it again, how you'd design it differently with a second engineer, what you'd do to make it multi-tenant.

---

## Story 2: Failure + Diagnosis — Something Went Wrong in Production

> **This is the most important story in your bank.** Interviewers weight it more heavily than any other. Why? Because it proves: you can handle adversity, you know how to debug systematically, and you learn from failure. A candidate who doesn't have a real failure story is either lying or hasn't shipped real software.

**What this story must show:**
- Something broke in production (not "we had a bug in development")
- You took ownership of the diagnosis and fix
- You show the actual debugging process (not "I found the bug" — show HOW you found it)
- You changed something to prevent recurrence
- You communicated appropriately throughout

### Template

**Situation:** [What was in production, what broke, what the impact was]

**Task:** I was responsible for diagnosing and resolving [specific incident].

**Action:**
- What you saw first (the symptom)
- How you approached diagnosis (what you checked, in what order, why)
- What you found (the root cause — be specific)
- What you did to fix it (interim mitigation + permanent fix)
- How you communicated during the incident
- What you changed to prevent recurrence

**Result:** [How long the incident lasted], [how you resolved it], [what changed after], [the learning]

---

### Worked Example: The Silent Data Loss Bug

**Situation:**
I was the FDE responsible for a carrier integration platform at a logistics SaaS company. One Tuesday morning, our operations team got a call from a large customer — they had 87 orders that had been "confirmed" in their OMS but never actually sent to the carrier. The orders were 14 hours old. Trucks were waiting at their warehouse with nothing to pick up.

**Task:**
I owned diagnosis and resolution. The customer was losing money in real-time (idle truck costs ~$150/hour, 3 trucks waiting). My manager was being called by the customer's VP of Operations. I had to diagnose, communicate status every 30 minutes, and resolve — in that order.

**Action:**

Step 1: Impact assessment before diving into code (critical — too many engineers skip this). I queried our order tracking table for any orders in "pending carrier booking" status older than 60 minutes. Found 87. All from the same customer, all between 10pm Tuesday and 2am Wednesday. I immediately told my manager: "87 orders, specific customer, specific window, trucks are waiting. Sending you a list of order IDs now."

Step 2: What happened between 10pm and 2am? I checked our SQS queue metrics — dead-letter queue had 87 messages in it. So the messages were processing but failing. I pulled the DLQ messages and checked the error: `FedEx API 401 Unauthorized`.

Step 3: Why 401 after months of working? I checked our secrets rotation schedule — we'd set up automatic API key rotation in AWS Secrets Manager 3 months ago. The key had rotated at 11:58pm. The new key was in Secrets Manager. But our Lambda was caching the old key in memory. Lambda functions cache environment variables for the lifetime of the execution environment — if the function is warm, it uses the cached key. When Secrets Manager rotated, the cached key went stale.

Step 4: Root cause confirmed. Our Lambda was: (1) loading the API key from Secrets Manager once on cold start, (2) caching it in a module-level variable, (3) never refreshing it. When the key rotated, every warm Lambda instance had a stale key. New instances (cold starts) got the new key and worked. But we had enough warm instances that most requests were hitting the stale key.

Step 5: Immediate mitigation. Manually triggered a Lambda redeploy to force all instances cold — they'd all pull the new key on restart. This took 3 minutes. Ordered the DLQ messages replayed. First batch went through immediately. Confirmed with the customer.

Step 6: Permanent fix. Added key refresh logic: instead of caching the key in a module variable, I added a refresh every 15 minutes using a timestamp check. Also added a specific 401 handler that immediately refreshes the key and retries once before failing — so even if the key rotates during a live execution, the next request self-heals.

Step 7: Process change. Added a test to our CI pipeline that rotates the test API key mid-test and verifies the integration handles it gracefully. Added a CloudWatch alarm: if DLQ depth > 0 for more than 5 minutes, page on-call immediately. Previously we had no DLQ alarm.

**Communication throughout:**
- T+0 (first aware): Message to manager with impact summary
- T+15: "Root cause identified: secrets rotation + Lambda key caching. Mitigation in progress."
- T+20: "Mitigation deployed. Replaying 87 orders now."
- T+35: "All 87 orders confirmed with FedEx. Trucks can start. Root cause and permanent fix in the postmortem, will send tonight."
- T+3 hours: Postmortem sent to customer's VP with full timeline, root cause, and what changed

**Result:**
Incident lasted 35 minutes from first page to resolution. All 87 orders processed. Customer was frustrated but appreciated the transparency and rapid resolution. The postmortem communication specifically helped — the VP forwarded it to her CEO as an example of how they want their vendors to operate.

What I'd do differently: I should have caught the secrets rotation risk at design time. Any external API key should have a rotation test in CI. This was a known class of failure and we didn't have protection for it.

---

### The Three Versions

**30-second version:**
"I had an incident where 87 customer orders were silently dropped for 14 hours — trucks were waiting. Root cause: our Lambda was caching API keys in memory and didn't handle secret rotation. I forced a cold restart in 3 minutes, replayed the DLQ, and added key-refresh logic and a DLQ depth alarm. The 35-minute resolution was driven by impact assessment first — knowing exactly which orders were affected before diving into the code."

**2-minute version:**
Situation → T+0 impact assessment → root cause discovery (401 → Secrets Manager rotation → Lambda cache) → 3-minute mitigation → permanent fix → 35 minutes total.

**5-minute version:**
Full story above, including the full debugging chain and communication timeline.

---

## Story 3: Navigating Ambiguity — Incomplete Requirements, Unclear Scope

**What this story must show:**
- The requirements were genuinely unclear (not just "we didn't have a design doc")
- You drove clarity actively rather than waiting for it
- You made explicit assumptions and documented them
- You moved forward without full information
- The outcome was good despite the ambiguity

### Template

**Situation:** [What the project was, why the requirements were unclear, what the stakes were]

**Task:** I was responsible for [specific deliverable] despite [specific ambiguity: conflicting stakeholders, missing info, unclear scope].

**Action:**
- What you did to get clarity (and what you did when clarity wasn't forthcoming)
- The assumptions you made, documented, and got sign-off on
- How you structured the work to minimize the cost of being wrong
- What changed mid-project and how you handled it

**Result:** [What shipped], [how the ambiguity resolved], [the learning]

---

### Worked Example: The "Modernize Our Integration" Project

**Situation:**
A mid-market manufacturer hired us to "modernize their integration between their ERP and their suppliers." That was the entire scope document: one sentence. I was assigned as the sole FDE. There were two internal stakeholders — the VP of Operations (wanted faster supplier confirmations) and the IT Director (wanted to replace their EDI system with REST APIs) — and they disagreed on what "modernize" meant.

**Task:**
I was responsible for delivering something in 8 weeks that made the VP and IT Director both happy, despite their conflicting definitions of success.

**Action:**

Week 1: I ran three separate discovery sessions — one with the VP of Operations, one with the IT Director, one with the actual people who used the integration (two procurement clerks). Their answers were entirely different:

- VP: "We need to know within 4 hours if a supplier accepted a PO, not 48 hours."
- IT Director: "We need to get off EDI, it costs us $3K/month in VAN fees and every change takes our EDI team 6 weeks."
- Procurement clerks: "We just want to stop manually entering confirmation numbers into the ERP when suppliers email them to us."

Three different problems. One project.

Week 1, end: Instead of choosing one, I wrote a one-page document: "Current State Assessment and Proposed Phasing." It described all three problems, their costs, and proposed phasing: Phase 1 (weeks 1-3) targets the procurement clerks' manual entry — it addresses the VP's 4-hour confirmation problem AND reduces EDI volume, satisfying part of the IT Director's goal. This document had a signature line. I sent it to both stakeholders and told them I needed both signatures before week 3 to stay on schedule.

This was a deliberate forcing function. By putting it on paper with a signature requirement, I moved the stakeholder conflict out of my head and into a meeting they had to have together.

They came back 4 days later with a counter-proposal: do all three phases but compress the timeline to 6 weeks (from 8). I pushed back in writing: "I can do Phase 1 in 3 weeks confidently. Phases 2 and 3 together require 8 weeks minimum given the EDI migration complexity. If we compress to 6, we risk shipping Phase 2 with insufficient testing. My recommendation: ship Phase 1 in 3 weeks, evaluate, then plan Phases 2-3 with a new timeline. If the 6-week total is a hard business constraint, I need to reduce scope in Phase 2."

They accepted the phased approach.

Week 2-4: Built Phase 1. A simple supplier portal (no EDI, no ERP write-back) where suppliers could log in and confirm POs. When they confirmed, the system triggered a webhook to the ERP (via a connector I built) that marked the PO as confirmed and emailed the procurement clerk instead of requiring manual entry. Completely avoided the EDI question for Phase 1 — it was out of scope and I'd documented that assumption explicitly.

**Key assumption I documented:** Phase 1 would not reduce EDI VAN fees — it would only reduce manual data entry. IT Director had to sign off that this was acceptable for Phase 1. He did.

Week 4: The IT Director changed his requirements. He'd talked to their EDI provider and found out they had a REST API bridge. Could I use that instead of building a custom connector? This changed my architecture. I evaluated the EDI provider's REST API — it was usable but had a 15-minute polling minimum, which violated the VP's "4-hour confirmation" requirement (she'd actually said she wanted confirmation within 30 minutes). I documented the conflict, asked both stakeholders to resolve it, and gave them a 48-hour deadline. They decided the 15-minute polling was OK and using the EDI provider's API was preferred. I updated the architecture and moved forward.

**Result:**
Phase 1 shipped at end of week 4 (1 week late due to the IT Director's mid-project change — I'd documented this risk). Supplier confirmation time dropped from 48 hours to under 30 minutes. Procurement clerk manual entry eliminated for the 8 suppliers in Phase 1 scope. Phase 2 kicked off with clearer requirements because the Phase 1 experience taught both stakeholders what they actually cared about.

What I'd do differently: I'd have structured the discovery as a joint session with both stakeholders in the room, instead of separate sessions. The conflicting requirements would have surfaced earlier and they'd have resolved them themselves instead of handing me the conflict to manage.

---

### The Three Versions

**30-second version:**
"I was handed an 'integration modernization' project with a one-sentence scope and two stakeholders who disagreed on what it meant. I wrote a one-page phasing document with a signature line as a forcing function — moved the stakeholder conflict out of my head and into a meeting they had to have. Both signed off on phasing, Phase 1 shipped 1 week late (due to a mid-project scope change, which I documented), and confirmation time dropped from 48 hours to 30 minutes."

**2-minute version:**
Situation → three-way stakeholder conflict → forcing function document → signature requirement → conflict resolved → Phase 1 shipped → outcome.

**5-minute version:**
Full story above.

---

## Story 4: Customer Communication — Delivering Bad News or Managing Expectations

**What this story must show:**
- You were the one who had to deliver bad news (not "my manager told them")
- You communicated in business terms, not tech jargon
- You took ownership rather than deflecting or blaming
- You proposed a path forward, not just the problem
- The relationship survived (ideally improved)

### Template

**Situation:** [What the bad news was, who the stakeholder was, what the stakes were]

**Task:** I was responsible for communicating [specific bad news] to [specific stakeholder type].

**Action:**
- What you said first (the acknowledgment)
- How you framed the impact (in business terms)
- What you didn't say (tech jargon, blame, excuses)
- What you proposed as the path forward
- How the conversation went
- What you did after

**Result:** [How the relationship ended up], [what you learned about communication]

---

### Worked Example: The Black Friday Integration Failure

**Situation:**
I'd built an integration between a DTC brand's Shopify store and their 3PL. We'd tested it thoroughly in staging. On Black Friday — their single highest-revenue day of the year — the integration started dropping orders at 9am. By 11am, 340 orders had been received by Shopify but not sent to the 3PL. The CEO was calling my manager. I was on a plane (it was Thanksgiving weekend). I landed, got the page, and had to call the CEO within 30 minutes.

**Task:**
I was the FDE who built and owned this integration. I had to call the CEO, explain what happened, and give them a path forward — while simultaneously diagnosing and fixing the issue.

**Action:**

The call. I called the CEO directly (not through an intermediary — this was the right call, not an instinct everyone has). My opening:

"[Name], this is [me], I built your integration. I know you have 340 orders that haven't reached your warehouse, and I know today is Black Friday. I'm going to walk you through exactly what happened and what we're doing about it, and I'm going to stay on this until every one of those orders is in your warehouse's queue. First — are your warehouse staff standing by or do I need to give you a time estimate so you can make a staffing decision?"

That last question was deliberate. CEOs making operational decisions need to know the ETA so they can staff appropriately. I led with what they needed to make their decision, not with my debugging process.

What happened: my 3PL integration used their SFTP endpoint. Their SFTP had a rate limit of 10 concurrent connections — something not in their documentation, but something I should have tested for. On Black Friday, 340 orders hit in 90 minutes, our connection pool maxed out, connections started failing silently, and orders went to our dead-letter queue. I'd set up a DLQ alarm — but the alarm threshold was "5 messages per hour" and the queue filled faster than the alarm could fire.

What I told the CEO (paraphrased): "Here's what happened: your 3PL's SFTP has a connection limit that's not documented, and your Black Friday volume exceeded it. Every one of your 340 orders is safe — they're in our retry queue and haven't been lost. I need 45 minutes to process them in batches that respect the connection limit. After that, your warehouse will have all 340 orders. I'm also changing the alarm threshold so this can't happen again today, and I'll fix the root cause over the weekend. I'll send you a status update every 15 minutes until it's done."

What I did NOT say:
- "Their documentation should have mentioned the rate limit" (blame shifting)
- "This is because of the architecture we had to use" (excuse)
- "There was a race condition in the connection pool management" (tech jargon)
- "I need to check with my team" (I owned it)

Result: 340 orders processed in 47 minutes. CEO got a status at 15 minutes, 30 minutes, and 47 minutes. On Saturday, I sent a written postmortem: what happened, what I fixed, what I'd change architecturally, and a commitment to test the rate limit behavior before the next major sale.

**Outcome:**
The CEO sent a reply: "This is exactly the kind of communication I want from a vendor. Most people would have disappeared. You stayed on it and kept me informed. The fact that it happened is a problem, the way you handled it is why we're renewing the contract."

What I'd do differently: I would have load-tested against production volume before Black Friday. Black Friday volumes are predictable — there's no excuse for not having tested at 10x normal load.

---

### The Three Versions

**30-second version:**
"Integration dropped 340 orders on Black Friday. I called the CEO directly, told them every order was safe in our retry queue, asked the specific question they needed answered for a staffing decision, gave them 15-minute status updates, and had all 340 orders in their warehouse in 47 minutes. The CEO renewed the contract specifically citing the communication."

**2-minute version:**
Situation → landing, getting the page → calling the CEO directly → leading with what they needed (ETA for staffing) → explaining impact without jargon → processing 340 orders in 47 minutes → postmortem → renewal.

**5-minute version:**
Full story above including the exact language used on the call.

---

## Story 5: Hard-Truth Pushback — Telling a Stakeholder They're Wrong

**What this story must show:**
- You disagreed with a customer or internal stakeholder on something substantive
- You pushed back clearly and constructively (not passive-aggressively)
- You were right (this matters — pushing back and being wrong is a different story)
- You preserved the relationship
- You had data or reasoning to support your position

### Template

**Situation:** [What the disagreement was about, who the other party was, what was at stake]

**Task:** I needed to push back on [specific thing] because [specific reason I believed it was wrong].

**Action:**
- How you framed the pushback (constructive, not combative)
- The specific evidence or reasoning you used
- How they responded
- How you handled their response (did they accept? did you escalate? did you find middle ground?)
- What happened

**Result:** [What was decided], [what the outcome was], [what the relationship looks like now]

---

### Worked Example: Pushing Back on the "One-Week MVP"

**Situation:**
I was in a scoping meeting with a VP of Logistics at a mid-market retailer and my company's account executive. The VP had an upcoming board meeting in 10 days and wanted to demonstrate a "live AI-powered shipment tracking dashboard" to the board. The AE had committed — before I was in the room — that we could do it in one week.

**Task:**
I had to tell the VP and the AE that one week was not possible for what they described, in a way that didn't crater the deal and didn't throw my AE under the bus.

**Action:**

I didn't push back in the meeting immediately. I asked a few clarifying questions first to confirm my concerns: "When you say 'live AI-powered tracking dashboard,' what specifically do you need to see in the demo? Is it tracking data for your current shipments, or synthetic data is OK for the board?" And: "How many carriers are we talking about?"

Their answers: real data from their top 3 carriers (FedEx, UPS, and a regional carrier named Eastern Freight), AI predicting late deliveries, and a dashboard the board could interact with live.

After the meeting (not in front of the VP), I pulled the AE aside: "I want to help you close this deal. A working dashboard with real data from 3 carriers in one week is not something I can do reliably — FedEx integration alone takes 3-4 days for the API, and we don't have a production key yet. If the demo fails in front of the board, we lose the deal anyway. Here's what I can do: a demo environment with real FedEx data only, synthetic data for UPS and Eastern, and a basic ETA prediction that we pre-compute. That's genuinely impressive, looks live, and I can build it in one week. After the board meeting, we do the full build in 4 more weeks with real data from all three carriers. Can we bring that proposal back to the VP?"

We brought it back together. I said to the VP: "I want to make sure what we demo to your board actually works. One week for three full carrier integrations with live AI is aggressive — if we rush it, we risk the dashboard failing during the demo itself, which would be worse than a polished demo with one carrier live and two in progress. Here's what I'd propose instead: [described the one-carrier live + synthetic approach]. Your board sees a working product with real FedEx data and ETA predictions. We finish the other two carriers in the following 4 weeks. What matters most to you for the board demo?"

The VP thought for a moment: "I want real data — doesn't have to be all three carriers. If FedEx is live and the other two are noted as 'in progress with go-live in 4 weeks,' that's fine."

I had my real constraints and the AE was now aligned.

**Result:**
Demo delivered in 7 days (one day over my estimate because the FedEx sandbox had a 48-hour activation delay I hadn't known about). Board demo worked. VP was promoted to SVP three months later and greenlit the full platform project. The AE told me afterward: "That's the right way to do it. I'll never commit a timeline without you in the room again."

What I'd do differently: I should have been in the scoping meeting before any commitment was made. I've since made it a non-negotiable that I'm present when technical timelines are discussed with customers.

---

### The Three Versions

**30-second version:**
"AE committed a one-week timeline to a VP without me in the room — three carrier integrations plus AI ETA predictions. I pulled the AE aside before going back to the VP, aligned on a realistic scope (one carrier live, two in progress), and re-presented together. VP accepted, demo worked, they became a flagship customer."

**2-minute version:**
Situation → clarifying questions to confirm scope problem → private alignment with AE first → re-presentation to VP with realistic alternative → VP accepts → demo worked.

**5-minute version:**
Full story above.

---

## Story 6: Integration Under Constraints — Technical Constraints, No Documentation, Legacy Systems

**What this story must show:**
- You worked within constraints rather than asking for them to be removed
- You found creative solutions when the clean path was blocked
- You showed technical ingenuity without recklessness
- The integration worked reliably despite the constraints

### Template

**Situation:** [What you were integrating, what the constraint was: no API, bad documentation, legacy system, rate limits, etc.]

**Task:** I was responsible for integrating [X] with [Y] despite [specific constraint].

**Action:**
- How you discovered the full scope of the constraint
- What approaches you considered (show you thought it through, not just did the first thing)
- What you chose and why
- What the hardest technical problem was and how you solved it
- What you built to make the integration maintainable despite the constraint

**Result:** [What shipped], [how the constraint was handled], [what the long-term maintainability looks like]

---

### Worked Example: The EDI-Only 3PL Integration

**Situation:**
A DTC apparel brand had grown to use a new 3PL for their West Coast fulfillment. The 3PL was a 20-year-old family business with excellent operations but completely legacy IT. Their only integration option was EDI over SFTP — X12 850 for purchase orders, X12 856 for advance ship notices, X12 997 for acknowledgments. Our customer had a Shopify store and a modern REST API-based OMS. They'd never touched EDI in their life.

**Task:**
I was responsible for getting Shopify orders into the 3PL's EDI system and getting shipment confirmations back into their OMS. Zero EDI expertise on my side at the start of the project.

**Action:**

Week 1: Research. I spent three days learning EDI — specifically X12 format. EDI is not a protocol, it's a message format standard. A 850 file is a fixed-format flat text file with segment delimiters. It looks like this:

```
ISA*00*          *00*          *ZZ*SENDER         *ZZ*RECEIVER       *220101*1200*U*00401*000000001*0*P*>~
GS*PO*SENDER*RECEIVER*20220101*1200*1*X*004010~
ST*850*0001~
BEG*00*SA*PO-12345**20220101~
...
```

I found the X12 850 specification ($200 from the official standards body — worth it), studied it for two days, and wrote my own Python library to generate compliant 850 files from JSON order data. I deliberately chose not to use an off-the-shelf EDI library — they're often overkill and the learning curve is similar. My library covered only what this 3PL required.

Week 2: Discovery with the 3PL's IT person (singular — they had one IT person). He gave me their EDI requirements document — 20 pages, PDF, last updated 2008. Most of it was still accurate. Three segments they required were not standard X12 — they were custom extensions. I spent two days reverse-engineering their expected format by looking at examples of EDI files from their other customers (they had samples on file). I built a parser for their response files (the 856 ASNs they'd send back) using the same approach.

Week 2-3: The SFTP connector. Their SFTP server had two quirks: (1) it expected files in a specific naming format (the file name had to encode the date, a sequence number, and the trading partner ID in a specific format), and (2) it had an undocumented 100KB file size limit per transaction. Most orders were well under 100KB, but orders with 20+ line items were not. I had to implement file splitting logic — if an 850 file exceeded 80KB (I used 80% of the limit), split it into multiple files with sequential numbering. This was not in any documentation; I found it by uploading a large test file and getting a silent rejection.

Week 3-4: Error handling and monitoring. The hardest part of EDI is error handling. The 3PL's system would send back 997 acknowledgments — but a 997 with a rejection code could come back anywhere from 5 minutes to 4 hours after you sent the 850. I couldn't use a synchronous success/failure model. I built a state machine:

- `SENT`: file uploaded to SFTP
- `ACKNOWLEDGED`: got a 997 back
- `ACCEPTED`: 997 had acceptance code
- `REJECTED`: 997 had rejection code → trigger alert, manual review queue
- `SHIPPED`: got the 856 ASN back (the 3PL confirming shipment)

Each state transition triggered an appropriate action and alert.

Week 4: Testing. I built a mock 3PL server that replicated their SFTP behavior and EDI format, so I could test the integration without hitting their (very slow, sometimes unreliable) test server. This was the single most valuable investment in the project — I could run full integration tests in CI in under a minute.

**Result:**
Integration went live in 5 weeks. Processing 150 orders/day reliably. Error rate on EDI generation: 0.2% (malformed data from the OMS). 856 ASN processing (turning 3PL confirmations into OMS tracking updates) works with an average 2-hour lag — that's the 3PL's processing speed, not ours.

Three months later, the 3PL upgraded to a newer SFTP server with slightly different behavior. Because I'd built a mock server that accurately replicated their behavior, the upgrade took 2 days to adapt instead of starting from scratch.

What I'd do differently: I would have asked for an actual 3PL-approved EDI test partner earlier. Reverse-engineering their format from a 2008 document was manageable but risky. In hindsight, the $500 test VAN account they offered would have saved me 4 days.

---

### The Three Versions

**30-second version:**
"I built an EDI integration for a customer connecting their modern Shopify OMS to a legacy 3PL with no REST API — X12 850/856/997 over SFTP. Learned EDI from scratch, reverse-engineered their custom segment extensions from example files, built a state machine to handle 4-hour async acknowledgment windows, and built a mock 3PL server for CI testing. Live in 5 weeks, 150 orders/day, still running."

**2-minute version:**
Situation → learning EDI in 3 days → reverse-engineering custom extensions → discovering the undocumented file size limit → state machine for async acknowledgments → mock server for CI → live in 5 weeks.

**5-minute version:**
Full story above.

---

## Building and Maintaining Your Story Bank

### The bank structure

Maintain these six stories in a document. For each, write the 2-minute version word-for-word. Know the 30-second version by heart. Have the 5-minute version ready for follow-ups.

### Rotating your stories

As you do more projects (like the capstone), add them to the bank. The capstone project from this course (LogiSync AI) is your primary technical ownership story and likely your primary integration-under-constraints story.

### Handling "tell me about a time you..." questions

Map each question type to a story:

| Question | Story |
|----------|-------|
| Tell me about a time you built something complex | Story 1: Ownership |
| Tell me about a time something went wrong | Story 2: Failure |
| Tell me about a time you had unclear requirements | Story 3: Ambiguity |
| Tell me about a time you had a difficult customer conversation | Story 4: Communication |
| Tell me about a time you disagreed with a stakeholder | Story 5: Pushback |
| Tell me about a time you worked around technical constraints | Story 6: Constraints |
| Tell me about a time you showed initiative | Story 1: Ownership |
| Tell me about a time you had to deliver bad news | Story 4: Communication |
| Tell me about a time you failed | Story 2: Failure |

### The follow-up trap

The hardest follow-up in behavioral interviews: **"What would you do differently?"**

This is where candidates either:
(a) Say "nothing, it went well" — which signals you don't learn from experience
(b) Give a vague non-answer — "I'd communicate more"
(c) Give an honest, specific answer — this is what they're looking for

For every story, have a specific, honest "would do differently":

- Story 1: "I'd have asked for 2 weeks of sandbox time before committing to the timeline."
- Story 2: "I'd have load-tested the rate limit behavior before Black Friday. That's a known class of failure."
- Story 3: "I'd have run joint discovery sessions instead of separate ones — forced the stakeholder conflict to surface early."
- Story 4: "I'd have set up the DLQ alarm with a much lower threshold — 5 messages, not 50."
- Story 5: "I should have been in the scoping meeting before any commitment was made."
- Story 6: "I'd have paid $500 for the test VAN account and saved 4 days of reverse-engineering."

These "would do differently" answers are gold. They show learning, they show honesty, and they show the kind of judgment that prevents the same mistake twice. Interviewers hire people who improve.

### The "we" audit

Read your stories out loud. Every time you say "we," stop. Replace it with "I" or restructure the sentence. If you genuinely can't own the action — find a different story. 

The test: could your interviewers call your former manager and ask "did [your name] own this completely?" and the answer would be yes? If yes, you're clear. If the manager would say "well, the team did it," you don't have full ownership and the story will fall apart.

---

## Quick Story Construction Checklist

Before you tell any story in an interview, run it through this checklist in your head:

- [ ] Is this 100% mine? (no "we")
- [ ] Do I have a specific technical detail that proves I actually did it?
- [ ] Do I have a specific number in the result? (not "it improved performance" but "it went from 4 hours to 5 minutes")
- [ ] Do I have an honest "what I'd do differently"?
- [ ] Can I tell it in 30 seconds without losing the point?
- [ ] Is the customer impact clear? (what changed for a real person?)

If you can't check all six boxes, keep practicing until you can.
