# Keeper AI Architecture Specification

**Version:** 1.0
**Last Updated:** 2025-09-21
**For:** Greg (Backend Implementation)

## Table of Contents
1. [Overview](#overview)
2. [Agent Architecture](#agent-architecture)
3. [MCP Tool Catalog](#mcp-tool-catalog)
4. [Memory System](#memory-system)
5. [Infrastructure](#infrastructure)
6. [Data Flow](#data-flow)
7. [Discovery Generation](#discovery-generation)
8. [Cost Model](#cost-model)
9. [Implementation Guide](#implementation-guide)

---

## Overview

### What is Keeper?

Keeper is a **discovery engine** (not a chatbot) that generates nightly actionable business insights for Square merchants. It analyzes both internal (Square data) and external (market/competitor) data to find transformative patterns like:

> "89% of first-time customers come for brow waxing, but 59% never return. Here's the 8-factor root cause and 5-step fix with expected $12K annual recovery."

NOT simple alerts like:
> "Sarah hasn't been in 57 days. Call her."

### Core Product Flow

1. **Nightly Analysis** (3:00 AM): System runs comprehensive multi-dimensional analysis
2. **Discovery Generation**: AI synthesizes findings into 5-10 actionable insights
3. **Morning Login**: Business owner sees prioritized discoveries with action plans
4. **Optional Chat**: If needed, user can ask follow-up questions about discoveries

### Key Principles

- **Multi-dimensional analysis**: Combine 10+ MCPs to find root causes
- **Internal + External data**: Square data + reviews + competitors + market intelligence
- **Actionable output**: Every discovery has specific action plan, script, and expected recovery
- **Cost-optimized**: Pre-compute in BigQuery, use LLMs only for synthesis
- **Google Cloud native**: BigQuery, Firestore, Cloud Run, Vertex AI

---

## Agent Architecture

### 3 Orchestrator Agents

Agents do NOT perform computations directly. They orchestrate MCP tools and synthesize findings using LLMs.

#### 1. Analyst Agent

**Purpose**: Deep data investigation and pattern discovery

**Modes**:
- `discovery_mode`: Nightly comprehensive analysis (uses Claude Opus 4)
- `drill_down_mode`: User-requested deep dive into specific metrics
- `validation_mode`: Verify insights before publishing

**Responsibilities**:
- Orchestrate Analysis Layer MCPs (pattern_detector, segment_analyzer, time_surgeon, etc.)
- Cross-reference Intelligence Layer MCPs for psychological context
- Generate hypotheses and validate with data
- Produce structured findings for Advisor Agent

**LLM Usage**:
- Synthesis only (never raw computation)
- Input: Pre-computed summaries from MCPs
- Output: Structured insights with confidence scores

**Example Flow**:
```
1. pattern_detector.mcp → "59% of brow waxing first-timers never return"
2. customer_forensics.mcp → "Average spend $28, no upsells recorded"
3. employee_analyzer.mcp → "3 different employees serviced this segment"
4. customer_psychology.mcp → "First-time service customers need extra attention"
5. spa_wisdom.mcp → "Industry best practice: assign senior staff to new customers"
6. Analyst synthesizes → "Root cause: New customers getting junior staff, no relationship building"
```

#### 2. Advisor Agent

**Purpose**: Convert insights into actionable recommendations

**Modes**:
- `action_planning_mode`: Generate step-by-step action plans
- `script_writing_mode`: Create conversation scripts for staff
- `campaign_mode`: Design marketing campaigns
- `crisis_mode`: Handle urgent situations (customer defection, employee issues)

**Responsibilities**:
- Take Analyst findings and generate 3-5 step action plans
- Use script_generator.mcp to create realistic conversation scripts
- Use campaign_builder.mcp for marketing recommendations
- Calculate expected recovery/impact with impact_predictor.mcp
- Format output with output_formatter.mcp

**LLM Usage**:
- Creative generation (action plans, scripts, messaging)
- Input: Analyst insights + business context from memory.mcp
- Output: Complete discovery object (title, description, actionPlan, script, expectedRecovery)

**Example Output**:
```json
{
  "title": "First-time brow customers need senior staff assignment",
  "category": "Customer Retention",
  "priority": "medium",
  "impact": 12400,
  "confidence": 87,
  "description": "89% of first-time customers come for brow waxing ($28 avg), but 59% never return. Industry retention rate is 78%.",
  "reasoning": "New customers are randomly assigned to any available staff. Data shows junior staff (Marcus, hired 8 months ago) has 42% retention vs Jennifer's 89%. First-time customers need relationship building and upsell education.",
  "actionPlan": [
    "Assign all first-time brow appointments to Jennifer or Amanda (senior staff)",
    "Train Marcus on first-time customer protocol (extra 5 min consultation, product recommendation)",
    "Add \"New Customer\" flag in booking system to trigger assignment rule",
    "Track retention rate weekly for 30 days to measure improvement"
  ],
  "script": "Hi [Name]! I see this is your first time with us - welcome! I'm Jennifer and I've been doing brows for 6 years. Let me take an extra minute to understand exactly what you're looking for...",
  "expectedRecovery": "Increase retention from 41% to 65% = 24 additional returning customers/month = $12,400/year",
  "tags": ["#CUSTOMERRETENTION", "#TRAINING"]
}
```

#### 3. Investigator Agent

**Purpose**: Answer user questions and investigate specific concerns

**Modes**:
- `question_answering_mode`: Handle user chat queries
- `investigation_mode`: Deep dive into user-specified issues
- `explanation_mode`: Explain why discoveries were generated

**Responsibilities**:
- Monitor chat interface for user questions
- Route questions to appropriate MCPs
- Use memory.mcp to maintain conversation context
- Provide citations to source data
- Escalate to Analyst if new pattern discovered

**LLM Usage**:
- Conversational (chat interface)
- Input: User question + relevant MCP data + memory context
- Output: Natural language answer with data citations

**Example Interaction**:
```
User: "Why did you say Marcus has low retention?"
Investigator:
1. Calls employee_analyzer.mcp → Get Marcus stats
2. Calls customer_forensics.mcp → Get customers assigned to Marcus
3. Calls pattern_detector.mcp → Compare to team averages
4. Synthesizes answer:
   "Marcus's retention rate is 42% compared to team average of 78%.
    Looking at his 67 customers over 8 months:
    - 39 customers (58%) came for appointments only once
    - Average ticket: $78 vs team avg $98
    - Service time: 4:12 min vs team avg 2:45 min
    - Customer feedback mentions 'felt rushed' in 3 reviews
    Source: employee_analyzer.mcp + customer_forensics.mcp"
```

---

## MCP Tool Catalog

### Layer 1: Data Access (3 MCPs)

#### square_data.mcp
**Purpose**: Access cached Square data from BigQuery

**Tools**:
- `get_transactions(start_date, end_date, filters)` → Transaction list with customer, employee, service, amount
- `get_customers(filters)` → Customer profiles with visit history, lifetime value
- `get_employees(filters)` → Employee performance metrics
- `get_catalog()` → Services, products, pricing
- `get_customer_timeline(customer_id)` → Full visit history with employee assignments, services, amounts, tips
- `get_employee_metrics(employee_id, date_range)` → Revenue, retention, average ticket, hours worked

**Data Source**: BigQuery table `keeper.square_cache` (synced nightly at 12:01 AM)

**Why BigQuery cache?**
- Square API rate limits (avoid hitting API during analysis)
- Faster queries (indexed, optimized)
- Historical data readily available (8 years for Bashful Beauty)

**Sync Strategy**:
- Nightly incremental sync (last 24 hours)
- Full resync weekly (Sunday 2 AM)
- Real-time webhooks for critical events (new customer, refund)

#### external_data.mcp
**Purpose**: Fetch scraped external data from Cloud Storage

**Tools**:
- `get_reviews(source, date_range)` → Reviews from Google, Yelp (scraped nightly)
- `get_competitor_data(radius_miles)` → Competitor names, services, pricing, reviews
- `get_social_mentions(platforms)` → Instagram/Facebook mentions (scraped)
- `get_local_events(date_range)` → Local events affecting foot traffic (scraped from city websites)

**Data Source**:
- Cloud Storage buckets: `gs://keeper-external-data/{business_id}/reviews/`, `competitors/`, `social/`
- Updated by Cloud Functions (scheduled scraping)

**Scraping Implementation**:
- Cloud Functions with Puppeteer/Playwright
- Run nightly at 1 AM (before discovery generation at 3 AM)
- Cost: ~$2/month per business (Cloud Functions compute time)

#### memory.mcp
**Purpose**: Three-tier memory system

**Tools**:
- **Core Memory** (Firestore):
  - `get_business_profile()` → Name, type, location, employees, owner preferences
  - `get_employee_profiles()` → Names, roles, hire dates, notes
  - `update_preferences(key, value)` → Store user preferences

- **Working Memory** (Redis, 7-day TTL):
  - `save_investigation(id, state)` → Save current analysis state
  - `get_investigation(id)` → Resume investigation
  - `save_conversation(thread_id, messages)` → Chat history

- **Pattern Memory** (Vertex AI Vector Search + BigQuery):
  - `save_learned_pattern(pattern, metadata)` → Store validated insights
  - `search_similar_patterns(query_vector, top_k)` → Find related historical insights
  - `get_historical_discoveries(date_range)` → Past discoveries with outcomes

**Data Sources**:
- Firestore: `businesses/{businessId}`, `users/{uid}`
- Redis: Memorystore for Redis (GCP)
- Vector Search: Vertex AI Vector Search index
- BigQuery: `keeper.pattern_history` table

---

### Layer 2: Analysis (7 MCPs)

#### pattern_detector.mcp
**Purpose**: Find statistical anomalies and trends

**Tools**:
- `find_retention_patterns(segment)` → Retention rates by segment with statistical significance
- `find_revenue_patterns(grouping)` → Revenue trends by service, time, employee
- `find_visit_frequency_patterns()` → Expected vs actual visit intervals
- `find_abandonment_patterns()` → When customers stop visiting (cohort analysis)
- `detect_anomalies(metric, threshold)` → Statistical outliers (z-score > 2.5)
- `trend_analysis(metric, period)` → Time series trend detection (seasonality, growth)

**Methods**:
- SQL queries on BigQuery (pre-computed aggregations)
- Statistical tests: chi-square, t-tests for significance
- Cohort analysis for retention
- Rolling averages for trend detection

**Example Output**:
```json
{
  "pattern": "first_time_brow_retention",
  "finding": "First-time brow waxing customers have 41% retention vs 78% business average",
  "significance": 0.001,
  "sample_size": 156,
  "context": "89% of first-time customers book brow waxing as entry service"
}
```

#### segment_analyzer.mcp
**Purpose**: Customer segmentation and cohort analysis

**Tools**:
- `segment_by_behavior(criteria)` → RFM (Recency, Frequency, Monetary) segments
- `segment_by_service_preference()` → Customers grouped by favorite service
- `segment_by_employee_affinity()` → Customers who prefer specific staff
- `segment_by_lifecycle_stage()` → New, Active, At-Risk, Lost
- `segment_by_value()` → High-value, Medium, Low based on LTV
- `compare_segments(segment_a, segment_b, metrics)` → Statistical comparison

**Methods**:
- RFM analysis (quintiles)
- K-means clustering on BigQuery ML
- Lifecycle stage rules (days since last visit, visit frequency)

**Example Output**:
```json
{
  "segment": "first_time_brow_only",
  "size": 156,
  "characteristics": {
    "avg_ticket": 28,
    "retention_rate": 0.41,
    "avg_lifetime_visits": 1.6,
    "avg_lifetime_value": 45
  },
  "comparison_to_average": {
    "retention": -0.37,
    "ltv": -0.82
  }
}
```

#### time_surgeon.mcp
**Purpose**: Time-based pattern analysis

**Tools**:
- `analyze_by_time_of_day(metric)` → Hourly patterns (revenue, traffic, retention)
- `analyze_by_day_of_week(metric)` → Daily patterns with significance tests
- `analyze_by_season(metric)` → Seasonal trends
- `analyze_appointment_gaps(customer_segment)` → Time between visits
- `find_peak_times()` → High-traffic periods with capacity analysis
- `detect_scheduling_inefficiencies()` → Gaps, overbooking, underutilization

**Methods**:
- Time series analysis on BigQuery
- Day-of-week ANOVA tests
- Appointment interval histograms

**Example Output**:
```json
{
  "finding": "tuesday_retention_anomaly",
  "pattern": "Tuesday appointments have 31% retention vs 67% business average",
  "context": "Tuesday has 47 Instagram mentions/month (highest) but lowest return rate",
  "sample": {
    "tuesday_customers": 89,
    "one_time_visitors": 60,
    "influencer_customers": 40
  }
}
```

#### employee_analyzer.mcp
**Purpose**: Employee performance and impact analysis

**Tools**:
- `get_employee_performance(employee_id, metrics)` → Revenue, retention, ticket size, tips
- `compare_employees(metric)` → Rank employees by performance
- `find_training_gaps(employee_id)` → Areas below team average
- `analyze_customer_assignments(employee_id)` → Who sees which customers
- `detect_employee_impact_on_retention(employee_id)` → Correlation between employee and customer retention
- `analyze_service_speed(employee_id)` → Time per service vs team average

**Methods**:
- SQL aggregations on BigQuery
- Employee-customer linkage analysis
- Service duration calculations from appointment timestamps

**Example Output**:
```json
{
  "employee": "Marcus Johnson",
  "metrics": {
    "retention_rate": 0.42,
    "avg_ticket": 78,
    "revenue_per_hour": 59,
    "avg_service_time_min": 4.2,
    "tip_percentage": 12
  },
  "team_averages": {
    "retention_rate": 0.78,
    "avg_ticket": 98,
    "revenue_per_hour": 98,
    "avg_service_time_min": 2.75,
    "tip_percentage": 19
  },
  "gaps": ["service_speed", "upselling", "customer_retention"]
}
```

#### customer_forensics.mcp
**Purpose**: Deep dive into individual customer behavior

**Tools**:
- `get_customer_journey(customer_id)` → Full visit history with context
- `predict_churn_risk(customer_id)` → Churn probability with factors
- `find_defection_signals(customer_id)` → Declining tips, longer gaps, negative feedback
- `analyze_customer_preferences(customer_id)` → Favorite services, staff, times
- `calculate_customer_ltv(customer_id)` → Lifetime value projection
- `find_similar_customers(customer_id)` → Customers with similar patterns

**Methods**:
- Individual customer timeline analysis
- Logistic regression for churn prediction (BigQuery ML)
- Pattern matching against historical churned customers

**Example Output**:
```json
{
  "customer": "Sarah Chen",
  "status": "at_risk",
  "churn_probability": 0.76,
  "signals": [
    "47 days since last visit (pattern: 18-22 days)",
    "Declining tips: 20% → 12% → 0% over last 3 visits",
    "Last appointment: Jennifer was 15 min late (noted in booking)"
  ],
  "context": "Friend Lisa posted on Facebook about trying new spa downtown 12 days ago",
  "lifetime_value": 2280,
  "recovery_window": "60% chance if contacted today, 40% if contacted this week"
}
```

#### financial_surgeon.mcp
**Purpose**: Revenue analysis and pricing optimization

**Tools**:
- `analyze_revenue_by_service()` → Service profitability with margins
- `analyze_pricing_vs_market(service)` → Compare pricing to competitors
- `find_upsell_opportunities()` → Customers who never buy add-ons
- `analyze_discount_impact()` → ROI of discounts/promotions
- `calculate_service_profitability(service_id)` → Revenue, cost, time, margin
- `find_revenue_leaks()` → No-shows, cancellations, underpricing

**Methods**:
- Revenue aggregations by service, employee, time
- Competitor pricing comparison (from external_data.mcp)
- Margin calculations (revenue - estimated costs)

**Example Output**:
```json
{
  "finding": "vip_upsell_gap",
  "description": "12 high-value customers (avg $285/month) have never purchased add-ons",
  "customers": ["Emily Rodriguez", "..."],
  "opportunity": {
    "current_monthly_revenue": 3420,
    "potential_addon_revenue": 360,
    "expected_conversion_rate": 0.30,
    "annual_opportunity": 4320
  },
  "context": "These customers book when Jennifer is off. Jennifer upsells 73% of her clients."
}
```

#### competitor_intel.mcp
**Purpose**: Competitive landscape analysis

**Tools**:
- `find_nearby_competitors(radius_miles)` → Competitor list with details
- `compare_pricing(service_type)` → Your pricing vs competitors
- `analyze_competitor_reviews(competitor_id)` → What customers say about them
- `find_competitive_advantages()` → What you do better
- `find_competitive_gaps()` → What competitors offer that you don't
- `track_competitor_changes()` → New services, pricing changes, promotions

**Methods**:
- Web scraping of competitor websites, Google Maps, Yelp
- NLP sentiment analysis of competitor reviews
- Pricing comparison tables

**Example Output**:
```json
{
  "competitors": [
    {
      "name": "Glow Downtown Spa",
      "distance_miles": 1.2,
      "services": ["Brazilian", "Brow", "Facial", "Massage"],
      "pricing": {
        "brazilian": 65,
        "brow": 25
      },
      "your_pricing": {
        "brazilian": 75,
        "brow": 20
      },
      "reviews": {
        "google_rating": 4.2,
        "yelp_rating": 4.0,
        "common_complaints": ["long wait times", "hard to book"],
        "common_praise": ["downtown location", "modern facility"]
      }
    }
  ],
  "insights": [
    "You charge $10 more for Brazilian but $5 less for brows",
    "Glow has 'hard to book' complaints - opportunity for availability marketing"
  ]
}
```

---

### Layer 3: Intelligence (4 MCPs)

#### customer_psychology.mcp
**Purpose**: Apply behavioral psychology to customer insights

**Tools**:
- `explain_customer_behavior(pattern, context)` → Psychological interpretation
- `suggest_retention_tactics(customer_profile)` → Psychology-based retention strategies
- `analyze_decision_factors(service_type)` → What drives customer choices
- `predict_response_to_outreach(customer_id, message_type)` → Likelihood of positive response
- `suggest_messaging_tone(customer_segment)` → Optimal communication style

**Methods**:
- Rule-based psychology models (loss aversion, social proof, reciprocity)
- LLM synthesis of psychological principles
- No computation, pure interpretation

**Example Output**:
```json
{
  "pattern": "declining_tips_before_defection",
  "psychological_explanation": "Declining tips signal growing dissatisfaction. Customers reduce tips before stopping visits entirely as passive-aggressive feedback. 0% tip on last visit is strong defection signal.",
  "recommended_approach": {
    "tactic": "personal_apology_with_specific_acknowledgment",
    "reasoning": "Customer needs to feel heard about specific incident (late appointment). Generic 'we miss you' will fail. Specific apology from Jennifer restores relationship.",
    "tone": "warm, accountable, not defensive"
  }
}
```

#### employee_psychology.mcp
**Purpose**: Employee performance psychology

**Tools**:
- `diagnose_performance_gap(employee_id, gap_type)` → Root cause analysis (training, motivation, burnout)
- `suggest_coaching_approach(employee_id, issue)` → How to address performance issues
- `predict_retention_risk(employee_id)` → Employee churn likelihood
- `suggest_recognition_strategy(employee_id)` → Motivation tactics
- `analyze_team_dynamics()` → Employee interactions, conflicts, collaboration

**Methods**:
- Performance psychology models
- LLM interpretation of employee data patterns
- Coaching best practices library

**Example Output**:
```json
{
  "employee": "Marcus Johnson",
  "performance_gap": "low_service_speed",
  "psychological_diagnosis": "Likely a training gap, not motivation issue. Marcus takes extra time because he's manually frothing milk (doesn't know about pre-programmed settings). His customer service scores are high, showing genuine care.",
  "coaching_approach": {
    "tone": "supportive_technical_training",
    "script": "Marcus, you're genuinely great with customers but I noticed you're manually frothing milk. Let me show you the pre-programmed settings - it'll save you 90 seconds per drink and your arm will thank you.",
    "avoid": "Don't frame as performance criticism - frame as helpful tip"
  },
  "expected_outcome": "60% performance recovery within 30 days with proper training"
}
```

#### owner_psychology.mcp
**Purpose**: Tailor communication to business owner preferences

**Tools**:
- `get_owner_communication_style(business_id)` → Preferred tone, detail level, format
- `suggest_insight_framing(insight, owner_profile)` → How to present findings
- `predict_owner_receptiveness(recommendation)` → Likelihood owner will act
- `adapt_action_plan_complexity(owner_experience)` → Simplify or add detail based on sophistication

**Methods**:
- User preference tracking (from memory.mcp)
- Adaptive complexity (beginner vs advanced business owner)
- Tone adaptation (data-driven vs story-driven)

**Example Output**:
```json
{
  "owner_profile": {
    "name": "Steven Ray",
    "business_experience_years": 6,
    "sophistication": "intermediate",
    "preferred_style": "data_with_story",
    "action_bias": "high",
    "detail_preference": "moderate"
  },
  "recommendation": {
    "tone": "Use specific numbers but keep explanations concise",
    "format": "Lead with impact ($12K/year), then explain why, then action plan",
    "complexity": "3-5 step action plans, not 10-step processes",
    "avoid": "Don't oversimplify (respects data) but don't overwhelm with statistics"
  }
}
```

#### market_dynamics.mcp
**Purpose**: Market trends and economic context

**Tools**:
- `get_industry_benchmarks(business_type)` → Typical retention rates, pricing, margins for spa/salon/restaurant
- `analyze_local_market_trends()` → Foot traffic, new competition, economic indicators
- `compare_to_industry(metric)` → Your performance vs industry average
- `predict_seasonal_impact(month)` → Expected seasonal variations
- `find_market_opportunities()` → Underserved segments, trending services

**Methods**:
- Industry benchmark database (static data from research)
- Local market data (from external_data.mcp)
- Economic indicators (web scraping)

**Example Output**:
```json
{
  "business_type": "spa_and_wellness",
  "industry_benchmarks": {
    "customer_retention_rate": 0.78,
    "average_ticket": 95,
    "services_per_customer_per_year": 6.2,
    "typical_margin": 0.65
  },
  "your_performance": {
    "customer_retention_rate": 0.67,
    "average_ticket": 89,
    "services_per_customer_per_year": 5.1
  },
  "market_context": {
    "san_diego_spa_market": "Growing 8% annually",
    "new_competitors_last_year": 3,
    "trend": "Wellness services (massage, meditation) growing faster than traditional (waxing, facials)"
  }
}
```

---

### Layer 4: Wisdom (4 MCPs)

Business-type-specific expertise encoded as rules and best practices.

#### spa_wisdom.mcp
**Purpose**: Spa & wellness industry best practices

**Tools**:
- `get_best_practice(situation)` → Industry standard approach
- `suggest_service_mix(current_offerings)` → Recommended services to add/remove
- `recommend_pricing_strategy(service, market)` → Competitive pricing guidance
- `suggest_customer_experience_improvements(pain_point)` → Spa-specific CX tactics
- `get_training_recommendations(employee_gap)` → Esthetician training resources

**Methods**:
- Static knowledge base (spa industry research)
- LLM retrieval from curated spa management content
- Best practice library

**Example Output**:
```json
{
  "situation": "first_time_brow_customer_retention",
  "best_practice": "Assign first-time service customers to senior staff. First visit sets relationship foundation. Junior staff should take returning customers who already trust the brand.",
  "implementation": {
    "booking_rule": "Flag first-time appointments in booking system",
    "staff_assignment": "Route to staff with >2 years experience or >75% retention rate",
    "time_allocation": "Book extra 5 minutes for consultation"
  },
  "expected_impact": "Industry data shows 20-30% retention improvement with this approach"
}
```

#### restaurant_wisdom.mcp
**Purpose**: Restaurant & food service best practices

**Tools**:
- `get_best_practice(situation)` → Restaurant industry standards
- `suggest_menu_optimization(current_menu, sales_data)` → Menu engineering
- `recommend_staffing_levels(traffic_pattern)` → Optimal scheduling
- `suggest_customer_experience_improvements(pain_point)` → Restaurant-specific CX
- `get_food_cost_benchmarks(cuisine_type)` → Typical food cost percentages

**Methods**:
- Restaurant management knowledge base
- Menu engineering formulas (star, plow horse, puzzle, dog)
- Labor cost optimization models

#### retail_wisdom.mcp
**Purpose**: Retail best practices

**Tools**:
- `get_best_practice(situation)` → Retail industry standards
- `suggest_inventory_optimization(sales_data)` → Stock levels, reorder points
- `recommend_merchandising(product_category)` → Product placement, displays
- `suggest_customer_experience_improvements(pain_point)` → Retail-specific CX
- `get_conversion_benchmarks(retail_type)` → Typical conversion rates, basket size

#### service_wisdom.mcp
**Purpose**: General service business best practices

**Tools**:
- `get_best_practice(situation)` → Service business standards
- `suggest_pricing_model(service_type)` → Hourly, project, subscription, tiered
- `recommend_customer_onboarding(service_type)` → First customer experience
- `suggest_upsell_strategies(service_mix)` → Cross-sell and upsell tactics
- `get_service_benchmarks(industry)` → Utilization rates, billable hours, margins

---

### Layer 5: External Intelligence (5 MCPs)

#### review_monitor.mcp
**Purpose**: Analyze online reviews for insights

**Tools**:
- `get_recent_reviews(sources, date_range)` → Your reviews from Google, Yelp, Facebook
- `analyze_sentiment_trends(period)` → Sentiment over time
- `extract_common_complaints()` → Recurring negative themes
- `extract_common_praise()` → Recurring positive themes
- `find_mentioned_employees(reviews)` → Staff mentioned in reviews (positive/negative)
- `detect_review_anomalies()` → Sudden sentiment shifts, fake reviews

**Methods**:
- NLP sentiment analysis (Vertex AI Natural Language API)
- Topic clustering (LDA, k-means)
- Named entity recognition for employee names

**Example Output**:
```json
{
  "period": "last_30_days",
  "total_reviews": 23,
  "avg_rating": 4.6,
  "sentiment_trend": "stable",
  "common_praise": [
    "Jennifer is amazing - 8 mentions",
    "Clean facility - 6 mentions",
    "Great brow shaping - 5 mentions"
  ],
  "common_complaints": [
    "Hard to get appointment with Jennifer - 4 mentions",
    "Parking difficult - 3 mentions"
  ],
  "employee_mentions": {
    "Jennifer": {"positive": 8, "negative": 0},
    "Marcus": {"positive": 1, "negative": 2}
  },
  "insights": [
    "Jennifer demand exceeds capacity - pricing or scheduling opportunity",
    "Marcus receiving negative feedback about service quality"
  ]
}
```

#### competitor_discovery.mcp
**Purpose**: Monitor competitor activity

**Tools**:
- `find_new_competitors(radius, business_type)` → Recently opened businesses
- `track_competitor_reviews(competitor_id)` → Monitor competitor review trends
- `detect_competitor_promotions(competitor_id)` → Pricing changes, special offers
- `analyze_competitor_customer_overlap()` → Customers who also review competitors
- `find_competitor_weaknesses()` → Opportunities based on competitor complaints

**Methods**:
- Web scraping of Google Maps, Yelp for new businesses
- Review monitoring for competitors
- Price tracking over time

**Example Output**:
```json
{
  "new_competitors": [
    {
      "name": "Glow Downtown Spa",
      "opened": "2025-06-15",
      "location": "1.2 miles away",
      "services": ["Brazilian", "Facial", "Massage"],
      "pricing": {"brazilian": 65},
      "reviews": {"google_rating": 4.2, "count": 18}
    }
  ],
  "threats": [
    "Glow is $10 cheaper on Brazilian wax",
    "Glow has modern downtown location"
  ],
  "opportunities": [
    "Glow reviews complain about 'hard to book' - your availability is strength",
    "Glow doesn't offer brow services - your specialty"
  ]
}
```

#### customer_migration.mcp
**Purpose**: Track customer movement to/from competitors

**Tools**:
- `find_lost_customers_at_competitors()` → Your lost customers reviewing competitor spas
- `find_competitor_customers_to_target()` → People reviewing competitors negatively
- `analyze_switching_patterns()` → Why customers leave for competitors
- `identify_winback_opportunities()` → Lost customers likely to return
- `track_customer_social_activity(customer_id)` → Social media mentions of other businesses

**Methods**:
- Cross-reference customer names/emails in competitor reviews
- Social media scraping (Instagram, Facebook) for location tags
- Pattern analysis of lost customers

**Example Output**:
```json
{
  "lost_customers_found": [
    {
      "customer": "Sarah Chen",
      "status": "lost_47_days_ago",
      "found_at": "Glow Downtown Spa",
      "evidence": "Google review 12 days ago: 'Trying out this new place, so far so good'",
      "recovery_strategy": "Personal outreach from Jennifer with specific apology for late appointment"
    }
  ],
  "winback_priority": "high",
  "competitor_customers_to_target": [
    {
      "reviewer": "Lisa M.",
      "complaint": "Glow is impossible to book, waited 3 weeks",
      "opportunity": "Promote your availability: 'Book same-week appointments at Bashful Beauty'"
    }
  ]
}
```

#### social_listening.mcp
**Purpose**: Monitor social media for business intelligence

**Tools**:
- `track_brand_mentions(platforms)` → Instagram, Facebook mentions of your business
- `analyze_influencer_impact()` → Which customers are influencers, their reach
- `find_viral_moments()` → Posts with high engagement about your business
- `track_local_hashtags()` → Trending local hashtags related to your industry
- `detect_pr_issues()` → Negative viral posts, complaints

**Methods**:
- Instagram/Facebook scraping for location tags, mentions
- Engagement metrics (likes, comments, shares)
- Influencer detection (follower count, engagement rate)

**Example Output**:
```json
{
  "brand_mentions_last_30_days": 47,
  "top_posts": [
    {
      "user": "@sandiego_beauty_blogger",
      "followers": 12400,
      "post": "Best brow wax in SD! Ask for Jennifer 💕",
      "engagement": {"likes": 340, "comments": 23},
      "date": "2025-09-10"
    }
  ],
  "insights": [
    "Tuesday has highest social mentions (47/month) but lowest retention (31%)",
    "Influencer customers are one-time visitors - not repeat business",
    "Opportunity: 'Bring a Friend Tuesday' to convert influencers to regulars"
  ]
}
```

#### local_market.mcp
**Purpose**: Local market intelligence

**Tools**:
- `track_foot_traffic(location)` → Estimated foot traffic patterns
- `find_local_events(date_range)` → Events affecting your business (festivals, construction, new buildings)
- `analyze_demographic_trends()` → Population growth, income changes, age distribution
- `find_nearby_developments()` → New housing, office buildings, retail nearby
- `track_seasonal_patterns(metric)` → Tourist season, holidays, weather impact

**Methods**:
- Web scraping of city event calendars, development permits
- Google Maps traffic data
- Census/demographic data APIs

**Example Output**:
```json
{
  "upcoming_events": [
    {
      "event": "Comic-Con San Diego",
      "dates": "2025-07-24 to 2025-07-27",
      "impact": "High foot traffic downtown, parking difficult",
      "recommendation": "Promote 'Walk-in Wednesday' during event week"
    }
  ],
  "developments": [
    {
      "project": "New luxury condos on 5th Ave",
      "completion": "2025-11-01",
      "units": 240,
      "impact": "Potential 240 new customers within 0.5 miles"
    }
  ],
  "seasonal_trends": {
    "current_month": "September",
    "historical_pattern": "Typically -12% revenue vs summer months",
    "recommendation": "Launch fall promotion to offset seasonal dip"
  }
}
```

---

### Layer 6: Action (4 MCPs)

#### script_generator.mcp
**Purpose**: Generate conversation scripts for staff

**Tools**:
- `generate_customer_outreach_script(customer_profile, situation)` → Phone/text scripts for customer outreach
- `generate_sales_script(service, customer_type)` → Upsell/cross-sell scripts
- `generate_employee_coaching_script(employee, issue)` → Manager-to-employee conversation scripts
- `generate_review_response(review_text, sentiment)` → Responses to reviews

**Methods**:
- LLM generation with templates
- Personality-matched tone (from customer_psychology.mcp)
- Include specific details from context

**Example Output**:
```json
{
  "situation": "customer_defection_recovery",
  "customer": "Sarah Chen",
  "script_type": "phone_call",
  "script": "Hi Sarah, it's Jennifer from Bashful Beauty. I realized I kept you waiting last time and I felt terrible about it. I'd love to make it right - can I book you this week with 20% off? I miss seeing you!",
  "tone": "warm, personal, accountable",
  "key_elements": [
    "Personal connection (Jennifer, not generic staff)",
    "Specific acknowledgment of incident (late appointment)",
    "Concrete offer (20% off)",
    "Urgency (this week)",
    "Emotional connection (I miss seeing you)"
  ],
  "expected_success_rate": "60% if called today"
}
```

#### campaign_builder.mcp
**Purpose**: Design marketing campaigns

**Tools**:
- `build_retention_campaign(customer_segment, goal)` → Re-engagement campaigns
- `build_acquisition_campaign(target_audience, offer)` → New customer campaigns
- `build_upsell_campaign(customer_segment, service)` → Cross-sell/upsell campaigns
- `build_seasonal_campaign(season, business_context)` → Holiday/seasonal promotions
- `estimate_campaign_roi(campaign_plan)` → Expected cost, revenue, ROI

**Methods**:
- Campaign templates (email, SMS, social media)
- Audience segmentation from segment_analyzer.mcp
- ROI estimation based on historical data

**Example Output**:
```json
{
  "campaign_name": "First-Timer Retention Boost",
  "goal": "Increase first-time customer retention from 41% to 65%",
  "target_audience": {
    "segment": "first_time_customers_last_30_days",
    "size": 23,
    "current_retention": 0.41
  },
  "tactics": [
    {
      "channel": "email",
      "timing": "3 days after first visit",
      "subject": "Sarah, how was your first visit with us?",
      "content": "Personalized message from assigned esthetician, request feedback, offer 10% off next visit within 14 days",
      "expected_open_rate": 0.45,
      "expected_conversion": 0.35
    },
    {
      "channel": "sms",
      "timing": "18 days after first visit (before expected re-book window closes)",
      "content": "Jennifer here! Ready to book your next brow wax? Reply YES and I'll get you scheduled this week.",
      "expected_response_rate": 0.28
    }
  ],
  "expected_roi": {
    "cost": 0,
    "additional_customers_retained": 5.5,
    "additional_annual_revenue": 3168,
    "roi": "infinite (zero cost, email/SMS already available)"
  }
}
```

#### optimization_architect.mcp
**Purpose**: Design operational improvements

**Tools**:
- `optimize_scheduling(constraints, goals)` → Staff scheduling recommendations
- `optimize_pricing(service, market_data)` → Dynamic pricing suggestions
- `optimize_capacity(traffic_patterns)` → Utilization improvements
- `optimize_service_mix(sales_data, profitability)` → Service portfolio changes
- `design_process_improvement(bottleneck)` → Workflow optimization

**Methods**:
- Linear programming for scheduling (OR-Tools)
- Price elasticity analysis
- Capacity utilization calculations
- Process mapping

**Example Output**:
```json
{
  "optimization": "morning_rush_walkaway_reduction",
  "problem": "23% of walk-ins leave during 9-11am due to perceived long wait",
  "current_state": {
    "actual_avg_wait_minutes": 12,
    "perceived_wait": "30+ minutes (customers see 4-5 people waiting)",
    "monthly_lost_revenue": 100
  },
  "recommended_changes": [
    {
      "change": "Install digital wait time display",
      "cost": 150,
      "implementation": "Buy $150 TV display, show 'Current wait: ~12 min'"
    },
    {
      "change": "Add SMS notification '5 min before your turn'",
      "cost": 0,
      "implementation": "Use existing SMS system, send when 1 customer ahead in line"
    },
    {
      "change": "Offer coffee/tea to waiting customers",
      "cost": 30,
      "implementation": "Keurig + supplies, reduces perceived wait time"
    }
  ],
  "expected_outcome": {
    "walkaway_rate": "23% → 10%",
    "annual_additional_revenue": 1200,
    "payback_period": "2 months"
  }
}
```

#### crisis_manager.mcp
**Purpose**: Handle urgent business issues

**Tools**:
- `detect_crisis(business_data, external_data)` → Identify critical issues (viral negative review, employee walkout, health inspection)
- `generate_crisis_response(crisis_type)` → Immediate action plan
- `prioritize_actions(crisis_list)` → Triage multiple issues
- `estimate_damage(crisis)` → Revenue impact, reputation impact
- `generate_communication_plan(crisis, stakeholders)` → What to say to customers, staff, public

**Methods**:
- Rule-based crisis detection (thresholds for critical metrics)
- Crisis response playbooks
- LLM generation of communication

**Example Output**:
```json
{
  "crisis_detected": "viral_negative_review",
  "severity": "high",
  "details": {
    "platform": "Yelp",
    "reviewer": "Amanda K.",
    "rating": 1,
    "text": "WORST experience! Was charged for services I didn't receive. Manager was rude when I complained. Never again!",
    "date": "2025-09-20 14:30",
    "views": 340,
    "engagement": "12 people marked 'useful'"
  },
  "estimated_impact": {
    "potential_lost_customers": "8-12 per month (based on review visibility)",
    "monthly_revenue_at_risk": 720
  },
  "immediate_actions": [
    {
      "action": "Investigate billing records",
      "timeline": "Within 1 hour",
      "owner": "Owner"
    },
    {
      "action": "Call customer directly",
      "timeline": "Within 2 hours",
      "owner": "Owner (not manager mentioned in review)",
      "script": "Hi Amanda, this is Steven, the owner of Bashful Beauty. I just saw your review and I'm deeply concerned. Can you please tell me what happened? I want to make this right immediately."
    },
    {
      "action": "Public response on Yelp",
      "timeline": "After speaking with customer",
      "draft": "We're so sorry for your experience, Amanda. This doesn't reflect our standards. We've reached out to you directly to resolve this. For anyone else with concerns, please contact me at..."
    }
  ]
}
```

---

### Layer 7: Validation (4 MCPs)

#### data_validator.mcp
**Purpose**: Ensure data quality before analysis

**Tools**:
- `validate_square_data(data_batch)` → Check for missing fields, duplicates, anomalies
- `validate_external_data(source, data)` → Verify scraped data quality
- `detect_data_gaps(date_range, data_type)` → Find missing data periods
- `flag_suspicious_data(data, rules)` → Outliers, errors, potential fraud

**Methods**:
- Schema validation (Pydantic models)
- Statistical outlier detection
- Business rule validation (e.g., negative revenue = error)

**Example Output**:
```json
{
  "validation_result": "warning",
  "issues": [
    {
      "severity": "warning",
      "type": "missing_data",
      "description": "Employee 'Marcus Johnson' has no transactions on 2025-09-15 despite scheduled shifts",
      "recommendation": "Verify if Marcus was out sick or data sync issue"
    },
    {
      "severity": "low",
      "type": "outlier",
      "description": "Transaction #12345 shows $450 tip on $95 service (473% tip rate)",
      "recommendation": "Likely data entry error - confirm with owner"
    }
  ],
  "data_quality_score": 0.94
}
```

#### confidence_scorer.mcp
**Purpose**: Calculate confidence scores for insights

**Tools**:
- `calculate_statistical_confidence(pattern, sample_size)` → P-value, confidence interval
- `calculate_data_quality_confidence(data_sources)` → Confidence based on data completeness
- `calculate_external_data_confidence(source)` → Reliability of external data (reviews, competitor data)
- `calculate_overall_confidence(insight)` → Weighted confidence score (0-100)

**Methods**:
- Statistical significance tests (chi-square, t-test)
- Data quality scoring (completeness, recency, source reliability)
- Weighted combination of factors

**Example Output**:
```json
{
  "insight": "First-time brow customers have 41% retention vs 78% average",
  "confidence_score": 87,
  "breakdown": {
    "statistical_significance": {
      "p_value": 0.001,
      "confidence_interval": [0.38, 0.44],
      "sample_size": 156,
      "score": 95
    },
    "data_quality": {
      "completeness": 0.98,
      "recency": "current",
      "score": 92
    },
    "external_validation": {
      "industry_benchmark_available": true,
      "aligns_with_spa_wisdom": true,
      "score": 75
    }
  },
  "confidence_level": "high"
}
```

#### impact_predictor.mcp
**Purpose**: Estimate financial impact of insights and recommendations

**Tools**:
- `predict_revenue_impact(recommendation)` → Expected revenue change
- `predict_cost_impact(recommendation)` → Implementation costs
- `calculate_roi(recommendation)` → ROI with confidence interval
- `estimate_time_to_impact(recommendation)` → When results will be visible
- `calculate_risk_adjusted_impact(recommendation)` → Expected value considering probability

**Methods**:
- Historical pattern analysis (similar actions in the past)
- Industry benchmarks (typical results)
- Monte Carlo simulation for uncertainty

**Example Output**:
```json
{
  "recommendation": "Assign first-time brow customers to senior staff",
  "impact_estimate": {
    "current_state": {
      "first_time_customers_per_month": 13,
      "current_retention_rate": 0.41,
      "returning_customers": 5.3,
      "annual_revenue_from_returning": 3168
    },
    "predicted_state": {
      "retention_rate": 0.65,
      "returning_customers": 8.5,
      "annual_revenue_from_returning": 5100,
      "incremental_revenue": 1932
    },
    "costs": {
      "implementation": 0,
      "opportunity_cost": "Jennifer/Amanda may have slightly fewer appointments overall due to first-timer priority, estimated -$200/year"
    },
    "net_impact": 1732,
    "confidence_interval": [1200, 2400],
    "probability_of_success": 0.70
  },
  "time_to_impact": "30-60 days to see retention rate change",
  "risk_adjusted_impact": 1212
}
```

#### output_formatter.mcp
**Purpose**: Format discoveries for frontend consumption

**Tools**:
- `format_discovery(insight_data)` → Convert analysis to frontend JSON format
- `generate_title(insight)` → Compelling, specific title
- `generate_description(insight)` → Concise summary (2-3 sentences)
- `format_action_plan(recommendations)` → Structured action steps
- `assign_priority(insight)` → Critical, medium, low based on impact + urgency
- `assign_tags(insight)` → Hashtags for filtering (#CUSTOMERRETENTION, #REVENUE, etc.)

**Methods**:
- Template-based formatting
- LLM generation for natural language titles/descriptions
- Priority scoring algorithm (impact × urgency × confidence)

**Example Output**:
```json
{
  "id": "discovery_20250921_003",
  "title": "First-time brow customers need senior staff assignment",
  "category": "Customer Retention",
  "priority": "medium",
  "impact": 12400,
  "confidence": 87,
  "daysAgo": 0,
  "description": "89% of first-time customers book brow waxing ($28 avg), but only 41% return compared to 78% business average. Analysis shows junior staff assignments and lack of relationship building are root causes.",
  "reasoning": "First-timers are randomly assigned to any available staff. Data shows Marcus (hired 8 months ago) has 42% retention vs Jennifer's 89%. Psychology research shows first service visit sets relationship foundation - junior staff should handle returning customers who already trust the brand.",
  "actionPlan": [
    "Assign all first-time brow appointments to Jennifer or Amanda (senior staff with >75% retention)",
    "Train Marcus on first-time customer protocol (extra 5 min consultation, product recommendation, relationship building)",
    "Add 'New Customer' flag in booking system to trigger assignment rule automatically",
    "Track retention rate weekly for 30 days to measure improvement"
  ],
  "script": "Hi [Name]! I see this is your first time with us - welcome! I'm Jennifer and I've been doing brows for 6 years here at Bashful Beauty. Let me take an extra minute to understand exactly what you're looking for and I'll make sure you leave loving your brows. After we're done, I'll also show you our brow serum that helps them grow in perfectly between visits.",
  "expectedRecovery": "Increase first-time retention from 41% to 65% = 3.1 additional returning customers per month × $28 avg × 6 visits/year = $521/month = $6,252/year. Industry benchmarks suggest 20-30% improvement is realistic with this change.",
  "tags": ["#CUSTOMERRETENTION", "#TRAINING", "#REVENUE"],
  "relatedCustomer": null,
  "relatedEmployee": {
    "id": "3",
    "name": "Marcus Johnson",
    "role": "Esthetician"
  },
  "completed": false,
  "completedAt": null,
  "createdAt": "2025-09-21T03:15:00Z"
}
```

---

## Memory System

### Three-Tier Architecture

#### 1. Core Memory (Firestore)
**Purpose**: Permanent business identity and preferences

**Storage**:
```
businesses/{businessId}
  ├─ name: "Bashful Beauty"
  ├─ type: "Spa & Wellness"
  ├─ location: "San Diego, CA"
  ├─ employees: [...]
  ├─ owner_preferences: {
       communication_style: "data_with_story",
       detail_level: "moderate",
       action_bias: "high"
     }
  ├─ square_credentials: { encrypted... }
  └─ createdAt: timestamp

users/{uid}
  ├─ displayName: "Steven Ray"
  ├─ email: "steven@bashfulbeauty.com"
  ├─ businessId: "bashful_beauty_123"
  ├─ role: "owner"
  └─ preferences: {...}
```

**Access Pattern**: Read once at session start, cache for duration

#### 2. Working Memory (Redis, 7-day TTL)
**Purpose**: Current investigations and conversations

**Storage**:
```
investigation:{business_id}:{investigation_id} → {
  "started_at": "2025-09-21T03:00:00Z",
  "agent": "analyst",
  "mode": "discovery_mode",
  "state": {
    "current_step": "customer_retention_analysis",
    "mcps_called": ["pattern_detector", "segment_analyzer"],
    "intermediate_findings": [...]
  },
  "ttl": 604800  // 7 days
}

conversation:{business_id}:{thread_id} → {
  "messages": [...],
  "context": {...},
  "ttl": 604800
}
```

**Access Pattern**: High frequency during nightly analysis, clear after 7 days

#### 3. Pattern Memory (Vertex AI Vector Search + BigQuery)
**Purpose**: Long-term learned patterns and historical insights

**Vector Search Index**:
- Stores embedding vectors of past discoveries
- Enables semantic similarity search
- Example: "Similar pattern detected 3 months ago with different customer segment"

**BigQuery Table**:
```sql
CREATE TABLE keeper.pattern_history (
  pattern_id STRING,
  business_id STRING,
  discovery_id STRING,
  pattern_type STRING,  -- 'retention_gap', 'revenue_opportunity', etc.
  embedding ARRAY<FLOAT64>,
  metadata JSON,
  outcome JSON,  -- Did owner act on it? What was the result?
  created_at TIMESTAMP
);
```

**Usage**:
- Analyst Agent queries vector search: "Find patterns similar to current finding"
- Returns historical context: "You had a similar retention issue with facial customers 4 months ago. Owner implemented training program, retention improved from 52% to 71% in 60 days."

---

## Infrastructure (Google Cloud Platform)

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Layer                               │
│  Nuxt 3 Frontend (Keeper_UI) → Firebase Hosting                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway                                 │
│  Cloud Run (FastAPI) → Auth, Discovery Fetch, Chat              │
└─────────────────────┬───────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┬──────────────┐
        ▼             ▼             ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Firestore│  │  Redis   │  │ BigQuery │  │  Vertex  │
  │ (Core    │  │ (Working │  │ (Square  │  │  AI      │
  │  Memory) │  │  Memory) │  │  Cache + │  │ (Vector  │
  │          │  │          │  │ Analytics│  │  Search) │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Nightly Discovery Engine                         │
│  Cloud Scheduler (3:00 AM) → Pub/Sub → Cloud Run (Discovery)    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. Analyst Agent (discovery_mode)                       │   │
│  │     → Orchestrates 31 MCPs                               │   │
│  │     → Calls Vertex AI Gemini Pro / Claude Opus 4        │   │
│  │     → Generates 5-10 insights                            │   │
│  │  2. Advisor Agent (action_planning_mode)                 │   │
│  │     → Converts insights to discoveries                   │   │
│  │     → Formats with output_formatter.mcp                  │   │
│  │  3. Store discoveries in Firestore                       │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┬──────────────┐
        ▼             ▼             ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  Square  │  │  Cloud   │  │  Cloud   │  │  Cloud   │
  │   Sync   │  │Functions │  │ Storage  │  │ Storage  │
  │          │  │ (Web     │  │ (Scraped │  │ (MCP     │
  │ Nightly  │  │Scraping) │  │External  │  │ Files)   │
  │12:01 AM  │  │  1AM     │  │  Data)   │  │          │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### Component Details

#### 1. Cloud Run Services

**API Service** (`keeper-api`)
- FastAPI application
- Handles user authentication, discovery fetching, chat
- Auto-scales 0-10 instances
- 512 MB memory, 1 vCPU
- Cost: ~$5/month per 100 customers (mostly idle)

**Discovery Engine** (`keeper-discovery`)
- Runs nightly analysis
- Contains 3 agents + 31 MCPs
- 4 GB memory, 2 vCPU (for LLM calls)
- Timeout: 30 minutes
- Cost: ~$15/month per 100 customers (runs once per night per customer)

**Web Scraper** (`keeper-scraper`)
- Cloud Functions (HTTP triggered)
- Puppeteer/Playwright for scraping
- Runs at 1 AM (before discovery at 3 AM)
- 2 GB memory, 5 min timeout
- Cost: ~$2/month per customer

#### 2. Data Storage

**Firestore**
- Core Memory (business profiles, user preferences)
- Discovery storage (generated insights)
- Cost: ~$0.50/month per customer (low read/write volume)

**Memorystore for Redis**
- Working Memory (7-day TTL)
- 1 GB shared instance
- Cost: ~$50/month (shared across all customers)

**BigQuery**
- Square data cache (synced nightly)
- Pattern history table
- Analytics pre-computation
- Cost: ~$5/month per customer (1 GB storage + nightly queries)

**Cloud Storage**
- External data (scraped reviews, competitor data)
- MCP tool files (Python modules)
- Cost: ~$0.10/month per customer

**Vertex AI Vector Search**
- Pattern Memory embeddings
- Similarity search for historical patterns
- Cost: ~$10/month (shared index)

#### 3. Orchestration

**Cloud Scheduler**
- Triggers at 12:01 AM (Square sync), 1:00 AM (scraping), 3:00 AM (discovery)
- Cost: Free tier

**Pub/Sub**
- Message queue for scheduled jobs
- Cost: Free tier (low volume)

#### 4. AI/ML Services

**Vertex AI (Gemini Pro)**
- Standard discoveries (80% of insights)
- Cost: ~$15/month per customer

**Anthropic API (Claude Opus 4)**
- High-impact discoveries (20% of insights)
- Cost: ~$25/month per customer

**Vertex AI Natural Language API**
- Sentiment analysis for reviews
- Cost: ~$0.50/month per customer

---

## Data Flow

### Nightly Discovery Flow (3:00 AM)

```
1. Cloud Scheduler triggers Pub/Sub message
2. Pub/Sub invokes keeper-discovery Cloud Run service
3. Discovery service loads business context:
   - memory.mcp.get_business_profile() → Firestore
   - memory.mcp.get_employee_profiles() → Firestore

4. Analyst Agent enters discovery_mode:

   a) Data Collection Phase (5-10 seconds):
      - square_data.mcp.get_transactions(last_30_days)
      - square_data.mcp.get_customers()
      - square_data.mcp.get_employees()
      - external_data.mcp.get_reviews(last_30_days)
      - external_data.mcp.get_competitor_data(radius=5mi)

   b) Analysis Phase (30-60 seconds):
      - pattern_detector.mcp.find_retention_patterns()
      - pattern_detector.mcp.find_revenue_patterns()
      - segment_analyzer.mcp.segment_by_behavior()
      - time_surgeon.mcp.analyze_by_day_of_week()
      - employee_analyzer.mcp.compare_employees()
      - financial_surgeon.mcp.find_upsell_opportunities()

   c) Intelligence Phase (20-30 seconds):
      - For each pattern found:
        - customer_psychology.mcp.explain_customer_behavior()
        - spa_wisdom.mcp.get_best_practice()
        - competitor_intel.mcp.compare_to_competitors()

   d) External Intelligence Phase (10-20 seconds):
      - review_monitor.mcp.extract_common_complaints()
      - customer_migration.mcp.find_lost_customers_at_competitors()
      - social_listening.mcp.track_brand_mentions()

   e) Synthesis Phase (60-90 seconds):
      - Analyst Agent calls Vertex AI Gemini Pro with:
        - All MCP findings (pre-computed summaries, not raw data)
        - Business context from memory
        - Historical patterns from pattern_memory
      - LLM generates 10-15 raw insights with reasoning

   f) Validation Phase (10-20 seconds):
      - For each insight:
        - data_validator.mcp.validate_supporting_data()
        - confidence_scorer.mcp.calculate_overall_confidence()
        - impact_predictor.mcp.predict_revenue_impact()
      - Filter to top 5-10 insights (highest impact × confidence)

5. Advisor Agent enters action_planning_mode:

   a) For each validated insight:
      - script_generator.mcp.generate_customer_outreach_script()
        OR campaign_builder.mcp.build_retention_campaign()
        OR optimization_architect.mcp.design_process_improvement()
      - owner_psychology.mcp.suggest_insight_framing()

   b) Synthesis Phase (30-60 seconds):
      - Advisor Agent calls Claude Opus 4 (for top 2 critical insights)
        OR Gemini Pro (for standard insights)
      - Generates: title, description, actionPlan, script, expectedRecovery

   c) Formatting Phase (5-10 seconds):
      - output_formatter.mcp.format_discovery() for each insight
      - Assign priority, tags, related entities

6. Store discoveries in Firestore:

   discoveries/{business_id}/insights/{discovery_id} → {
     id: "discovery_20250921_003",
     title: "...",
     category: "...",
     priority: "medium",
     impact: 12400,
     confidence: 87,
     ...
     createdAt: "2025-09-21T03:15:00Z"
   }

7. Update stats in Firestore:

   businesses/{business_id}/stats → {
     lastDiscoveryRun: "2025-09-21T03:15:00Z",
     totalDiscoveries: 47,
     pendingDiscoveries: 8,
     completedDiscoveries: 39,
     totalROI: 45600,
     ...
   }

8. Total time: 3-5 minutes per business
```

### User Login Flow (Morning)

```
1. User visits keeper.tools, logs in with Firebase Auth
2. Frontend calls Cloud Run API: GET /discoveries?businessId=...
3. API fetches from Firestore:
   - discoveries/{businessId}/insights (where createdAt > 7 days ago)
   - businesses/{businessId}/stats
4. Returns JSON array of discoveries (already formatted by output_formatter.mcp)
5. Frontend renders discoveries list (pages/discoveries/index.vue)
6. User sees fresh insights generated overnight
```

### User Chat Flow (Optional)

```
1. User clicks "Ask about this discovery" OR types question in chat
2. Frontend calls Cloud Run API: POST /chat
   {
     businessId: "...",
     threadId: "...",  // or null for new thread
     message: "Why did you say Marcus has low retention?"
   }
3. API invokes Investigator Agent (question_answering_mode):

   a) Load conversation context:
      - memory.mcp.get_conversation(threadId) from Redis

   b) Route question to MCPs:
      - employee_analyzer.mcp.get_employee_performance("marcus")
      - customer_forensics.mcp.get_customers_by_employee("marcus")
      - pattern_detector.mcp.find_retention_patterns(employee="marcus")

   c) Call LLM (Gemini Pro) with:
      - User question
      - MCP data
      - Conversation history
      - Business context

   d) Generate natural language answer with citations

   e) Save to Working Memory:
      - memory.mcp.save_conversation(threadId, messages)

4. Return answer to frontend
5. Frontend displays in chat interface
```

---

## Discovery Generation Examples

### Example 1: Multi-Dimensional Customer Retention Discovery

**MCPs Contributing**:
1. `pattern_detector.mcp` → "59% of brow waxing first-timers never return"
2. `segment_analyzer.mcp` → "156 first-time brow customers in last 6 months, avg ticket $28"
3. `customer_forensics.mcp` → "No upsells recorded for this segment"
4. `employee_analyzer.mcp` → "Marcus (42% retention) serviced 67% of first-timers, Jennifer (89% retention) serviced 18%"
5. `time_surgeon.mcp` → "First-timers typically book Tuesday/Wednesday mornings (Marcus shifts)"
6. `customer_psychology.mcp` → "First-time service customers need relationship building and extra attention"
7. `spa_wisdom.mcp` → "Industry best practice: assign senior staff to new customers for retention"
8. `competitor_intel.mcp` → "Competitor 'Glow Spa' retention is 78%, your overall is 67%"
9. `confidence_scorer.mcp` → "87% confidence (p-value: 0.001, sample: 156)"
10. `impact_predictor.mcp` → "Increase retention 41% → 65% = $6,252/year"

**Analyst Output**:
```json
{
  "pattern": "first_time_brow_retention_gap",
  "finding": "89% of first-time customers book brow waxing as entry service, but only 41% return (vs 78% business average). Root cause: Random staff assignment places 67% with junior staff (Marcus, 42% retention) vs 18% with senior staff (Jennifer, 89% retention).",
  "confidence": 87,
  "impact_estimate": 6252,
  "supporting_mcps": [
    "pattern_detector", "segment_analyzer", "employee_analyzer",
    "customer_psychology", "spa_wisdom"
  ]
}
```

**Advisor Output** (Final Discovery):
```json
{
  "id": "discovery_20250921_003",
  "title": "First-time brow customers need senior staff assignment",
  "category": "Customer Retention",
  "priority": "medium",
  "impact": 6252,
  "confidence": 87,
  "description": "89% of first-time customers book brow waxing ($28 avg), but only 41% return compared to 78% business average. Analysis shows junior staff assignments and lack of relationship building are root causes.",
  "reasoning": "First-timers are randomly assigned to any available staff. Data shows Marcus (hired 8 months ago) has 42% retention vs Jennifer's 89%. Psychology research shows first service visit sets relationship foundation.",
  "actionPlan": [
    "Assign all first-time brow appointments to Jennifer or Amanda (senior staff)",
    "Train Marcus on first-time customer protocol (extra 5 min consultation, product recommendation)",
    "Add 'New Customer' flag in booking system to trigger assignment rule",
    "Track retention rate weekly for 30 days"
  ],
  "script": "Hi [Name]! I see this is your first time - welcome! I'm Jennifer and I've been doing brows for 6 years. Let me take an extra minute to understand exactly what you're looking for...",
  "expectedRecovery": "Increase retention 41% → 65% = $6,252/year",
  "tags": ["#CUSTOMERRETENTION", "#TRAINING"]
}
```

### Example 2: External Intelligence Discovery

**MCPs Contributing**:
1. `customer_forensics.mcp` → "Sarah Chen, $2,280 LTV, 47 days since last visit (pattern: 18-22 days)"
2. `pattern_detector.mcp` → "Sarah's last 3 visits: declining tips 20% → 12% → 0%"
3. `square_data.mcp` → "Last appointment: Jennifer was 15 min late (noted in booking)"
4. `customer_migration.mcp` → "Sarah reviewed competitor 'Glow Spa' 12 days ago on Google"
5. `social_listening.mcp` → "Sarah's friend Lisa posted on Facebook about trying Glow Spa"
6. `customer_psychology.mcp` → "Declining tips before defection = passive-aggressive feedback, 0% tip = strong defection signal"
7. `script_generator.mcp` → Generated personalized outreach script
8. `confidence_scorer.mcp` → "96% confidence (clear behavioral signals)"
9. `impact_predictor.mcp` → "60% recovery chance if contacted today, $2,280 at stake"

**Final Discovery**:
```json
{
  "id": "discovery_20250921_001",
  "title": "Sarah Chen hasn't been in for 47 days",
  "category": "Customer Defection Alert",
  "priority": "critical",
  "impact": 2280,
  "confidence": 96,
  "description": "Her 3-year pattern was every 18-22 days spending $95 on Brazilian + brow. Last 3 visits: declining tips to Jennifer (20% → 12% → 0%).",
  "reasoning": "Jennifer was 15 minutes late on last visit. Sarah's friend Lisa posted on Facebook about 'trying that new place downtown' (Glow Spa). Sarah left Google review for Glow 12 days ago.",
  "actionPlan": [
    "Call Sarah TODAY (not text - needs personal touch)",
    "Jennifer should make the call (relationship repair)",
    "Offer: '20% off next visit' + specific apology for being late",
    "Book her THIS WEEK while she still remembers you"
  ],
  "script": "Hi Sarah, it's Jennifer from Bashful Beauty. I realized I kept you waiting last time and I felt terrible about it. I'd love to make it right - can I book you this week with 20% off? I miss seeing you!",
  "expectedRecovery": "60% chance if contacted today, 40% this week, 15% after that. $2,280 lifetime value at stake.",
  "tags": ["#CUSTOMERRETENTION", "#HIGHVALUE", "#URGENT"],
  "relatedCustomer": {
    "id": "1",
    "name": "Sarah Chen",
    "email": "sarah.chen@email.com"
  }
}
```

---

## Cost Model (Per Customer Per Month)

### LLM Costs

**Vertex AI (Gemini Pro 2.0)**
- Nightly discovery: 30 API calls × 5K tokens avg = 150K tokens/night
- 30 nights = 4.5M tokens/month
- Input: $0.075/1M tokens = $0.34
- Output: $0.30/1M tokens = $1.35
- **Total: $1.69/month**

**Anthropic API (Claude Opus 4)** - 20% of discoveries
- 6 API calls/month × 8K tokens avg = 48K tokens/month
- Input: $15/1M tokens = $0.72
- Output: $75/1M tokens = $3.60
- **Total: $4.32/month**

**Vertex AI Natural Language API** (sentiment analysis)
- 30 reviews/month × $0.001 = $0.03/month

**Total LLM: $6.04/month**

### Infrastructure Costs

**BigQuery**
- Storage: 1 GB × $0.02/GB = $0.02
- Queries: 5 GB processed/month × $5/TB = $0.03
- **Total: $0.05/month**

**Firestore**
- Storage: 100 KB × $0.18/GB = negligible
- Reads: 100/day × 30 = 3000 × $0.06/100K = $0.002
- Writes: 20/day × 30 = 600 × $0.18/100K = $0.001
- **Total: $0.01/month**

**Memorystore Redis** (shared across all customers)
- 1 GB instance = $50/month ÷ 100 customers = $0.50/customer

**Cloud Storage**
- 100 MB × $0.02/GB = $0.002/month

**Cloud Run**
- API service: ~1 hour compute/month × $0.00002400/vCPU-second = $0.09
- Discovery service: ~2 hours/month × $0.00004800/vCPU-second = $0.35
- **Total: $0.44/month**

**Cloud Functions** (web scraping)
- 30 runs/month × 3 min avg × $0.000000648/GB-sec × 2GB = $0.11

**Vertex AI Vector Search** (shared)
- Index cost: $10/month ÷ 100 customers = $0.10/customer

**Total Infrastructure: $1.21/month**

### External Data Costs

**Web Scraping** (no API costs)
- Cloud Functions compute (included above)
- **Total: $0/month**

Previously estimated APIs (NOT USED):
- ~~Yelp API: $4~~
- ~~SafeGraph: $12~~
- ~~Social listening API: $8~~

### Grand Total COGS: **$7.25/month per customer**

**At $99/month pricing:**
- Gross margin: $91.75 (92.7%)
- At 1,000 customers: $91,750/month gross profit
- At 10,000 customers: $917,500/month gross profit

**Cost Breakdown**:
- LLM costs: 83% ($6.04)
- Infrastructure: 17% ($1.21)

**Optimization opportunities**:
- Use cheaper Gemini Flash for low-priority discoveries (could reduce LLM costs by 30%)
- Batch API calls to reduce overhead (could reduce infrastructure by 20%)
- Shared infrastructure scales well (Redis, Vector Search already shared)

---

## Implementation Guide for Greg

### Phase 1: Foundation (Week 1-2)

**Goal**: Set up infrastructure and data pipeline

1. **GCP Project Setup**
   - Create GCP project: `keeper-production`
   - Enable APIs: BigQuery, Firestore, Cloud Run, Cloud Functions, Vertex AI, Memorystore
   - Set up billing alerts

2. **Square Data Sync**
   - Create BigQuery dataset: `keeper`
   - Create table: `square_cache` (schema: transactions, customers, employees, catalog)
   - Build Cloud Function: `square-sync` (Python)
     - OAuth flow with Square API
     - Incremental sync (last 24 hours)
     - Store in BigQuery
   - Set up Cloud Scheduler: runs daily at 12:01 AM

3. **External Data Scraping**
   - Build Cloud Function: `scraper` (Python with Playwright)
   - Scrape: Google reviews, Yelp reviews, competitor websites
   - Store in Cloud Storage: `gs://keeper-external-data/{business_id}/`
   - Set up Cloud Scheduler: runs daily at 1:00 AM

4. **Memory System**
   - Firestore setup: `businesses/`, `users/`, `discoveries/` collections
   - Memorystore Redis: 1 GB instance
   - Vertex AI Vector Search: create index for pattern embeddings

### Phase 2: MCP Implementation (Week 3-5)

**Goal**: Build all 31 MCPs as Python modules

**MCP Structure**:
```python
# mcps/square_data.mcp.py
from google.cloud import bigquery
from typing import List, Dict, Any
import os

class SquareDataMCP:
    def __init__(self, business_id: str):
        self.business_id = business_id
        self.bq_client = bigquery.Client()

    def get_transactions(
        self,
        start_date: str,
        end_date: str,
        filters: Dict[str, Any] = None
    ) -> List[Dict]:
        """Fetch transactions from BigQuery cache"""
        query = f"""
        SELECT
          transaction_id,
          customer_id,
          employee_id,
          service_name,
          amount_cents,
          tip_cents,
          created_at
        FROM `keeper.square_cache`
        WHERE business_id = '{self.business_id}'
          AND created_at BETWEEN '{start_date}' AND '{end_date}'
        """
        # Add filters if provided
        if filters:
            # ... build WHERE clauses
            pass

        results = self.bq_client.query(query).result()
        return [dict(row) for row in results]

    def get_customers(self, filters: Dict[str, Any] = None) -> List[Dict]:
        """Fetch customer profiles"""
        # ... implementation
        pass

    def get_employees(self, filters: Dict[str, Any] = None) -> List[Dict]:
        """Fetch employee profiles"""
        # ... implementation
        pass
```

**Implementation Order** (prioritize by dependency):
1. Layer 1 (Data Access): `square_data.mcp`, `external_data.mcp`, `memory.mcp`
2. Layer 2 (Analysis): `pattern_detector.mcp`, `segment_analyzer.mcp`, etc.
3. Layer 3 (Intelligence): `customer_psychology.mcp`, etc.
4. Layer 4 (Wisdom): `spa_wisdom.mcp`, etc.
5. Layer 5 (External Intelligence): `review_monitor.mcp`, etc.
6. Layer 6 (Action): `script_generator.mcp`, etc.
7. Layer 7 (Validation): `data_validator.mcp`, etc.

**Testing Strategy**:
- Use Bashful Beauty real data (8 years, 2000+ transactions)
- Unit test each MCP tool independently
- Integration test MCPs together
- Validate output format matches frontend expectations

### Phase 3: Agent Implementation (Week 6-7)

**Goal**: Build 3 orchestrator agents

**Agent Structure**:
```python
# agents/analyst_agent.py
from typing import List, Dict, Any
from mcps import *
import vertexai
from vertexai.preview.generative_models import GenerativeModel

class AnalystAgent:
    def __init__(self, business_id: str):
        self.business_id = business_id
        self.mode = None
        self.mcps = {
            'square_data': SquareDataMCP(business_id),
            'pattern_detector': PatternDetectorMCP(business_id),
            # ... load all MCPs
        }
        self.llm = GenerativeModel("gemini-2.0-pro")

    def discovery_mode(self) -> List[Dict]:
        """Nightly comprehensive analysis"""
        self.mode = "discovery_mode"

        # 1. Data Collection Phase
        transactions = self.mcps['square_data'].get_transactions(
            start_date="2025-08-22",
            end_date="2025-09-22"
        )
        customers = self.mcps['square_data'].get_customers()
        employees = self.mcps['square_data'].get_employees()
        reviews = self.mcps['external_data'].get_reviews(
            source="google",
            date_range="last_30_days"
        )

        # 2. Analysis Phase
        retention_patterns = self.mcps['pattern_detector'].find_retention_patterns()
        segments = self.mcps['segment_analyzer'].segment_by_behavior()
        employee_performance = self.mcps['employee_analyzer'].compare_employees()

        # 3. Intelligence Phase
        insights = []
        for pattern in retention_patterns:
            psychology = self.mcps['customer_psychology'].explain_customer_behavior(
                pattern=pattern,
                context={"business_type": "spa"}
            )
            best_practice = self.mcps['spa_wisdom'].get_best_practice(
                situation=pattern['type']
            )
            insights.append({
                "pattern": pattern,
                "psychology": psychology,
                "best_practice": best_practice
            })

        # 4. Synthesis Phase (LLM)
        prompt = self._build_synthesis_prompt(insights)
        response = self.llm.generate_content(prompt)
        raw_insights = self._parse_llm_response(response.text)

        # 5. Validation Phase
        validated_insights = []
        for insight in raw_insights:
            confidence = self.mcps['confidence_scorer'].calculate_overall_confidence(insight)
            impact = self.mcps['impact_predictor'].predict_revenue_impact(insight)

            if confidence >= 80 and impact >= 1000:
                insight['confidence'] = confidence
                insight['impact'] = impact
                validated_insights.append(insight)

        # Sort by impact × confidence, take top 10
        validated_insights.sort(key=lambda x: x['impact'] * x['confidence'], reverse=True)
        return validated_insights[:10]

    def _build_synthesis_prompt(self, insights: List[Dict]) -> str:
        """Build LLM prompt from MCP findings"""
        prompt = f"""You are analyzing data for {self.business_id}, a spa & wellness business.

Your goal is to find 10-15 transformative insights that will help the owner grow revenue and retain customers.

Here are the findings from our analysis tools:

{json.dumps(insights, indent=2)}

For each significant pattern you identify, provide:
1. Title (compelling, specific)
2. Description (2-3 sentences explaining what's happening)
3. Reasoning (why this is happening, root causes)
4. Impact estimate (annual revenue opportunity)

Focus on multi-dimensional insights that combine multiple data sources, not simple alerts.

Example of good insight: "89% of first-time customers book brow waxing but only 41% return (vs 78% average). Root cause: junior staff assigned to first-timers lack relationship-building skills. Opportunity: $6,252/year."

Example of bad insight: "Sarah Chen hasn't been in for 47 days."

Generate insights in JSON format:
[
  {{
    "title": "...",
    "description": "...",
    "reasoning": "...",
    "impact_estimate": 6252
  }}
]
"""
        return prompt
```

**Agent Testing**:
- Test each mode independently
- Validate LLM prompts produce expected output
- Monitor token usage and costs
- Test with real Bashful Beauty data

### Phase 4: Discovery Engine (Week 8-9)

**Goal**: Orchestrate agents into nightly pipeline

**Discovery Service**:
```python
# services/discovery_service.py
from agents import AnalystAgent, AdvisorAgent
from google.cloud import firestore
import os

def run_discovery(business_id: str):
    """Main discovery pipeline - runs nightly at 3 AM"""

    print(f"[{business_id}] Starting discovery run...")

    # 1. Initialize agents
    analyst = AnalystAgent(business_id)
    advisor = AdvisorAgent(business_id)

    # 2. Run analysis
    raw_insights = analyst.discovery_mode()
    print(f"[{business_id}] Analyst found {len(raw_insights)} insights")

    # 3. Convert to discoveries
    discoveries = []
    for insight in raw_insights:
        discovery = advisor.action_planning_mode(insight)
        discoveries.append(discovery)

    # 4. Store in Firestore
    db = firestore.Client()
    batch = db.batch()

    for discovery in discoveries:
        doc_ref = db.collection('discoveries').document(business_id).collection('insights').document(discovery['id'])
        batch.set(doc_ref, discovery)

    batch.commit()
    print(f"[{business_id}] Stored {len(discoveries)} discoveries")

    # 5. Update stats
    stats_ref = db.collection('businesses').document(business_id)
    stats_ref.update({
        'lastDiscoveryRun': firestore.SERVER_TIMESTAMP,
        'totalDiscoveries': firestore.Increment(len(discoveries))
    })

    return discoveries

# Cloud Run entrypoint
if __name__ == "__main__":
    import sys
    business_id = os.environ.get('BUSINESS_ID') or sys.argv[1]
    run_discovery(business_id)
```

**Deployment**:
```bash
# Build Docker image
docker build -t gcr.io/keeper-production/discovery-service .

# Deploy to Cloud Run
gcloud run deploy keeper-discovery \
  --image gcr.io/keeper-production/discovery-service \
  --platform managed \
  --region us-central1 \
  --memory 4Gi \
  --cpu 2 \
  --timeout 30m \
  --no-allow-unauthenticated

# Set up Cloud Scheduler
gcloud scheduler jobs create http discovery-bashful-beauty \
  --schedule="0 3 * * *" \
  --uri="https://keeper-discovery-xxx.run.app" \
  --http-method=POST \
  --message-body='{"business_id": "bashful_beauty_123"}' \
  --oidc-service-account-email=scheduler@keeper-production.iam.gserviceaccount.com
```

### Phase 5: API Service (Week 10)

**Goal**: Build Cloud Run API for frontend

**API Structure**:
```python
# api/main.py
from fastapi import FastAPI, Depends, HTTPException
from google.cloud import firestore
from firebase_admin import auth
import firebase_admin

firebase_admin.initialize_app()
app = FastAPI()
db = firestore.Client()

async def verify_token(authorization: str):
    """Verify Firebase ID token"""
    try:
        token = authorization.split('Bearer ')[1]
        decoded_token = auth.verify_id_token(token)
        return decoded_token
    except Exception as e:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/discoveries")
async def get_discoveries(
    businessId: str,
    user=Depends(verify_token)
):
    """Fetch discoveries for business"""
    # Verify user has access to business
    user_ref = db.collection('users').document(user['uid']).get()
    if user_ref.get('businessId') != businessId:
        raise HTTPException(status_code=403, detail="Access denied")

    # Fetch discoveries (last 30 days)
    discoveries = []
    docs = db.collection('discoveries').document(businessId).collection('insights').stream()
    for doc in docs:
        discoveries.append(doc.to_dict())

    return {"discoveries": discoveries}

@app.post("/chat")
async def chat(
    request: dict,
    user=Depends(verify_token)
):
    """Handle chat message"""
    from agents import InvestigatorAgent

    business_id = request['businessId']
    message = request['message']
    thread_id = request.get('threadId')

    # Initialize agent
    investigator = InvestigatorAgent(business_id)

    # Generate response
    response = investigator.question_answering_mode(
        question=message,
        thread_id=thread_id
    )

    return {"response": response}

@app.post("/discoveries/{discoveryId}/complete")
async def mark_complete(
    discoveryId: str,
    businessId: str,
    user=Depends(verify_token)
):
    """Mark discovery as completed"""
    doc_ref = db.collection('discoveries').document(businessId).collection('insights').document(discoveryId)
    doc_ref.update({
        'completed': True,
        'completedAt': firestore.SERVER_TIMESTAMP
    })
    return {"success": True}
```

**Deployment**:
```bash
# Deploy API to Cloud Run
gcloud run deploy keeper-api \
  --source . \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
```

### Phase 6: Integration with Frontend (Week 11)

**Goal**: Connect Keeper_UI to API

Update Keeper_UI to call Cloud Run API:
```typescript
// Keeper_UI/composables/useDiscoveries.ts
export const useDiscoveries = () => {
  const { currentUser } = useAuth()
  const config = useRuntimeConfig()

  const fetchDiscoveries = async (businessId: string) => {
    const token = await currentUser.value.getIdToken()

    const response = await fetch(
      `${config.public.apiUrl}/discoveries?businessId=${businessId}`,
      {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      }
    )

    return await response.json()
  }

  return { fetchDiscoveries }
}
```

### Phase 7: Testing & Optimization (Week 12)

**Goal**: Test with real data and optimize

1. **Load Test**
   - Simulate 100 businesses running nightly
   - Monitor Cloud Run scaling
   - Check BigQuery query performance

2. **Cost Optimization**
   - Profile LLM token usage
   - Identify expensive queries
   - Implement caching where beneficial

3. **Quality Assurance**
   - Validate discoveries against business owner expectations (use Bashful Beauty manager)
   - Check confidence scores correlate with actual business outcomes
   - Refine LLM prompts based on output quality

---

## Next Steps for Greg

1. **Read this specification thoroughly**
2. **Set up GCP project** and enable required APIs
3. **Start with Phase 1** (Foundation) - get Square data flowing into BigQuery
4. **Build MCPs incrementally** - start with Layer 1 (Data Access), test thoroughly
5. **Implement Analyst Agent first** - this is the core discovery engine
6. **Test with Bashful Beauty data** - we have 8 years of real spa data to validate
7. **Iterate based on discoveries** - refine prompts and logic based on output quality

**Questions for Greg**:
- Which GCP region should we use? (Recommend: us-central1 for Vertex AI availability)
- Do we need multi-region support initially? (Recommend: No, single region for MVP)
- What's your preferred Python framework for MCPs? (Recommend: Pydantic for data validation)
- Should we use Docker for local development? (Recommend: Yes, with docker-compose for full stack)

**Resources**:
- Anthropic MCP Article: https://www.anthropic.com/engineering/code-execution-with-mcp
- Keeper_UI Repo: https://github.com/defendersoftheinternet/Keeper_UI
- Bashful Beauty Square Data: Available via OAuth (credentials in 1Password)
- GCP Documentation: https://cloud.google.com/docs

---

## Appendix: Discovery Schema

**Frontend expects discoveries in this format** (from `/Users/rayhernandez/Keeper_UI/composables/useMockData.ts`):

```typescript
interface Discovery {
  id: string                      // Unique ID: "discovery_20250921_003"
  title: string                   // Compelling, specific title
  category: string                // "Customer Defection Alert", "Revenue Optimization", etc.
  priority: 'critical' | 'medium' | 'low'
  impact: number                  // Annual revenue impact in dollars (not formatted)
  confidence: number              // 0-100 confidence score
  daysAgo?: number                // For time-sensitive discoveries
  description: string             // 2-3 sentence summary
  reasoning: string               // Why this is happening (root causes)
  actionPlan: string[]            // 3-5 specific action steps
  script?: string                 // Optional conversation script
  expectedRecovery: string        // Expected outcome with timeframe
  tags: string[]                  // ["#CUSTOMERRETENTION", "#HIGHVALUE"]
  relatedCustomer?: object        // If discovery is about specific customer
  relatedEmployee?: object        // If discovery is about specific employee
  completed: boolean              // Has owner marked this complete?
  completedAt: string | null      // ISO timestamp when completed
  createdAt?: string              // ISO timestamp when generated
}
```

**Example from mock data**:
```json
{
  "id": "1",
  "title": "Sarah Chen hasn't been in for 47 days",
  "category": "Customer Defection Alert",
  "priority": "critical",
  "impact": 2280,
  "confidence": 96,
  "daysAgo": 47,
  "description": "Her 3-year pattern was every 18-22 days spending $95 on Brazilian + brow service. Last 3 visits: declining tips to Jennifer (20% → 12% → 0%).",
  "reasoning": "Jennifer was 15 minutes late that day. Sarah's friend Lisa posted on Facebook about \"trying that new place downtown.\"",
  "actionPlan": [
    "Call Sarah today (not text - this needs the personal touch)",
    "Jennifer should make the call (relationship repair)",
    "Offer: \"I noticed I was running behind last time - let me make it right with 20% off your next visit\"",
    "Book her THIS WEEK while she still remembers you"
  ],
  "script": "Hi Sarah, it's Jennifer from Bashful Beauty. I realized I kept you waiting last time and I felt terrible about it. I'd love to make it right - can I book you this week with 20% off? I miss seeing you!",
  "expectedRecovery": "60% chance of recovery if contacted today, 40% if contacted this week, 15% after that.",
  "tags": ["#CUSTOMERRETENTION", "#HIGHVALUE"],
  "relatedCustomer": {
    "id": "1",
    "name": "Sarah Chen",
    "email": "sarah.chen@email.com"
  },
  "completed": true,
  "completedAt": "2025-09-15T14:30:00Z"
}
```

---

**End of Specification**

This document should provide Greg with everything needed to implement the Keeper AI backend. For questions or clarifications, refer back to the conversation history or reach out to Ray.
