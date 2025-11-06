# Keeper AI Backend - Cookbook

**The complete implementation guide for Keeper's AI-powered business intelligence engine.**

Keeper is a discovery engine that analyzes Square merchant data to generate nightly actionable business insights. It combines internal transaction data with external market intelligence to help small business owners grow revenue and retain customers.

---

## 🚀 Quick Start

**New to the project? Start here:**

1. 📘 Read **[KEEPER_ARCHITECTURE.md](KEEPER_ARCHITECTURE.md)** - Complete system design (agents, MCPs, infrastructure)
2. 🏃 Follow **[QUICK_START_GUIDE.md](QUICK_START_GUIDE.md)** - Get up and running with Bashful Beauty data
3. 🗺️ Check **[IMPLEMENTATION_ROADMAP.md](IMPLEMENTATION_ROADMAP.md)** - Phase-by-phase implementation guide

---

## 📚 Documentation

### Core Documentation

| Document | Purpose | For |
|----------|---------|-----|
| **[KEEPER_ARCHITECTURE.md](KEEPER_ARCHITECTURE.md)** | Complete architecture specification with all 3 agents and 31 MCPs | Understanding the system design |
| **[QUICK_START_GUIDE.md](QUICK_START_GUIDE.md)** | Step-by-step setup and first implementation | Getting started quickly |
| **[IMPLEMENTATION_ROADMAP.md](IMPLEMENTATION_ROADMAP.md)** | Phase-by-phase implementation guide with deliverables | Project planning |
| **[REPOSITORY_STRUCTURE.md](REPOSITORY_STRUCTURE.md)** | Recommended code organization and file structure | Setting up the codebase |
| **[FRONTEND_BACKEND_MAPPING.md](FRONTEND_BACKEND_MAPPING.md)** | Maps frontend pages/components to backend MCPs | Understanding what to build for the UI |

---

## 🏗️ System Overview

### What is Keeper?

Keeper is a **discovery engine** (not a chatbot) that:
- Runs nightly analysis on Square merchant data (8+ years of transactions, customers, employees)
- Combines internal data with external intelligence (reviews, competitors, social media)
- Generates 5-10 actionable insights with specific action plans, scripts, and ROI estimates
- Delivers insights through a beautiful web interface (Keeper_UI)

**Example Discovery:**
> "89% of first-time customers book brow waxing but only 41% return (vs 78% average). Root cause: junior staff assigned to first-timers lack relationship-building skills. **Action:** Assign first-time appointments to Jennifer (89% retention). **Expected outcome:** +24 returning customers/month = $12,400/year."

### Architecture

**3 Orchestrator Agents:**
1. **Analyst Agent** - Deep data investigation and pattern discovery
2. **Advisor Agent** - Convert insights into actionable recommendations
3. **Investigator Agent** - Answer user questions and investigate concerns

**31 MCP Tools** organized in 7 layers:
1. **Data Access** (3 MCPs) - Square data, external data, memory
2. **Analysis** (7 MCPs) - Pattern detection, segmentation, employee/customer analysis
3. **Intelligence** (4 MCPs) - Psychology, market dynamics, owner preferences
4. **Wisdom** (4 MCPs) - Industry best practices (spa, restaurant, retail, service)
5. **External Intelligence** (5 MCPs) - Reviews, competitors, social listening
6. **Action** (4 MCPs) - Script generation, campaigns, optimizations
7. **Validation** (4 MCPs) - Data quality, confidence scoring, impact prediction

**Tech Stack:**
- **Python 3.11+** - Core backend language
- **Google Cloud Platform** - BigQuery, Firestore, Cloud Run, Vertex AI
- **LLMs** - Vertex AI Gemini Pro (synthesis), Anthropic Claude Opus 4 (high-impact)
- **FastAPI** - REST API for frontend
- **Playwright** - Web scraping for external data

---

## 🎯 Project Goals

### MVP Success Criteria
- ✅ Nightly discoveries generated for Bashful Beauty (test business)
- ✅ 5-10 actionable insights with confidence >80%, impact >$1K
- ✅ Keeper_UI showing real discoveries from backend
- ✅ Business owner validates discoveries lead to improvements
- ✅ Cost per customer < $8/month (LLM + infrastructure)
- ✅ System runs reliably without manual intervention

### Long-term Vision
- 100+ small businesses using Keeper
- Average discovery quality rating > 4.5 stars
- Measurable ROI for customers (recovered revenue, retained customers)
- Scale to 1000+ businesses at 92%+ gross margin

---

## 💡 Key Insights from Planning

### Why This Architecture?

**Multi-Dimensional Analysis:**
Keeper doesn't just find "Sarah hasn't visited in 47 days." It finds:
- Sarah's 3-year visit pattern (every 18-22 days)
- Declining tips before defection (20% → 12% → 0%)
- Context: Jennifer was late on last appointment
- External: Sarah reviewed competitor "Glow Spa" 12 days ago
- Psychology: Declining tips = passive-aggressive feedback
- Action: Jennifer calls Sarah today with specific apology + 20% off
- Expected: 60% recovery chance, $2,280 lifetime value at stake

**Cost Optimization:**
- Pre-compute analytics in BigQuery (no LLM needed)
- Use LLMs only for synthesis and generation
- Web scraping for external data ($0 API costs)
- Dynamic model selection (Gemini Flash for standard, Opus 4 for critical)
- Result: $7.25/month COGS vs $99/month pricing = 92% margin

---

## 🧪 Testing with Real Data

**Bashful Beauty Spa** (San Diego, CA)
- 8 years of Square transaction data (2017-present)
- 200+ customers, 8 employees, 25 services
- $42,800 monthly revenue
- Real patterns to validate: retention issues, employee performance gaps, upsell opportunities

**Why This Matters:**
Testing with real business data ensures discoveries are accurate and actionable. Fake data won't reveal the complexity of real-world patterns.

---

## 📋 Implementation Phases

### Phase 1: Foundation
- Set up GCP infrastructure
- Sync 8 years of Bashful Beauty Square data to BigQuery
- Build first MCP: `square_data.mcp`

### Phase 2: Core MCPs
- Implement all Layer 1 (Data Access) and Layer 2 (Analysis) MCPs
- Test each MCP with real data
- Verify output quality

### Phase 3: Analyst Agent
- Build Analyst Agent with discovery_mode
- Integrate Vertex AI Gemini Pro
- Generate first real discoveries

### Phase 4: Advisor Agent
- Build Advisor Agent with action planning
- Implement Action Layer MCPs (scripts, campaigns)
- Generate complete discoveries with action plans

### Phase 5: API & Integration
- Build FastAPI REST API
- Deploy to Cloud Run
- Integrate with Keeper_UI frontend

### Phase 6: External Intelligence
- Build web scraping for reviews, competitors, social
- Enhance discoveries with external context

### Phase 7: Testing & Optimization
- Comprehensive testing (unit, integration, e2e)
- Performance and cost optimization
- Production readiness

---

## 🔗 Related Repositories

- **[Keeper_UI](https://github.com/defendersoftheinternet/Keeper_UI)** - Nuxt 3 frontend (Vue, Tailwind, Firebase)
- **Keeper_Cookbook** (this repo) - AI backend implementation guide

---

## 📞 Resources

- **Anthropic MCP Article:** https://www.anthropic.com/engineering/code-execution-with-mcp
- **Square API Docs:** https://developer.squareup.com/docs
- **Google Cloud Docs:** https://cloud.google.com/docs
- **Vertex AI Docs:** https://cloud.google.com/vertex-ai/docs

---

## 🙋 Questions?

Check the documentation:
1. **"How does the system work?"** → [KEEPER_ARCHITECTURE.md](KEEPER_ARCHITECTURE.md)
2. **"How do I get started?"** → [QUICK_START_GUIDE.md](QUICK_START_GUIDE.md)
3. **"What do I build first?"** → [IMPLEMENTATION_ROADMAP.md](IMPLEMENTATION_ROADMAP.md)
4. **"Which MCPs power which UI components?"** → [FRONTEND_BACKEND_MAPPING.md](FRONTEND_BACKEND_MAPPING.md)
5. **"How should I organize my code?"** → [REPOSITORY_STRUCTURE.md](REPOSITORY_STRUCTURE.md)

---

**Let's build something transformative for small business owners.** 🚀

—Ray & Claude
