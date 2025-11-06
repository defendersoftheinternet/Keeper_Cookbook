# Keeper Repository Structure

This document outlines the recommended repository structure for the Keeper AI backend (Keeper_Cookbook).

## Recommended Structure

```
Keeper_Cookbook/
├── README.md                          # Repository overview and quick start
├── ARCHITECTURE.md                    # Complete architecture spec (see KEEPER_ARCHITECTURE.md)
├── docker-compose.yml                 # Local development environment
├── .env.example                       # Environment variables template
├── requirements.txt                   # Python dependencies
│
├── agents/                            # 3 Orchestrator Agents
│   ├── __init__.py
│   ├── analyst_agent.py              # Deep data investigation and pattern discovery
│   ├── advisor_agent.py              # Convert insights to actionable recommendations
│   ├── investigator_agent.py         # Answer user questions and investigate concerns
│   └── base_agent.py                 # Shared agent functionality
│
├── mcps/                              # 31 MCP Tools (organized by layer)
│   ├── __init__.py
│   │
│   ├── data_access/                  # Layer 1: Data Access (3 MCPs)
│   │   ├── __init__.py
│   │   ├── square_data.mcp.py       # Access cached Square data from BigQuery
│   │   ├── external_data.mcp.py     # Fetch scraped external data
│   │   └── memory.mcp.py            # Three-tier memory system
│   │
│   ├── analysis/                     # Layer 2: Analysis (7 MCPs)
│   │   ├── __init__.py
│   │   ├── pattern_detector.mcp.py  # Statistical anomalies and trends
│   │   ├── segment_analyzer.mcp.py  # Customer segmentation and cohort analysis
│   │   ├── time_surgeon.mcp.py      # Time-based pattern analysis
│   │   ├── employee_analyzer.mcp.py # Employee performance analysis
│   │   ├── customer_forensics.mcp.py # Deep dive into individual customer behavior
│   │   ├── financial_surgeon.mcp.py  # Revenue analysis and pricing optimization
│   │   └── competitor_intel.mcp.py   # Competitive landscape analysis
│   │
│   ├── intelligence/                 # Layer 3: Intelligence (4 MCPs)
│   │   ├── __init__.py
│   │   ├── customer_psychology.mcp.py   # Behavioral psychology for customers
│   │   ├── employee_psychology.mcp.py   # Employee performance psychology
│   │   ├── owner_psychology.mcp.py      # Tailor communication to owner preferences
│   │   └── market_dynamics.mcp.py       # Market trends and economic context
│   │
│   ├── wisdom/                       # Layer 4: Wisdom (4 MCPs)
│   │   ├── __init__.py
│   │   ├── spa_wisdom.mcp.py        # Spa & wellness industry best practices
│   │   ├── restaurant_wisdom.mcp.py # Restaurant & food service best practices
│   │   ├── retail_wisdom.mcp.py     # Retail best practices
│   │   └── service_wisdom.mcp.py    # General service business best practices
│   │
│   ├── external_intelligence/       # Layer 5: External Intelligence (5 MCPs)
│   │   ├── __init__.py
│   │   ├── review_monitor.mcp.py    # Analyze online reviews for insights
│   │   ├── competitor_discovery.mcp.py  # Monitor competitor activity
│   │   ├── customer_migration.mcp.py    # Track customer movement to/from competitors
│   │   ├── social_listening.mcp.py      # Monitor social media for business intelligence
│   │   └── local_market.mcp.py          # Local market intelligence
│   │
│   ├── action/                       # Layer 6: Action (4 MCPs)
│   │   ├── __init__.py
│   │   ├── script_generator.mcp.py  # Generate conversation scripts for staff
│   │   ├── campaign_builder.mcp.py  # Design marketing campaigns
│   │   ├── optimization_architect.mcp.py # Design operational improvements
│   │   └── crisis_manager.mcp.py    # Handle urgent business issues
│   │
│   └── validation/                   # Layer 7: Validation (4 MCPs)
│       ├── __init__.py
│       ├── data_validator.mcp.py    # Ensure data quality before analysis
│       ├── confidence_scorer.mcp.py # Calculate confidence scores for insights
│       ├── impact_predictor.mcp.py  # Estimate financial impact
│       └── output_formatter.mcp.py  # Format discoveries for frontend
│
├── services/                         # Core Services
│   ├── __init__.py
│   ├── discovery_service.py         # Nightly discovery pipeline (orchestrates agents)
│   ├── square_sync_service.py       # Sync Square data to BigQuery
│   ├── scraper_service.py           # Web scraping for external data
│   └── chat_service.py              # Handle chat interactions
│
├── api/                              # FastAPI REST API
│   ├── __init__.py
│   ├── main.py                      # FastAPI app entrypoint
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── discoveries.py           # GET /discoveries, POST /discoveries/:id/complete
│   │   ├── chat.py                  # POST /chat
│   │   └── health.py                # GET /health
│   ├── middleware/
│   │   ├── __init__.py
│   │   └── auth.py                  # Firebase token verification
│   └── models/
│       ├── __init__.py
│       ├── discovery.py             # Pydantic models for discoveries
│       └── chat.py                  # Pydantic models for chat
│
├── scripts/                          # Utility Scripts
│   ├── deploy.sh                    # Deploy to Cloud Run
│   ├── setup_gcp.sh                 # Initialize GCP infrastructure
│   ├── seed_data.sh                 # Load test data
│   └── run_local_discovery.py       # Run discovery locally for testing
│
├── config/                           # Configuration Files
│   ├── __init__.py
│   ├── settings.py                  # Application settings (env vars, constants)
│   ├── bigquery_schemas.py          # BigQuery table schemas
│   └── firestore_schemas.py         # Firestore document schemas
│
├── utils/                            # Shared Utilities
│   ├── __init__.py
│   ├── bigquery_client.py           # BigQuery helper functions
│   ├── firestore_client.py          # Firestore helper functions
│   ├── redis_client.py              # Redis helper functions
│   ├── square_client.py             # Square API client wrapper
│   ├── llm_client.py                # Vertex AI / Anthropic client wrappers
│   └── logging.py                   # Structured logging setup
│
├── tests/                            # Tests
│   ├── __init__.py
│   ├── unit/                        # Unit tests for individual MCPs
│   │   ├── mcps/
│   │   ├── agents/
│   │   └── services/
│   ├── integration/                 # Integration tests
│   │   ├── test_discovery_pipeline.py
│   │   ├── test_square_sync.py
│   │   └── test_scraper.py
│   └── fixtures/                    # Test data
│       ├── square_data.json
│       ├── external_data.json
│       └── expected_discoveries.json
│
├── infrastructure/                   # Infrastructure as Code
│   ├── terraform/                   # Terraform configs for GCP
│   │   ├── main.tf
│   │   ├── bigquery.tf
│   │   ├── firestore.tf
│   │   ├── cloud_run.tf
│   │   └── cloud_scheduler.tf
│   └── scripts/
│       ├── setup_bigquery.sh
│       ├── setup_firestore.sh
│       └── setup_scheduler.sh
│
├── notebooks/                        # Jupyter notebooks for analysis
│   ├── explore_bashful_beauty_data.ipynb
│   ├── test_mcp_outputs.ipynb
│   └── validate_discoveries.ipynb
│
├── docs/                             # Additional Documentation
│   ├── MCP_GUIDE.md                 # How to write new MCPs
│   ├── AGENT_GUIDE.md               # How agents work
│   ├── DEPLOYMENT.md                # Deployment instructions
│   ├── TESTING.md                   # Testing strategies
│   └── TROUBLESHOOTING.md           # Common issues and solutions
│
├── .github/                          # GitHub Actions
│   └── workflows/
│       ├── test.yml                 # Run tests on PR
│       ├── deploy-staging.yml       # Deploy to staging on merge to main
│       └── deploy-production.yml    # Deploy to production on release
│
├── Dockerfile                        # Docker image for Cloud Run
├── .dockerignore
├── .gitignore
└── pyproject.toml                    # Python project metadata (Poetry)
```

## Data Flow Between Components

```
┌─────────────────────────────────────────────────────────────────┐
│                     User Login (Morning)                         │
│  Keeper_UI → api/routes/discoveries.py → Firestore              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  Nightly Discovery Flow (3 AM)                   │
│  1. Cloud Scheduler triggers Pub/Sub                             │
│  2. services/discovery_service.py starts                         │
│  3. Loads agents/analyst_agent.py                                │
│  4. Analyst calls mcps/data_access/square_data.mcp.py            │
│  5. Analyst calls mcps/analysis/*.mcp.py (all 7 MCPs)            │
│  6. Analyst calls mcps/intelligence/*.mcp.py (context)           │
│  7. Analyst calls Vertex AI Gemini Pro (synthesis)               │
│  8. Analyst calls mcps/validation/*.mcp.py (validate)            │
│  9. Loads agents/advisor_agent.py                                │
│ 10. Advisor calls mcps/action/*.mcp.py (generate plans)          │
│ 11. Advisor calls Claude Opus 4 / Gemini Pro (generate)          │
│ 12. Advisor calls mcps/validation/output_formatter.mcp.py        │
│ 13. services/discovery_service.py stores in Firestore            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Square Data Sync (12:01 AM)                   │
│  1. Cloud Scheduler triggers services/square_sync_service.py     │
│  2. Fetches data via utils/square_client.py                      │
│  3. Stores in BigQuery via utils/bigquery_client.py              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   Web Scraping (1:00 AM)                         │
│  1. Cloud Scheduler triggers services/scraper_service.py         │
│  2. Scrapes Google, Yelp, competitors with Playwright            │
│  3. Stores in Cloud Storage (gs://keeper-external-data/)         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       User Chat                                  │
│  1. Keeper_UI → api/routes/chat.py                               │
│  2. Loads agents/investigator_agent.py                           │
│  3. Investigator calls relevant MCPs                             │
│  4. Investigator calls Gemini Pro (answer)                       │
│  5. Response returned to Keeper_UI                               │
└─────────────────────────────────────────────────────────────────┘
```

## Key Files Explained

### agents/analyst_agent.py
**Purpose**: Deep data investigation and pattern discovery

**Key Methods**:
- `discovery_mode()` - Nightly comprehensive analysis (main method)
- `drill_down_mode(topic)` - User-requested deep dive
- `validation_mode(insight)` - Verify insights before publishing

**Dependencies**:
- All 31 MCPs (loads dynamically based on analysis needs)
- Vertex AI Gemini Pro for synthesis
- Pattern Memory (vector search) for historical context

### agents/advisor_agent.py
**Purpose**: Convert insights into actionable recommendations

**Key Methods**:
- `action_planning_mode(insight)` - Generate 3-5 step action plans
- `script_writing_mode(situation)` - Create conversation scripts
- `campaign_mode(goal)` - Design marketing campaigns

**Dependencies**:
- Action Layer MCPs (script_generator, campaign_builder, optimization_architect)
- Validation Layer MCPs (output_formatter)
- Claude Opus 4 for high-impact discoveries
- Gemini Pro for standard discoveries

### agents/investigator_agent.py
**Purpose**: Answer user questions and investigate specific concerns

**Key Methods**:
- `question_answering_mode(question, thread_id)` - Handle chat queries
- `investigation_mode(topic)` - Deep dive into user-specified issues
- `explanation_mode(discovery_id)` - Explain why discoveries were generated

**Dependencies**:
- Relevant MCPs based on question type
- Working Memory (Redis) for conversation context
- Gemini Pro for conversational responses

### mcps/data_access/square_data.mcp.py
**Purpose**: Access cached Square data from BigQuery

**Key Methods**:
- `get_transactions(start_date, end_date, filters)` - Fetch transactions
- `get_customers(filters)` - Customer profiles with visit history
- `get_employees(filters)` - Employee performance metrics
- `get_catalog()` - Services, products, pricing
- `get_customer_timeline(customer_id)` - Full visit history

**Data Source**: BigQuery table `keeper.square_cache`

### services/discovery_service.py
**Purpose**: Orchestrate nightly discovery pipeline

**Key Methods**:
- `run_discovery(business_id)` - Main entrypoint (called by Cloud Scheduler)
- `_initialize_agents(business_id)` - Load Analyst and Advisor agents
- `_store_discoveries(discoveries)` - Save to Firestore
- `_update_stats(business_id, discoveries)` - Update business stats

**Flow**:
1. Initialize agents
2. Run Analyst.discovery_mode()
3. For each insight, run Advisor.action_planning_mode()
4. Store discoveries in Firestore
5. Update business stats

### services/square_sync_service.py
**Purpose**: Sync Square data to BigQuery (runs nightly at 12:01 AM)

**Key Methods**:
- `sync_business(business_id)` - Sync one business
- `_fetch_square_data(business_id, start_date)` - Fetch from Square API
- `_transform_data(raw_data)` - Transform to BigQuery schema
- `_load_to_bigquery(data)` - Insert into BigQuery

**Strategy**:
- Incremental sync (last 24 hours)
- Full resync weekly (Sunday 2 AM)
- Handle Square API rate limits
- Store OAuth tokens in Firestore (encrypted)

### services/scraper_service.py
**Purpose**: Scrape external data (reviews, competitors) via Playwright

**Key Methods**:
- `scrape_reviews(business_id)` - Google, Yelp reviews
- `scrape_competitors(business_id, radius)` - Competitor data
- `scrape_social(business_id)` - Instagram/Facebook mentions
- `_store_to_cloud_storage(data, path)` - Save to GCS

**Strategy**:
- Use Playwright (headless browser)
- Rotate user agents to avoid detection
- Cache results in Cloud Storage
- Run nightly at 1 AM (before discovery at 3 AM)

### api/main.py
**Purpose**: FastAPI REST API for Keeper_UI

**Key Routes**:
- `GET /discoveries?businessId=...` - Fetch discoveries
- `POST /discoveries/:id/complete` - Mark discovery as completed
- `POST /chat` - Handle chat message
- `GET /health` - Health check

**Authentication**: Firebase ID token verification

## Environment Variables

Required environment variables (see `.env.example`):

```bash
# Google Cloud
GCP_PROJECT_ID=keeper-production
GCP_REGION=us-central1

# BigQuery
BIGQUERY_DATASET=keeper
BIGQUERY_TABLE_SQUARE_CACHE=square_cache
BIGQUERY_TABLE_PATTERN_HISTORY=pattern_history

# Firestore
FIRESTORE_DATABASE=(default)

# Redis
REDIS_HOST=10.0.0.3  # Memorystore IP
REDIS_PORT=6379

# Cloud Storage
GCS_BUCKET_EXTERNAL_DATA=keeper-external-data
GCS_BUCKET_MCP_FILES=keeper-mcp-files

# Vertex AI
VERTEX_AI_LOCATION=us-central1
VERTEX_AI_MODEL_GEMINI=gemini-2.0-pro
VERTEX_AI_VECTOR_SEARCH_INDEX=keeper-patterns

# Anthropic API
ANTHROPIC_API_KEY=sk-ant-...

# Square API (per business, stored in Firestore)
# SQUARE_ACCESS_TOKEN and SQUARE_REFRESH_TOKEN managed via OAuth

# Application
LOG_LEVEL=INFO
ENVIRONMENT=production  # or staging, development
```

## Development Workflow

### 1. Local Development Setup

```bash
# Clone repository
git clone https://github.com/defendersoftheinternet/Keeper_Cookbook.git
cd Keeper_Cookbook

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your values

# Start local services (BigQuery emulator, Firestore emulator, Redis)
docker-compose up -d

# Run tests
pytest tests/

# Run discovery locally (against real Bashful Beauty data)
python scripts/run_local_discovery.py --business-id bashful_beauty_123
```

### 2. Adding a New MCP

See `docs/MCP_GUIDE.md` for detailed instructions.

```python
# mcps/analysis/new_analyzer.mcp.py
from typing import List, Dict, Any
from utils.bigquery_client import BigQueryClient

class NewAnalyzerMCP:
    def __init__(self, business_id: str):
        self.business_id = business_id
        self.bq = BigQueryClient()

    def analyze_something(self, params: Dict[str, Any]) -> List[Dict]:
        """Analyze something and return insights"""
        # Implementation here
        pass
```

### 3. Testing

```bash
# Unit tests (fast, no external dependencies)
pytest tests/unit/

# Integration tests (requires GCP credentials)
pytest tests/integration/

# Test specific MCP
pytest tests/unit/mcps/test_pattern_detector.py

# Test with real Bashful Beauty data
pytest tests/integration/test_discovery_pipeline.py --use-real-data
```

### 4. Deployment

```bash
# Deploy to staging
./scripts/deploy.sh staging

# Deploy to production
./scripts/deploy.sh production

# Deploy specific service
gcloud run deploy keeper-api --source ./api --region us-central1
```

## Next Steps for Implementation

1. **Set up GCP project** (see `infrastructure/scripts/setup_*.sh`)
2. **Implement Layer 1 MCPs** (data access) first
3. **Build Analyst Agent** with discovery_mode
4. **Test with Bashful Beauty data** to validate outputs
5. **Implement remaining MCPs** layer by layer
6. **Build Advisor Agent** and output formatting
7. **Deploy to Cloud Run** and test end-to-end
8. **Integrate with Keeper_UI** frontend

## Resources

- **Main Architecture Spec**: See `KEEPER_ARCHITECTURE.md` in Keeper_UI repo
- **Anthropic MCP Article**: https://www.anthropic.com/engineering/code-execution-with-mcp
- **GCP Documentation**: https://cloud.google.com/docs
- **Square API Docs**: https://developer.squareup.com/docs
- **Keeper_UI Repo**: https://github.com/defendersoftheinternet/Keeper_UI
