# Keeper AI Backend - Implementation Guide

**Project**: Keeper AI Discovery Engine
**Target**: Small businesses using Square (< $1M revenue)
**Tech Stack**: Python, Google Cloud, Vertex AI Gemini Pro, Anthropic Claude Opus 4

---

## Overview

This guide breaks down the implementation of Keeper's AI backend into manageable phases. Each phase builds on the previous one and includes concrete deliverables, testing strategies, and success criteria.

**Key Documents**:
- 📘 **Architecture Spec**: `KEEPER_ARCHITECTURE.md` - Complete system design
- 📁 **Repository Structure**: `REPOSITORY_STRUCTURE.md` - Code organization
- 🚀 **Quick Start**: `QUICK_START_GUIDE.md` - Get up and running

---

## Implementation Phases

| Phase | Goal | Success Criteria |
|-------|------|------------------|
| **Phase 1** | Foundation & Data Pipeline | Bashful Beauty data in BigQuery |
| **Phase 2** | Core MCPs (Layers 1-2) | All analysis MCPs working |
| **Phase 3** | Analyst Agent | Nightly discoveries generated |
| **Phase 4** | Advisor Agent & Actions | Complete discoveries with action plans |
| **Phase 5** | API & Integration | Keeper_UI showing real discoveries |
| **Phase 6** | External Intelligence | Reviews, competitors, social data |
| **Phase 7** | Testing & Optimization | Production-ready, cost-optimized |

---

## Phase 1: Foundation & Data Pipeline

### Goals
- Set up Google Cloud infrastructure
- Sync Bashful Beauty's 8 years of Square data to BigQuery
- Establish development environment

### Tasks

#### GCP Setup
- [ ] Create GCP project `keeper-production`
- [ ] Enable required APIs (BigQuery, Firestore, Cloud Run, Vertex AI, etc.)
- [ ] Create service account for development
- [ ] Set up BigQuery dataset and tables (`keeper.square_cache`, `keeper.pattern_history`)
- [ ] Set up Firestore database
- [ ] Create Cloud Storage buckets (`keeper-external-data`, `keeper-mcp-files`)
- [ ] Set up local development environment (Python 3.11, Docker, dependencies)

**Deliverables**:
- ✅ GCP project fully configured
- ✅ BigQuery tables created with correct schemas
- ✅ Service account credentials working locally
- ✅ `.env` file with all environment variables

**Testing**: Run `gcloud projects describe keeper-production` and verify all APIs enabled

---

#### Square Data Sync
- [ ] Implement `utils/square_client.py` - Square API wrapper
- [ ] Implement `services/square_sync_service.py` - Sync service
- [ ] Create Square OAuth flow (get access token from 1Password)
- [ ] Run full sync for Bashful Beauty (8 years of data)
- [ ] Verify data quality in BigQuery
- [ ] Create indexes on BigQuery tables for performance

**Deliverables**:
- ✅ 2000+ Bashful Beauty transactions in BigQuery
- ✅ Customer and employee data synced
- ✅ Service catalog synced
- ✅ Incremental sync working (last 24 hours)

**Testing**:
```sql
-- Verify transaction count
SELECT COUNT(*) FROM keeper.square_cache WHERE business_id = 'bashful_beauty_123';
-- Should return 2000+

-- Verify date range
SELECT MIN(created_at), MAX(created_at) FROM keeper.square_cache WHERE business_id = 'bashful_beauty_123';
-- Should show 2017-present

-- Verify data quality
SELECT COUNT(DISTINCT customer_id) as customers,
       COUNT(DISTINCT employee_id) as employees,
       COUNT(DISTINCT service_id) as services
FROM keeper.square_cache WHERE business_id = 'bashful_beauty_123';
-- Should show ~200 customers, 8 employees, 25 services
```

**Success Criteria**:
- ✅ 100% of Bashful Beauty historical data synced
- ✅ No missing transactions
- ✅ Incremental sync runs in < 30 seconds
- ✅ Data quality score > 95%

---

## Phase 2: Core MCPs - Layers 1-2

### Goals
- Build all Data Access MCPs (Layer 1)
- Build all Analysis MCPs (Layer 2)
- Test each MCP individually with real data

### Tasks

#### Layer 1 - Data Access MCPs (3 MCPs)

**square_data.mcp**
- [ ] Implement `mcps/data_access/square_data.mcp.py`
- [ ] Tools: `get_transactions()`, `get_customers()`, `get_employees()`, `get_catalog()`, `get_customer_timeline()`
- [ ] Test with Bashful Beauty data
- [ ] Add caching layer (Redis) for frequently accessed data

**external_data.mcp**
- [ ] Implement `mcps/data_access/external_data.mcp.py`
- [ ] Tools: `get_reviews()`, `get_competitor_data()`, `get_social_mentions()` (stub implementations for now)
- [ ] Set up Cloud Storage bucket structure

**memory.mcp**
- [ ] Implement `mcps/data_access/memory.mcp.py`
- [ ] Tools: `get_business_profile()`, `get_employee_profiles()`, `save_investigation()`, `get_investigation()`
- [ ] Set up Firestore collections (`businesses/`, `users/`)
- [ ] Set up Redis for Working Memory (7-day TTL)
- [ ] Create Bashful Beauty business profile in Firestore

**Deliverables**:
- ✅ All 3 Layer 1 MCPs implemented and tested
- ✅ Unit tests passing for each MCP
- ✅ Bashful Beauty business profile in Firestore

**Testing**: Create `tests/unit/mcps/test_layer1.py` with tests for each tool

---

#### Layer 2 - Analysis MCPs (7 MCPs)

**pattern_detector.mcp**
- [ ] Implement `mcps/analysis/pattern_detector.mcp.py`
- [ ] Tools: `find_retention_patterns()`, `find_revenue_patterns()`, `find_visit_frequency_patterns()`, `detect_anomalies()`
- [ ] Use SQL + statistical methods (z-scores, significance tests)
- [ ] Test on Bashful Beauty data

**segment_analyzer.mcp**
- [ ] Implement `mcps/analysis/segment_analyzer.mcp.py`
- [ ] Tools: `segment_by_behavior()`, `segment_by_service_preference()`, `segment_by_lifecycle_stage()`
- [ ] Use RFM analysis, clustering
- [ ] Test segmentation accuracy

**time_surgeon.mcp**
- [ ] Implement `mcps/analysis/time_surgeon.mcp.py`
- [ ] Tools: `analyze_by_time_of_day()`, `analyze_by_day_of_week()`, `analyze_appointment_gaps()`
- [ ] Test time-based patterns

**employee_analyzer.mcp**
- [ ] Implement `mcps/analysis/employee_analyzer.mcp.py`
- [ ] Tools: `get_employee_performance()`, `compare_employees()`, `find_training_gaps()`
- [ ] Test on Bashful Beauty employees (Jennifer, Amanda, Marcus, Sarah)

**customer_forensics.mcp**
- [ ] Implement `mcps/analysis/customer_forensics.mcp.py`
- [ ] Tools: `get_customer_journey()`, `predict_churn_risk()`, `find_defection_signals()`
- [ ] Build churn prediction model (BigQuery ML logistic regression)
- [ ] Test on known churned customers

**financial_surgeon.mcp**
- [ ] Implement `mcps/analysis/financial_surgeon.mcp.py`
- [ ] Tools: `analyze_revenue_by_service()`, `find_upsell_opportunities()`, `calculate_service_profitability()`
- [ ] Test margin calculations

**competitor_intel.mcp**
- [ ] Implement `mcps/analysis/competitor_intel.mcp.py`
- [ ] Tools: `find_nearby_competitors()`, `compare_pricing()` (stub for now, will implement scraping in Phase 6)

**Integration Testing**
- [ ] Test all Layer 2 MCPs together
- [ ] Verify outputs are consistent
- [ ] Document expected output formats

**Deliverables**:
- ✅ All 7 Layer 2 MCPs implemented
- ✅ Comprehensive test suite
- ✅ Example discovery data generated

**Success Criteria**:
- ✅ All MCPs return data for Bashful Beauty
- ✅ Unit test coverage > 80%
- ✅ No data quality issues
- ✅ Performance < 5 seconds per MCP tool call

---

## Phase 3: Analyst Agent

### Goals
- Build Analyst Agent with discovery_mode
- Integrate Vertex AI Gemini Pro for synthesis
- Generate first real discoveries

### Tasks

#### Analyst Agent Core

**Base Agent Implementation**
- [ ] Implement `agents/base_agent.py` - Shared functionality
- [ ] Implement `agents/analyst_agent.py` skeleton
- [ ] Set up MCP loading mechanism
- [ ] Create agent configuration (which MCPs to load, in what order)

**Discovery Mode Implementation**
- [ ] Implement `discovery_mode()` method
- [ ] Orchestrate Layer 1 MCPs (data collection)
- [ ] Orchestrate Layer 2 MCPs (analysis)
- [ ] Aggregate findings into structured format

**Add Intelligence Layer (Stub)**
- [ ] Create stub implementations for Intelligence MCPs (Layer 3)
- [ ] `customer_psychology.mcp`, `spa_wisdom.mcp` (basic rules, no LLM yet)
- [ ] Test discovery pipeline without LLM synthesis

**Deliverables**:
- ✅ Analyst Agent generating structured findings
- ✅ All Layer 1 and Layer 2 MCPs orchestrated correctly
- ✅ Sample discoveries (without LLM synthesis yet)

---

#### LLM Integration

**Vertex AI Setup**
- [ ] Set up Vertex AI credentials
- [ ] Implement `utils/llm_client.py` - Wrapper for Gemini Pro and Claude Opus 4
- [ ] Test basic LLM calls (prompt → response)
- [ ] Implement token counting and cost tracking

**Synthesis Implementation**
- [ ] Build synthesis prompts for Analyst Agent
- [ ] Convert MCP findings → LLM context
- [ ] Call Gemini Pro with aggregated findings
- [ ] Parse LLM response into structured insights
- [ ] Test on real Bashful Beauty data

**Validation Layer**
- [ ] Implement `mcps/validation/confidence_scorer.mcp.py`
- [ ] Implement `mcps/validation/impact_predictor.mcp.py`
- [ ] Filter insights by confidence × impact threshold
- [ ] Generate top 5-10 discoveries

**Deliverables**:
- ✅ Analyst Agent calling Gemini Pro successfully
- ✅ LLM generating meaningful insights from MCP data
- ✅ Validation filtering working (high-confidence insights only)
- ✅ Sample discoveries that match quality expected by frontend

**Testing**:
```python
# Test Analyst Agent end-to-end
agent = AnalystAgent('bashful_beauty_123')
insights = agent.discovery_mode()

assert len(insights) >= 5, "Should generate at least 5 insights"
assert all(i['confidence'] >= 80 for i in insights), "All insights should have 80+ confidence"
assert all(i['impact'] >= 1000 for i in insights), "All insights should have $1K+ impact"
```

**Success Criteria**:
- ✅ Analyst Agent generates 5-10 high-quality insights
- ✅ LLM synthesis is coherent and actionable
- ✅ Insights pass validation (80+ confidence, $1K+ impact)
- ✅ Runtime < 3 minutes per business
- ✅ LLM cost < $1.50 per discovery run

---

## Phase 4: Advisor Agent & Actions

### Goals
- Build Advisor Agent to convert insights → actionable discoveries
- Implement Action Layer MCPs (scripts, campaigns, optimizations)
- Generate complete discovery objects for frontend

### Tasks

#### Advisor Agent Core

**Advisor Agent Implementation**
- [ ] Implement `agents/advisor_agent.py`
- [ ] Mode: `action_planning_mode(insight)` - Take Analyst insight, generate action plan
- [ ] Mode: `script_writing_mode(situation)` - Generate conversation scripts
- [ ] Mode: `campaign_mode(goal)` - Design marketing campaigns

**Action Layer MCPs - Part 1**
- [ ] Implement `mcps/action/script_generator.mcp.py`
- [ ] Tools: `generate_customer_outreach_script()`, `generate_sales_script()`, `generate_employee_coaching_script()`
- [ ] Test script quality with real scenarios

**Action Layer MCPs - Part 2**
- [ ] Implement `mcps/action/campaign_builder.mcp.py`
- [ ] Tools: `build_retention_campaign()`, `build_acquisition_campaign()`, `estimate_campaign_roi()`
- [ ] Implement `mcps/action/optimization_architect.mcp.py`
- [ ] Tools: `optimize_scheduling()`, `optimize_pricing()`, `design_process_improvement()`

**Integration**
- [ ] Connect Analyst → Advisor pipeline
- [ ] For each Analyst insight, call Advisor.action_planning_mode()
- [ ] Test end-to-end discovery generation

**Deliverables**:
- ✅ Advisor Agent working
- ✅ Action plans generated for insights
- ✅ Scripts are realistic and actionable
- ✅ End-to-end pipeline: Analyst → Advisor → Discoveries

---

#### Output Formatting & Storage

**Output Formatting**
- [ ] Implement `mcps/validation/output_formatter.mcp.py`
- [ ] Tool: `format_discovery(insight_data)` - Convert to frontend JSON schema
- [ ] Match exact schema from `Keeper_UI/composables/useMockData.ts`
- [ ] Test formatting accuracy

**Firestore Storage**
- [ ] Create Firestore schema for discoveries
- [ ] Collection: `discoveries/{businessId}/insights/{discoveryId}`
- [ ] Implement storage in `services/discovery_service.py`
- [ ] Add discovery stats tracking

**Discovery Service**
- [ ] Implement complete `services/discovery_service.py`
- [ ] Orchestrate: Analyst → Advisor → Format → Store
- [ ] Add error handling and logging
- [ ] Test nightly run simulation

**Local Testing**
- [ ] Run complete discovery pipeline locally
- [ ] Generate discoveries for Bashful Beauty
- [ ] Verify discoveries in Firestore
- [ ] Check discovery quality against mock data examples

**Deliverables**:
- ✅ Complete discovery_service working end-to-end
- ✅ Discoveries stored in Firestore
- ✅ Discoveries match frontend schema exactly
- ✅ Quality matches or exceeds mock data

**Testing**:
```bash
# Run full discovery pipeline
python services/discovery_service.py --business-id bashful_beauty_123

# Verify in Firestore
gcloud firestore query --collection-ids=insights --filter='business_id=bashful_beauty_123'

# Should see 5-10 discoveries with complete data
```

**Success Criteria**:
- ✅ Discovery pipeline runs successfully
- ✅ Discoveries have all required fields
- ✅ Action plans are 3-5 specific steps
- ✅ Scripts are personalized and realistic
- ✅ expectedRecovery includes timeframe and ROI
- ✅ Ray/Steven (owner) reviews discoveries and confirms quality

---

## Phase 5: API & Frontend Integration

### Goals
- Build FastAPI REST API
- Deploy to Cloud Run
- Integrate Keeper_UI frontend with real backend

### Tasks

**FastAPI Setup**
- [ ] Implement `api/main.py` - FastAPI app
- [ ] Implement `api/routes/discoveries.py` - GET /discoveries, POST /discoveries/:id/complete
- [ ] Implement `api/routes/health.py` - GET /health
- [ ] Implement `api/middleware/auth.py` - Firebase token verification
- [ ] Test locally with Postman/curl

**Chat Endpoint (Simple)**
- [ ] Implement basic `agents/investigator_agent.py`
- [ ] Mode: `question_answering_mode(question)` - Answer simple questions about discoveries
- [ ] Implement `api/routes/chat.py` - POST /chat
- [ ] Test chat flow

**Cloud Run Deployment**
- [ ] Create `Dockerfile` for API service
- [ ] Build Docker image
- [ ] Deploy to Cloud Run
- [ ] Set up environment variables in Cloud Run
- [ ] Test deployed API

**Frontend Integration**
- [ ] Update `Keeper_UI/composables/useDiscoveries.ts` to call Cloud Run API
- [ ] Update `Keeper_UI/composables/useAuth.ts` to get Firebase token
- [ ] Test login → fetch discoveries flow
- [ ] Verify discoveries render correctly

**End-to-End Testing**
- [ ] Run discovery pipeline → Store in Firestore
- [ ] Login to Keeper_UI → See real discoveries
- [ ] Mark discovery as complete → Verify in Firestore
- [ ] Test chat → Ask question about discovery
- [ ] Fix any integration issues

**Deliverables**:
- ✅ FastAPI deployed to Cloud Run
- ✅ Keeper_UI showing real discoveries from Bashful Beauty data
- ✅ Mark complete flow working
- ✅ Basic chat working

**Success Criteria**:
- ✅ Frontend and backend fully integrated
- ✅ Discoveries render correctly
- ✅ No CORS or auth issues
- ✅ Response time < 500ms for GET /discoveries

---

## Phase 6: External Intelligence

### Goals
- Build web scraping for external data (reviews, competitors, social)
- Implement External Intelligence MCPs (Layer 5)
- Enhance discoveries with external context

### Tasks

**Web Scraper Service**
- [ ] Implement `services/scraper_service.py`
- [ ] Use Playwright for scraping
- [ ] Methods: `scrape_google_reviews()`, `scrape_yelp_reviews()`, `scrape_competitors()`
- [ ] Store in Cloud Storage
- [ ] Deploy to Cloud Functions

**External Intelligence MCPs - Part 1**
- [ ] Implement `mcps/external_intelligence/review_monitor.mcp.py`
- [ ] Tools: `get_recent_reviews()`, `analyze_sentiment_trends()`, `extract_common_complaints()`
- [ ] Implement `mcps/external_intelligence/competitor_discovery.mcp.py`
- [ ] Tools: `find_new_competitors()`, `track_competitor_reviews()`

**External Intelligence MCPs - Part 2**
- [ ] Implement `mcps/external_intelligence/customer_migration.mcp.py`
- [ ] Tool: `find_lost_customers_at_competitors()` - Cross-reference customer names in competitor reviews
- [ ] Implement `mcps/external_intelligence/social_listening.mcp.py`
- [ ] Tools: `track_brand_mentions()`, `analyze_influencer_impact()`

**Integration with Analyst**
- [ ] Update Analyst Agent to call External Intelligence MCPs
- [ ] Add external data to discovery context
- [ ] Test enhanced discoveries (internal + external data)

**Scheduling**
- [ ] Set up Cloud Scheduler for scraping (daily at 1 AM)
- [ ] Test scheduled scraping
- [ ] Verify external data flows into discoveries

**Deliverables**:
- ✅ Web scraping working for Google reviews, Yelp, competitors
- ✅ External Intelligence MCPs implemented
- ✅ Discoveries enhanced with external context
- ✅ Example: "Sarah Chen left 4-star review at Glow Spa 12 days ago"

**Success Criteria**:
- ✅ Scraping runs successfully without being blocked
- ✅ External data accuracy > 90%
- ✅ Discoveries include external insights
- ✅ Cost < $2/month per business for scraping

---

## Phase 7: Testing & Optimization

### Goals
- Comprehensive testing (unit, integration, end-to-end)
- Performance optimization
- Cost optimization
- Production readiness

### Tasks

**Comprehensive Testing**
- [ ] Write unit tests for all MCPs (target: 80% coverage)
- [ ] Write integration tests for agent pipelines
- [ ] Write end-to-end test (Square sync → Discovery → API → Frontend)
- [ ] Test with multiple business types (spa, restaurant, retail)

**Performance Optimization**
- [ ] Profile discovery pipeline (identify bottlenecks)
- [ ] Optimize BigQuery queries (add indexes, reduce data scanned)
- [ ] Implement caching (Redis) for frequently accessed data
- [ ] Parallelize MCP calls where possible
- [ ] Target: Discovery runtime < 3 minutes per business

**Cost Optimization**
- [ ] Analyze LLM token usage
- [ ] Optimize prompts (reduce tokens while maintaining quality)
- [ ] Implement dynamic model selection (Gemini Flash for low-priority, Opus 4 for critical)
- [ ] Batch API calls to reduce overhead
- [ ] Target: LLM cost < $6/month per business

**Production Setup**
- [ ] Set up Cloud Scheduler for nightly discovery (3 AM)
- [ ] Set up Cloud Scheduler for Square sync (12:01 AM)
- [ ] Set up Cloud Scheduler for scraping (1 AM)
- [ ] Configure monitoring and alerting (Cloud Monitoring)
- [ ] Set up error logging (Cloud Logging)
- [ ] Create runbook for common issues

**Acceptance Testing**
- [ ] Run discovery pipeline for Bashful Beauty
- [ ] Steven/manager reviews discoveries for quality
- [ ] Validate action plans are actionable
- [ ] Verify discoveries lead to business improvements
- [ ] Get sign-off from Ray

**Deliverables**:
- ✅ Comprehensive test suite (80%+ coverage)
- ✅ Performance optimized (< 3 min per business)
- ✅ Cost optimized (< $8/month COGS)
- ✅ Production infrastructure ready
- ✅ Monitoring and alerting configured
- ✅ Discoveries approved by business owner

**Success Criteria**:
- ✅ All tests passing
- ✅ Discovery quality meets or exceeds expectations
- ✅ Performance meets SLA (< 3 min runtime)
- ✅ Cost per customer < $8/month
- ✅ System runs reliably for 7 consecutive nights
- ✅ Zero critical bugs

---

## Post-MVP: Future Enhancements

After completing the MVP, prioritize these enhancements:

### Scale & Reliability
- [ ] Multi-business support (not just Bashful Beauty)
- [ ] Onboarding flow (new business signup → data sync → first discoveries)
- [ ] Pattern Memory implementation (Vertex AI Vector Search)
- [ ] Historical outcome tracking (did discoveries lead to results?)

### Advanced Features
- [ ] Investigator Agent enhancements (deep chat, explanations)
- [ ] Crisis Manager MCP (viral negative review detection)
- [ ] Industry Wisdom MCPs for restaurant, retail (currently just spa)
- [ ] Campaign builder automation (actually send emails/SMS)

### Intelligence Improvements
- [ ] Advanced customer psychology (personalization)
- [ ] Predictive analytics (forecast revenue, churn)
- [ ] A/B testing framework (test action plan variations)
- [ ] Multi-language support

---

## Key Metrics to Track

### Development Metrics
- **Code coverage**: Target 80%+
- **MCPs implemented**: 31 total
- **Agents implemented**: 3 total
- **Test pass rate**: 100%

### Performance Metrics
- **Discovery runtime**: < 3 minutes per business
- **API response time**: < 500ms for GET /discoveries
- **LLM latency**: < 10 seconds per synthesis call
- **BigQuery query time**: < 2 seconds per query

### Cost Metrics
- **LLM cost**: < $6/month per business
- **Infrastructure cost**: < $2/month per business
- **Total COGS**: < $8/month per business
- **Gross margin**: > 90% at $99/month pricing

### Quality Metrics
- **Discovery confidence**: Average 85%+
- **Discovery accuracy**: 90%+ validated by business owners
- **Actionability score**: 95%+ of discoveries have clear action plans
- **Business owner satisfaction**: 4.5+ stars

---

## Risk Management

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Square API rate limits | Medium | High | Cache data in BigQuery, use webhooks |
| LLM hallucinations | Medium | High | Validation layer, confidence scoring |
| BigQuery costs exceed budget | Low | Medium | Query optimization, caching |
| Web scraping blocked | Medium | Medium | Rotate user agents, use proxies if needed |
| Vertex AI latency > 10s | Low | Medium | Batch requests, use streaming |

### Product Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Discoveries not actionable | Medium | High | Test with real business owner, iterate |
| Insights are obvious (no value) | Medium | High | Multi-dimensional analysis, external data |
| Action plans too complex | Low | Medium | Keep to 3-5 steps, test with users |
| Users don't act on discoveries | Medium | High | Better UX, reminders, success stories |

### Business Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| COGS > revenue | Low | Critical | Cost optimization, efficient LLM usage |
| Square changes API | Low | High | Monitor API changelog, build abstraction layer |
| Competitor launches similar product | Medium | Medium | Speed to market, superior quality |

---

## Success Definition

**MVP is successful if**:
1. ✅ Discoveries are generated nightly for Bashful Beauty
2. ✅ Steven (owner) or manager can log in and see 5-10 actionable insights
3. ✅ Discoveries are accurate, insightful, and lead to business improvements
4. ✅ System runs reliably without manual intervention
5. ✅ Cost per customer < $8/month
6. ✅ Ray approves quality and gives green light to onboard more businesses

**Long-term success if**:
1. ✅ 100+ businesses using Keeper within 6 months
2. ✅ Average discovery quality rating > 4.5 stars
3. ✅ Customers report measurable ROI (recovered revenue, retained customers)
4. ✅ System scales to 1000+ businesses without infrastructure changes

---

## Resources

- **Architecture**: `KEEPER_ARCHITECTURE.md`
- **Repository Structure**: `REPOSITORY_STRUCTURE.md`
- **Quick Start**: `QUICK_START_GUIDE.md`
- **Anthropic MCP**: https://www.anthropic.com/engineering/code-execution-with-mcp
- **Keeper_UI Repo**: https://github.com/defendersoftheinternet/Keeper_UI
- **Square API**: https://developer.squareup.com/docs
- **GCP Docs**: https://cloud.google.com/docs

---

**Let's build something transformative for small business owners.** 🚀

—Ray & Claude
