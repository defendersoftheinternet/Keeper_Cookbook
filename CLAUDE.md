# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Keeper_Cookbook is the complete implementation guide and architecture documentation for Keeper's AI-powered business intelligence engine. This is a **documentation repository** - it contains comprehensive specifications, guides, and architecture designs for building Keeper's backend, but does not contain actual implementation code.

**What Keeper Does:**
Keeper is a discovery engine that analyzes Square merchant data to generate nightly actionable business insights. It combines internal transaction data with external market intelligence to help small business owners grow revenue and retain customers.

## Repository Purpose

This repository serves as the **"cookbook"** - a complete reference for implementing Keeper's backend:
- System architecture specifications (3 agents, 31 MCP tools)
- Implementation roadmaps and guides
- Data flow diagrams and integration patterns
- Quick start guides with real example code
- Frontend-backend mapping documentation

**Note:** The actual backend implementation code will live in a separate repository. This is the planning and specification repository.

## Key Documentation Files

### Core Architecture
- **[KEEPER_ARCHITECTURE.md](KEEPER_ARCHITECTURE.md)** (85KB, 2335 lines) - Complete system design
  - 3 Orchestrator Agents (Analyst, Advisor, Investigator)
  - 31 MCP Tools organized in 7 layers
  - Infrastructure on Google Cloud Platform
  - Discovery generation flow
  - Cost model ($7.25/month per customer)
  - Complete implementation examples

### Getting Started
- **[QUICK_START_GUIDE.md](QUICK_START_GUIDE.md)** (24KB, 847 lines) - Step-by-step setup
  - Environment setup (Python 3.11+, GCP, Docker)
  - Square data sync to BigQuery (8 years of Bashful Beauty data)
  - First MCP implementation (square_data.mcp)
  - Testing with real data

### Implementation Planning
- **[IMPLEMENTATION_ROADMAP.md](IMPLEMENTATION_ROADMAP.md)** (24KB) - Phase-by-phase guide
  - Phase 1: Foundation (GCP setup, data sync)
  - Phase 2: Core MCPs (Layer 1-2 implementation)
  - Phase 3-7: Agents, API, external intelligence, testing

### Code Organization
- **[REPOSITORY_STRUCTURE.md](REPOSITORY_STRUCTURE.md)** (21KB, 466 lines) - Recommended file structure
  - `/agents/` - 3 orchestrator agents
  - `/mcps/` - 31 MCP tools organized by layer
  - `/services/` - Core services (discovery, sync, scraper)
  - `/api/` - FastAPI REST API
  - `/infrastructure/` - Terraform configs
  - Data flow diagrams between components

### UI Integration
- **[FRONTEND_BACKEND_MAPPING.md](FRONTEND_BACKEND_MAPPING.md)** (27KB) - Maps UI components to backend MCPs

### Project Overview
- **[README.md](README.md)** (8KB, 193 lines) - Repository introduction and navigation

## System Architecture (High-Level)

### 3 Orchestrator Agents
1. **Analyst Agent** - Deep data investigation (uses Claude Opus 4 for synthesis)
   - `discovery_mode()` - Nightly comprehensive analysis
   - `drill_down_mode()` - User-requested deep dives
   - `validation_mode()` - Verify insights before publishing

2. **Advisor Agent** - Convert insights to actionable recommendations
   - `action_planning_mode()` - Generate 3-5 step action plans
   - `script_writing_mode()` - Create conversation scripts
   - `campaign_mode()` - Design marketing campaigns

3. **Investigator Agent** - Answer user questions via chat
   - `question_answering_mode()` - Handle chat queries
   - `investigation_mode()` - Deep dive into user topics
   - `explanation_mode()` - Explain why discoveries were generated

### 31 MCP Tools (7 Layers)
1. **Data Access (3 MCPs)** - square_data.mcp, external_data.mcp, memory.mcp
2. **Analysis (7 MCPs)** - pattern_detector.mcp, segment_analyzer.mcp, employee_analyzer.mcp, etc.
3. **Intelligence (4 MCPs)** - customer_psychology.mcp, employee_psychology.mcp, market_dynamics.mcp, etc.
4. **Wisdom (4 MCPs)** - spa_wisdom.mcp, restaurant_wisdom.mcp, retail_wisdom.mcp, service_wisdom.mcp
5. **External Intelligence (5 MCPs)** - review_monitor.mcp, competitor_discovery.mcp, social_listening.mcp, etc.
6. **Action (4 MCPs)** - script_generator.mcp, campaign_builder.mcp, optimization_architect.mcp, crisis_manager.mcp
7. **Validation (4 MCPs)** - data_validator.mcp, confidence_scorer.mcp, impact_predictor.mcp, output_formatter.mcp

### Tech Stack
- **Language:** Python 3.11+
- **Cloud:** Google Cloud Platform (BigQuery, Firestore, Cloud Run, Vertex AI)
- **LLMs:** Vertex AI Gemini Pro (synthesis), Anthropic Claude Opus 4 (high-impact discoveries)
- **API:** FastAPI for REST endpoints
- **Data:** BigQuery (Square data cache), Firestore (discoveries, business profiles)
- **Memory:** Redis (7-day working memory), Vertex AI Vector Search (pattern memory)
- **Scraping:** Playwright for external data (reviews, competitors, social media)

## Key Concepts

### Discovery Flow (Nightly at 3:00 AM)
1. **Data Collection** (5-10s) - Fetch Square data, external data from BigQuery/Cloud Storage
2. **Analysis** (30-60s) - Run all Analysis Layer MCPs to find patterns
3. **Intelligence** (20-30s) - Apply psychological and industry context
4. **External Intelligence** (10-20s) - Review analysis, competitor tracking, social listening
5. **Synthesis** (60-90s) - Analyst Agent calls Gemini Pro with all findings
6. **Validation** (10-20s) - Confidence scoring, impact prediction
7. **Action Planning** (30-60s) - Advisor Agent generates scripts, campaigns
8. **Formatting** (5-10s) - Convert to frontend JSON format
9. **Storage** - Save discoveries to Firestore

**Total time:** 3-5 minutes per business

### Example Discovery
```json
{
  "id": "discovery_20250921_003",
  "title": "First-time brow customers need senior staff assignment",
  "category": "Customer Retention",
  "priority": "medium",
  "impact": 6252,
  "confidence": 87,
  "description": "89% of first-time customers book brow waxing ($28 avg), but only 41% return compared to 78% business average.",
  "reasoning": "First-timers randomly assigned to any staff. Marcus (42% retention) serviced 67% vs Jennifer (89% retention) at 18%.",
  "actionPlan": [
    "Assign all first-time brow appointments to Jennifer or Amanda",
    "Train Marcus on first-time customer protocol",
    "Add 'New Customer' flag in booking system",
    "Track retention rate weekly for 30 days"
  ],
  "script": "Hi [Name]! I see this is your first time - welcome! I'm Jennifer...",
  "expectedRecovery": "Increase retention 41% → 65% = $6,252/year",
  "tags": ["#CUSTOMERRETENTION", "#TRAINING"]
}
```

### Test Data
**Bashful Beauty Spa** (San Diego, CA) - Real business used for testing
- 8 years of Square transaction data (2017-present)
- 200+ customers, 8 employees, 25 services
- $42,800 monthly revenue
- Used to validate all discoveries are accurate and actionable

## Working with This Repository

### Reading the Documentation
When implementing features, always start with these documents in order:
1. **KEEPER_ARCHITECTURE.md** - Understand the complete system design
2. **REPOSITORY_STRUCTURE.md** - Know where code should live
3. **QUICK_START_GUIDE.md** - See working example implementations
4. **IMPLEMENTATION_ROADMAP.md** - Follow the phase-by-phase plan

### Understanding MCP Structure
MCPs are Python classes that:
- Inherit from `BaseMCP` (provides BigQuery, Firestore, Redis clients)
- Contain focused methods for specific analysis tasks
- Return structured dictionaries (not raw data)
- Pre-compute analytics in BigQuery (never raw SQL in LLM prompts)
- Example location: `mcps/data_access/square_data.mcp.py`

### Understanding Agent Structure
Agents are orchestrators that:
- Load relevant MCPs dynamically
- Call multiple MCPs to gather context
- Synthesize findings using LLMs (Gemini Pro or Claude Opus 4)
- Do NOT perform computations directly (delegate to MCPs)
- Example location: `agents/analyst_agent.py`

### Cost Model
Target: **$7.25/month per customer**
- LLM costs: $6.04 (83%)
  - Gemini Pro: $1.69 (standard discoveries)
  - Claude Opus 4: $4.32 (20% of high-impact discoveries)
- Infrastructure: $1.21 (17%)
  - BigQuery: $0.05, Firestore: $0.01, Cloud Run: $0.44, etc.

Pricing: **$99/month** = **92.7% gross margin**

### Infrastructure
All on Google Cloud Platform:
- **BigQuery:** Square data cache (synced nightly at 12:01 AM)
- **Firestore:** Business profiles, discoveries, user data
- **Memorystore Redis:** Working memory (7-day TTL)
- **Cloud Run:** API service + discovery engine
- **Cloud Functions:** Web scraping (reviews, competitors)
- **Cloud Scheduler:** Triggers nightly jobs
- **Vertex AI:** Gemini Pro LLM, Vector Search for pattern memory

## Important Notes

### This is a Documentation Repository
When asked to "implement" features:
1. Reference the appropriate section in KEEPER_ARCHITECTURE.md
2. Show example code from QUICK_START_GUIDE.md
3. Explain the design rationale
4. Do NOT create actual implementation files (unless explicitly requested for documentation examples)

### Multi-Dimensional Analysis
Keeper's value comes from combining 10+ MCPs to find root causes:
- Not: "Sarah hasn't visited in 47 days"
- But: "Sarah's 3-year pattern (18-22 days) broken, declining tips before defection (20%→12%→0%), Jennifer was late last visit, Sarah reviewed competitor 12 days ago, psychology: passive-aggressive feedback, expected recovery: 60% if contacted today"

### LLM Usage Philosophy
- Pre-compute all analytics in BigQuery (SQL queries, aggregations)
- Use LLMs ONLY for synthesis and natural language generation
- Never send raw transaction data to LLMs
- Always send pre-computed summaries from MCPs
- Result: Low token counts, fast responses, low cost

### External Data Strategy
- Web scraping (Playwright) instead of expensive APIs
- $0/month for reviews, competitor data, social listening
- Previously estimated $24/month in API costs - eliminated
- Scraper runs at 1:00 AM before discovery at 3:00 AM

## Related Repositories

- **Keeper_UI** - Nuxt 3 frontend (the UI that displays discoveries)
  - Located at: `/Users/concordsteve/Projects/Keeper-App-UI/`
  - Vue 3, TypeScript, Tailwind CSS
  - Expects discoveries in specific JSON format (see KEEPER_ARCHITECTURE.md Appendix)

## Vocabulary

- **Discovery:** A single actionable insight generated by the system (not "alert" or "notification")
- **MCP:** Model Context Protocol - Tools that agents use to access data and perform analysis
- **Agent:** Orchestrator that calls MCPs and synthesizes findings with LLMs
- **Working Memory:** Redis cache with 7-day TTL for current investigations
- **Core Memory:** Firestore permanent storage for business profiles
- **Pattern Memory:** Vector Search + BigQuery for historical insights
- **Bashful Beauty:** Real spa business in San Diego used as test data

## Common Tasks

### Finding Architecture Details
All architectural details are in KEEPER_ARCHITECTURE.md:
- Agent responsibilities and modes
- All 31 MCP tools with methods and examples
- Complete data flow diagrams
- Infrastructure component details
- Discovery schema format

### Finding Implementation Examples
QUICK_START_GUIDE.md contains working example code for:
- Setting up GCP infrastructure
- Syncing Square data to BigQuery
- Implementing the first MCP (square_data.mcp)
- Testing with real Bashful Beauty data

### Understanding Data Flow
REPOSITORY_STRUCTURE.md has complete data flow diagrams:
- Nightly discovery flow (12:01 AM sync → 1:00 AM scraping → 3:00 AM discovery)
- User login flow (fetch discoveries from Firestore)
- Chat flow (Investigator Agent)

### Mapping UI to Backend
FRONTEND_BACKEND_MAPPING.md shows which MCPs power which UI pages

## Resources

- **Anthropic MCP Documentation:** https://www.anthropic.com/engineering/code-execution-with-mcp
- **Square API Docs:** https://developer.squareup.com/docs
- **Google Cloud Docs:** https://cloud.google.com/docs
- **Vertex AI Gemini:** https://cloud.google.com/vertex-ai/docs/generative-ai/model-reference/gemini
- **BigQuery:** https://cloud.google.com/bigquery/docs
- **Firestore:** https://cloud.google.com/firestore/docs
