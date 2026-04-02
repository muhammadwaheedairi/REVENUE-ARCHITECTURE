# REVENUE-ARCHITECTURE.md
## Digital FTE Framework — Claw Kingdom Signal-to-Revenue System
### Built by: WaheedAI Solutions | Muhammad Waheed (Lead Engineer)

---

## EXECUTIVE SUMMARY

Most AI builds capture a signal and update a database field. That's not revenue.

Revenue happens when a signal triggers **ownership**, **a follow-up sequence**, and a
**feedback loop** — all within milliseconds. This document proves our Digital FTE
framework closes the four gaps Claw Kingdom identified.

---

## THE FOUR GAPS WE SOLVE

```
Client Said:                          Our System Does:
─────────────────────────────────     ──────────────────────────────────────
"Signal layer = commercial value"  →  Every QR scan = structured Kafka event
"Context→Action gap kills builds"  →  pgvector context decides action in <50ms
"Follow-up must tie to outcomes"   →  Agent SDK triggers ownership + sequence
"Need pipeline movement + owner"   →  Postgres stage + Kafka ownership notify
```

---

## SECTION 1 — SIGNAL-TO-REVENUE MAPPING

### How a Raw QR Scan Becomes a Commercial Event

A physical QR scan at a Claw Machine is blind by default. Our ingestion layer
makes it commercially visible in 3 steps:

```
PHYSICAL WORLD                    DIGITAL SIGNAL LAYER
──────────────                    ────────────────────

  [Claw Machine]                  FastAPI Webhook Endpoint
  QR Scan Event       ──────────► POST /api/v1/signal/ingest
                                         │
                                         ▼
                               Kafka Topic: fte.signals.raw
                               Payload:
                               {
                                 signal_id: "uuid",
                                 source: "qr_scan",
                                 machine_id: "CLW-042",
                                 customer_id: "inferred or null",
                                 timestamp: "ISO-8601",
                                 location: "floor_zone_B",
                                 metadata: { attempts: 3, session_ms: 45000 }
                               }
                                         │
                                         ▼
                            Signal Enrichment Consumer
                            (Postgres lookup: known customer?)
                                         │
                               ┌─────────┴──────────┐
                               ▼                    ▼
                           KNOWN               UNKNOWN
                        [VIP / History]     [New Lead Signal]
                               │                    │
                               ▼                    ▼
                    Kafka: fte.signals.enriched  (both paths merge here)
                    {
                      ...original,
                      commercial_intent: "HIGH / MEDIUM / NEW",
                      lead_stage: "HOT / WARM / COLD / NEW",
                      recommended_action: "DIRECT_SALES / NURTURE / DISCOUNT_TRIGGER"
                    }
```

**Every scan is now a Commercial Event — not just a log entry.**

---

## SECTION 2 — THE MISSING LINK: CONTEXT-TO-ACTION

### The Gap Most Builds Fall Into

```
TYPICAL BUILD (broken):
  Signal ──► CRM Stage Update ──► Nothing else happens
                                  (no owner, no follow-up, revenue stalls)

OUR BUILD (closed loop):
  Signal ──► Context Layer ──► Action Layer ──► Owner Assigned + Follow-up Live
```

### How We Decide "Who Owns This Lead" and "What Follow-Up Fires"

```
CONTEXT LAYER — PostgreSQL + pgvector
──────────────────────────────────────

  customers table:
    - visit_count, last_visit, total_spend
    - failed_attempts (frustration score)
    - preferred_channel (WhatsApp / Email / Web)

  pgvector semantic search:
    - "Find customers similar to this behavior pattern"
    - Returns: intent_score, churn_risk, upsell_probability

  Decision Matrix:
  ┌────────────────────┬───────────────────┬──────────────────────────┐
  │ Context Signal     │ Intent Tag        │ Action Triggered         │
  ├────────────────────┼───────────────────┼──────────────────────────┤
  │ 3+ failed attempts │ FRUSTRATED_HOT    │ Instant discount WhatsApp│
  │ VIP + 7 day gap    │ RETENTION_RISK    │ Manager alert + callback │
  │ First scan ever    │ NEW_LEAD          │ Welcome + nurture seq.   │
  │ High spend history │ UPSELL_READY      │ Direct Sales bot assign  │
  │ Scan + no purchase │ ABANDONED         │ 30-min follow-up trigger │
  └────────────────────┴───────────────────┴──────────────────────────┘


ACTION LAYER — OpenAI Agents SDK (This Is The Ownership Layer)
───────────────────────────────────────────────────────────────

  Agent receives enriched signal from Kafka.
  Agent does NOT just update a field.
  Agent executes:

    Step 1: Assign Lead Owner
    ─────────────────────────
    if intent == "FRUSTRATED_HOT" or "UPSELL_READY":
        → assign_to: floor_staff_agent (human escalation)
        → Kafka publish: fte.ownership.assigned
        → Staff gets WhatsApp ping in <5 seconds

    if intent == "NEW_LEAD" or "ABANDONED":
        → assign_to: sales_sub_agent (automated)
        → Sub-agent begins follow-up sequence autonomously

    Step 2: Trigger Follow-Up Sequence
    ────────────────────────────────────
    Channel selected from customer preference (WhatsApp / Gmail / Web)
    Sequence type from intent tag:
      - DISCOUNT  → immediate one-time offer message
      - NURTURE   → 3-message drip over 48 hours
      - CALLBACK  → manager notified + calendar slot sent
      - RETENTION → personalized win-back flow

    Step 3: Log Ownership Event
    ────────────────────────────
    Postgres INSERT into pipeline_events:
    {
      signal_id, customer_id, intent_tag,
      owner_assigned, follow_up_type,
      timestamp, channel_used
    }
```

**This is the "Missing Link." The Context Layer reads history.
The Action Layer creates accountability. No lead goes unowned.**

---

## SECTION 3 — PIPELINE OWNERSHIP LOGIC

### Stage Entry

```
NEW LEAD (first scan, no history):
  lead_stage = "NEW"
  owner = "nurture_bot"
  sequence = "onboarding_3_step"
  visibility = "new_leads dashboard"

HOT LEAD (3+ visits, no conversion):
  lead_stage = "HOT"
  owner = floor_staff OR sales_bot (intent-based)
  sequence = "high_intent_close"
  visibility = "priority_pipeline dashboard"
  SLA = 15 minutes (escalates if no action)

VIP AT RISK (high spend + long gap):
  lead_stage = "RETENTION"
  owner = "senior_manager_agent"
  sequence = "win_back_personal"
  visibility = "retention_alerts dashboard"
  SLA = 5 minutes
```

### Instant Ownership Flow

```
Signal Enriched
      │
      ▼
  OpenAI Agent evaluates intent
      │
      ├──► Human Escalation Path:
      │    Kafka: fte.ownership.human
      │    → Floor staff WhatsApp ping
      │    → Dashboard highlights lead RED
      │    → 15-min SLA timer starts
      │
      └──► Bot Ownership Path:
           Kafka: fte.ownership.bot
           → Sub-agent begins sequence
           → Dashboard highlights lead YELLOW
           → Auto-resolves or escalates after sequence
```

### Feedback Loops — Revenue Visibility for Management

```
Every action updates Postgres pipeline_events in real-time:

  Event Types logged:
  ─────────────────────────────────────────────────────────
  SIGNAL_RECEIVED     → raw scan captured
  INTENT_TAGGED       → context layer classified
  OWNER_ASSIGNED      → human or bot assigned
  SEQUENCE_STARTED    → follow-up initiated
  SEQUENCE_RESPONDED  → customer replied or clicked
  STAGE_ADVANCED      → lead moved HOT→CLOSED or churned
  REVENUE_LOGGED      → purchase confirmed
  ─────────────────────────────────────────────────────────

Management Dashboard (Next.js):
  - Live pipeline view: NEW / HOT / RETENTION / CLOSED
  - Per-machine revenue attribution
  - Owner performance (bot vs human close rate)
  - Sequence effectiveness (which triggers convert best)
  - Daily/weekly revenue impact per signal source
```

---

## SECTION 4 — INFRASTRUCTURE FOR SCALE

### Why AWS EKS + Kafka = 100% Reliability at Scale

```
CLAW KINGDOM SCALE CHALLENGE:
  Hundreds of physical machines
  Peak hours: simultaneous scan bursts
  Zero tolerance for dropped signals

OUR INFRASTRUCTURE ANSWER:

  ┌─────────────────────────────────────────────────────────┐
  │                    AWS EKS CLUSTER                       │
  │                                                          │
  │  ┌──────────────┐   ┌──────────────┐  ┌──────────────┐  │
  │  │  FastAPI Pods │   │  Agent Pods  │  │ Next.js Pods │  │
  │  │  (ingestion) │   │  (OpenAI SDK)│  │  (dashboard) │  │
  │  │  replicas: 3 │   │  replicas: 5 │  │  replicas: 2 │  │
  │  └──────┬───────┘   └──────┬───────┘  └──────────────┘  │
  │         │                  │                              │
  │         ▼                  ▼                              │
  │  ┌─────────────────────────────────────┐                 │
  │  │         Apache Kafka (3.7.0)         │                 │
  │  │  Topics:                             │                 │
  │  │  fte.signals.raw                     │                 │
  │  │  fte.signals.enriched                │                 │
  │  │  fte.ownership.assigned              │                 │
  │  │  fte.sequences.triggered             │                 │
  │  │  fte.pipeline.events                 │                 │
  │  │  Partitions: 12 | Replication: 3    │                 │
  │  └─────────────────────┬───────────────┘                 │
  │                        │                                  │
  │                        ▼                                  │
  │  ┌─────────────────────────────────────┐                 │
  │  │   PostgreSQL + pgvector (Neon.tech)  │                 │
  │  │   - customers, leads, pipeline_events│                 │
  │  │   - vector embeddings for intent     │                 │
  │  │   - connection pooling: pgBouncer    │                 │
  │  └─────────────────────────────────────┘                 │
  └─────────────────────────────────────────────────────────┘

RELIABILITY GUARANTEES:
  ✓ Kafka partitions = parallel processing (no queue bottleneck)
  ✓ EKS auto-scaling = burst scan handling during peak hours
  ✓ Replication factor 3 = zero message loss even if 2 brokers fail
  ✓ Neon.tech serverless Postgres = auto-scale, no cold starts
  ✓ Dead Letter Queue on every Kafka topic = failed events retried
```

---

## SECTION 5 — REVENUE IMPACT TABLE

```
┌──────────────────────┬───────────────────────────────────┬─────────────────────────────────────┐
│ SIGNAL               │ COMMERCIAL LOGIC                  │ REVENUE OUTCOME                     │
├──────────────────────┼───────────────────────────────────┼─────────────────────────────────────┤
│ QR Scan — 1st time   │ New lead created, nurture starts  │ Brand awareness → 2nd visit likely  │
│                      │ Welcome message via WhatsApp       │ +15-25% return rate (avg)           │
├──────────────────────┼───────────────────────────────────┼─────────────────────────────────────┤
│ QR Scan — 3 failed   │ FRUSTRATED_HOT intent triggered   │ Instant discount prevents churn     │
│ attempts             │ Discount offer sent in <5 sec     │ Converts frustration → purchase     │
├──────────────────────┼───────────────────────────────────┼─────────────────────────────────────┤
│ VIP — 7+ day gap     │ RETENTION_RISK flagged            │ Win-back campaign before they leave │
│                      │ Manager alert + personal offer    │ Retaining VIP = 5-10x new lead $    │
├──────────────────────┼───────────────────────────────────┼─────────────────────────────────────┤
│ Scan + no purchase   │ ABANDONED — 30-min follow-up      │ Re-engagement while intent is hot   │
│ (session drop-off)   │ Personalized incentive sent       │ +30% recovery rate on warm leads    │
├──────────────────────┼───────────────────────────────────┼─────────────────────────────────────┤
│ High spend history   │ UPSELL_READY → direct sales bot  │ Premium package / loyalty offer     │
│                      │ assigned immediately               │ Higher AOV per visit                │
├──────────────────────┼───────────────────────────────────┼─────────────────────────────────────┤
│ Staff assigned lead  │ Human SLA: 15 min response        │ Personal touch on high-value leads  │
│ (human escalation)   │ Dashboard RED alert until done    │ Accountability = no lead left cold  │
├──────────────────────┼───────────────────────────────────┼─────────────────────────────────────┤
│ Sequence completed   │ Feedback logged to Postgres       │ Management sees what converts       │
│ (all types)          │ A/B data on best-performing flows │ Continuous revenue optimization     │
└──────────────────────┴───────────────────────────────────┴─────────────────────────────────────┘
```

---

## CLOSING STATEMENT

> "Most builds stop at the signal. Ours starts there."

The Digital FTE framework treats every QR scan as a revenue event with
an owner, a sequence, and a feedback loop. Claw Kingdom's pipeline will
not just capture leads — it will move them, own them, and close them.

**Stack Summary:**
- Ingestion: FastAPI + Kafka (Apache 3.7.0)
- Context: PostgreSQL + pgvector (Neon.tech)
- Action: OpenAI Agents SDK (multi-channel routing)
- Channels: WhatsApp (Twilio) + Gmail + Web Form
- Infrastructure: AWS EKS + Kubernetes
- Dashboard: Next.js 16

---
*Document prepared by Muhammad Waheed — WaheedAI Solutions*
*Contact: muhammadwaheedairi@gmail.com*
*GitHub: muhammadwaheedairi*
