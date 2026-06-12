# FDE Interview Skill #1: Problem Decomposition

> **This is the highest-weighted skill in FDE hiring.** Technical ability gets you in the room. Decomposition determines if you get the offer. Most candidates fail here — not because they lack knowledge, but because they skip the discipline.

---

## What Decomposition Actually Is in an FDE Interview

The interviewer gives you a vague customer problem — deliberately vague — and watches how you think. They are not watching for the right answer. There is no right answer. They are watching:

1. **Do you clarify before proposing?** (Most candidates don't.)
2. **Do you break the problem into logical pieces?** (Or do you treat it as one blob?)
3. **Can you identify the minimum viable thing that proves the value?** (Not "Phase 1" — the actual minimum.)
4. **Do you think in customer-first terms or engineering terms?** (The wrong framing is obvious immediately.)

### What the interviewer is actually scoring

| Dimension | Bad signal | Good signal |
|-----------|-----------|-------------|
| Problem framing | "So I'd build a FastAPI service that..." | "Before I propose anything, help me understand..." |
| Question quality | 20 questions, covering edge cases | 3 targeted questions that unlock the whole problem |
| Decomposition | Treats problem as one blob | Breaks into data flow / systems / users |
| MVP instinct | Phases 1 through 5, detailed | One happy path, end-to-end, in production |
| Iteration logic | Adds features randomly | Prioritizes by customer impact |
| Confidence | Freezes on ambiguity | States assumptions explicitly, moves forward |

### What fails people in this round

**Failure mode 1: Jumping to the solution**
You hear "automate our order-to-cash" and immediately start designing a system. The interviewer watches you build the wrong thing confidently. You don't get the offer.

**Failure mode 2: Asking 20 clarifying questions**
You ask about every edge case before understanding the core problem. The interviewer sees someone who needs complete information before acting — that's not an FDE, that's a spec-driven engineer.

**Failure mode 3: Losing the thread**
You ask good questions, get good answers, then forget what you were trying to understand. The clarifying questions become their own conversation and you never actually decompose anything.

**Failure mode 4: Technology-first framing**
"What's your tech stack?" is a bad first question. "What does success look like for your team?" is a good first question. Technology is downstream of the business problem.

**Failure mode 5: Skipping the success metric**
You can design the perfect system for the wrong outcome. "What does working look like to you?" takes 10 seconds and reorients everything.

---

## The Decomposition Loop

Drill this loop until it is a reflex. It takes about 45 seconds per cycle. Every FDE interview question runs through this loop.

```
CLARIFY → DECOMPOSE → MVP → ITERATE
    ↑                              |
    └──────────────────────────────┘
```

### Phase 1: CLARIFY (2-3 questions, 5-10 minutes)

You are not trying to understand everything. You are trying to understand enough to decompose intelligently. Three questions, not twenty.

**The four question types — pick the most important three:**

**Volume/Scale question:**
"How many [X] per [day/week]? What's peak vs. average?"
This unlocks: whether you're building for 100 orders or 100,000. Changes the architecture completely.

**Current state question:**
"Walk me through what happens today, step by step."
This unlocks: where the actual pain is, what systems exist, what's already working that you shouldn't break.

**Success metric question:**
"What does 'working' look like to you? How would you measure success?"
This unlocks: the real outcome they care about. Often different from the stated problem.

**Constraint question:**
"Are there systems we must integrate with? Anything that can't change?"
This unlocks: the fixed points your design must work around.

**How to ask questions without sounding like an interrogation:**

BAD:
> "Question 1: What is your order volume? Question 2: What systems do you use? Question 3: What is your budget? Question 4: Who are the stakeholders?"

GOOD:
> "Before I start designing, I want to make sure I understand the problem clearly. Can you walk me through what happens today when an order comes in — step by step? And what breaks in that process?"

Then, after they answer:

> "And what would success look like — like, if this is working perfectly 6 months from now, what's measurably different?"

Then one more targeted question based on what they said.

You've asked three questions. You understand the problem. Now decompose.

### Phase 2: DECOMPOSE (break into components, 10-15 minutes)

Every customer problem in logistics/supply chain breaks into three layers. Work through all three.

**Layer 1: Data flow**
- What data moves? (orders, shipments, inventory levels, tracking events, invoices)
- From where? (ERP, e-commerce platform, warehouse system, carrier API, customer)
- To where? (same options, other direction)
- How does it move today? (manual, batch file, API, email)
- What's wrong with that? (latency, errors, missing fields, manual touch points)

**Layer 2: Systems involved**
- What systems does the customer already have?
- What systems do they need to connect?
- Which systems are fixed (must integrate as-is) vs. flexible?
- Where does the integration live? (who owns the contract?)

**Layer 3: Users**
- Who does what, when?
- What's their technical level?
- What's their pain today?
- What do they need to do differently?

**The decomposition output — say it out loud:**

> "OK, so the way I'm seeing this: there are three parts to the problem. First, there's the data problem — orders are sitting in Shopify but your 3PL doesn't know about them until someone exports a CSV manually every two hours. Second, there's the integration problem — your 3PL has an SFTP interface, not an API, so we need something to translate. Third, there's the visibility problem — once the order is at the 3PL, you have no idea what's happening until it ships. Each of these is solvable independently, but the most painful one is the 2-hour delay. Does that match your understanding?"

That last question — "does that match your understanding?" — is critical. You're confirming your decomposition before building on it.

### Phase 3: MVP (the smallest thing that proves the value)

This is where most candidates make a second mistake: they describe a full product roadmap as the MVP. That's a Phase 1, not an MVP.

An MVP has exactly one purpose: prove that the core value hypothesis is true in production, with real data.

**The MVP filter:**
- One happy path, end-to-end
- Real data, not synthetic
- In production (or staging with production data), not demo
- Measurably better than the current state

**How to identify the MVP:**

Ask yourself: "What is the ONE thing — if it works — that proves the rest is worth building?"

Example: customer wants full supply chain visibility.
- Phase 1 thinking: build the tracking dashboard, the alert system, the SLA monitoring, the carrier integration, the reporting
- MVP thinking: get one shipment's tracking events into their system automatically, displayed to one user, with one alert working

The MVP is not "Phase 1 with fewer features." The MVP is the smallest end-to-end thing that works with real data.

**How to say the MVP:**

> "For the MVP, I'd do this: one integration, one carrier, one happy path — an order comes in Shopify, it gets automatically sent to your 3PL's SFTP within 60 seconds, and you get a Slack notification confirming it went through. That's it. Two weeks. Once that's working with real orders, we expand to the other carriers and add error handling. The reason to start there: it proves the core plumbing works and your team gets immediate value on your highest-volume carrier."

Notice what that MVP statement has:
- Concrete scope (one carrier, one happy path)
- Concrete timeline (two weeks)
- Concrete success metric (60 seconds, Slack notification)
- Rationale (proves the plumbing, immediate value)
- Explicit next step (other carriers, error handling)

### Phase 4: ITERATE (what comes next, in what order)

Iteration is not "adding features." It's a prioritized sequence based on customer impact.

**The iteration framework:**
1. What breaks the MVP? (error handling, edge cases)
2. What does the next 20% of users need? (second carrier, second use case)
3. What makes it operationally robust? (monitoring, alerts, retries)
4. What expands the value? (new integrations, new users, reporting)

**How to present iterations:**

> "After the MVP is stable — probably week 3-4 — I'd add: error handling for when the SFTP is down (retry queue, alert), then add your second carrier, then add the reverse direction so your OMS gets shipment confirmations back from the 3PL. After that we have a solid bidirectional sync and can talk about the visibility dashboard."

---

## Worked Example 1: "We want to automate our order-to-cash process"

### The BAD approach

**Interviewer:** "Our CFO wants to automate the order-to-cash process. Can you design that?"

**Candidate (BAD):** "Sure! So order-to-cash covers order management, invoicing, payment processing, and collections. I'd build a workflow engine using Apache Airflow with DAGs for each stage. We'd integrate with Salesforce for order capture, SAP for invoicing, and Stripe for payment processing. The architecture would have a message queue between each stage for reliability, probably SQS, and we'd use Lambda for the processing functions. The whole thing could be containerized in ECS with an RDS backend for state management. We'd need about 3 months to build and it would reduce DSO by about 20%..."

**Why this fails:**
- Zero clarifying questions
- Assumes Salesforce, SAP, Stripe — may be completely wrong
- Assumes the bottleneck is technical automation, not, say, invoice disputes
- Gives a timeline commitment without understanding scope
- The "20% DSO reduction" number came from nowhere
- The interviewer is watching someone build the wrong thing confidently

---

**The GOOD approach:**

**Interviewer:** "Our CFO wants to automate the order-to-cash process. Can you design that?"

**Candidate (GOOD):** "Before I design anything, I want to understand the problem clearly — order-to-cash covers a lot of ground. Can you walk me through what the current process looks like, step by step? And where does it most often break down or create pain?"

**Interviewer:** "Sure. Orders come in from our sales team in Salesforce, then someone manually enters them into SAP, then our billing team generates invoices from SAP about a week after the order closes, then we follow up on payments manually."

**Candidate:** "OK, so there are at least two manual touch points — the Salesforce-to-SAP entry and the invoice timing. When you say the CFO wants to automate this, what does success look like to them? Is it about reducing DSO, reducing headcount, reducing errors in the entry — what's the business driver?"

**Interviewer:** "Mainly reducing DSO. We're sitting at 47 days and the CFO wants to get to 30. The manual invoice generation is the biggest delay — sometimes it takes 10 days after close."

**Candidate:** "Got it. And the Salesforce-to-SAP entry — how many orders per day, and what's the error rate on that manual entry today?"

**Interviewer:** "About 50 orders a day, and we see maybe 3-5 entry errors per week that cause invoice problems."

**Candidate:** "OK, let me make sure I have this right: you have two problems. Problem 1 is invoice latency — invoices go out 7-10 days after close instead of same-day or next-day. That's probably 15-20 of your 47 DSO days. Problem 2 is entry errors that delay payment because the invoice has wrong data. The Salesforce-to-SAP sync is the root cause of both. Does that sound right?"

**Interviewer:** "Yes, exactly."

**Candidate:** "OK. So here's how I'd decompose this. There are three components: the Salesforce-SAP sync, the invoice trigger, and the error handling. For the MVP, I'd focus on just one: auto-generating the SAP order within 5 minutes of a Salesforce opportunity closing as Won. That addresses the latency and eliminates the manual entry errors in one shot. Invoice generation can then be triggered automatically once the SAP order exists. The MVP is: Salesforce webhook → transform → SAP API call → invoice trigger. Two weeks to build, one week to test with real orders. If that works, DSO should drop 10-15 days immediately. Then we add error handling, duplicate detection, and expand to other order types."

**Why this works:**
- Two targeted clarifying questions unlocked the real problem
- The decomposition named the components explicitly
- The MVP is concrete, time-boxed, and tied to the business metric (DSO)
- The iteration plan is logical and sequenced

---

## Worked Example 2: "Our 3PL keeps losing shipments, can AI help?"

### The BAD approach

**Candidate (BAD):** "Absolutely, AI can help with this. I'd build an ML model that classifies shipments by risk level based on historical loss data. We'd use a Random Forest classifier initially, then upgrade to a neural network as we get more data. The training data would be your historical shipments labeled by whether they were lost or delivered, and we'd add features like carrier, route, weight, dimensions, and weather data. The model outputs a risk score and high-risk shipments get additional tracking or insurance. Accuracy should be around 85-90%..."

**Why this fails:**
- "Losing shipments" is completely ambiguous — they could mean literally lost (never delivered), misrouted, not in the system, delayed, stolen — AI is not the right answer for most of these
- No clarifying questions
- Jumps straight to ML before understanding if ML is even the right tool
- 85-90% accuracy number invented on the spot

---

**The GOOD approach:**

**Candidate (GOOD):** "When you say 'losing shipments' — can you describe what that looks like concretely? Like, a shipment goes out and then what happens?"

**Interviewer:** "We send the shipment data to our 3PL, they pick it up, and then sometimes we just... don't hear anything. The customer calls us asking where their order is, and we can't see it in our system."

**Candidate:** "OK, so the shipment physically moves but you lose visibility in your system — you don't have the tracking events feeding back into your OMS. Is that right? Or do shipments actually fail to arrive at customers?"

**Interviewer:** "Mostly the visibility problem. We don't get tracking updates reliably. Sometimes shipments do get lost but that's maybe 2% of volume."

**Candidate:** "And what does your current tracking setup look like — are you polling the carrier's API, getting webhooks, or is your 3PL supposed to be feeding you status updates?"

**Interviewer:** "Our 3PL sends us a daily file with shipment statuses. It's pretty unreliable."

**Candidate:** "Got it. So this isn't an AI problem at all — this is a data pipeline reliability problem. The root cause is that you're dependent on a daily batch file from your 3PL, which is fragile and slow. AI could theoretically help predict which shipments will have visibility gaps, but the better fix is getting real-time carrier tracking directly. Here's how I'd decompose it: First, set up direct carrier webhook or API polling — bypass your 3PL for tracking data. Second, reconcile that against what your 3PL sends so you catch discrepancies. Third, add an alert when a shipment hasn't had a status update in more than 12 hours. That eliminates 95% of your 'lost shipment' calls without any AI. The 2% that actually go missing — that's a carrier claims process problem, and yes, AI could eventually help triage those, but that's not where I'd start."

**Why this works:**
- First question completely reframed the problem (not lost shipments, lost visibility)
- Second question confirmed the framing
- Third question exposed the root cause (daily batch file)
- The candidate explicitly said "this isn't an AI problem" — that's a mark of maturity
- The decomposition is clear and sequenced
- The actual AI use case is correctly deferred

---

## Worked Example 3: "We want to connect our ERP to our carrier APIs"

### The BAD approach

**Candidate (BAD):** "Great, so we'd build an integration layer. I'd use FastAPI for the middleware, with a connector for each carrier — FedEx, UPS, DHL. The ERP would push orders to our API, we'd transform them to the carrier's format, call the carrier API to book the shipment and get a label, and push the tracking number back to the ERP. We'd use Redis for caching and PostgreSQL for state. The whole thing would run on AWS ECS with auto-scaling. Timeline is probably 6-8 weeks for all three carriers."

**Why this fails:**
- "Which ERP?" matters enormously — SAP vs. NetSuite vs. a home-grown system have completely different integration patterns
- Didn't ask which carriers
- Didn't ask what the integration should do (just labels? also rates? also tracking?)
- The 6-8 week estimate is made up
- Didn't ask if this exists today and what's wrong with it

---

**The GOOD approach:**

**Candidate (GOOD):** "Before I design this — a few things I need to understand. First, which ERP are we talking about, and does it have a modern API or are we dealing with EDI or SFTP exports?"

**Interviewer:** "It's NetSuite. It has APIs but they're... complicated."

**Candidate:** "OK, NetSuite — so we can use SuiteScript or the REST API. And which carriers? All of them, or specific ones you need first?"

**Interviewer:** "FedEx and UPS are the most important. We also use a regional carrier for the Midwest but they don't have an API."

**Candidate:** "And what's the scope of the integration? Just shipment booking and label creation? Or also rate shopping, tracking events coming back into NetSuite, something else?"

**Interviewer:** "Booking and labels first. Tracking events back into NetSuite would be nice."

**Candidate:** "And what happens today? Is someone manually logging into the carrier portals and copying tracking numbers back into NetSuite?"

**Interviewer:** "Yes. It takes about 2 hours a day across our team."

**Candidate:** "OK. So the core problem is manual data entry — someone books in FedEx, gets a label, then has to re-enter the tracking number in NetSuite. And the regional carrier has no API so there's no way to eliminate that piece automatically. Let me decompose this: there are three components. NetSuite order extraction — when an order is fulfilled in NetSuite, we need to pull it into the integration. Carrier booking — call FedEx/UPS API with the shipment details, get a label and tracking number. NetSuite write-back — post the tracking number and shipment cost back to the NetSuite sales order. The MVP is: NetSuite webhook or polling → transform order data → FedEx label creation → write tracking number back to NetSuite. One carrier, one happy path, no edge cases. Three weeks. That eliminates the 2 hours a day immediately. Then we add UPS, then the tracking event callbacks, then the rate shopping. The regional carrier — we'd build a manual queue UI that makes the manual process faster and ensures the tracking number still gets into NetSuite."

**Why this works:**
- Three targeted questions revealed the ERP, carriers, scope, and current state
- The manual work was quantified (2 hours/day)
- The MVP is single-carrier, single-direction, concrete scope
- The regional carrier problem was called out explicitly — not ignored, not overpromised
- The iteration plan is logical

---

## Worked Example 4: "We need real-time visibility into our supply chain"

### The BAD approach

**Candidate (BAD):** "Real-time visibility is a great use case. I'd build a control tower application — a centralized dashboard that aggregates data from all your carriers, 3PLs, and suppliers. We'd use a streaming architecture with Kafka for real-time event processing, a React frontend for the dashboard, and TimescaleDB for the time-series data. We could add ML-based ETD predictions and exception alerting. This would typically take 4-6 months to build properly."

**Why this fails:**
- "Real-time visibility" is a product category, not a problem statement
- Building a control tower from scratch when commercial options exist (project44, FourKites, Descartes) is often wrong
- No clarification on what "visibility" means to them
- No clarification on what "real-time" means (every minute? every hour? it matters)
- 4-6 months is a major commitment with no requirements

---

**The GOOD approach:**

**Candidate (GOOD):** "Real-time visibility is a broad term — I want to make sure I understand what you actually need. What's the specific pain you're feeling today? What happens, and who is it happening to?"

**Interviewer:** "Our customer service team gets 200 calls a day asking 'where's my order.' They have to call the carrier, wait on hold, get the update, then call the customer back. It takes forever."

**Candidate:** "So the primary use case is customer service — they need to be able to look up a shipment and immediately see where it is, without calling the carrier. Is the issue that you have no tracking data at all, or that the data exists but isn't accessible to your CS team?"

**Interviewer:** "The data kind of exists — we can log into each carrier's website — but we have 12 carriers and it's a different website for each one."

**Candidate:** "Got it. So it's an access problem, not a data problem. You need a unified interface where a CS rep can type a tracking number or order number and get the current status, regardless of which carrier. What's the volume — how many active shipments at any given time?"

**Interviewer:** "About 5,000 at any given moment."

**Candidate:** "OK. And does this need to be built from scratch, or have you looked at tools like project44 or FourKites? Because what you're describing is exactly what those tools do out of the box, and they have integrations with all 12 carriers already. I want to make sure we're not rebuilding something you can buy."

**Interviewer:** "We looked at project44 — it's $80K a year and we're not ready for that."

**Candidate:** "Understood. So we're building a lighter version. Here's how I'd decompose it: You need three things. First, a carrier integration layer — something that can query each of your 12 carriers' APIs given a tracking number and return a normalized status. Second, a lookup UI — a simple web page where CS reps type an order number and see status. Third, a data layer — store the tracking events so you're not hitting 12 carrier APIs in real-time on every search. For the MVP: two carriers (your highest-volume ones), the lookup UI, and a polling job that updates statuses every 30 minutes. That handles maybe 70% of your CS calls. Then we add carriers one at a time. The 30-minute polling is probably good enough — 'real-time' for your CS use case probably means 'within 30 minutes of the carrier scanning it,' not 'within seconds.'"

**Why this works:**
- The first question reframed from abstract (real-time visibility) to concrete (CS team, 200 calls/day)
- The build vs. buy question was asked explicitly — and answered
- The "real-time" requirement was challenged (is 30 minutes sufficient?)
- The MVP is 70% of the problem, not 100%, and that's OK

---

## Worked Example 5: "Can you build a chatbot for our warehouse staff?"

### The BAD approach

**Candidate (BAD):** "A chatbot for warehouse staff is a great use case for LLMs. I'd use Claude or GPT-4 as the underlying model with RAG on your warehouse documentation. The frontend could be a simple chat interface on tablets distributed across the warehouse floor. We'd embed your SOPs, inventory data, and shift schedules as context. Accuracy should be around 90%..."

**Why this fails:**
- "Chatbot" is a solution, not a problem — why do they want it?
- RAG on warehouse docs might be completely wrong if the real need is task lookup
- Tablets in a warehouse — have you been in a warehouse? The UI requirements are completely different
- 90% accuracy is made up and concerning — 10% wrong answers in a warehouse context could be dangerous

---

**The GOOD approach:**

**Candidate (GOOD):** "Before I design anything — tell me what the chatbot would be used for. Like, what does a warehouse worker actually need to do today that you're hoping the chatbot helps with?"

**Interviewer:** "They're always asking their supervisors questions — like where is this SKU located, what's the pick priority for this order, how do I handle a damaged item return."

**Candidate:** "OK, so three different use cases: inventory location lookup, order pick prioritization, and returns handling guidance. How are they getting those answers today? And how many workers would be using this?"

**Interviewer:** "They either ask their supervisor or walk to a terminal to check the WMS. We have about 80 workers across two shifts."

**Candidate:** "And the WMS — does it have an API, or would the chatbot need to screen-scrape? Because 'where is SKU X located' is a database query — you don't necessarily need LLM for that, you need a fast lookup interface."

**Interviewer:** "The WMS has a pretty bad API. Limited, but it exists."

**Candidate:** "OK. Here's how I'd think about this. You actually have two different types of requests mixed together. The first type — 'where is SKU X?' and 'what's the pick priority for order Y?' — these are database queries. They don't need an LLM. They need a fast mobile interface with a barcode scanner that pulls from the WMS API. An LLM adds latency and hallucination risk to what should be a deterministic lookup. The second type — 'how do I handle a damaged return?' — that's a procedures/policy question where LLM + RAG on your SOPs could genuinely help. For the MVP, I'd separate these: build the WMS lookup interface first (two weeks, no AI, 60 of your 80 workers benefit immediately), then add the SOP chatbot as a separate feature for the policy questions. The interface for warehouse workers needs to be optimized for gloves and poor lighting — large buttons, simple text, minimal typing. What devices are they currently using?"

**Why this works:**
- The first question extracted the actual use cases (three distinct ones)
- The candidate challenged the LLM assumption — correctly identified that lookup questions don't need AI
- The WMS API question was critical — drives the whole architecture
- The form factor question was asked (warehouse context matters enormously)
- The MVP cleanly separates the deterministic and generative parts

---

## Daily Drill Format

This is the most important practice habit in this entire course. Do it every day for two weeks before your interview.

### The daily drill protocol

1. Pick a vague problem (use the list below or make one up)
2. Set a 10-minute timer
3. Go through the loop **out loud, by yourself** (narrate your thinking)
4. Record yourself on your phone
5. Listen back, score yourself on the rubric below
6. Do it again, better

**Why out loud?** Because in the interview you will be narrating your thinking. Silent practice does not build that muscle. The reflex only sets under the conditions you practice in.

**Why record yourself?** Because you will be surprised how you actually sound vs. how you think you sound. "Um" and "like" are invisible until you hear them. Weak questions sound different when you're listening rather than asking.

### Drill problems (use these)

Pull one randomly:

1. "We want to reduce our freight costs by 20%"
2. "Our warehouse picking accuracy is only 98% and we need 99.9%"
3. "We want to give our suppliers visibility into their POs"
4. "Our returns process takes 14 days and customers are angry"
5. "We need to automate our purchase order approval process"
6. "Our inventory forecasting is wrong — we're always over or understocked"
7. "We want to build a customer-facing tracking portal"
8. "Our customs clearance keeps delaying shipments — can we fix it?"
9. "We want to connect our Shopify store to our 3PL automatically"
10. "Our ERP data quality is terrible — wrong addresses, duplicate records"
11. "We need to automate our proof of delivery collection"
12. "Our demand planning team spends all week in Excel — can AI help?"
13. "We want real-time alerts when a shipment will be late"
14. "Can we use AI to improve our carrier selection?"
15. "We need to digitize our paper-based receiving process"

### Self-scoring rubric

After each drill, score yourself 1-5 on each dimension:

| Dimension | 1 | 3 | 5 |
|-----------|---|---|---|
| First move | Jumped straight to solution | Asked one good question | Opened with "help me understand the problem" |
| Question quality | Asked about tech/stack first | Mixed business and tech questions | All questions targeted business pain |
| Problem reframing | Accepted the stated problem | Partially reframed | Correctly identified the real problem behind the stated one |
| Decomposition clarity | No explicit decomposition | Rough decomposition | Named components, data flows, and users explicitly |
| MVP concreteness | "Phase 1 with features" | Concrete but too large | Smallest working end-to-end thing |
| Business framing | Talked in tech terms | Mixed | Tied everything to customer outcome |

Score 25+: Interview-ready on this dimension
Score 15-24: Keep drilling
Score below 15: Read this entire document again, then drill

### The "rubber duck" version of decomposition

Before a practice drill, tell the problem to your rubber duck (or a friend who doesn't know logistics) in 60 seconds. If they can tell you what success looks like after your 60 seconds, you understand the problem. If they can't, you haven't understood it yet.

---

## Common Traps and How to Avoid Them

### Trap 1: Solutioning too early

**The symptom:** Your first sentence is "I'd build..."

**The fix:** Write "CLARIFY" on a sticky note and put it next to your screen. Before every problem, your first words must start with "before I propose anything" or "help me understand."

**What to say instead:** "Before I design anything, I want to make sure I understand the problem clearly. [First targeted question]"

### Trap 2: Asking about technology before the business problem

**The symptom:** "What's your tech stack?" is your first question.

**The problem:** Technology is downstream of the business problem. Asking about it first signals that you're going to engineer a solution rather than solve a business problem.

**The fix:** Make your first two questions about business outcomes, current process, or success metrics. Ask about technology only after you understand what you're trying to accomplish.

### Trap 3: Making scope commitments without understanding constraints

**The symptom:** "We can build this in 4 weeks" before you know what "this" actually is.

**The problem:** Timeline estimates without scope are lies. And in an interview, making up timelines shows poor judgment.

**The fix:** Always scope before estimating. "Once I understand the full scope, I'd estimate 4-6 weeks for the MVP — and that depends on [assumption 1], [assumption 2], and [assumption 3]."

### Trap 4: Skipping the success metric

**The symptom:** You design a system without knowing what success looks like.

**The problem:** You can build a technically perfect system that doesn't address what the customer actually cares about.

**The fix:** Ask "what does success look like to you?" before you design anything. Then tie every design decision to that metric. "The reason I'm proposing this approach is that it directly addresses your goal of getting DSO below 30 days."

### Trap 5: Treating "AI" as the answer before understanding the question

**The symptom:** Customer mentions AI/ML, you immediately start designing an ML system.

**The problem:** AI is a solution technology, not a problem category. Most problems that customers describe as "AI problems" are actually data pipeline problems, process problems, or integration problems where a deterministic solution works better.

**The fix:** Ask "what would this AI system actually do, step by step?" If the answer is a database lookup, build a database lookup. If the answer is genuinely probabilistic or generative, then AI might be right.

### Trap 6: Ignoring the humans in the system

**The symptom:** Your decomposition has systems and data but no people.

**The problem:** Every system has users. Users have constraints: technical literacy, time, context, physical environment (warehouse floors are not office environments). Ignoring users produces unusable systems.

**The fix:** Add a "users" layer to every decomposition. "The warehouse workers using this are on their feet, have gloved hands, and are in a loud environment. That changes every UI decision."

### Trap 7: The "phase 1" trap

**The symptom:** Your MVP is labeled "Phase 1" and has 10 features.

**The problem:** An MVP is not Phase 1. Phase 1 is a scope reduction. An MVP is the minimum end-to-end thing that proves the core value hypothesis.

**The fix:** Ask yourself: "What is the single thing that, if it works, proves everything else is worth building?" That is your MVP. Everything else is iteration.

---

## Advanced: Decomposition Under Pressure

When the interviewer starts pushing back, don't abandon your structure. Use it as an anchor.

**Scenario:** You're mid-decomposition and the interviewer says "but what if the customer needs this in one week?"

**Bad response:** Panic, compress your design, promise something smaller and vague.

**Good response:** "That constraint changes the scope significantly. If one week is a hard deadline, the MVP shrinks to: can we get one data point flowing end-to-end? Not production-ready, but a proof of concept with real data that we iterate on. The risk is that a one-week build has no error handling and will need significant hardening in week 2-3. Is the one-week goal a business requirement or an arbitrary deadline?"

That last question is important. Sometimes constraints are real, sometimes they're tests. An FDE pushes back on constraints that look arbitrary — respectfully.

**Scenario:** The interviewer plays a customer who keeps changing requirements mid-decomposition.

**Bad response:** Accept every change, try to design for everything.

**Good response:** "I want to make sure we don't lose the thread here. We started with [original problem]. You've added [new requirement] and [new requirement]. Those are all valid, but if I try to solve all three at once, we won't have anything working in 4 weeks. Can we agree on which one is most painful today, build that, and add the others in sequence?"

This is what a real FDE does. They protect the customer from scope creep — including the customer's own scope creep.

---

## Summary: The Reflex You're Building

By the time you walk into the interview, the decomposition loop should be a reflex. Not a procedure you look up. A reflex.

The test: when someone says a vague thing to you — in the interview, at dinner, anywhere — your first instinct should be to ask "what does that actually mean and what does success look like?"

That reflex takes 40 hours of practice to build. Start today.

```
Every problem → CLARIFY (3 questions)
             → DECOMPOSE (data, systems, users)  
             → MVP (smallest end-to-end working thing)
             → ITERATE (sequenced by customer impact)
```

That's it. Everything else in the interview flows from this.
