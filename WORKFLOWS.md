# Marketing System Workflows

## Development Workflow

### Branch Strategy

We follow **GitHub Flow** with protection rules:

```
main (protected)
  ├─ feature/campaign-ab-testing
  ├─ feature/segment-performance
  ├─ bugfix/email-rendering
  └─ hotfix/rate-limit-issue
```

### Branch Naming Convention
- **Features**: `feature/short-description`
- **Bug Fixes**: `bugfix/issue-description`
- **Hotfixes**: `hotfix/critical-issue`
- **Refactoring**: `refactor/component-name`

### Commit Message Format
```
type(scope): subject

body (optional)

footer (optional)
```

**Types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

**Examples**:
```
feat(campaigns): add A/B testing support for email campaigns
fix(segmentation): resolve race condition in segment refresh
docs(api): update campaign API endpoint documentation
```

### Pull Request Process

1. **Create Branch**
   ```bash
   git checkout -b feature/new-feature
   ```

2. **Make Changes & Commit**
   ```bash
   git add .
   git commit -m "feat(scope): description"
   ```

3. **Push & Create PR**
   ```bash
   git push origin feature/new-feature
   ```
   - Open PR in GitHub
   - Fill out PR template
   - Link related issues

4. **Code Review**
   - At least 1 approval required
   - All checks must pass
   - No merge conflicts

5. **Merge**
   - Squash and merge to main
   - Delete branch after merge

### PR Template
```markdown
## Description
Brief description of changes

## Type of Change
- [ ] New feature
- [ ] Bug fix
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No new warnings
```

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Marketing System CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Type check
        run: npm run type-check
      
      - name: Run unit tests
        run: npm run test:unit
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          REDIS_URL: redis://localhost:6379
      
      - name: Run integration tests
        run: npm run test:integration
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'

  build:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/ionoi-inc/marketing-system:latest
            ghcr.io/ionoi-inc/marketing-system:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - name: Deploy to staging
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/staging/deployment.yaml
            k8s/staging/service.yaml
          images: |
            ghcr.io/ionoi-inc/marketing-system:${{ github.sha }}
          namespace: marketing-staging
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          BASE_URL: https://staging-marketing.ionoi.io

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to production (blue-green)
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/production/deployment.yaml
            k8s/production/service.yaml
          images: |
            ghcr.io/ionoi-inc/marketing-system:${{ github.sha }}
          namespace: marketing-production
          strategy: blue-green
      
      - name: Health check
        run: |
          curl -f https://marketing.ionoi.io/health || exit 1
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Marketing System deployed to production",
              "version": "${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## Testing Strategy

### Test Pyramid

```
        /\
       /E2E\         (5%)  - Full user flows
      /------\
     /Integ  \       (25%) - Service interactions
    /----------\
   /   Unit     \    (70%) - Component logic
  /--------------\
```

### Unit Tests

**Coverage Requirements**: Minimum 80%

**Framework**: Jest + TypeScript

**Example**:
```typescript
describe('CampaignService', () => {
  let service: CampaignService;
  let mockRepository: jest.Mocked<CampaignRepository>;

  beforeEach(() => {
    mockRepository = {
      create: jest.fn(),
      findById: jest.fn(),
      update: jest.fn(),
    } as any;
    
    service = new CampaignService(mockRepository);
  });

  describe('createCampaign', () => {
    it('should create a campaign with valid data', async () => {
      const campaignData = {
        name: 'Test Campaign',
        type: 'email',
        segmentId: 'seg_123',
      };
      
      mockRepository.create.mockResolvedValue({
        id: 'cmp_123',
        ...campaignData,
      });

      const result = await service.createCampaign(campaignData);

      expect(result.id).toBe('cmp_123');
      expect(mockRepository.create).toHaveBeenCalledWith(campaignData);
    });

    it('should throw error for invalid campaign type', async () => {
      const campaignData = {
        name: 'Test Campaign',
        type: 'invalid',
        segmentId: 'seg_123',
      };

      await expect(service.createCampaign(campaignData))
        .rejects.toThrow('Invalid campaign type');
    });
  });
});
```

**Run Command**:
```bash
npm run test:unit
npm run test:unit -- --coverage
npm run test:unit -- --watch
```

### Integration Tests

**Purpose**: Test service interactions, database queries, API endpoints

**Framework**: Jest + Supertest

**Example**:
```typescript
describe('Campaign API', () => {
  let app: Express;
  let db: Database;

  beforeAll(async () => {
    db = await setupTestDatabase();
    app = createApp(db);
  });

  afterAll(async () => {
    await db.close();
  });

  describe('POST /api/v1/campaigns', () => {
    it('should create a campaign', async () => {
      const response = await request(app)
        .post('/api/v1/campaigns')
        .set('Authorization', `Bearer ${testToken}`)
        .send({
          name: 'Integration Test Campaign',
          type: 'email',
          segmentId: 'seg_123',
        })
        .expect(201);

      expect(response.body.id).toBeDefined();
      expect(response.body.name).toBe('Integration Test Campaign');
    });

    it('should return 401 without auth', async () => {
      await request(app)
        .post('/api/v1/campaigns')
        .send({
          name: 'Test Campaign',
        })
        .expect(401);
    });
  });
});
```

**Run Command**:
```bash
npm run test:integration
```

### End-to-End Tests

**Purpose**: Test complete user workflows

**Framework**: Playwright

**Example**:
```typescript
test('Create and launch email campaign', async ({ page }) => {
  // Login
  await page.goto('https://staging-marketing.ionoi.io/login');
  await page.fill('[name="email"]', 'test@ionoi.io');
  await page.fill('[name="password"]', 'testpass');
  await page.click('button[type="submit"]');

  // Navigate to campaigns
  await page.click('text=Campaigns');
  await page.click('text=Create Campaign');

  // Fill campaign form
  await page.fill('[name="name"]', 'E2E Test Campaign');
  await page.selectOption('[name="type"]', 'email');
  await page.selectOption('[name="segment"]', 'seg_123');
  await page.fill('[name="subject"]', 'Test Subject');
  await page.fill('[name="body"]', 'Test email body');

  // Launch campaign
  await page.click('button:has-text("Launch")');
  
  // Verify success
  await expect(page.locator('.toast-success')).toContainText('Campaign launched');
});
```

**Run Command**:
```bash
npm run test:e2e
npm run test:e2e -- --headed  # Visual mode
```

### Load Testing

**Tool**: k6

**Scenario**:
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp to 200
    { duration: '5m', target: 200 },  // Stay at 200
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% requests < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  const response = http.get('https://api.ionoi.io/v1/campaigns');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

**Run Command**:
```bash
k6 run load-test.js
```

## Monitoring and Alerts

### Health Checks

**Endpoint**: `GET /health`

**Response**:
```json
{
  "status": "healthy",
  "version": "1.2.3",
  "uptime": 3600,
  "checks": {
    "database": "healthy",
    "redis": "healthy",
    "kafka": "healthy",
    "email_service": "healthy"
  }
}
```

**Kubernetes Probes**:
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
```

### Metrics Collection

**Tool**: Prometheus

**Exposed Metrics**:
```
# Request metrics
http_requests_total{method, path, status}
http_request_duration_seconds{method, path}

# Business metrics
campaigns_created_total
campaigns_launched_total
emails_sent_total
emails_opened_total
emails_clicked_total

# System metrics
nodejs_heap_size_used_bytes
nodejs_eventloop_lag_seconds
database_connections_active
redis_commands_processed_total
```

**Scrape Config**:
```yaml
scrape_configs:
  - job_name: 'marketing-system'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: marketing-system
        action: keep
```

### Alerting Rules

**Prometheus Alerts**:
```yaml
groups:
  - name: marketing-system
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} (threshold: 0.05)"

      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.95, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Slow API response time"
          description: "P95 response time is {{ $value }}s"

      - alert: CampaignSendRateLow
        expr: |
          rate(emails_sent_total[5m]) < 100
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Campaign send rate below threshold"
          description: "Only {{ $value }} emails/min (expected: 1000+)"

      - alert: DatabaseConnectionPoolExhausted
        expr: |
          database_connections_active / database_connections_max > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool nearly exhausted"
```

### Dashboards

**Grafana Dashboards**:

1. **System Overview**
   - Request rate & latency
   - Error rate
   - CPU & Memory usage
   - Database performance

2. **Campaign Metrics**
   - Active campaigns
   - Send rate
   - Open/click rates
   - Conversion tracking

3. **Business KPIs**
   - Revenue attribution
   - ROI by channel
   - Customer engagement trends
   - Segment performance

### Logging

**Format**: Structured JSON logs

**Example**:
```json
{
  "timestamp": "2026-02-11T07:00:00Z",
  "level": "info",
  "service": "marketing-system",
  "version": "1.2.3",
  "request_id": "req_abc123",
  "user_id": "user_456",
  "message": "Campaign launched successfully",
  "context": {
    "campaign_id": "cmp_789",
    "segment_size": 50000,
    "type": "email"
  }
}
```

**Log Levels**:
- **ERROR**: System errors, failed operations
- **WARN**: Degraded performance, retries
- **INFO**: Normal operations, campaign launches
- **DEBUG**: Detailed debugging information

**Retention**:
- **Hot storage** (Elasticsearch): 7 days
- **Warm storage** (S3): 90 days
- **Cold storage** (Glacier): 2 years

## Runbooks

### Campaign Not Sending

**Symptoms**:
- Campaigns stuck in "active" status
- No emails being sent
- Send rate dropped to zero

**Investigation**:
1. Check queue depth: `kubectl exec -it rabbitmq-0 -- rabbitmqctl list_queues`
2. Check consumer status: `kubectl logs -l app=campaign-worker`
3. Check email service status: `curl https://api.sendgrid.com/v3/health`

**Resolution**:
- If queue backed up: Scale workers `kubectl scale deployment campaign-worker --replicas=10`
- If email service down: Switch to backup provider in config
- If database connection issues: Check connection pool settings

### Database Performance Degradation

**Symptoms**:
- Slow API responses
- High database CPU
- Query timeouts

**Investigation**:
1. Check slow queries: `SELECT * FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;`
2. Check connection count: `SELECT count(*) FROM pg_stat_activity;`
3. Check replication lag: `SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) FROM pg_stat_replication;`

**Resolution**:
- Add missing indexes
- Optimize slow queries
- Scale read replicas
- Enable query result caching

## Release Process

### Versioning

**Semantic Versioning**: `MAJOR.MINOR.PATCH`

- **MAJOR**: Breaking changes
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes

### Release Checklist

- [ ] All tests passing
- [ ] Documentation updated
- [ ] CHANGELOG.md updated
- [ ] Version bumped in package.json
- [ ] Git tag created
- [ ] Docker image built and tagged
- [ ] Staging deployment successful
- [ ] E2E tests pass on staging
- [ ] Production deployment approved
- [ ] Post-deployment health checks pass
- [ ] Release notes published

### Rollback Procedure

```bash
# Quick rollback to previous version
kubectl rollout undo deployment/marketing-system -n production

# Rollback to specific revision
kubectl rollout undo deployment/marketing-system --to-revision=5 -n production

# Monitor rollback status
kubectl rollout status deployment/marketing-system -n production
```

## Support & Resources

- **Development Guide**: `/docs/development.md`
- **API Documentation**: `https://api-docs.ionoi.io/marketing`
- **Slack Channel**: `#marketing-system-dev`
- **On-call Rotation**: PagerDuty schedule
- **Incident Management**: `/docs/incidents.md`
