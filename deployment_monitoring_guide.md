# Referral System Deployment Guide

## 1. Prerequisites

### System Requirements
- Node.js v14+
- MongoDB v4.4+
- Redis v6+
- Minimum 4GB RAM
- 2 vCPUs
- 20GB Storage

### Environment Variables
```bash
# Database
MONGODB_URI=mongodb://localhost:27017/neurolov
MONGODB_POOL_SIZE=10

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_password
REDIS_PREFIX=ref:

# Referral System
REFERRAL_MAX_CHAIN_DEPTH=3
REFERRAL_REWARD_PROCESSING_INTERVAL=5000
REFERRAL_QUEUE_CLEANUP_INTERVAL=3600000
REFERRAL_CACHE_TTL=300
REFERRAL_MAX_RETRIES=3

# Monitoring
PROMETHEUS_ENABLED=true
METRICS_PORT=9090
```

## 2. Database Setup

### MongoDB Indexes
```javascript
// Run these commands in MongoDB shell
db.users.createIndex({ "referralCode": 1 }, { unique: true, sparse: true })
db.users.createIndex({ "referralStats.totalEarned": -1 })
db.users.createIndex({ "referredBy": 1, "createdAt": -1 })
db.referrals.createIndex({ "referrer": 1, "tier": 1, "status": 1 })
db.referrals.createIndex({ "code": 1, "status": 1 })
db.referralTransactions.createIndex({ "userId": 1, "status": 1, "createdAt": -1 })
```

### Redis Configuration
```bash
maxmemory 2gb
maxmemory-policy allkeys-lru
appendonly yes
```

## 3. Deployment Steps

### 1. Pre-deployment Checks
```bash
# Verify environment
npm run verify-env

# Run tests
npm run test:referral

# Verify database connection
npm run verify-db

# Check Redis connection
npm run verify-redis
```

### 2. Database Migration
```bash
# Backup existing data
npm run backup

# Run migration
npm run migrate:referral

# Verify migration
npm run verify-migration
```

### 3. Service Deployment
```bash
# Deploy core services
npm run deploy:referral

# Start background jobs
npm run start:jobs

# Initialize caching
npm run init:cache

# Start metrics collection
npm run start:metrics
```

## 4. Monitoring Setup

### Prometheus Configuration
```yaml
scrape_configs:
  - job_name: 'referral_metrics'
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: '/metrics'
    scrape_interval: 15s
```

### Grafana Dashboards
1. Referral Performance Dashboard
   - Chain depth distribution
   - Reward distribution rates
   - Queue sizes
   - Processing times

2. Error Monitoring Dashboard
   - Failed transactions
   - Retry rates
   - Error types distribution
   - Chain integrity issues

### Alert Rules
```yaml
groups:
  - name: referral_alerts
    rules:
      - alert: HighFailureRate
        expr: rate(referral_failures_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
      - alert: QueueBacklog
        expr: referral_queue_size > 1000
        for: 10m
        labels:
          severity: warning
```

## 5. Performance Tuning

### MongoDB Optimization
```javascript
{
  wtimeout: 2500,
  poolSize: 10,
  useCreateIndex: true,
  useFindAndModify: false,
  retryWrites: true,
  readPreference: 'secondaryPreferred'
}
```

### Redis Optimization
```bash
maxclients 10000
timeout 300
tcp-keepalive 60
```

### Node.js Settings
```bash
NODE_OPTIONS="--max-old-space-size=4096"
UV_THREADPOOL_SIZE=8
```

## 6. Scaling Considerations

### Horizontal Scaling
1. Redis Cluster Setup
2. MongoDB Sharding
3. Load Balancer Configuration

### Vertical Scaling
1. Memory Requirements
   - Base: 4GB
   - Per 10k users: +2GB
   - Per 100k transactions: +4GB

2. CPU Requirements
   - Base: 2 vCPUs
   - Per 10k concurrent users: +2 vCPUs

## 7. Backup Strategy

### Daily Backups
```bash
# MongoDB
mongodump --db neurolov --collection referrals
mongodump --db neurolov --collection referralTransactions

# Redis
redis-cli SAVE
```

### Retention Policy
- Daily backups: 7 days
- Weekly backups: 4 weeks
- Monthly backups: 6 months

## 8. Recovery Procedures

### System Recovery
```bash
# Restore from backup
npm run restore:referral -- --backup=<backup_date>

# Verify data integrity
npm run verify:integrity

# Rebuild indexes
npm run rebuild:indexes
```

### Chain Repair
```bash
# Check chain integrity
npm run check:chains

# Repair broken chains
npm run repair:chains

# Verify repairs
npm run verify:chains
```

## 9. Maintenance Procedures

### Daily Tasks
1. Monitor queue sizes
2. Check error rates
3. Verify reward distributions
4. Review system metrics

### Weekly Tasks
1. Clean up old transactions
2. Verify chain integrity
3. Update statistics
4. Review performance metrics

### Monthly Tasks
1. Full system audit
2. Clean up old data
3. Review and optimize indexes
4. Update monitoring thresholds

## 10. Troubleshooting Guide

### Common Issues
1. Failed Reward Distribution
   - Check transaction logs
   - Verify user existence
   - Check chain integrity

2. Chain Integrity Issues
   - Run chain verification
   - Check for circular references
   - Verify depth constraints

3. Performance Issues
   - Review index usage
   - Check queue sizes
   - Monitor cache hit rates

Would you like me to provide more details about any specific aspect of the deployment or additional information about specific components?
