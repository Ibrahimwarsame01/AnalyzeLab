# AnalyzeLab

A governed multi-agent AI fleet that verifies pharmaceutical products, investigates suspect/counterfeit drugs, and produces a regulator-grade audit trail built on Google ADK and the Gemini Enterprise Agent Platform (GEAP) for the **Fortified Enterprise Fleet** track.

## Problem

Counterfeit and diverted drugs are actively entering the legitimate U.S. pharmaceutical supply chain. Under the Drug Supply Chain Security Act (DSCSA), trading partners are legally required to verify products electronically and investigate suspect items within a strict window but in practice, verification is mechanical (does this serial number match?) with no reasoning layer, and failures are often caught only after a shipment has already arrived. AnalyzeLab adds the missing reasoning-and-governance layer on top of existing verification infrastructure.

Full problem statement and architecture: see [`AnalyzeLabPRD.md`](AnalyzeLabPRD.md).

## Live Demo

🔗 **[Try AnalyzeLab live](https://your-deployed-url-here.web.app)** *(update this link once deployed)*

## Architecture

```
Incoming Product Event
        │
        ▼
 ┌─────────────────┐
 │ Orchestrator     │  routes events, escalates anomalies
 │ Agent            │
 └────────┬─────────┘
          │
   ┌──────┴───────┐
   ▼              ▼
┌──────────┐  ┌───────────────┐
│Verification│  │ Investigation │  reasons over anomalies,
│Agent       │─▶│ Agent         │  uses Memory Bank for
│(checks     │  │               │  multi-day cases
│ registry)  │  └───────┬───────┘
└────────────┘          │
                         ▼
                 ┌───────────────┐
                 │ Reporting     │  drafts FDA suspect/
                 │ Agent         │  illegitimate product
                 │               │  notification
                 └───────────────┘

All inter-agent + external calls routed through Agent Gateway.
All traffic screened by Model Armor.
Every agent has a distinct Agent Identity.
All agents cataloged in Agent Registry.
Every action traced via Agent Observability (OpenTelemetry → Cloud Trace).
```

Simulated counterparty (manufacturer/trading-partner) is a plain REST/Cloud Function endpoint mimicking an EPCIS/VRS-style system intentionally *not* another AI agent, matching real-world DSCSA infrastructure where most trading partners run traditional systems.

## Reliability Safeguards

Prompt instructions alone don't stop an agent from ignoring a rule — every safeguard below is enforced outside the model, at the infrastructure or workflow layer:

- **Scoped permissions, not polite instructions —** each agent's write access is restricted at the IAM level (e.g. Verification's identity has no Firestore write grant at all), not just told "don't write" in its prompt
- **Persistent case state —** multi-day investigations read/write constraints to Firestore/Memory Bank each turn, instead of relying on the model to remember them in context
- **Verified, not just claimed, completion —** actions like "product verified" or "notification drafted" are independently confirmed with a readback, not trusted at face value
- **Human approval gate before filing —** the Reporting Agent can only produce a draft notification; a human must approve it before it's finalized, and the model has no path to bypass that gate

Full detail in [`AnalyzeLabPRD.md`](AnalyzeLabPRD.md), Section 6.5.

## Tech Stack

- **Agent framework:** Google ADK (Python)
- **Model:** Gemini 3.5 Flash (default), Gemini Pro (reserved for deep reasoning in Investigation Agent)
- **Deployment:** Cloud Run via Agent Runtime
- **Data:** Firestore (registry, transactions, case records), Vertex AI Memory Bank (cross-session investigation context)
- **Governance:** Agent Identity, Agent Registry, Agent Gateway, Model Armor, Agent Observability
- **Communication:** A2A protocol between agents
- **Frontend:** React dashboard, deployed on Vercel (data via Supabase)

## Repo Structure

```
analyzelab/
├── agents/
│   ├── orchestrator/     # routes events, escalates to investigation
│   ├── verification/     # checks products against mock registry
│   ├── investigation/    # reasons over anomalies, uses Memory Bank
│   └── reporting/        # drafts FDA notification artifacts
├── mock_registry/        # simulated EPCIS/VRS counterparty endpoint
├── frontend/              # dashboard UI
├── docs/                  # PRD, architecture diagram, submission materials
├── scripts/               # deploy scripts, seed data scripts
└── README.md
```

## Local Setup

### Prerequisites
- Python 3.11+
- `gcloud` CLI installed and authenticated
- A GCP project with billing enabled

### Install

```bash
git clone <your-repo-url>
cd analyzelab
python -m venv venv
source venv/bin/activate
pip install google-adk --break-system-packages
pip install -r requirements.txt
```

### Configure

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export GOOGLE_GENAI_USE_VERTEXAI=True
```

### Run locally

```bash
# Start the mock registry (simulated trading partner)
python mock_registry/server.py

# In a separate terminal, run the orchestrator agent
cd agents/orchestrator
adk run .
```

### Deploy backend to Cloud Run

```bash
bash scripts/deploy.sh
```

## License

All rights reserved. This code is provided for hackathon judging purposes only and may not be used, copied, or distributed without permission.
