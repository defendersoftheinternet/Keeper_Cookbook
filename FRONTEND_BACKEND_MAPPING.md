# Keeper Frontend-Backend Mapping

This document maps each page and component in **Keeper_UI** to the specific **MCPs** that provide the data. Use this as a reference when implementing the backend to ensure you're building the right tools for the frontend.

---

## Page-by-Page Breakdown

### 1. Landing Page (`pages/index.vue`)

**Purpose**: Marketing page with ROI calculator, testimonials, pricing

**Data Needed**:
- Static content (no backend needed)
- ROI calculator uses client-side JavaScript

**MCPs**: None (static page)

---

### 2. Dashboard (`pages/dashboard.vue`)

**Purpose**: Main overview showing hero insight, key metrics, and top opportunities

**File**: `/Users/rayhernandez/Keeper_UI/pages/dashboard.vue`

#### Components on Page:

##### A. Hero Insight Card (Top Priority Discovery)
**What it shows**: Single most important discovery with large impact
**Data needed**:
```typescript
{
  title: string,
  category: string,
  priority: 'critical',
  impact: number,
  description: string,
  actionPlan: string[]
}
```

**MCPs to call**:
1. **Read from Firestore** (no MCP, direct query):
   ```
   discoveries/{businessId}/insights
   WHERE priority = 'critical'
   ORDER BY (impact * confidence) DESC
   LIMIT 1
   ```

**Example**:
```typescript
// This data comes from the nightly discovery run
// Analyst Agent → Advisor Agent → output_formatter.mcp → Firestore
const heroInsight = await firestore
  .collection('discoveries')
  .doc(businessId)
  .collection('insights')
  .where('priority', '==', 'critical')
  .where('completed', '==', false)
  .orderBy('impact', 'desc')
  .limit(1)
  .get()
```

---

##### B. Stats Cards (Revenue Metrics)
**What it shows**:
- Total Revenue
- Revenue at Risk
- Hidden Opportunities
- Tasks Count

**Mock data location**: `composables/useMockData.ts` - `stats` object

**MCPs to call**:

**Card 1: Total Revenue**
- **MCP**: `square_data.mcp.get_transactions()`
- **Calculation**: Sum of all transactions in current month
```python
# Backend calculation (stored in Firestore stats)
transactions = square_data.get_transactions(
    start_date='2025-09-01',
    end_date='2025-09-30'
)
total_revenue = sum(txn['amount_cents'] for txn in transactions) / 100
```

**Card 2: Revenue at Risk**
- **MCP**: `customer_forensics.mcp.predict_churn_risk()`
- **Calculation**: Sum of LTV for customers with churn_probability > 0.5
```python
# Get at-risk customers
customers = square_data.get_customers()
at_risk_revenue = 0

for customer in customers:
    churn_risk = customer_forensics.predict_churn_risk(customer['customer_id'])
    if churn_risk['churn_probability'] > 0.5:
        at_risk_revenue += customer['lifetime_value_cents'] / 100
```

**Card 3: Hidden Opportunities**
- **MCP**: `financial_surgeon.mcp.find_upsell_opportunities()`
- **Calculation**: Sum of potential revenue from all discovered opportunities
```python
# Get upsell opportunities
opportunities = financial_surgeon.find_upsell_opportunities()
hidden_revenue = sum(opp['annual_opportunity'] for opp in opportunities)
```

**Card 4: Tasks Count**
- **MCP**: None (read from Firestore discoveries)
- **Calculation**: Count of pending discoveries
```python
# Count pending discoveries
pending_count = firestore.collection('discoveries')
  .doc(businessId)
  .collection('insights')
  .where('completed', '==', False)
  .count()
```

**Data structure to return**:
```typescript
{
  totalRevenue: 42800,
  revenueAtRisk: 12908,
  hiddenOpportunities: 12228,
  tasksCount: 5,
  urgentTasks: 2,
  averageConfidence: 90
}
```

---

##### C. More Opportunities Section (Discovery Cards)
**What it shows**: Grid of 4 discovery cards (non-critical discoveries)

**Data needed**: Same as discoveries page (see below), but limited to 4 and excluding the hero insight

**MCPs to call**:
1. **Read from Firestore**:
   ```
   discoveries/{businessId}/insights
   WHERE completed = false
   WHERE priority != 'critical'
   ORDER BY (impact * confidence) DESC
   LIMIT 4
   ```

---

### 3. Discoveries Page (`pages/discoveries/index.vue`)

**Purpose**: Full list of all discoveries with filtering and sorting

**File**: `/Users/rayhernandez/Keeper_UI/pages/discoveries/index.vue`

#### Components on Page:

##### A. ROI Counter Card (Header)
**What it shows**:
- Estimated Total ROI (from completed discoveries)
- Completed count
- Pending count

**Mock data**: `composables/useMockData.ts` - `stats.completedROI`, `stats.pendingROI`

**MCPs to call**: None (calculated from Firestore discoveries)

**Backend calculation**:
```python
# Calculate in discovery_service.py when storing discoveries
all_discoveries = firestore.collection('discoveries')
  .doc(businessId)
  .collection('insights')
  .get()

completed_roi = sum(
    d['impact'] for d in all_discoveries if d['completed']
)
pending_roi = sum(
    d['impact'] for d in all_discoveries if not d['completed']
)
completed_count = sum(1 for d in all_discoveries if d['completed'])
pending_count = sum(1 for d in all_discoveries if not d['completed'])

# Store in Firestore business stats
firestore.collection('businesses').doc(businessId).update({
    'stats': {
        'completedROI': completed_roi,
        'pendingROI': pending_roi,
        'completedCount': completed_count,
        'pendingCount': pending_count
    }
})
```

**API Endpoint**: `GET /stats?businessId={businessId}`

**Response**:
```typescript
{
  completedROI: 5880,    // Sum of impact from completed discoveries
  pendingROI: 12228,     // Sum of impact from pending discoveries
  completedCount: 2,
  pendingCount: 3,
  totalCount: 5
}
```

---

##### B. Discovery Cards (List)
**What it shows**: All discoveries with filtering (All, Urgent, Revenue, Customers, Team, Completed)

**Mock data**: `composables/useMockData.ts` - `insights` array

**Data structure needed**: See full schema in [Discovery Schema](#discovery-schema) below

**MCPs that generate discoveries** (called during nightly run):

**For discovery: "Sarah Chen hasn't been in for 47 days"**
1. `customer_forensics.mcp.predict_churn_risk(customer_id='sarah_chen')` → Returns churn probability, signals
2. `customer_forensics.mcp.get_customer_journey(customer_id='sarah_chen')` → Returns visit history, declining tips
3. `square_data.mcp.get_customer_timeline(customer_id='sarah_chen')` → Transaction history
4. `customer_migration.mcp.find_lost_customers_at_competitors()` → "Sarah reviewed Glow Spa 12 days ago"
5. `customer_psychology.mcp.explain_customer_behavior()` → "Declining tips = passive-aggressive feedback"
6. `script_generator.mcp.generate_customer_outreach_script()` → "Hi Sarah, it's Jennifer..."
7. `impact_predictor.mcp.predict_revenue_impact()` → "$2,280 at stake, 60% recovery if contacted today"
8. `confidence_scorer.mcp.calculate_overall_confidence()` → 96% confidence
9. `output_formatter.mcp.format_discovery()` → Final JSON for frontend

**For discovery: "Tuesday is secretly losing $129 every week"**
1. `time_surgeon.mcp.analyze_by_day_of_week('retention')` → "Tuesday has 31% retention vs 67% average"
2. `social_listening.mcp.track_brand_mentions()` → "Tuesday has 47 Instagram mentions/month"
3. `segment_analyzer.mcp.segment_by_behavior()` → "67% are one-time influencer customers"
4. `pattern_detector.mcp.find_retention_patterns()` → Statistical significance
5. `campaign_builder.mcp.build_retention_campaign()` → "Bring a Friend Tuesday" strategy
6. `impact_predictor.mcp.predict_revenue_impact()` → "$6,708/year if retention improves 31% → 55%"
7. `output_formatter.mcp.format_discovery()` → Final JSON

**For discovery: "Marcus needs immediate attention"**
1. `employee_analyzer.mcp.get_employee_performance('marcus')` → "42% retention, $59/hour"
2. `employee_analyzer.mcp.compare_employees()` → "Team average: 78% retention, $98/hour"
3. `customer_forensics.mcp.get_customers_by_employee('marcus')` → 67 customers, analyze patterns
4. `employee_psychology.mcp.diagnose_performance_gap()` → "Training gap, not motivation issue"
5. `employee_psychology.mcp.suggest_coaching_approach()` → "Show pre-programmed settings, supportive tone"
6. `script_generator.mcp.generate_employee_coaching_script()` → "Marcus, you're great with customers but..."
7. `impact_predictor.mcp.predict_revenue_impact()` → "$3,600/year, 60% recovery with training"
8. `output_formatter.mcp.format_discovery()` → Final JSON

**For discovery: "12 VIP customers never buy add-ons"**
1. `financial_surgeon.mcp.find_upsell_opportunities()` → "12 customers, avg $285/month, $0 add-ons"
2. `segment_analyzer.mcp.segment_by_value()` → Identify high-value customers
3. `employee_analyzer.mcp.get_employee_performance('jennifer')` → "Jennifer upsells 73% of clients"
4. `employee_analyzer.mcp.get_employee_performance('amanda')` → "Amanda upsells 18% of clients"
5. `customer_forensics.mcp.analyze_customer_preferences()` → "These 12 book when Jennifer is off"
6. `campaign_builder.mcp.build_upsell_campaign()` → "Train Amanda on Jennifer's script"
7. `impact_predictor.mcp.predict_revenue_impact()` → "$4,320/year with 30% conversion"
8. `output_formatter.mcp.format_discovery()` → Final JSON

**API Endpoint**: `GET /discoveries?businessId={businessId}`

**Response**:
```typescript
{
  discoveries: [
    {
      id: "discovery_20250921_001",
      title: "Sarah Chen hasn't been in for 47 days",
      category: "Customer Defection Alert",
      priority: "critical",
      impact: 2280,
      confidence: 96,
      daysAgo: 47,
      description: "Her 3-year pattern was every 18-22 days spending $95...",
      reasoning: "Jennifer was 15 minutes late that day...",
      actionPlan: [
        "Call Sarah today (not text - this needs the personal touch)",
        "Jennifer should make the call (relationship repair)",
        "Offer: '20% off next visit' + specific apology",
        "Book her THIS WEEK"
      ],
      script: "Hi Sarah, it's Jennifer from Bashful Beauty...",
      expectedRecovery: "60% chance if contacted today, 40% this week, 15% after",
      tags: ["#CUSTOMERRETENTION", "#HIGHVALUE"],
      relatedCustomer: {
        id: "1",
        name: "Sarah Chen",
        email: "sarah.chen@email.com",
        avatar: "SC",
        lifetimeValue: 2280
      },
      completed: false,
      completedAt: null,
      createdAt: "2025-09-21T03:15:00Z"
    },
    // ... more discoveries
  ]
}
```

---

##### C. Filters
**What it shows**: All, Urgent, Revenue, Customers, Team, Completed

**Filtering logic** (client-side):
```typescript
// Filter on frontend based on discovery fields
const filters = {
  'all': discoveries,
  'urgent': discoveries.filter(d => d.priority === 'critical' && !d.completed),
  'revenue': discoveries.filter(d =>
    (d.category.includes('Revenue') || d.category.includes('Opportunity')) && !d.completed
  ),
  'customers': discoveries.filter(d => d.category.includes('Customer') && !d.completed),
  'team': discoveries.filter(d => d.category.includes('Employee') && !d.completed),
  'completed': discoveries.filter(d => d.completed)
}
```

**Categories to assign** (in `output_formatter.mcp`):
- "Customer Defection Alert" → Customers filter
- "Customer Retention" → Customers filter
- "Revenue Optimization" → Revenue filter
- "Hidden Revenue Opportunity" → Revenue filter
- "Employee Performance" → Team filter
- "Operational Efficiency" → General
- "Crisis Alert" → Urgent filter (priority='critical')

---

### 4. Discovery Detail Page (`pages/discoveries/[id].vue`)

**Purpose**: Full detail view of a single discovery

**Data needed**: Same as discovery card, but may want to show additional context

**MCPs for additional detail** (optional enhancements):
1. `customer_forensics.mcp.get_customer_journey(customer_id)` → Full timeline for relatedCustomer
2. `employee_analyzer.mcp.get_employee_metrics(employee_id)` → Full stats for relatedEmployee
3. `pattern_detector.mcp.find_similar_patterns()` → "Similar pattern detected 3 months ago"
4. `memory.mcp.search_similar_patterns()` → Historical context

**API Endpoint**: `GET /discoveries/:id?businessId={businessId}`

**Response**: Single discovery object (same schema as list)

---

### 5. Chat Page (`pages/chats.vue`)

**Purpose**: Chat interface to ask questions about discoveries

**Data needed**: Chat messages with AI responses

**MCPs to call** (when user sends message):

**User question**: "Why did you say Marcus has low retention?"

**Agent**: `InvestigatorAgent.question_answering_mode(question)`

**MCPs called by Investigator**:
1. `employee_analyzer.mcp.get_employee_performance('marcus')` → Get Marcus stats
2. `employee_analyzer.mcp.compare_employees()` → Get team averages for comparison
3. `customer_forensics.mcp.get_customers_by_employee('marcus')` → Get Marcus's customers
4. `pattern_detector.mcp.find_retention_patterns(employee='marcus')` → Statistical analysis
5. **Synthesize answer with LLM** (Gemini Pro)

**API Endpoint**: `POST /chat`

**Request**:
```typescript
{
  businessId: "bashful_beauty_123",
  threadId: "thread_abc123",  // or null for new conversation
  message: "Why did you say Marcus has low retention?"
}
```

**Response**:
```typescript
{
  response: "Marcus's retention rate is 42% compared to team average of 78%. Looking at his 67 customers over 8 months:\n- 39 customers (58%) came for appointments only once\n- Average ticket: $78 vs team avg $98\n- Service time: 4:12 min vs team avg 2:45 min\n- Customer feedback mentions 'felt rushed' in 3 reviews\n\nSource: employee_analyzer.mcp + customer_forensics.mcp",
  threadId: "thread_abc123",
  sources: [
    { mcp: "employee_analyzer", tool: "get_employee_performance" },
    { mcp: "customer_forensics", tool: "get_customers_by_employee" }
  ]
}
```

---

## Discovery Schema (Complete)

This is the **exact schema** that `output_formatter.mcp` must produce for the frontend.

**Source**: `/Users/rayhernandez/Keeper_UI/composables/useMockData.ts` (lines 131-234)

```typescript
interface Discovery {
  // Core identification
  id: string                      // Format: "discovery_YYYYMMDD_NNN" (e.g., "discovery_20250921_001")
  createdAt: string               // ISO timestamp: "2025-09-21T03:15:00Z"

  // Display information
  title: string                   // Compelling, specific (e.g., "Sarah Chen hasn't been in for 47 days")
  category: string                // One of:
                                  // - "Customer Defection Alert"
                                  // - "Customer Retention"
                                  // - "Revenue Optimization"
                                  // - "Hidden Revenue Opportunity"
                                  // - "Employee Performance"
                                  // - "Operational Efficiency"
                                  // - "Crisis Alert"

  // Priority and impact
  priority: 'critical' | 'medium' | 'low'
  impact: number                  // Annual revenue impact in dollars (not formatted, no $)
                                  // Example: 2280 (not "$2,280" or "2280K")
  confidence: number              // 0-100 confidence score (integer)
  daysAgo?: number                // Optional: For time-sensitive discoveries (e.g., 47)

  // Content
  description: string             // 2-3 sentence summary of what's happening
                                  // Example: "Her 3-year pattern was every 18-22 days spending $95 on Brazilian + brow service. Last 3 visits: declining tips to Jennifer (20% → 12% → 0%)."

  reasoning: string               // Why this is happening (root causes)
                                  // Example: "Jennifer was 15 minutes late that day. Sarah's friend Lisa posted on Facebook about 'trying that new place downtown.'"

  actionPlan: string[]            // 3-5 specific action steps
                                  // Example:
                                  // [
                                  //   "Call Sarah today (not text - this needs the personal touch)",
                                  //   "Jennifer should make the call (relationship repair)",
                                  //   "Offer: '20% off next visit' + specific apology",
                                  //   "Book her THIS WEEK while she still remembers you"
                                  // ]

  script?: string                 // Optional: Exact conversation script
                                  // Example: "Hi Sarah, it's Jennifer from Bashful Beauty. I realized I kept you waiting last time and I felt terrible about it. I'd love to make it right - can I book you this week with 20% off? I miss seeing you!"

  expectedRecovery: string        // Expected outcome with timeframe and probability
                                  // Example: "60% chance of recovery if contacted today, 40% if contacted this week, 15% after that."

  // Metadata
  tags: string[]                  // Hashtags for filtering
                                  // Examples: ["#CUSTOMERRETENTION", "#HIGHVALUE", "#URGENT"]

  // Related entities (optional)
  relatedCustomer?: {
    id: string,
    name: string,
    email: string,
    avatar?: string,              // Initials (e.g., "SC")
    lifetimeValue?: number        // In dollars
  }

  relatedEmployee?: {
    id: string,
    name: string,
    role: string,
    avatar?: string               // Initials (e.g., "MJ")
  }

  // Completion tracking
  completed: boolean              // Has owner marked this complete?
  completedAt: string | null      // ISO timestamp when completed, or null
}
```

**Example Complete Discovery**:
```json
{
  "id": "discovery_20250921_001",
  "title": "Sarah Chen hasn't been in for 47 days",
  "category": "Customer Defection Alert",
  "priority": "critical",
  "impact": 2280,
  "confidence": 96,
  "daysAgo": 47,
  "description": "Her 3-year pattern was every 18-22 days spending $95 on Brazilian + brow service. Last 3 visits: declining tips to Jennifer (20% → 12% → 0%).",
  "reasoning": "Jennifer was 15 minutes late that day. Sarah's friend Lisa posted on Facebook about 'trying that new place downtown.'",
  "actionPlan": [
    "Call Sarah today (not text - this needs the personal touch)",
    "Jennifer should make the call (relationship repair)",
    "Offer: 'I noticed I was running behind last time - let me make it right with 20% off your next visit'",
    "Book her THIS WEEK while she still remembers you"
  ],
  "script": "Hi Sarah, it's Jennifer from Bashful Beauty. I realized I kept you waiting last time and I felt terrible about it. I'd love to make it right - can I book you this week with 20% off? I miss seeing you!",
  "expectedRecovery": "60% chance of recovery if contacted today, 40% if contacted this week, 15% after that.",
  "tags": ["#CUSTOMERRETENTION", "#HIGHVALUE"],
  "relatedCustomer": {
    "id": "1",
    "name": "Sarah Chen",
    "email": "sarah.chen@email.com",
    "avatar": "SC",
    "lifetimeValue": 2280
  },
  "completed": false,
  "completedAt": null,
  "createdAt": "2025-09-21T03:15:00Z"
}
```

---

## API Endpoints Summary

### For Greg to Implement

| Endpoint | Method | Purpose | Data Source |
|----------|--------|---------|-------------|
| `/discoveries` | GET | Get all discoveries for business | Firestore: `discoveries/{businessId}/insights` |
| `/discoveries/:id` | GET | Get single discovery | Firestore: `discoveries/{businessId}/insights/{discoveryId}` |
| `/discoveries/:id/complete` | POST | Mark discovery as completed | Update Firestore document |
| `/stats` | GET | Get business stats (ROI, counts) | Firestore: `businesses/{businessId}/stats` |
| `/chat` | POST | Send chat message, get AI response | InvestigatorAgent → MCPs → LLM |
| `/health` | GET | API health check | Return `{ status: 'ok' }` |

---

## MCP Usage Frequency

**Most frequently used MCPs** (called for almost every discovery):
1. `square_data.mcp` - Get transactions, customers, employees (every discovery)
2. `pattern_detector.mcp` - Find statistical patterns (every discovery)
3. `confidence_scorer.mcp` - Calculate confidence (every discovery)
4. `impact_predictor.mcp` - Estimate ROI (every discovery)
5. `output_formatter.mcp` - Format for frontend (every discovery)

**Frequently used MCPs** (called for many discoveries):
6. `segment_analyzer.mcp` - Customer segmentation (60% of discoveries)
7. `employee_analyzer.mcp` - Employee performance (40% of discoveries)
8. `customer_forensics.mcp` - Individual customer analysis (50% of discoveries)
9. `script_generator.mcp` - Generate scripts (70% of discoveries)
10. `customer_psychology.mcp` - Explain behavior (60% of discoveries)

**Occasionally used MCPs** (called for specific discovery types):
11. `time_surgeon.mcp` - Time-based patterns (20% of discoveries)
12. `financial_surgeon.mcp` - Revenue analysis (30% of discoveries)
13. `review_monitor.mcp` - Review insights (15% of discoveries)
14. `customer_migration.mcp` - Find lost customers at competitors (10% of discoveries)
15. `spa_wisdom.mcp` - Industry best practices (40% of spa discoveries)

**Rarely used MCPs** (called only for special cases):
16. `crisis_manager.mcp` - Urgent issues (5% of discoveries)
17. `social_listening.mcp` - Social media insights (10% of discoveries)
18. `competitor_discovery.mcp` - New competitors (5% of discoveries)

---

## Testing Checklist for Greg

When implementing MCPs and API, verify:

### Data Accuracy
- [ ] Discovery `impact` values match MCP calculations
- [ ] Discovery `confidence` scores are realistic (80-99 range)
- [ ] `relatedCustomer` data matches Square customer data
- [ ] `relatedEmployee` data matches Square employee data

### Schema Compliance
- [ ] All required fields present in discovery object
- [ ] `id` format is correct: `discovery_YYYYMMDD_NNN`
- [ ] `priority` is one of: `critical`, `medium`, `low`
- [ ] `category` matches one of the defined categories
- [ ] `actionPlan` is array of 3-5 strings
- [ ] `tags` are formatted with `#` prefix
- [ ] `impact` is a number (not string, not formatted)
- [ ] `createdAt` is ISO timestamp string

### API Testing
- [ ] `GET /discoveries` returns array of discoveries
- [ ] `GET /discoveries/:id` returns single discovery
- [ ] `POST /discoveries/:id/complete` updates Firestore
- [ ] `GET /stats` returns correct ROI calculations
- [ ] `POST /chat` returns coherent AI response
- [ ] All endpoints verify Firebase authentication
- [ ] CORS headers allow Keeper_UI origin

### Frontend Integration
- [ ] Dashboard hero insight displays correctly
- [ ] Discovery cards render without errors
- [ ] Filters work (All, Urgent, Revenue, etc.)
- [ ] Mark complete flow updates UI
- [ ] Chat messages appear correctly
- [ ] ROI counter animates with real data

---

## Example Data Flow (End-to-End)

### Scenario: "Sarah Chen Defection Discovery"

**1. Nightly Discovery Run (3 AM)**
```
Cloud Scheduler → triggers → services/discovery_service.py
  ↓
discovery_service loads agents/analyst_agent.py
  ↓
Analyst enters discovery_mode()
  ↓
Calls square_data.mcp.get_customers()
  → Returns: Sarah Chen, last_visit: 47 days ago, LTV: $2,280
  ↓
Calls customer_forensics.mcp.predict_churn_risk('sarah_chen')
  → Returns: { churn_probability: 0.76, signals: ['47 days', 'declining tips'] }
  ↓
Calls customer_forensics.mcp.get_customer_journey('sarah_chen')
  → Returns: [visit1, visit2, visit3] with declining tips: 20% → 12% → 0%
  ↓
Calls customer_migration.mcp.find_lost_customers_at_competitors()
  → Returns: Sarah reviewed "Glow Spa" 12 days ago
  ↓
Calls customer_psychology.mcp.explain_customer_behavior()
  → Returns: "Declining tips = passive-aggressive feedback"
  ↓
Analyst calls Vertex AI Gemini Pro
  → Input: All MCP findings
  → Output: Insight with title, description, reasoning
  ↓
Calls confidence_scorer.mcp.calculate_overall_confidence()
  → Returns: 96% confidence
  ↓
Calls impact_predictor.mcp.predict_revenue_impact()
  → Returns: $2,280 at stake, 60% recovery chance
  ↓
Analyst hands off to Advisor Agent
  ↓
Advisor enters action_planning_mode(insight)
  ↓
Calls script_generator.mcp.generate_customer_outreach_script()
  → Returns: "Hi Sarah, it's Jennifer from Bashful Beauty..."
  ↓
Advisor calls Claude Opus 4
  → Input: Insight + script + business context
  → Output: Complete action plan (4 steps)
  ↓
Calls output_formatter.mcp.format_discovery()
  → Returns: Complete discovery JSON (matches frontend schema)
  ↓
discovery_service stores in Firestore
  → Collection: discoveries/bashful_beauty_123/insights/discovery_20250921_001
```

**2. User Logs In (8 AM)**
```
User opens Keeper_UI → pages/dashboard.vue
  ↓
Frontend calls GET /discoveries?businessId=bashful_beauty_123
  ↓
API (FastAPI on Cloud Run) verifies Firebase token
  ↓
API queries Firestore: discoveries/bashful_beauty_123/insights
  ↓
Returns JSON array of discoveries
  ↓
Frontend receives discoveries, renders on page
  ↓
Hero insight card shows: "Sarah Chen hasn't been in for 47 days"
  ↓
User clicks "View Details" → navigateTo('/discoveries/discovery_20250921_001')
  ↓
Discovery detail page shows full actionPlan, script, expectedRecovery
  ↓
User clicks "Mark Complete" → POST /discoveries/discovery_20250921_001/complete
  ↓
API updates Firestore: { completed: true, completedAt: '2025-09-21T08:15:00Z' }
  ↓
Frontend updates UI (discovery moves to "Completed" filter)
  ↓
ROI counter updates (adds $2,280 to completedROI)
```

**3. User Asks Follow-up Question (8:30 AM)**
```
User clicks "Ask about this discovery" → opens chat
  ↓
User types: "How do you know Sarah went to Glow Spa?"
  ↓
Frontend calls POST /chat
  → { businessId, threadId, message }
  ↓
API invokes agents/investigator_agent.py
  ↓
Investigator enters question_answering_mode()
  ↓
Calls customer_migration.mcp.find_lost_customers_at_competitors()
  → Returns: Sarah left Google review for "Glow Spa" on 2025-09-09
  ↓
Calls social_listening.mcp.track_brand_mentions()
  → Returns: Sarah's friend Lisa mentioned "new place downtown" on Facebook
  ↓
Investigator calls Gemini Pro
  → Input: Question + MCP data + conversation history
  → Output: "We found Sarah left a 4-star Google review for Glow Spa on Sept 9th. Additionally, her friend Lisa (who we identified from past Instagram tags) posted on Facebook about trying a new spa downtown around the same time. Source: customer_migration.mcp + social_listening.mcp"
  ↓
API returns response to frontend
  ↓
Frontend displays in chat interface
```

---

## Final Notes for Greg

1. **Start simple**: Implement `square_data.mcp` first, get real data flowing
2. **Test incrementally**: After each MCP, verify output matches expected schema
3. **Use Bashful Beauty data**: 8 years of real spa data for testing
4. **Match the schema exactly**: Frontend expects specific field names and types
5. **Focus on quality**: Better to have 5 great discoveries than 20 mediocre ones
6. **Iterate with feedback**: Show Ray/Steven discoveries, refine based on their input

**Questions while building?**
- Check `KEEPER_ARCHITECTURE.md` for detailed MCP specs
- Check `QUICK_START_GUIDE.md` for implementation steps
- Check `REPOSITORY_STRUCTURE.md` for code organization
- Check this document for frontend integration

Good luck! 🚀
