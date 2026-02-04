# HypoText Infrastructure Cost Analysis

> Technical cost documentation for AWS infrastructure scaling
> Last Updated: February 2026

---

## Table of Contents

1. [Current Infrastructure](#current-infrastructure)
2. [Pricing Tiers](#pricing-tiers)
3. [Capacity Thresholds](#capacity-thresholds)
4. [Cost Scenarios](#cost-scenarios)
5. [Profitability Analysis](#profitability-analysis)
6. [Scaling Recommendations](#scaling-recommendations)

---

## Current Infrastructure

### Production Stack (v2)

| Component | Service | Specification | Monthly Cost |
|-----------|---------|---------------|--------------|
| API Server | AWS App Runner | 0.5 vCPU, 2GB RAM | ~$25 |
| Database | AWS RDS PostgreSQL | db.t3.micro (2 vCPU burstable, 1GB) | ~$15 |
| Storage | AWS S3 | Standard, us-east-1 | ~$0.023/GB |
| Renderer | AWS Lambda | 10GB memory, container image | Pay per use |
| Frontend | Vercel | Free tier | $0 |
| CI/CD | GitHub Actions | Free tier (2000 min/month) | $0 |



## Pricing Tiers

### Customer Pricing

| Tier | Bot Visits/Month | Price/Month |
|------|------------------|-------------|
| **Starter** | 1K - 100K | $49 |
| **Growth** | 100K - 3M | $299 |

### Service Limits per Tier

| Tier | Sites | Pages per Site | Render Frequency |
|------|-------|----------------|------------------|
| Starter | 1 | 10 | Weekly |
| Growth | Unlimited | 10 per site | Weekly |

---

## Capacity Thresholds

### Current Infrastructure Limits

#### App Runner (0.5 vCPU, 2GB RAM)

| Metric | Capacity |
|--------|----------|
| Concurrent requests | 50-100 per instance |
| Requests/second | 20-40 per instance |
| Auto-scaling range | 1-25 instances (default) |
| Max throughput | 500-1000 req/sec (25 instances) |

#### RDS db.t3.micro (2 vCPU burstable, 1GB RAM)

| Metric | Capacity |
|--------|----------|
| Max connections | ~85 |
| Baseline CPU | 10% (burstable to 100%) |
| CPU credits | 12/hour |
| Sustained queries/sec | 100-200 |
| Storage IOPS | 3,000 (gp3 baseline) |

### Critical Scaling Thresholds

| Bot Visits/Month | Req/Sec (Avg) | Status | Action Required |
|------------------|---------------|--------|-----------------|
| < 500K | ~0.2 | ✅ Comfortable | None |
| 500K - 2M | 0.2 - 0.8 | ✅ Fine | Monitor metrics |
| 2M - 5M | 0.8 - 2 | ⚠️ Caution | Watch RDS CPU credits |
| **5M - 10M** | 2 - 4 | 🔴 **First threshold** | Upgrade RDS to db.t3.small |
| 10M - 50M | 4 - 20 | 🔴 Scale both | db.t3.medium + 1 vCPU App Runner |
| 50M - 100M | 20 - 40 | 🔴 Major upgrade | db.r6g.large + 2 vCPU |
| **100M+** | 40+ | 🔴 Architecture change | Read replicas + caching |

---

## Cost Scenarios

### Scenario 1: 100 Sites × 1M Visits Each

**Configuration:**
- Sites: 100
- Pages per site: 10
- Total pages: 1,000
- Traffic per site: 1M visits/month
- **Total traffic: 100M visits/month**
- Render frequency: Weekly
- Lambda invocations: 4,000/month

**Traffic Pattern:**
```
100M visits ÷ 30 days ÷ 24 hrs ÷ 3600 sec = 38.5 req/sec average
Peak (3x): 115 req/sec
```

**Recommended Infrastructure:**

| Component | Specification |
|-----------|---------------|
| App Runner | 2 vCPU, 4GB × 5-10 instances |
| RDS | db.r6g.large (2 vCPU, 16GB) |
| ElastiCache | cache.t3.medium |
| S3 | Standard |

**Monthly Cost Breakdown:**

| Component | Calculation | Cost |
|-----------|-------------|------|
| App Runner | 2 vCPU × $0.064 × 730h × 5 + 4GB × $0.007 × 730h × 5 | $570 |
| RDS Primary | db.r6g.large | $365 |
| RDS Storage | 100GB gp3 | $12 |
| ElastiCache | cache.t3.medium | $47 |
| S3 Storage | 1,000 pages × 100KB = 100MB | $0.02 |
| S3 Requests | 100M GETs (90% cache → 10M actual) | $4 |
| Lambda | 4,000 × 10s × 10GB × $0.0000166667 | $7 |
| Data Transfer | 100M × 50KB = 5TB outbound | $450 |
| **Total** | | **$1,455** |

**Unit Economics:**
```
Cost per site: $1,455 ÷ 100 = $14.55/site/month
Cost per million visits: $14.55
```

---

### Scenario 2: 1,000 Sites × 1M Visits Each

**Configuration:**
- Sites: 1,000
- Pages per site: 10
- Total pages: 10,000
- Traffic per site: 1M visits/month
- **Total traffic: 1B visits/month**
- Render frequency: Weekly
- Lambda invocations: 40,000/month

**Traffic Pattern:**
```
1B visits ÷ 30 days ÷ 24 hrs ÷ 3600 sec = 385 req/sec average
Peak (3x): 1,155 req/sec
```

**Recommended Infrastructure:**

| Component | Specification |
|-----------|---------------|
| App Runner | 4 vCPU, 8GB × 20-30 instances |
| RDS Primary | db.r6g.2xlarge (8 vCPU, 64GB) |
| Read Replicas | 2× db.r6g.large |
| ElastiCache | r6g.large cluster |
| CloudFront | CDN for snapshots |

**Monthly Cost Breakdown:**

| Component | Calculation | Cost |
|-----------|-------------|------|
| App Runner | 4 vCPU × $0.064 × 730h × 20 + 8GB × $0.007 × 730h × 20 | $4,555 |
| RDS Primary | db.r6g.2xlarge | $1,459 |
| RDS Read Replicas | 2× db.r6g.large | $730 |
| RDS Storage | 500GB gp3 | $60 |
| ElastiCache | r6g.large | $219 |
| S3 Storage | 10,000 pages × 100KB = 1GB | $0.23 |
| S3 Requests | 1B GETs (95% CDN cache → 50M actual) | $20 |
| CloudFront | 50TB transfer | $4,250 |
| Lambda | 40,000 × 10s × 10GB × $0.0000166667 | $67 |
| Data Transfer | Included in CloudFront | $0 |
| **Total** | | **$11,360** |

**Unit Economics:**
```
Cost per site: $11,360 ÷ 1,000 = $11.36/site/month
Cost per million visits: $11.36
```

---

### Cost Comparison Summary

| Metric | 100 Sites | 1,000 Sites |
|--------|-----------|-------------|
| Total pages | 1,000 | 10,000 |
| Total traffic | 100M/month | 1B/month |
| Requests/sec (avg) | 38.5 | 385 |
| Lambda invocations | 4,000 | 40,000 |
| **Monthly cost** | **$1,455** | **$11,360** |
| Cost per site | $14.55 | $11.36 |
| Cost per 1M visits | $14.55 | $11.36 |

**Economy of Scale:**
```
100 sites → 1,000 sites = 10x customers
Cost increase: $1,455 → $11,360 = 7.8x
Savings at scale: ~22% per site
```

---

## Profitability Analysis

### Revenue vs Cost

#### 100 Sites (Growth Tier @ $299/site)

| Metric | Value |
|--------|-------|
| Revenue | 100 × $299 = $29,900/month |
| Infrastructure Cost | $1,455/month |
| **Gross Profit** | **$28,445/month** |
| **Margin** | **95.1%** |

#### 1,000 Sites (Growth Tier @ $299/site)

| Metric | Value |
|--------|-------|
| Revenue | 1,000 × $299 = $299,000/month |
| Infrastructure Cost | $11,360/month |
| **Gross Profit** | **$287,640/month** |
| **Margin** | **96.2%** |

### Break-Even Analysis

**Cost to serve one Growth customer (1M visits):** ~$11-15/month
**Price charged:** $299/month
**Profit per customer:** ~$285/month

**Minimum customers to cover infrastructure:**

| Scenario | Break-even Customers |
|----------|---------------------|
| 100 Sites infrastructure | $1,455 ÷ $299 = **5 customers** |
| 1,000 Sites infrastructure | $11,360 ÷ $299 = **38 customers** |

### Mixed Tier Distribution

Assuming realistic customer mix:
- 70% Starter ($49, avg 50K visits)
- 30% Growth ($299, avg 1M visits)

| Sites | Starter Revenue | Growth Revenue | Total Revenue | Cost | Profit |
|-------|-----------------|----------------|---------------|------|--------|
| 100 | 70 × $49 = $3,430 | 30 × $299 = $8,970 | $12,400 | ~$500 | $11,900 |
| 1,000 | 700 × $49 = $34,300 | 300 × $299 = $89,700 | $124,000 | ~$4,000 | $120,000 |

### Annual Projections

| Scenario | Monthly Profit | Annual Profit |
|----------|----------------|---------------|
| 100 Growth sites | $28,445 | **$341,340** |
| 1,000 Growth sites | $287,640 | **$3,451,680** |
| 100 Mixed sites | $11,900 | **$142,800** |
| 1,000 Mixed sites | $120,000 | **$1,440,000** |

---

## Scaling Recommendations

### Infrastructure Upgrade Path

| Customer Count | Traffic | Infrastructure Changes |
|----------------|---------|----------------------|
| 1-50 | < 50M | Current setup (db.t3.micro, 0.5 vCPU) |
| 50-100 | 50-100M | Upgrade RDS to db.t3.small |
| 100-300 | 100-300M | db.r6g.large + 1 vCPU App Runner |
| 300-500 | 300-500M | Add ElastiCache, increase instances |
| 500-1000 | 500M-1B | db.r6g.2xlarge + read replica + CloudFront |
| 1000+ | 1B+ | Full distributed architecture |

### Cost Optimization Strategies

| Strategy | Potential Savings | When to Apply |
|----------|-------------------|---------------|
| Reserved Instances (RDS) | 30% (~$650/mo at 1K sites) | 1-year commitment |
| Savings Plans (Compute) | 20% (~$900/mo at 1K sites) | 1-year commitment |
| CloudFront caching | 40% on data transfer | At 100+ sites |
| ElastiCache for hot URLs | 30% RDS load reduction | At 50+ sites |
| Batch Lambda rendering | 15% Lambda cost | At 1000+ pages |

**Optimized Costs (with commitments):**

| Scenario | Standard | Optimized | Savings |
|----------|----------|-----------|---------|
| 100 Sites | $1,455 | ~$1,100 | 24% |
| 1,000 Sites | $11,360 | ~$8,500 | 25% |

---

## AWS Pricing Reference

### App Runner

| Resource | Price |
|----------|-------|
| vCPU | $0.064/vCPU-hour |
| Memory | $0.007/GB-hour |
| Provisioned instances | Additional charges apply |

### RDS PostgreSQL

| Instance Type | vCPU | Memory | Price/month |
|---------------|------|--------|-------------|
| db.t3.micro | 2 (burst) | 1 GB | $15 |
| db.t3.small | 2 (burst) | 2 GB | $25 |
| db.t3.medium | 2 (burst) | 4 GB | $50 |
| db.r6g.large | 2 | 16 GB | $365 |
| db.r6g.xlarge | 4 | 32 GB | $730 |
| db.r6g.2xlarge | 8 | 64 GB | $1,459 |

### Lambda

| Resource | Price |
|----------|-------|
| Requests | $0.20 per 1M requests |
| Duration | $0.0000166667 per GB-second |
| Provisioned concurrency | Additional charges |

### S3

| Resource | Price |
|----------|-------|
| Storage | $0.023/GB/month |
| PUT requests | $0.005 per 1,000 |
| GET requests | $0.0004 per 1,000 |

### Data Transfer

| Volume | Price |
|--------|-------|
| First 10 TB | $0.09/GB |
| Next 40 TB | $0.085/GB |
| Next 100 TB | $0.07/GB |
| Over 150 TB | $0.05/GB |

### CloudFront

| Volume | Price |
|--------|-------|
| First 10 TB | $0.085/GB |
| Next 40 TB | $0.080/GB |
| Next 100 TB | $0.060/GB |

---

## Summary

### Key Metrics

| Metric | Value |
|--------|-------|
| Minimum viable cost | ~$40/month (current setup) |
| Cost per Growth customer | ~$11-15/month |
| Gross margin at scale | 95-96% |
| Break-even customers | 5-38 (depends on infrastructure tier) |

### Critical Thresholds

| Threshold | Traffic | Action |
|-----------|---------|--------|
| First upgrade | 5M visits/month | Upgrade RDS |
| Major scaling | 100M visits/month | Add caching + replicas |
| Architecture change | 1B+ visits/month | Distributed system |

---


