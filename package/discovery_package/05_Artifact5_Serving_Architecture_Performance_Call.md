# Artifact #5: Agent Serving Architecture, Performance & Production Readiness Discovery Call

## Session Metadata

* **Session Title:** Cymbal Bank — Production Serving Architecture, Runtime Performance, FinOps & Security Gate Review
* **Participants:**
  * **Vikram Patel** — VP of Platform Engineering & Site Reliability (Cymbal Bank)
  * **Dana Hughes** — Director of Cloud FinOps & IT Economics (Cymbal Bank)
  * **Sam Lin** — Chief Information Security Officer / Principal Security Architect (Cymbal Bank)
  * **Marcus Vance** — VP of Global Compliance Operations (Cymbal Bank)
  * **Lead Customer Engineer (Facilitator)**

---

## Verbatim Discovery Transcript

**[00:00:15] Lead Customer Engineer:**
Thanks everyone for joining this follow-up technical session. In our earlier sessions, we aligned on the functional domain rules—settlement disruptions, FFIEC BSA/AML CDD hold legality, correspondent payment re-routing, and Reg E / Reg Z statutory compensation rules. Today, Vikram, Dana, and Sam wanted to dive into how this autonomous agent will actually run in production: the serving architecture, how we make it fast and scalable under core-banking outage load, how we keep inference and compute costs under control, how it orchestrates tool calls, and how we lock down the serving perimeter. Vikram, do you want to kick us off?

**[00:01:02] Vikram Patel (VP of Platform Engineering & SRE):**
Gladly. Look, building a prototype compliance agent that works on a developer's laptop for a single analyst is easy. Running an enterprise regulatory compliance and dispute agent in production during a systemic clearing freeze is a completely different beast. My first major concern is **responsive, real-time interaction**. When an impacted customer is standing at a branch kiosk or chatting on our mobile banking app while settlement cutoffs are closing, or when a COC dispatcher is holding a high-value clearing batch waiting for an FFIEC CDD legality ruling, the system cannot feel sluggish.

**[00:01:48] Marcus Vance (VP of Global Compliance Operations):**
Amen to that. If an account holder or a branch specialist asks a question and sits staring at a spinning wheel while the agent silently ponders three different federal regulations and four backend core-banking systems before spitting out a wall of text, they're going to abandon the app and get right back into the contact center phone queue. They need immediate visual feedback that the system is working and streaming answers back in real time.

**[00:02:20] Vikram Patel (VP of Platform Engineering & SRE):**
Exactly. To achieve **low-latency inference**, our serving architecture has to attack delay at every layer. First, we cannot afford infrastructure cold-start penalties when a sudden clearing outage hits a regional hub and customers open the mobile banking app all at once. Second, not every turn in a conversation requires our heaviest, deepest reasoning model. If a customer just says, *"What is the status of my wire transfer?"* or *"Thanks, confirm that fee waiver,"* routing that through a massive reasoning model is pure latency waste. We need a way to dynamically tier or route requests so simple intents take a fast path, while complex multi-regulation Reg E and Reg Z disputes use deeper reasoning. And third, the agent spends way too much time repeatedly digesting the exact same massive regulatory manuals and deposit policy documents on every single turn. If hundreds of account holders on the same delayed settlement batch are asking about their options, why should the model re-process those huge policy preambles from scratch every single time?

**[00:03:35] Lead Customer Engineer:**
That connects directly to **throughput maximization** as well. Vikram, how does your team think about traffic patterns between normal "steady-state" banking days and major hub clearing disruptions?

**[00:03:52] Vikram Patel (VP of Platform Engineering & SRE):**
Banking compliance and dispute traffic isn't a smooth curve; it's a step-function shockwave. On a normal afternoon, traffic is steady and modest. The moment a regional grid brownout or an authentication gateway fault freezes a major clearing hub, every account holder across dozens of delayed settlement batches hits the mobile app at the exact same instant. Our serving layer has to absorb massive, sudden concurrency spikes without dropping connections or suffering from head-of-line blocking where one slow, multi-step regulatory investigation starves out dozens of quick dispute status checks. We also worry about hitting model inference capacity ceilings or regional quota exhaustion when a surge hits. How do we architect the serving layer and model endpoints so we maximize concurrent throughput, distribute load resiliently, and gracefully prioritize interactive traffic over background batch work?

**[00:04:55] Dana Hughes (Director of Cloud FinOps & IT Economics):**
And let me jump in right there, because Vikram's "always-ready, high-throughput" wish list is usually what blows up my quarterly cloud budget! From a **cost optimization** standpoint, I have three non-negotiable concerns.
First, I refuse to pay for peak outage-level compute and dedicated inference capacity sitting idle during the ninety percent of the week when clearing rails are running smoothly. The serving architecture must scale down economically during off-peak hours while remaining responsive when demand surges.
Second, token and inference consumption can spiral out of control if we aren't disciplined. If an agent gets stuck in a reasoning loop, or if we feed bloated conversation histories, massive system instructions, and huge raw database dumps into the model on every turn, our unit cost per resolved compliance case will destroy the business case.
Third, if thousands of customers ask substantially identical questions during the same executive fee-waiver event, we shouldn't be paying for redundant full-price model evaluations over and over. I expect the CE's proposal to detail concrete architectural methods for **cost optimization** across compute elasticity, context/prompt reuse, semantic or response caching, and model routing.

**[00:06:22] Lead Customer Engineer:**
That balance between low latency, high throughput, and FinOps discipline is critical. Now let's talk about what happens *inside* an agent turn when it has to reach out to Cymbal Bank's internal systems—**optimizing for tool interaction**. Vikram, where do your current integrations bog down?

**[00:06:45] Vikram Patel (VP of Platform Engineering & SRE):**
Honestly, tool interaction is our biggest hidden bottleneck. To resolve a single settlement disruption or billing error dispute, the agent often has to consult multiple operational tools: settlement batch audit status, FFIEC CDD duty-window trackers, customer/analyst RBAC profiles, partner liquidity inventory, and provisional credit eligibility engines.
Right now, if an orchestrator calls those tools one after another sequentially—wait for batch status, then call CDD tracker, then call account lookup, then call partner directory—the cumulative round-trip latency is unbearable. Most of those read lookups are independent and should be dispatched concurrently in parallel!
Beyond parallelism, our legacy Core Banking PSS and compliance ledger backends are fragile and verbose:
1. **Payload Bloat:** When you query a legacy Core Banking dispute API, it dumps a massive nested payload full of historical ISO 20022 clearing codes and irrelevant metadata. If you feed that raw response straight back into the model's context window, it slows down inference and drives up Dana's token bill. The architecture needs a pattern to filter, trim, or summarize tool outputs before they hit the model.
2. **Slow or Flaky Downstream APIs:** If the third-party partner liquidity or SWIFT gpi tracking API hangs or slows down during an outage, it cannot lock up the entire agent serving thread or crash the customer's session. We need non-blocking asynchronous execution, strict timeouts, connection reuse, circuit breaking, and safe fallback behaviors.
3. **Retry Safety:** If a tool call times out halfway through issuing a fee-waiver voucher or re-routing a wire transfer, a naive agent retry could accidentally issue duplicate vouchers or double-settle a payment. Tool interactions must be designed with idempotency and state checkpoints.

**[00:08:40] Sam Lin (CISO / Principal Security Architect):**
Speaking of tool interactions and serving architecture, this is where InfoSec draws a hard line. Look at what you're proposing: an autonomous, internet-reachable serving architecture that takes natural-language input from anyone on the internet and connects directly to our internal enterprise banking APIs and customer data stores. I need the design proposal to explicitly **identify the key security considerations for the selected serving architecture and define concrete mitigation strategies** for each one.

**[00:09:18] Lead Customer Engineer:**
Sam, could you walk through the specific threat vectors your security architecture review board will be scrutinizing in the CE's design proposal?

**[00:09:30] Sam Lin (CISO / Principal Security Architect):**
We group them into five critical serving and runtime threat surfaces:
1. **Adversarial Input & Prompt/Instruction Hijacking:** Adversaries—or even clever customers—will try to manipulate the agent into ignoring Reg E / Reg Z rules, approving unauthorized high-value provisional credit payouts, answering off-corpus investment/tax questions, or leaking internal operational prompts and proprietary compliance playbooks. What runtime inspection and guardrail layers sit in front of—and behind—the model inference call to detect and block adversarial inputs and policy-violating outputs without adding massive latency?
2. **Sensitive Data Leakage in Transit, Context & Logs:** Customers routinely paste debit/credit card numbers, passport identifiers, and personal financial info into chat windows. At the same time, internal tools return sensitive account records. How does the serving architecture prevent sensitive PII and payment data from entering the model context, persisting in shared caches or telemetry logs, or leaking across concurrent user sessions?
3. **Agent Identity, Confused Deputy & Over-Privileged Tool Execution:** This is my biggest architectural fear. If the agent runtime authenticates to our backend tools using a single, broad "super-service-account," a manipulated agent could query or modify records belonging to *other* customers or bypass RBAC role tiers (`analyst`, `senior_counsel`, `audit_reviewer`)—a classic confused deputy attack. How does the serving architecture enforce least-privilege identity propagation, scoped token exchange, and fine-grained authorization so the agent can only invoke tools and access rows strictly authorized for the currently authenticated user?
4. **Unrestricted Network Egress & Supply-Chain/Code Execution Risks:** If the agent runtime or its tool execution environment is compromised, could an attacker exfiltrate financial data out to an external internet server? How is the serving network perimeter isolated so the runtime can only communicate with approved internal banking APIs and model endpoints over private, encrypted paths?
5. **Service Exhaustion & Automated Abuse at the Edge:** During a clearing outage, malicious bots or scrapers could flood our public agent endpoints to exhaust our model quotas and deny service to real account holders—or rack up a massive inference bill for Dana. What edge protection, rate-limiting, and identity verification mitigations sit in front of the agent serving gateway?

**[00:12:05] Marcus Vance (VP of Global Compliance Operations):**
So to sum it up for your engineering team: don't just give us a list of buzzwords. Show us a cohesive **agent serving architecture** and explain the **specific technical methods** you're choosing to deliver **low-latency real-time inference**, **maximize throughput** during outage surges, **optimize costs**, **streamline tool interactions**, and **mitigate every single one of Sam's security risks**.

---
