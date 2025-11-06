# Keeper Architecture - Mermaid Diagrams

This document contains visual architecture diagrams for the Keeper AI system.

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "User Layer"
        UI[Keeper UI<br/>Nuxt 3 Frontend]
    end

    subgraph "API Gateway - Cloud Run"
        API[FastAPI REST API<br/>- GET /discoveries<br/>- POST /chat<br/>- POST /complete]
    end

    subgraph "Discovery Engine - Cloud Run"
        DE[Discovery Service<br/>Runs nightly at 3 AM]

        subgraph "3 Orchestrator Agents"
            ANALYST[Analyst Agent<br/>Deep Investigation]
            ADVISOR[Advisor Agent<br/>Action Planning]
            INVESTIGATOR[Investigator Agent<br/>Chat Handler]
        end
    end

    subgraph "31 MCP Tools"
        subgraph "Layer 1: Data Access"
            MCP1[square_data.mcp]
            MCP2[external_data.mcp]
            MCP3[memory.mcp]
        end

        subgraph "Layer 2: Analysis"
            MCP4[pattern_detector.mcp]
            MCP5[segment_analyzer.mcp]
            MCP6[employee_analyzer.mcp]
            MCP7[customer_forensics.mcp]
            MCP8[financial_surgeon.mcp]
            MCP9[time_surgeon.mcp]
            MCP10[competitor_intel.mcp]
        end

        subgraph "Layer 3: Intelligence"
            MCP11[customer_psychology.mcp]
            MCP12[employee_psychology.mcp]
            MCP13[owner_psychology.mcp]
            MCP14[market_dynamics.mcp]
        end

        subgraph "Layer 4: Wisdom"
            MCP15[spa_wisdom.mcp]
            MCP16[restaurant_wisdom.mcp]
            MCP17[retail_wisdom.mcp]
            MCP18[service_wisdom.mcp]
        end

        subgraph "Layer 5: External Intelligence"
            MCP19[review_monitor.mcp]
            MCP20[competitor_discovery.mcp]
            MCP21[customer_migration.mcp]
            MCP22[social_listening.mcp]
            MCP23[local_market.mcp]
        end

        subgraph "Layer 6: Action"
            MCP24[script_generator.mcp]
            MCP25[campaign_builder.mcp]
            MCP26[optimization_architect.mcp]
            MCP27[crisis_manager.mcp]
        end

        subgraph "Layer 7: Validation"
            MCP28[data_validator.mcp]
            MCP29[confidence_scorer.mcp]
            MCP30[impact_predictor.mcp]
            MCP31[output_formatter.mcp]
        end
    end

    subgraph "Data Storage - GCP"
        BQ[(BigQuery<br/>Square Cache<br/>Pattern History)]
        FS[(Firestore<br/>Discoveries<br/>Business Profiles)]
        REDIS[(Redis<br/>Working Memory<br/>7-day TTL)]
        GCS[(Cloud Storage<br/>External Data<br/>Scraped Content)]
        VS[(Vector Search<br/>Pattern Embeddings)]
    end

    subgraph "AI/ML Services"
        GEMINI[Vertex AI<br/>Gemini Pro<br/>$1.69/month]
        CLAUDE[Anthropic API<br/>Claude Opus 4<br/>$4.32/month]
    end

    subgraph "Data Pipeline"
        SYNC[Square Sync<br/>12:01 AM daily]
        SCRAPER[Web Scraper<br/>Playwright<br/>1:00 AM daily]
    end

    subgraph "Orchestration"
        SCHEDULER[Cloud Scheduler]
        PUBSUB[Pub/Sub]
    end

    %% User interactions
    UI -->|HTTPS| API
    API -->|Fetch| FS
    API -->|Chat| INVESTIGATOR

    %% Nightly discovery flow
    SCHEDULER -->|Trigger| PUBSUB
    PUBSUB -->|Invoke| DE
    DE --> ANALYST
    ANALYST -->|Insights| ADVISOR
    ADVISOR -->|Discoveries| FS

    %% Agents use MCPs
    ANALYST -.->|Uses| MCP1
    ANALYST -.->|Uses| MCP4
    ANALYST -.->|Uses| MCP11
    ANALYST -.->|Uses| MCP19
    ADVISOR -.->|Uses| MCP24
    ADVISOR -.->|Uses| MCP28
    INVESTIGATOR -.->|Uses| MCP1

    %% Agents use LLMs
    ANALYST -->|Synthesis| GEMINI
    ANALYST -->|High-impact| CLAUDE
    ADVISOR -->|Generation| GEMINI
    ADVISOR -->|Critical| CLAUDE
    INVESTIGATOR -->|Chat| GEMINI

    %% MCPs access data
    MCP1 --> BQ
    MCP2 --> GCS
    MCP3 --> FS
    MCP3 --> REDIS
    MCP3 --> VS
    MCP4 --> BQ
    MCP5 --> BQ

    %% Data pipeline
    SCHEDULER -->|12:01 AM| SYNC
    SCHEDULER -->|1:00 AM| SCRAPER
    SYNC -->|Insert| BQ
    SCRAPER -->|Upload| GCS

    style ANALYST fill:#4A90E2
    style ADVISOR fill:#7B68EE
    style INVESTIGATOR fill:#50C878
    style GEMINI fill:#FFD700
    style CLAUDE fill:#FF6B6B
```

## 2. MCP Layer Organization

```mermaid
graph LR
    subgraph "Data Sources"
        SQUARE[Square API<br/>Transactions<br/>Customers<br/>Employees]
        WEB[Web Scraping<br/>Reviews<br/>Competitors<br/>Social Media]
    end

    subgraph "Layer 1: Data Access"
        L1A[square_data.mcp<br/>Access BigQuery cache]
        L1B[external_data.mcp<br/>Fetch scraped data]
        L1C[memory.mcp<br/>3-tier memory]
    end

    subgraph "Layer 2: Analysis"
        L2A[pattern_detector.mcp<br/>Statistical anomalies]
        L2B[segment_analyzer.mcp<br/>Customer cohorts]
        L2C[employee_analyzer.mcp<br/>Performance metrics]
        L2D[customer_forensics.mcp<br/>Individual behavior]
        L2E[financial_surgeon.mcp<br/>Revenue analysis]
        L2F[time_surgeon.mcp<br/>Time patterns]
        L2G[competitor_intel.mcp<br/>Market landscape]
    end

    subgraph "Layer 3: Intelligence"
        L3A[customer_psychology.mcp<br/>Behavior interpretation]
        L3B[employee_psychology.mcp<br/>Performance diagnosis]
        L3C[owner_psychology.mcp<br/>Communication style]
        L3D[market_dynamics.mcp<br/>Industry benchmarks]
    end

    subgraph "Layer 4: Wisdom"
        L4A[spa_wisdom.mcp<br/>Spa best practices]
        L4B[restaurant_wisdom.mcp<br/>Restaurant standards]
        L4C[retail_wisdom.mcp<br/>Retail expertise]
        L4D[service_wisdom.mcp<br/>Service business]
    end

    subgraph "Layer 5: External Intelligence"
        L5A[review_monitor.mcp<br/>Sentiment analysis]
        L5B[competitor_discovery.mcp<br/>Competitor tracking]
        L5C[customer_migration.mcp<br/>Customer movement]
        L5D[social_listening.mcp<br/>Social mentions]
        L5E[local_market.mcp<br/>Local events]
    end

    subgraph "Layer 6: Action"
        L6A[script_generator.mcp<br/>Conversation scripts]
        L6B[campaign_builder.mcp<br/>Marketing campaigns]
        L6C[optimization_architect.mcp<br/>Process improvements]
        L6D[crisis_manager.mcp<br/>Urgent issues]
    end

    subgraph "Layer 7: Validation"
        L7A[data_validator.mcp<br/>Data quality]
        L7B[confidence_scorer.mcp<br/>Confidence scores]
        L7C[impact_predictor.mcp<br/>Financial impact]
        L7D[output_formatter.mcp<br/>JSON formatting]
    end

    subgraph "Output"
        DISC[Discovery Object<br/>JSON format for UI]
    end

    SQUARE --> L1A
    WEB --> L1B

    L1A --> L2A
    L1A --> L2B
    L1A --> L2C
    L1B --> L2G

    L2A --> L3A
    L2C --> L3B

    L2A --> L4A

    L1B --> L5A
    L1B --> L5B

    L3A --> L6A
    L2A --> L6B

    L2A --> L7B
    L6A --> L7D

    L7D --> DISC

    style L1A fill:#E3F2FD
    style L1B fill:#E3F2FD
    style L1C fill:#E3F2FD
    style L2A fill:#FFF3E0
    style L2B fill:#FFF3E0
    style L2C fill:#FFF3E0
    style L2D fill:#FFF3E0
    style L2E fill:#FFF3E0
    style L2F fill:#FFF3E0
    style L2G fill:#FFF3E0
    style L3A fill:#F3E5F5
    style L3B fill:#F3E5F5
    style L3C fill:#F3E5F5
    style L3D fill:#F3E5F5
    style L4A fill:#E8F5E9
    style L4B fill:#E8F5E9
    style L4C fill:#E8F5E9
    style L4D fill:#E8F5E9
    style L5A fill:#FFF9C4
    style L5B fill:#FFF9C4
    style L5C fill:#FFF9C4
    style L5D fill:#FFF9C4
    style L5E fill:#FFF9C4
    style L6A fill:#FFEBEE
    style L6B fill:#FFEBEE
    style L6C fill:#FFEBEE
    style L6D fill:#FFEBEE
    style L7A fill:#E0F2F1
    style L7B fill:#E0F2F1
    style L7C fill:#E0F2F1
    style L7D fill:#E0F2F1
    style DISC fill:#4CAF50
```

## 3. Nightly Discovery Flow (Sequence Diagram)

```mermaid
sequenceDiagram
    participant SCHED as Cloud Scheduler
    participant SYNC as Square Sync
    participant SCRAPER as Web Scraper
    participant BQ as BigQuery
    participant GCS as Cloud Storage
    participant DE as Discovery Engine
    participant ANALYST as Analyst Agent
    participant MCPs as MCP Tools
    participant GEMINI as Gemini Pro
    participant ADVISOR as Advisor Agent
    participant CLAUDE as Claude Opus 4
    participant FS as Firestore

    Note over SCHED: 12:01 AM - Data Sync
    SCHED->>SYNC: Trigger Square sync
    SYNC->>BQ: Fetch last 24 hours
    SYNC->>BQ: Insert/Update transactions
    Note over BQ: Square data cached

    Note over SCHED: 1:00 AM - Web Scraping
    SCHED->>SCRAPER: Trigger scraper
    SCRAPER->>GCS: Scrape Google reviews
    SCRAPER->>GCS: Scrape Yelp reviews
    SCRAPER->>GCS: Scrape competitor sites
    SCRAPER->>GCS: Scrape social media
    Note over GCS: External data ready

    Note over SCHED: 3:00 AM - Discovery Generation
    SCHED->>DE: Trigger discovery run
    DE->>ANALYST: Start discovery_mode()

    Note over ANALYST,MCPs: Data Collection Phase (5-10s)
    ANALYST->>MCPs: square_data.get_transactions()
    MCPs->>BQ: Query transactions
    BQ-->>MCPs: Return data
    MCPs-->>ANALYST: Transaction list

    ANALYST->>MCPs: external_data.get_reviews()
    MCPs->>GCS: Fetch reviews
    GCS-->>MCPs: Return reviews
    MCPs-->>ANALYST: Review data

    Note over ANALYST,MCPs: Analysis Phase (30-60s)
    ANALYST->>MCPs: pattern_detector.find_retention_patterns()
    MCPs->>BQ: Run SQL analysis
    BQ-->>MCPs: Pattern results
    MCPs-->>ANALYST: Retention patterns

    ANALYST->>MCPs: segment_analyzer.segment_by_behavior()
    MCPs->>BQ: RFM analysis
    BQ-->>MCPs: Segments
    MCPs-->>ANALYST: Customer segments

    ANALYST->>MCPs: employee_analyzer.compare_employees()
    MCPs->>BQ: Performance metrics
    BQ-->>MCPs: Employee stats
    MCPs-->>ANALYST: Performance data

    Note over ANALYST,MCPs: Intelligence Phase (20-30s)
    ANALYST->>MCPs: customer_psychology.explain_behavior()
    MCPs-->>ANALYST: Psychological context

    ANALYST->>MCPs: spa_wisdom.get_best_practice()
    MCPs-->>ANALYST: Industry best practices

    Note over ANALYST,GEMINI: Synthesis Phase (60-90s)
    ANALYST->>GEMINI: Synthesize all MCP findings
    Note over GEMINI: Process 150K tokens<br/>Generate 10-15 insights
    GEMINI-->>ANALYST: Raw insights with reasoning

    Note over ANALYST,MCPs: Validation Phase (10-20s)
    loop For each insight
        ANALYST->>MCPs: confidence_scorer.calculate()
        MCPs-->>ANALYST: Confidence: 87%

        ANALYST->>MCPs: impact_predictor.predict_revenue()
        MCPs-->>ANALYST: Impact: $6,252/year
    end

    ANALYST-->>DE: 10 validated insights

    Note over ADVISOR: Action Planning (30-60s per insight)
    DE->>ADVISOR: Convert to discoveries

    loop For each insight
        ADVISOR->>MCPs: script_generator.generate()
        MCPs-->>ADVISOR: Conversation script

        ADVISOR->>MCPs: campaign_builder.build()
        MCPs-->>ADVISOR: Marketing campaign

        alt High-impact discovery
            ADVISOR->>CLAUDE: Generate action plan
            Note over CLAUDE: 8K tokens<br/>$0.72 cost
            CLAUDE-->>ADVISOR: Detailed action plan
        else Standard discovery
            ADVISOR->>GEMINI: Generate action plan
            GEMINI-->>ADVISOR: Action plan
        end

        ADVISOR->>MCPs: output_formatter.format()
        MCPs-->>ADVISOR: JSON discovery object
    end

    ADVISOR-->>DE: 10 formatted discoveries

    Note over DE,FS: Storage (1-2s)
    DE->>FS: Store all discoveries
    DE->>FS: Update business stats
    FS-->>DE: Success

    Note over DE: Total time: 3-5 minutes
```

## 4. Agent Architecture

```mermaid
graph TB
    subgraph "Analyst Agent - Deep Investigation"
        A1[discovery_mode<br/>Nightly analysis]
        A2[drill_down_mode<br/>User deep dive]
        A3[validation_mode<br/>Verify insights]
    end

    subgraph "Advisor Agent - Action Planning"
        B1[action_planning_mode<br/>Generate action plans]
        B2[script_writing_mode<br/>Create scripts]
        B3[campaign_mode<br/>Design campaigns]
        B4[crisis_mode<br/>Handle urgencies]
    end

    subgraph "Investigator Agent - Chat Interface"
        C1[question_answering_mode<br/>Handle queries]
        C2[investigation_mode<br/>Deep dive topics]
        C3[explanation_mode<br/>Explain discoveries]
    end

    subgraph "MCP Access"
        D1[Data Access MCPs<br/>square_data, external_data, memory]
        D2[Analysis MCPs<br/>7 analysis tools]
        D3[Intelligence MCPs<br/>4 intelligence tools]
        D4[Action MCPs<br/>4 action tools]
        D5[Validation MCPs<br/>4 validation tools]
    end

    subgraph "LLM Services"
        E1[Vertex AI Gemini Pro<br/>Standard synthesis]
        E2[Claude Opus 4<br/>High-impact discoveries]
    end

    A1 --> D1
    A1 --> D2
    A1 --> D3
    A1 --> E1
    A1 --> E2

    B1 --> D4
    B1 --> D5
    B1 --> E1
    B1 --> E2

    C1 --> D1
    C1 --> D2
    C1 --> E1

    style A1 fill:#4A90E2
    style A2 fill:#4A90E2
    style A3 fill:#4A90E2
    style B1 fill:#7B68EE
    style B2 fill:#7B68EE
    style B3 fill:#7B68EE
    style B4 fill:#7B68EE
    style C1 fill:#50C878
    style C2 fill:#50C878
    style C3 fill:#50C878
```

## 5. Data Storage Architecture

```mermaid
graph TB
    subgraph "Core Memory - Firestore"
        FM1[(businesses collection<br/>name, type, location<br/>owner preferences)]
        FM2[(users collection<br/>displayName, email<br/>businessId, role)]
        FM3[(discoveries collection<br/>insights, metadata<br/>completed status)]
    end

    subgraph "Working Memory - Redis (7-day TTL)"
        WM1[investigation state<br/>current analysis<br/>intermediate findings]
        WM2[conversation threads<br/>chat history<br/>context]
    end

    subgraph "Pattern Memory"
        PM1[(Vector Search<br/>embedding vectors<br/>semantic similarity)]
        PM2[(BigQuery<br/>pattern_history table<br/>historical insights)]
    end

    subgraph "Data Cache"
        DC1[(BigQuery<br/>square_cache table<br/>8 years transactions)]
        DC2[(Cloud Storage<br/>external data<br/>reviews, competitors)]
    end

    subgraph "Agents"
        AG1[Analyst Agent]
        AG2[Advisor Agent]
        AG3[Investigator Agent]
    end

    AG1 -->|Read profile| FM1
    AG1 -->|Save state| WM1
    AG1 -->|Query similar patterns| PM1
    AG1 -->|Query transactions| DC1
    AG1 -->|Fetch external data| DC2

    AG2 -->|Read profile| FM1
    AG2 -->|Store discoveries| FM3
    AG2 -->|Save patterns| PM2

    AG3 -->|Read profile| FM1
    AG3 -->|Load conversation| WM2
    AG3 -->|Query data| DC1

    style FM1 fill:#E8F5E9
    style FM2 fill:#E8F5E9
    style FM3 fill:#E8F5E9
    style WM1 fill:#FFF3E0
    style WM2 fill:#FFF3E0
    style PM1 fill:#F3E5F5
    style PM2 fill:#F3E5F5
    style DC1 fill:#E3F2FD
    style DC2 fill:#E3F2FD
```

## 6. Cost Breakdown

```mermaid
pie title Monthly Cost per Customer ($7.25)
    "Gemini Pro (LLM)" : 1.69
    "Claude Opus 4 (LLM)" : 4.32
    "BigQuery" : 0.05
    "Firestore" : 0.01
    "Redis" : 0.50
    "Cloud Storage" : 0.002
    "Cloud Run" : 0.44
    "Cloud Functions" : 0.11
    "Vector Search" : 0.10
    "Natural Language API" : 0.03
```

## 7. Discovery Example Flow

```mermaid
graph LR
    subgraph "Input: MCP Findings"
        I1[pattern_detector:<br/>41% retention for<br/>first-time brow customers]
        I2[segment_analyzer:<br/>156 customers<br/>$28 avg ticket]
        I3[employee_analyzer:<br/>Marcus: 42% retention<br/>Jennifer: 89% retention]
        I4[customer_psychology:<br/>First-time needs<br/>relationship building]
        I5[spa_wisdom:<br/>Assign senior staff<br/>to new customers]
    end

    subgraph "Analyst Synthesis"
        S1[Root Cause:<br/>Junior staff assigned<br/>to first-timers]
        S2[Confidence: 87%<br/>Sample: 156<br/>P-value: 0.001]
    end

    subgraph "Advisor Action Planning"
        A1[Action Plan:<br/>4 specific steps]
        A2[Script:<br/>Jennifer greeting<br/>first-time customers]
        A3[Impact:<br/>$6,252/year<br/>24 customers retained]
    end

    subgraph "Output: Discovery"
        O1[Discovery Object<br/>JSON format<br/>Ready for UI]
    end

    I1 --> S1
    I2 --> S1
    I3 --> S1
    I4 --> S1
    I5 --> S1

    S1 --> S2
    S2 --> A1
    S2 --> A2
    S2 --> A3

    A1 --> O1
    A2 --> O1
    A3 --> O1

    style S1 fill:#4A90E2
    style S2 fill:#7B68EE
    style O1 fill:#4CAF50
```

## 8. User Interaction Flow

```mermaid
sequenceDiagram
    participant USER as Business Owner
    participant UI as Keeper UI
    participant API as FastAPI
    participant FS as Firestore
    participant INV as Investigator Agent
    participant MCP as MCP Tools
    participant GEMINI as Gemini Pro

    Note over USER,UI: Morning Login (8:00 AM)
    USER->>UI: Visit keeper.tools
    UI->>API: GET /discoveries?businessId=...
    API->>FS: Fetch discoveries
    FS-->>API: 10 discoveries (generated at 3 AM)
    API-->>UI: JSON array
    UI-->>USER: Display discoveries dashboard

    Note over USER,UI: User Reviews Discovery
    USER->>UI: Click discovery #3
    UI-->>USER: Show full details:<br/>- Description<br/>- Reasoning<br/>- Action Plan<br/>- Script

    Note over USER,UI: User Asks Question
    USER->>UI: "Why Marcus has low retention?"
    UI->>API: POST /chat
    API->>INV: question_answering_mode()

    INV->>MCP: employee_analyzer.get_performance("marcus")
    MCP-->>INV: Marcus stats

    INV->>MCP: customer_forensics.get_customers_by_employee()
    MCP-->>INV: Customer list

    INV->>GEMINI: Synthesize answer with citations
    GEMINI-->>INV: Natural language response

    INV-->>API: Answer with data sources
    API-->>UI: Chat message
    UI-->>USER: "Marcus's retention is 42%...<br/>Source: employee_analyzer.mcp"

    Note over USER,UI: User Marks Complete
    USER->>UI: Click "Mark as Complete"
    UI->>API: POST /discoveries/:id/complete
    API->>FS: Update completed=true
    FS-->>API: Success
    API-->>UI: Updated
    UI-->>USER: Show checkmark
```

## Notes

- All diagrams use Mermaid syntax and can be rendered in GitHub, GitLab, or any Mermaid-compatible viewer
- Color coding:
  - **Blue**: Analyst Agent (investigation)
  - **Purple**: Advisor Agent (action planning)
  - **Green**: Investigator Agent (chat), Output/Success states
  - **Yellow**: LLMs, External Intelligence
  - **Red**: Action/Crisis Management
  - **Light colors**: Different MCP layers
