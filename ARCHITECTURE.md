# Marketing System Architecture

## System Design Patterns

### Microservices Architecture
The Marketing System follows a microservices architecture with domain-driven design principles:

- **Campaign Service**: Manages campaign lifecycle and orchestration
- **Segmentation Service**: Handles audience segmentation and targeting
- **Content Service**: Manages content assets and personalization
- **Analytics Service**: Processes events and generates insights
- **Automation Service**: Executes workflow rules and triggers

### Event-Driven Architecture
All state changes are published as events to Kafka topics:

```
marketing.campaign.created
marketing.campaign.launched
marketing.campaign.completed
marketing.segment.refreshed
marketing.email.sent
marketing.email.opened
marketing.email.clicked
```

### CQRS Pattern
Command Query Responsibility Segregation for read-heavy operations:
- **Write Model**: PostgreSQL for transactional operations
- **Read Model**: ClickHouse for analytics queries
- **Synchronization**: Event-driven sync via Kafka

## Data Models

### Campaign Model
```typescript
interface Campaign {
  id: string;
  name: string;
  type: 'email' | 'sms' | 'push' | 'social' | 'multi-channel';
  status: 'draft' | 'scheduled' | 'active' | 'paused' | 'completed';
  segmentId: string;
  contentId: string;
  schedule: {
    startDate: Date;
    endDate?: Date;
    timezone: string;
    frequency?: string; // cron expression
  };
  budget: {
    total: number;
    spent: number;
    currency: string;
  };
  goals: {
    type: 'clicks' | 'conversions' | 'revenue';
    target: number;
  }[];
  createdBy: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### Segment Model
```typescript
interface Segment {
  id: string;
  name: string;
  description: string;
  type: 'static' | 'dynamic';
  criteria: {
    field: string;
    operator: 'equals' | 'contains' | 'gt' | 'lt' | 'in' | 'not_in';
    value: any;
  }[];
  logic: 'AND' | 'OR';
  size: number;
  lastRefreshed: Date;
  refreshFrequency: string; // cron expression
  createdAt: Date;
  updatedAt: Date;
}
```

### Event Model
```typescript
interface MarketingEvent {
  id: string;
  type: string;
  timestamp: Date;
  customerId: string;
  campaignId?: string;
  properties: Record<string, any>;
  metadata: {
    ipAddress?: string;
    userAgent?: string;
    deviceType?: string;
    location?: {
      country: string;
      city: string;
    };
  };
}
```

### Content Model
```typescript
interface Content {
  id: string;
  name: string;
  type: 'email' | 'sms' | 'push' | 'landing_page';
  version: number;
  subject?: string; // for email
  body: string;
  variables: string[]; // personalization variables
  variants: {
    id: string;
    name: string;
    weight: number; // for A/B testing
    body: string;
  }[];
  status: 'draft' | 'approved' | 'archived';
  createdAt: Date;
  updatedAt: Date;
}
```

## Database Schema

### PostgreSQL Tables

```sql
-- Campaigns Table
CREATE TABLE campaigns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  type VARCHAR(50) NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'draft',
  segment_id UUID REFERENCES segments(id),
  content_id UUID REFERENCES contents(id),
  schedule JSONB NOT NULL,
  budget JSONB NOT NULL,
  goals JSONB,
  created_by UUID NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP,
  CONSTRAINT campaigns_status_check CHECK (status IN ('draft', 'scheduled', 'active', 'paused', 'completed'))
);

CREATE INDEX idx_campaigns_status ON campaigns(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_campaigns_created_at ON campaigns(created_at DESC);
CREATE INDEX idx_campaigns_segment_id ON campaigns(segment_id);

-- Segments Table
CREATE TABLE segments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  type VARCHAR(50) NOT NULL,
  criteria JSONB NOT NULL,
  logic VARCHAR(10) NOT NULL DEFAULT 'AND',
  size BIGINT DEFAULT 0,
  last_refreshed TIMESTAMP,
  refresh_frequency VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP,
  CONSTRAINT segments_type_check CHECK (type IN ('static', 'dynamic'))
);

CREATE INDEX idx_segments_type ON segments(type) WHERE deleted_at IS NULL;
CREATE INDEX idx_segments_last_refreshed ON segments(last_refreshed);

-- Contents Table
CREATE TABLE contents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  type VARCHAR(50) NOT NULL,
  version INT DEFAULT 1,
  subject VARCHAR(500),
  body TEXT NOT NULL,
  variables TEXT[],
  variants JSONB,
  status VARCHAR(50) NOT NULL DEFAULT 'draft',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP,
  CONSTRAINT contents_status_check CHECK (status IN ('draft', 'approved', 'archived'))
);

CREATE INDEX idx_contents_type ON contents(type) WHERE deleted_at IS NULL;
CREATE INDEX idx_contents_status ON contents(status);
```

### ClickHouse Tables (Analytics)

```sql
-- Campaign Events
CREATE TABLE campaign_events (
  event_id UUID,
  event_type String,
  timestamp DateTime,
  customer_id UUID,
  campaign_id UUID,
  properties String, -- JSON
  metadata String,   -- JSON
  date Date DEFAULT toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (campaign_id, customer_id, timestamp);

-- Campaign Metrics (Materialized View)
CREATE MATERIALIZED VIEW campaign_metrics_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (campaign_id, date)
AS SELECT
  campaign_id,
  toDate(timestamp) as date,
  countIf(event_type = 'email.sent') as sent_count,
  countIf(event_type = 'email.opened') as open_count,
  countIf(event_type = 'email.clicked') as click_count,
  countIf(event_type = 'conversion') as conversion_count
FROM campaign_events
GROUP BY campaign_id, date;
```

## Scalability Considerations

### Horizontal Scaling
- **Stateless Services**: All services are stateless and can scale horizontally
- **Database Sharding**: Customer data sharded by customer_id hash
- **Read Replicas**: 3 read replicas for PostgreSQL with load balancing
- **Cache Layer**: Redis cluster for session and frequently accessed data

### Performance Optimization
- **Connection Pooling**: PgBouncer for database connection management
- **Query Optimization**: Indexes on all foreign keys and frequently queried fields
- **Batch Processing**: Campaign emails sent in batches of 1000
- **Async Processing**: Heavy operations queued to RabbitMQ

### Capacity Planning
- **Current**: 10M customers, 1000 active campaigns, 50M events/day
- **Target**: 100M customers, 10K active campaigns, 500M events/day
- **Growth Rate**: 3x year-over-year

### Auto-Scaling Rules
```yaml
services:
  campaign-service:
    min_replicas: 3
    max_replicas: 20
    target_cpu: 70%
    target_memory: 80%
  
  segmentation-service:
    min_replicas: 2
    max_replicas: 15
    target_cpu: 75%
  
  analytics-service:
    min_replicas: 5
    max_replicas: 30
    target_cpu: 65%
```

## Security Model

### Authentication & Authorization

#### OAuth 2.0 Flow
```
Client → Request Token → Auth Server
Auth Server → Validate Credentials → Issue JWT
Client → API Request + JWT → Resource Server
Resource Server → Validate JWT → Process Request
```

#### JWT Token Structure
```json
{
  "sub": "user-id",
  "email": "user@company.com",
  "roles": ["marketing_admin", "campaign_manager"],
  "permissions": ["campaign:create", "campaign:launch", "segment:read"],
  "tenant_id": "tenant-123",
  "exp": 1738368000
}
```

### Access Control

#### Role-Based Permissions
- **Marketing Admin**: Full access to all features
- **Campaign Manager**: Create/edit campaigns, view analytics
- **Content Editor**: Manage content assets only
- **Analyst**: Read-only access to analytics
- **Viewer**: Read-only access to campaigns and segments

### Data Protection

#### Encryption
- **At Rest**: AES-256 encryption for all databases
- **In Transit**: TLS 1.3 for all API communications
- **PII Tokenization**: Sensitive customer data tokenized using Vault

#### Data Retention
- **Campaign Data**: 3 years
- **Event Data**: 2 years (then aggregated)
- **Customer Data**: Retained per GDPR/CCPA requirements
- **Audit Logs**: 7 years

#### Compliance
- **GDPR**: Right to erasure, data portability, consent management
- **CCPA**: Do-not-sell tracking and enforcement
- **CAN-SPAM**: Unsubscribe links, sender identification
- **TCPA**: SMS consent and opt-out management

### Security Hardening
- **API Rate Limiting**: 1000 requests/minute per tenant
- **DDoS Protection**: Cloudflare WAF
- **Secrets Management**: HashiCorp Vault
- **Vulnerability Scanning**: Snyk for dependencies, Trivy for containers
- **Penetration Testing**: Quarterly external audits

## Performance Benchmarks

### API Response Times (p95)
- **Campaign Create**: 180ms
- **Campaign List**: 120ms
- **Segment Refresh** (1M records): 25 seconds
- **Analytics Query**: 300ms
- **Event Ingestion**: 50ms

### Throughput
- **Campaign Launches**: 100/minute
- **Emails Sent**: 10,000/second
- **Events Processed**: 50,000/second
- **API Requests**: 50,000 req/second

### Resource Utilization (Production)
- **CPU**: 45% average, 80% peak
- **Memory**: 60% average, 85% peak
- **Database**: 70% storage, 55% IOPS
- **Network**: 2 Gbps average, 8 Gbps peak

## Monitoring & Alerting

### Key Metrics
- **Service Health**: Uptime, response time, error rate
- **Campaign Performance**: Send rate, open rate, click rate, conversion rate
- **System Resources**: CPU, memory, disk, network
- **Database Performance**: Query time, connection pool, replication lag

### Alert Thresholds
- **Critical**: API error rate > 5%, service down > 2 minutes
- **Warning**: Response time p95 > 500ms, CPU > 80%
- **Info**: Campaign launched, segment refreshed

### Dashboards
- **Operations Dashboard**: Service health, infrastructure metrics
- **Marketing Dashboard**: Campaign performance, funnel metrics
- **Business Dashboard**: Revenue attribution, ROI analysis
