# 🧠 SynapseAI — Real-time Knowledge Mesh

**Kafka · MongoDB · pgvector · Gemini**

> A real-time, event-driven knowledge system that ingests Slack, GitHub, and Notion events through a secure webhook gateway, streams them via Kafka, normalizes and enriches them into MongoDB (truth store), generates Gemini embeddings, indexes retrieval in pgvector, and powers a dashboard plus bounded agent workflows with two-tier memory (Digests + Raw Docs).

---

## 📌 What is SynapseAI?

Teams generate massive amounts of context across tools like Slack, GitHub, and Notion — but that knowledge is fragmented, hard to search, and quickly forgotten.

**SynapseAI** acts as a **real-time knowledge mesh** that:

* Captures every important event as it happens
* Normalizes and enriches it into a canonical knowledge graph
* Builds semantic search with vector embeddings
* Maintains structured organizational memory (issues, decisions, signals)
* Powers dashboards and bounded AI agents with reliable evidence

Think of it as:

> **Event-driven ingestion → canonical truth store → vector retrieval → structured memory → explainable AI workflows**

---

## 🏗️ Architecture
<img width="1070" height="589" alt="Pasted Graphic" src="https://github.com/user-attachments/assets/ec71cd5e-8956-435f-a9fe-e7052125d1a1" />


### High-Level System Design

![SynapseAI Architecture](docs/architecture.png)

**Pipeline overview:**

Secure webhooks & delta sync → Kafka → normalize/enrich → MongoDB truth store → Gemini embeddings → pgvector retrieval → UI + bounded agents with two-tier memory.

---

## 🧩 Components

### 🎨 Frontend (UI)

**SynapseAI Dashboard — Next.js / React**

Features:

* Digests feed (known issues & decisions)
* Semantic search with explanation (“why it matched”)
* Trend analysis over time
* Drilldowns to Slack/GitHub/Notion permalinks

---

### 🔌 Source Systems (Real-Time)

* **Slack Events API**

  * Channels, DMs, reactions
* **GitHub Webhooks**

  * Issues, PRs, comments, pushes
* **Notion**

  * Polling + delta sync (hybrid due to webhook limitations)

Goal: **every meaningful change becomes an event**.

---

### 🚪 Webhook Gateway + Ingestion API

**Next.js API routes (TypeScript)**

Endpoints:

* `POST /webhooks/slack`
* `POST /webhooks/github`
* `POST /sync/notion` (delta sync)

Responsibilities:

* Verify signatures (Slack signing secret, GitHub HMAC)
* Deduplicate deliveries (`event_id`, `delivery_id`)
* Minimal parsing and validation
* Immediately publish raw payload to Kafka
  *(no heavy processing inside webhooks)*

---

### 📬 Event Queue / Stream

**Kafka**

Topics:

* `ingest.raw_events`
* `process.normalize`
* `process.enrich`
* `process.embed`
* `index.updates`

Reliability:

* DLQ topics for failures
* Replay + backfill support

Kafka is the backbone for:

* Scalability
* Fault tolerance
* Deterministic reprocessing

---

### ⚙️ Processing Layer (Stateless Workers)

All workers are Dockerized and stateless.

#### Worker A — Normalizer (Node.js / TypeScript)

* Converts tool payloads → canonical schema
* Writes:

  * Raw payload → `mongo.raw_events`
  * Canonical document → `mongo.documents`

---

#### Worker B — Enricher (Node.js / TS or Python)

* Entity extraction
* Signal detection
* Classification:

  * issue / decision / info
* Updates document metadata in MongoDB

---

#### Worker C — Embedder (Gemini)

* Generates embeddings for:

  * Canonical document text
  * Optional digest summaries
* Writes vectors into:

  * PostgreSQL + pgvector
* Stores `doc_id` references back to Mongo

---

### 🧾 System of Record (Truth + Audit)

**MongoDB**

Collections:

* `raw_events`

  * Full audit log
  * Replay support
* `documents`

  * Canonical normalized records
* `digests`

  * Structured organizational memory
* `processing_state`

  * Connector checkpoints
  * Backfill tracking

MongoDB is the **ground truth** of the system.

---

### 🔍 Vector DB / Retrieval Index

**PostgreSQL + pgvector**

Stores:

* Embeddings
* Retrieval metadata
* `doc_id` references back to Mongo

Used for:

* Semantic search
* Agent retrieval
* Similarity clustering

---

### 🤖 Query + Agent Layer

#### 🔎 Search API

`POST /search`

Flow:

* Embed query using Gemini
* pgvector similarity search + metadata filters
* Retrieve matching `doc_id`s
* Hydrate results from Mongo
* Return explanations + permalinks

---

#### 🧠 Issue Memory / Agent API

`POST /issues/classify`

Bounded reasoning workflow:

1. Retrieve top-k related digests
2. Compare with incoming doc
3. Classify as:

   * Duplicate
   * Regression
   * New issue
4. Write structured memory:

   * Digest updates
   * Evidence links to source docs

This prevents hallucinations and keeps agents grounded.

---

### 🔐 Auth & Security

* Webhook verification

  * Slack signing secret
  * GitHub HMAC
* UI authentication

  * OAuth / NextAuth / SSO (optional)
* Role-based access control

  * Org / workspace scoped

Security is enforced at both **ingestion** and **query** layers.

---

### 🧠 LLM / Model Layer

**Gemini via Google AI / Vertex AI**

Used for:

* Text embeddings (semantic vectors)
* Digest summarization
* Bounded classification tasks

LLMs never directly read raw databases —
all access is mediated through retrieval + evidence.

---

## 🔁 End-to-End Data Flow (Step-by-Step)

1. Slack/GitHub emits a webhook; Notion runs delta sync.
2. Webhook Gateway verifies signature, deduplicates, validates.
3. Gateway publishes raw payload to Kafka (`ingest.raw_events`).
4. Normalizer Worker:

   * Writes raw event to Mongo `raw_events`
   * Writes canonical doc to Mongo `documents`
5. Enricher Worker:

   * Extracts entities & signals
   * Classifies document type
   * Updates Mongo `documents`
6. Embedder Worker:

   * Generates Gemini embeddings
   * Writes vectors to pgvector
7. Index update messages published to Kafka for consistency.
8. Search API:

   * Embeds query
   * Vector search in pgvector
   * Hydrates details from Mongo
9. Agent API:

   * Retrieves top-k digests
   * Gemini performs bounded reasoning
   * Updates Mongo `digests` with evidence
10. UI Dashboard:

* Shows digests, trends, explanations
* Links back to original Slack/GitHub/Notion context

---

## 🚀 Deployment

### UI + API

* Next.js (TypeScript)
* Options:

  * Vercel
  * Docker on Kubernetes / ECS / VM

---

### Streaming

* Kafka cluster

  * Managed or self-hosted
  * Includes DLQ + retry topics

---

### Workers

* Dockerized Node.js / Python services
* Deployed on:

  * Kubernetes
  * ECS
  * VM scale sets
* Autoscale based on Kafka lag

---

### Databases

* **MongoDB**

  * Atlas or self-hosted
  * Truth store + audit log
* **PostgreSQL + pgvector**

  * Managed Postgres
  * Vector retrieval index

---

### Gemini

* Google AI APIs or Vertex AI
* Used by:

  * Embedder workers
  * Agent services

---

## 🎯 Why This Architecture Works

* ✅ Event-driven → no polling bottlenecks
* ✅ Kafka replay → safe backfills & reprocessing
* ✅ Canonical truth store → consistent semantics
* ✅ Vector DB separate from truth → scalable retrieval
* ✅ Bounded agents → explainable AI, not hallucinations
* ✅ Two-tier memory → fast reasoning + deep audit

This is designed for **enterprise-scale collaboration tools** where:

* Data volume is high
* Knowledge must remain explainable
* AI must operate with strong guardrails

