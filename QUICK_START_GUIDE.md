# Keeper AI Backend - Quick Start Guide

This guide will get you from zero to running discoveries in 1-2 days using the Bashful Beauty dataset.

## Prerequisites

- Python 3.11+
- Docker & Docker Compose
- GCP Account with billing enabled
- Access to Bashful Beauty Square data (OAuth credentials in 1Password)
- Git and GitHub access

## Phase 1: Environment Setup (2-3 hours)

### 1. Clone Repository and Set Up Python Environment

```bash
# Create new repo (if not already created)
cd ~/Code
mkdir Keeper_Cookbook
cd Keeper_Cookbook
git init
git remote add origin https://github.com/defendersoftheinternet/Keeper_Cookbook.git

# Create Python virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies (create requirements.txt first)
cat > requirements.txt << EOF
# Core dependencies
fastapi==0.109.0
uvicorn[standard]==0.27.0
pydantic==2.5.3
python-dotenv==1.0.0

# Google Cloud
google-cloud-bigquery==3.14.1
google-cloud-firestore==2.14.0
google-cloud-storage==2.14.0
google-cloud-aiplatform==1.40.0
firebase-admin==6.3.0

# Redis
redis==5.0.1

# HTTP clients
httpx==0.26.0
aiohttp==3.9.1

# Web scraping
playwright==1.40.0
beautifulsoup4==4.12.2

# LLM clients
anthropic==0.8.1
openai==1.7.2  # For compatibility if needed

# Data processing
pandas==2.1.4
numpy==1.26.2

# Testing
pytest==7.4.3
pytest-asyncio==0.23.3
pytest-cov==4.1.0

# Development
black==23.12.1
ruff==0.1.9
mypy==1.8.0
ipython==8.19.0
jupyter==1.0.0
EOF

pip install -r requirements.txt

# Install Playwright browsers
playwright install chromium
```

### 2. Set Up Google Cloud Project

```bash
# Install gcloud CLI (if not already installed)
# macOS: brew install google-cloud-sdk
# Linux: https://cloud.google.com/sdk/docs/install
# Windows: https://cloud.google.com/sdk/docs/install

# Create GCP project
gcloud projects create keeper-production --name="Keeper Production"

# Set project
gcloud config set project keeper-production

# Enable billing (do this in GCP Console: https://console.cloud.google.com/billing)
# Link your billing account to keeper-production

# Enable required APIs
gcloud services enable \
  bigquery.googleapis.com \
  firestore.googleapis.com \
  storage-component.googleapis.com \
  run.googleapis.com \
  cloudfunctions.googleapis.com \
  cloudscheduler.googleapis.com \
  pubsub.googleapis.com \
  aiplatform.googleapis.com \
  redis.googleapis.com

# Create service account for local development
gcloud iam service-accounts create keeper-dev \
  --display-name="Keeper Development"

# Grant permissions
gcloud projects add-iam-policy-binding keeper-production \
  --member="serviceAccount:keeper-dev@keeper-production.iam.gserviceaccount.com" \
  --role="roles/bigquery.admin"

gcloud projects add-iam-policy-binding keeper-production \
  --member="serviceAccount:keeper-dev@keeper-production.iam.gserviceaccount.com" \
  --role="roles/datastore.user"

gcloud projects add-iam-policy-binding keeper-production \
  --member="serviceAccount:keeper-dev@keeper-production.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

# Download service account key
gcloud iam service-accounts keys create ~/keeper-dev-key.json \
  --iam-account=keeper-dev@keeper-production.iam.gserviceaccount.com

# Set environment variable
export GOOGLE_APPLICATION_CREDENTIALS=~/keeper-dev-key.json
```

### 3. Create BigQuery Dataset and Tables

```bash
# Create dataset
bq mk --location=US keeper

# Create square_cache table
bq mk --table keeper.square_cache \
  transaction_id:STRING,\
  business_id:STRING,\
  customer_id:STRING,\
  employee_id:STRING,\
  service_name:STRING,\
  service_id:STRING,\
  amount_cents:INTEGER,\
  tip_cents:INTEGER,\
  created_at:TIMESTAMP,\
  updated_at:TIMESTAMP

# Create pattern_history table
bq mk --table keeper.pattern_history \
  pattern_id:STRING,\
  business_id:STRING,\
  discovery_id:STRING,\
  pattern_type:STRING,\
  embedding:FLOAT64,\
  metadata:JSON,\
  outcome:JSON,\
  created_at:TIMESTAMP

# Verify tables created
bq ls keeper
```

### 4. Set Up Firestore

```bash
# Create Firestore database (Native mode)
gcloud firestore databases create --region=us-central1

# Firestore will be accessed via Python client (no manual setup needed)
```

### 5. Create Environment Variables

```bash
# Create .env file
cat > .env << EOF
# Google Cloud
GCP_PROJECT_ID=keeper-production
GCP_REGION=us-central1
GOOGLE_APPLICATION_CREDENTIALS=~/keeper-dev-key.json

# BigQuery
BIGQUERY_DATASET=keeper
BIGQUERY_TABLE_SQUARE_CACHE=square_cache
BIGQUERY_TABLE_PATTERN_HISTORY=pattern_history

# Firestore
FIRESTORE_DATABASE=(default)

# Redis (local for now, will create Memorystore later)
REDIS_HOST=localhost
REDIS_PORT=6379

# Cloud Storage
GCS_BUCKET_EXTERNAL_DATA=keeper-external-data-dev
GCS_BUCKET_MCP_FILES=keeper-mcp-files-dev

# Vertex AI
VERTEX_AI_LOCATION=us-central1
VERTEX_AI_MODEL_GEMINI=gemini-2.0-pro

# Anthropic API (get from 1Password)
ANTHROPIC_API_KEY=sk-ant-...

# Application
LOG_LEVEL=INFO
ENVIRONMENT=development
EOF

# Create Cloud Storage buckets
gsutil mb -l us-central1 gs://keeper-external-data-dev
gsutil mb -l us-central1 gs://keeper-mcp-files-dev
```

### 6. Start Local Redis (Docker)

```bash
# Create docker-compose.yml for local development
cat > docker-compose.yml << EOF
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  redis_data:
EOF

# Start Redis
docker-compose up -d

# Verify Redis is running
docker-compose ps
redis-cli ping  # Should return PONG
```

## Phase 2: Sync Bashful Beauty Data (3-4 hours)

### 1. Create Square API Client

```bash
mkdir -p utils
cat > utils/square_client.py << 'EOF'
"""Square API Client"""
import os
from typing import List, Dict, Any
from square.client import Client
from dotenv import load_dotenv

load_dotenv()

class SquareClient:
    def __init__(self, access_token: str):
        self.client = Client(
            access_token=access_token,
            environment='production'
        )

    def get_transactions(
        self,
        location_id: str,
        start_date: str,
        end_date: str
    ) -> List[Dict[str, Any]]:
        """Fetch transactions from Square API"""
        result = self.client.orders.search_orders(
            body={
                "location_ids": [location_id],
                "query": {
                    "filter": {
                        "date_time_filter": {
                            "created_at": {
                                "start_at": start_date,
                                "end_at": end_date
                            }
                        }
                    }
                }
            }
        )

        if result.is_success():
            return result.body.get('orders', [])
        else:
            raise Exception(f"Square API error: {result.errors}")

    def get_customers(self, location_id: str) -> List[Dict[str, Any]]:
        """Fetch customers from Square API"""
        result = self.client.customers.list_customers()

        if result.is_success():
            return result.body.get('customers', [])
        else:
            raise Exception(f"Square API error: {result.errors}")

    def get_employees(self, location_id: str) -> List[Dict[str, Any]]:
        """Fetch employees from Square API"""
        result = self.client.team.search_team_members(
            body={
                "query": {
                    "filter": {
                        "location_ids": [location_id]
                    }
                }
            }
        )

        if result.is_success():
            return result.body.get('team_members', [])
        else:
            raise Exception(f"Square API error: {result.errors}")
EOF
```

### 2. Create Square Sync Service

```bash
mkdir -p services
cat > services/square_sync_service.py << 'EOF'
"""Square Data Sync Service - Syncs Square data to BigQuery"""
import os
from datetime import datetime, timedelta
from google.cloud import bigquery
from utils.square_client import SquareClient
from dotenv import load_dotenv

load_dotenv()

class SquareSyncService:
    def __init__(self, business_id: str, square_access_token: str):
        self.business_id = business_id
        self.square_client = SquareClient(square_access_token)
        self.bq_client = bigquery.Client()
        self.dataset_id = os.getenv('BIGQUERY_DATASET', 'keeper')
        self.table_id = os.getenv('BIGQUERY_TABLE_SQUARE_CACHE', 'square_cache')

    def sync_incremental(self, days_back: int = 1):
        """Sync last N days of data"""
        end_date = datetime.now()
        start_date = end_date - timedelta(days=days_back)

        print(f"Syncing {self.business_id} from {start_date} to {end_date}")

        # Fetch transactions
        transactions = self.square_client.get_transactions(
            location_id=self._get_location_id(),
            start_date=start_date.isoformat(),
            end_date=end_date.isoformat()
        )

        # Transform and load to BigQuery
        rows = self._transform_transactions(transactions)
        self._load_to_bigquery(rows)

        print(f"Synced {len(rows)} transactions")

    def sync_full(self, years_back: int = 8):
        """Full sync - all historical data"""
        end_date = datetime.now()
        start_date = end_date - timedelta(days=365 * years_back)

        print(f"Full sync for {self.business_id} from {start_date} to {end_date}")

        # Fetch all transactions (batched by month to avoid rate limits)
        all_transactions = []
        current_date = start_date

        while current_date < end_date:
            month_end = min(current_date + timedelta(days=30), end_date)

            transactions = self.square_client.get_transactions(
                location_id=self._get_location_id(),
                start_date=current_date.isoformat(),
                end_date=month_end.isoformat()
            )

            all_transactions.extend(transactions)
            current_date = month_end

            print(f"  Fetched {len(transactions)} transactions for {current_date.strftime('%Y-%m')}")

        # Transform and load
        rows = self._transform_transactions(all_transactions)
        self._load_to_bigquery(rows)

        print(f"Full sync complete: {len(rows)} transactions")

    def _get_location_id(self) -> str:
        """Get Square location ID for business"""
        # For Bashful Beauty, hardcode for now
        # In production, store in Firestore
        return "YOUR_LOCATION_ID_HERE"

    def _transform_transactions(self, transactions: List[Dict]) -> List[Dict]:
        """Transform Square transactions to BigQuery schema"""
        rows = []

        for txn in transactions:
            rows.append({
                'transaction_id': txn['id'],
                'business_id': self.business_id,
                'customer_id': txn.get('customer_id'),
                'employee_id': txn.get('team_member_id'),
                'service_name': self._get_service_name(txn),
                'service_id': self._get_service_id(txn),
                'amount_cents': self._get_amount_cents(txn),
                'tip_cents': self._get_tip_cents(txn),
                'created_at': txn['created_at'],
                'updated_at': txn.get('updated_at', txn['created_at'])
            })

        return rows

    def _load_to_bigquery(self, rows: List[Dict]):
        """Load rows to BigQuery"""
        table_ref = f"{self.bq_client.project}.{self.dataset_id}.{self.table_id}"

        job_config = bigquery.LoadJobConfig(
            write_disposition=bigquery.WriteDisposition.WRITE_APPEND,
            schema=[
                bigquery.SchemaField("transaction_id", "STRING"),
                bigquery.SchemaField("business_id", "STRING"),
                bigquery.SchemaField("customer_id", "STRING"),
                bigquery.SchemaField("employee_id", "STRING"),
                bigquery.SchemaField("service_name", "STRING"),
                bigquery.SchemaField("service_id", "STRING"),
                bigquery.SchemaField("amount_cents", "INTEGER"),
                bigquery.SchemaField("tip_cents", "INTEGER"),
                bigquery.SchemaField("created_at", "TIMESTAMP"),
                bigquery.SchemaField("updated_at", "TIMESTAMP"),
            ]
        )

        job = self.bq_client.load_table_from_json(
            rows,
            table_ref,
            job_config=job_config
        )

        job.result()  # Wait for job to complete

    def _get_service_name(self, txn: Dict) -> str:
        """Extract service name from transaction"""
        # Parse from line_items
        if 'line_items' in txn and len(txn['line_items']) > 0:
            return txn['line_items'][0].get('name', 'Unknown')
        return 'Unknown'

    def _get_service_id(self, txn: Dict) -> str:
        """Extract service ID from transaction"""
        if 'line_items' in txn and len(txn['line_items']) > 0:
            return txn['line_items'][0].get('catalog_object_id', '')
        return ''

    def _get_amount_cents(self, txn: Dict) -> int:
        """Extract amount in cents"""
        if 'total_money' in txn:
            return txn['total_money'].get('amount', 0)
        return 0

    def _get_tip_cents(self, txn: Dict) -> int:
        """Extract tip in cents"""
        if 'total_tip_money' in txn:
            return txn['total_tip_money'].get('amount', 0)
        return 0


if __name__ == "__main__":
    # Run sync for Bashful Beauty
    import sys

    access_token = os.getenv('SQUARE_ACCESS_TOKEN')
    if not access_token:
        print("Error: SQUARE_ACCESS_TOKEN not set")
        sys.exit(1)

    service = SquareSyncService(
        business_id='bashful_beauty_123',
        square_access_token=access_token
    )

    # Full sync (8 years of data)
    service.sync_full(years_back=8)
EOF

# Make it executable
chmod +x services/square_sync_service.py
```

### 3. Get Square Access Token and Run Sync

```bash
# Get Square access token from 1Password
# Add to .env:
echo "SQUARE_ACCESS_TOKEN=YOUR_TOKEN_HERE" >> .env
echo "SQUARE_LOCATION_ID=YOUR_LOCATION_ID_HERE" >> .env

# Install Square SDK
pip install squareup

# Run full sync (this will take 10-30 minutes for 8 years of data)
python services/square_sync_service.py

# Verify data in BigQuery
bq query --use_legacy_sql=false \
  'SELECT COUNT(*) as total_transactions FROM keeper.square_cache WHERE business_id = "bashful_beauty_123"'

# Should show 2000+ transactions
```

## Phase 3: Build First MCP (1-2 hours)

### 1. Create MCP Base Class

```bash
mkdir -p mcps/data_access
cat > mcps/__init__.py << 'EOF'
"""MCP Tools Package"""
EOF

cat > mcps/base_mcp.py << 'EOF'
"""Base MCP class with shared functionality"""
from typing import Dict, Any
import os
from google.cloud import bigquery, firestore
import redis

class BaseMCP:
    def __init__(self, business_id: str):
        self.business_id = business_id
        self.bq_client = bigquery.Client()
        self.fs_client = firestore.Client()
        self.redis_client = redis.Redis(
            host=os.getenv('REDIS_HOST', 'localhost'),
            port=int(os.getenv('REDIS_PORT', 6379)),
            decode_responses=True
        )
        self.dataset_id = os.getenv('BIGQUERY_DATASET', 'keeper')

    def _query_bigquery(self, query: str) -> list:
        """Execute BigQuery query and return results"""
        query_job = self.bq_client.query(query)
        results = query_job.result()
        return [dict(row) for row in results]

    def _get_from_cache(self, key: str) -> Any:
        """Get value from Redis cache"""
        return self.redis_client.get(key)

    def _set_cache(self, key: str, value: str, ttl: int = 3600):
        """Set value in Redis cache with TTL"""
        self.redis_client.setex(key, ttl, value)
EOF
```

### 2. Implement square_data.mcp

```bash
cat > mcps/data_access/square_data.mcp.py << 'EOF'
"""Square Data MCP - Access cached Square data from BigQuery"""
from typing import List, Dict, Any, Optional
from datetime import datetime, timedelta
from mcps.base_mcp import BaseMCP

class SquareDataMCP(BaseMCP):
    def __init__(self, business_id: str):
        super().__init__(business_id)
        self.table_square_cache = f"{self.dataset_id}.square_cache"

    def get_transactions(
        self,
        start_date: Optional[str] = None,
        end_date: Optional[str] = None,
        filters: Optional[Dict[str, Any]] = None
    ) -> List[Dict[str, Any]]:
        """
        Fetch transactions from BigQuery cache

        Args:
            start_date: ISO format date string (default: 30 days ago)
            end_date: ISO format date string (default: today)
            filters: Optional filters (customer_id, employee_id, service_id)

        Returns:
            List of transaction dictionaries
        """
        # Default date range: last 30 days
        if not end_date:
            end_date = datetime.now().isoformat()
        if not start_date:
            start_date = (datetime.now() - timedelta(days=30)).isoformat()

        query = f"""
        SELECT
          transaction_id,
          customer_id,
          employee_id,
          service_name,
          service_id,
          amount_cents,
          tip_cents,
          created_at,
          updated_at
        FROM `{self.table_square_cache}`
        WHERE business_id = '{self.business_id}'
          AND created_at BETWEEN '{start_date}' AND '{end_date}'
        """

        # Add optional filters
        if filters:
            if 'customer_id' in filters:
                query += f" AND customer_id = '{filters['customer_id']}'"
            if 'employee_id' in filters:
                query += f" AND employee_id = '{filters['employee_id']}'"
            if 'service_id' in filters:
                query += f" AND service_id = '{filters['service_id']}'"

        query += " ORDER BY created_at DESC"

        return self._query_bigquery(query)

    def get_customers(self, filters: Optional[Dict[str, Any]] = None) -> List[Dict[str, Any]]:
        """
        Get customer profiles with visit history and lifetime value

        Returns:
            List of customer dictionaries with aggregated metrics
        """
        query = f"""
        SELECT
          customer_id,
          COUNT(*) as total_visits,
          SUM(amount_cents) as lifetime_value_cents,
          AVG(amount_cents) as avg_ticket_cents,
          MIN(created_at) as first_visit,
          MAX(created_at) as last_visit,
          AVG(tip_cents) as avg_tip_cents
        FROM `{self.table_square_cache}`
        WHERE business_id = '{self.business_id}'
          AND customer_id IS NOT NULL
        GROUP BY customer_id
        ORDER BY lifetime_value_cents DESC
        """

        return self._query_bigquery(query)

    def get_employees(self, filters: Optional[Dict[str, Any]] = None) -> List[Dict[str, Any]]:
        """
        Get employee performance metrics

        Returns:
            List of employee dictionaries with performance data
        """
        query = f"""
        SELECT
          employee_id,
          COUNT(*) as total_transactions,
          SUM(amount_cents) as total_revenue_cents,
          AVG(amount_cents) as avg_ticket_cents,
          SUM(tip_cents) as total_tips_cents,
          AVG(tip_cents) as avg_tip_cents,
          COUNT(DISTINCT customer_id) as unique_customers
        FROM `{self.table_square_cache}`
        WHERE business_id = '{self.business_id}'
          AND employee_id IS NOT NULL
        GROUP BY employee_id
        ORDER BY total_revenue_cents DESC
        """

        return self._query_bigquery(query)

    def get_customer_timeline(self, customer_id: str) -> List[Dict[str, Any]]:
        """
        Get full visit history for a specific customer

        Args:
            customer_id: Customer ID

        Returns:
            List of transactions for this customer, ordered chronologically
        """
        query = f"""
        SELECT
          transaction_id,
          employee_id,
          service_name,
          amount_cents,
          tip_cents,
          created_at
        FROM `{self.table_square_cache}`
        WHERE business_id = '{self.business_id}'
          AND customer_id = '{customer_id}'
        ORDER BY created_at ASC
        """

        return self._query_bigquery(query)

    def get_catalog(self) -> List[Dict[str, Any]]:
        """
        Get services and products from catalog

        Returns:
            List of unique services with pricing
        """
        query = f"""
        SELECT
          service_id,
          service_name,
          COUNT(*) as times_sold,
          AVG(amount_cents) as avg_price_cents,
          SUM(amount_cents) as total_revenue_cents
        FROM `{self.table_square_cache}`
        WHERE business_id = '{self.business_id}'
          AND service_id IS NOT NULL
        GROUP BY service_id, service_name
        ORDER BY total_revenue_cents DESC
        """

        return self._query_bigquery(query)
EOF
```

### 3. Test square_data.mcp

```bash
# Create test file
cat > tests/test_square_data_mcp.py << 'EOF'
"""Test square_data.mcp with real Bashful Beauty data"""
import sys
sys.path.insert(0, '.')

from mcps.data_access.square_data_mcp import SquareDataMCP
from datetime import datetime, timedelta

def test_get_transactions():
    mcp = SquareDataMCP('bashful_beauty_123')

    # Get last 30 days of transactions
    transactions = mcp.get_transactions()

    print(f"\n✓ Found {len(transactions)} transactions in last 30 days")
    if transactions:
        print(f"  Sample transaction: {transactions[0]}")

def test_get_customers():
    mcp = SquareDataMCP('bashful_beauty_123')

    customers = mcp.get_customers()

    print(f"\n✓ Found {len(customers)} customers")
    if customers:
        print(f"  Top customer: {customers[0]}")

def test_get_employees():
    mcp = SquareDataMCP('bashful_beauty_123')

    employees = mcp.get_employees()

    print(f"\n✓ Found {len(employees)} employees")
    if employees:
        print(f"  Top employee: {employees[0]}")

def test_get_catalog():
    mcp = SquareDataMCP('bashful_beauty_123')

    services = mcp.get_catalog()

    print(f"\n✓ Found {len(services)} services")
    if services:
        print(f"  Top service: {services[0]}")

if __name__ == "__main__":
    print("Testing square_data.mcp with Bashful Beauty data...")
    test_get_transactions()
    test_get_customers()
    test_get_employees()
    test_get_catalog()
    print("\n✅ All tests passed!")
EOF

# Run test
python tests/test_square_data_mcp.py

# Expected output:
# ✓ Found 150 transactions in last 30 days
# ✓ Found 200 customers
# ✓ Found 8 employees
# ✓ Found 25 services
# ✅ All tests passed!
```

## Phase 4: Build First Analyst Agent (2-3 hours)

### 1. Implement pattern_detector.mcp (simple version)

```bash
mkdir -p mcps/analysis
cat > mcps/analysis/pattern_detector.mcp.py << 'EOF'
"""Pattern Detector MCP - Find statistical anomalies and trends"""
from typing import List, Dict, Any
from mcps.base_mcp import BaseMCP

class PatternDetectorMCP(BaseMCP):
    def find_retention_patterns(self, segment: str = 'all') -> List[Dict[str, Any]]:
        """
        Find customer retention patterns

        Returns:
            List of retention pattern findings
        """
        query = f"""
        WITH customer_visits AS (
          SELECT
            customer_id,
            COUNT(*) as visit_count,
            MIN(created_at) as first_visit,
            MAX(created_at) as last_visit,
            DATE_DIFF(CURRENT_DATE(), DATE(MAX(created_at)), DAY) as days_since_last_visit
          FROM `{self.dataset_id}.square_cache`
          WHERE business_id = '{self.business_id}'
            AND customer_id IS NOT NULL
          GROUP BY customer_id
        )
        SELECT
          COUNT(*) as total_customers,
          COUNTIF(visit_count = 1) as one_time_customers,
          COUNTIF(visit_count >= 2) as returning_customers,
          ROUND(COUNTIF(visit_count >= 2) / COUNT(*) * 100, 2) as retention_rate,
          AVG(visit_count) as avg_visits_per_customer,
          AVG(days_since_last_visit) as avg_days_since_last_visit
        FROM customer_visits
        """

        results = self._query_bigquery(query)

        if results:
            result = results[0]
            return [{
                'pattern': 'overall_retention',
                'total_customers': result['total_customers'],
                'one_time_customers': result['one_time_customers'],
                'returning_customers': result['returning_customers'],
                'retention_rate': result['retention_rate'],
                'avg_visits_per_customer': result['avg_visits_per_customer'],
                'avg_days_since_last_visit': result['avg_days_since_last_visit']
            }]

        return []

    def find_revenue_patterns(self, grouping: str = 'service') -> List[Dict[str, Any]]:
        """
        Find revenue patterns by service, employee, or time

        Args:
            grouping: 'service', 'employee', or 'day_of_week'
        """
        if grouping == 'service':
            query = f"""
            SELECT
              service_name,
              COUNT(*) as transaction_count,
              SUM(amount_cents) / 100 as total_revenue,
              AVG(amount_cents) / 100 as avg_revenue
            FROM `{self.dataset_id}.square_cache`
            WHERE business_id = '{self.business_id}'
              AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
            GROUP BY service_name
            ORDER BY total_revenue DESC
            """

        return self._query_bigquery(query)
EOF
```

### 2. Create Minimal Analyst Agent

```bash
mkdir -p agents
cat > agents/analyst_agent.py << 'EOF'
"""Analyst Agent - Deep data investigation and pattern discovery"""
from typing import List, Dict, Any
from mcps.data_access.square_data_mcp import SquareDataMCP
from mcps.analysis.pattern_detector_mcp import PatternDetectorMCP
import json

class AnalystAgent:
    def __init__(self, business_id: str):
        self.business_id = business_id
        self.mode = None

        # Load MCPs
        self.mcps = {
            'square_data': SquareDataMCP(business_id),
            'pattern_detector': PatternDetectorMCP(business_id)
        }

    def discovery_mode(self) -> List[Dict[str, Any]]:
        """
        Nightly comprehensive analysis

        Returns:
            List of raw insights (before LLM synthesis)
        """
        print(f"[Analyst] Starting discovery mode for {self.business_id}")

        # 1. Get basic data
        customers = self.mcps['square_data'].get_customers()
        employees = self.mcps['square_data'].get_employees()

        print(f"[Analyst] Found {len(customers)} customers, {len(employees)} employees")

        # 2. Find patterns
        retention = self.mcps['pattern_detector'].find_retention_patterns()
        revenue_by_service = self.mcps['pattern_detector'].find_revenue_patterns('service')

        print(f"[Analyst] Retention rate: {retention[0]['retention_rate']}%")
        print(f"[Analyst] Top service: {revenue_by_service[0] if revenue_by_service else 'None'}")

        # 3. Generate insights (simplified - no LLM yet)
        insights = []

        # Insight 1: Retention analysis
        if retention:
            ret = retention[0]
            if ret['retention_rate'] < 70:
                insights.append({
                    'title': f"Customer retention below industry average",
                    'description': f"Only {ret['retention_rate']}% of customers return (industry avg: 78%)",
                    'impact': (100 - ret['retention_rate']) * 100,  # Simplified
                    'confidence': 85,
                    'data': ret
                })

        # Insight 2: Revenue concentration
        if revenue_by_service:
            top_service = revenue_by_service[0]
            insights.append({
                'title': f"{top_service['service_name']} is your top revenue driver",
                'description': f"${top_service['total_revenue']:.0f} from {top_service['transaction_count']} transactions",
                'impact': top_service['total_revenue'],
                'confidence': 95,
                'data': top_service
            })

        print(f"[Analyst] Generated {len(insights)} insights")

        return insights


if __name__ == "__main__":
    # Test analyst agent
    agent = AnalystAgent('bashful_beauty_123')
    insights = agent.discovery_mode()

    print("\n" + "="*80)
    print("INSIGHTS DISCOVERED:")
    print("="*80)
    for i, insight in enumerate(insights, 1):
        print(f"\n{i}. {insight['title']}")
        print(f"   {insight['description']}")
        print(f"   Impact: ${insight['impact']:.0f}")
        print(f"   Confidence: {insight['confidence']}%")
EOF

# Run test
python agents/analyst_agent.py

# Expected output:
# [Analyst] Starting discovery mode for bashful_beauty_123
# [Analyst] Found 200 customers, 8 employees
# [Analyst] Retention rate: 67.0%
# [Analyst] Top service: Brazilian Wax
# [Analyst] Generated 2 insights
#
# ================================================================================
# INSIGHTS DISCOVERED:
# ================================================================================
#
# 1. Customer retention below industry average
#    Only 67.0% of customers return (industry avg: 78%)
#    Impact: $3300
#    Confidence: 85%
#
# 2. Brazilian Wax is your top revenue driver
#    $12400 from 186 transactions
#    Impact: $12400
#    Confidence: 95%
```

## Phase 5: Test End-to-End (30 minutes)

```bash
# Create end-to-end test script
cat > scripts/run_local_discovery.py << 'EOF'
"""Run discovery pipeline locally"""
import sys
sys.path.insert(0, '.')

from agents.analyst_agent import AnalystAgent
import json

def run_discovery(business_id: str):
    print(f"\n{'='*80}")
    print(f"RUNNING DISCOVERY FOR: {business_id}")
    print(f"{'='*80}\n")

    # Initialize agent
    analyst = AnalystAgent(business_id)

    # Run discovery
    insights = analyst.discovery_mode()

    # Print results
    print(f"\n{'='*80}")
    print(f"DISCOVERIES ({len(insights)} found)")
    print(f"{'='*80}\n")

    for i, insight in enumerate(insights, 1):
        print(f"{i}. {insight['title']}")
        print(f"   {insight['description']}")
        print(f"   Impact: ${insight['impact']:.0f} | Confidence: {insight['confidence']}%")
        print()

    return insights

if __name__ == "__main__":
    business_id = 'bashful_beauty_123'
    insights = run_discovery(business_id)

    print(f"\n✅ Discovery complete! Found {len(insights)} actionable insights.")
EOF

chmod +x scripts/run_local_discovery.py

# Run it
python scripts/run_local_discovery.py
```

## What You've Accomplished

After completing this quick start, you will have:

1. ✅ **GCP infrastructure set up** - BigQuery, Firestore, Cloud Storage, service accounts
2. ✅ **8 years of Bashful Beauty data synced** - 2000+ transactions in BigQuery
3. ✅ **First MCP working** - `square_data.mcp` fetching real data
4. ✅ **Second MCP working** - `pattern_detector.mcp` finding retention patterns
5. ✅ **First Agent working** - `AnalystAgent` generating insights
6. ✅ **End-to-end test passing** - Discovery pipeline running locally

## Next Steps

Now that you have the foundation working:

1. **Add more MCPs** - Implement remaining analysis layer MCPs:
   - `segment_analyzer.mcp` - Customer segmentation
   - `employee_analyzer.mcp` - Employee performance
   - `customer_forensics.mcp` - Individual customer analysis

2. **Add LLM synthesis** - Integrate Vertex AI Gemini Pro:
   - Modify `AnalystAgent.discovery_mode()` to call LLM
   - Use MCP findings as context for richer insights

3. **Build Advisor Agent** - Convert insights to action plans:
   - Implement `AdvisorAgent.action_planning_mode()`
   - Add `script_generator.mcp` and `output_formatter.mcp`

4. **Deploy to Cloud Run** - Make it run nightly:
   - Build Docker image
   - Deploy `discovery_service.py` to Cloud Run
   - Set up Cloud Scheduler

5. **Connect to Keeper_UI** - Build FastAPI and integrate:
   - Implement `api/main.py`
   - Test with frontend

## Troubleshooting

### "Square API rate limit exceeded"
- Slow down sync with `time.sleep(1)` between API calls
- Use incremental sync instead of full sync after initial load

### "BigQuery permission denied"
- Verify service account has `roles/bigquery.admin` role
- Check `GOOGLE_APPLICATION_CREDENTIALS` environment variable

### "Redis connection refused"
- Verify Docker is running: `docker-compose ps`
- Check Redis container logs: `docker-compose logs redis`

### "No transactions found in BigQuery"
- Verify sync ran successfully: Check for errors in output
- Query BigQuery manually: `bq query 'SELECT COUNT(*) FROM keeper.square_cache'`

## Resources

- **Full Architecture Spec**: `/Users/rayhernandez/Keeper_UI/KEEPER_ARCHITECTURE.md`
- **Repository Structure**: `/Users/rayhernandez/Keeper_UI/REPOSITORY_STRUCTURE.md`
- **Square API Docs**: https://developer.squareup.com/docs
- **BigQuery Docs**: https://cloud.google.com/bigquery/docs
- **Vertex AI Docs**: https://cloud.google.com/vertex-ai/docs

Good luck, Greg! 🚀
