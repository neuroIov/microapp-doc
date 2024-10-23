# Neurolov Compute Bot - Referral System Technical Documentation

## Table of Contents
1. System Overview
2. Architecture Design
3. Implementation Guide
4. Integration Points
5. Migration Strategy
6. Testing Protocol
7. Deployment Guide
8. Troubleshooting Guide
9. API Documentation
10. Monitoring & Maintenance

## 1. System Overview

### 1.1 Core Features
- Three-tier referral system (10%, 5%, 2.5% rewards)
- Real-time reward distribution
- Unique referral code generation
- Chain management
- Performance optimizations
- Error recovery

### 1.2 Technical Stack Requirements
- Node.js v14+
- MongoDB v4.4+
- Redis v6+
- Bull Queue
- Prometheus/Grafana (optional)

### 1.3 System Architecture
```plaintext
User Action → API Gateway → Referral Service → Queue System → Reward Distribution
                                  ↓
                            Cache Layer
                                  ↓
                         MongoDB + Redis
```

## 2. Architecture Design

### 2.1 Data Models

#### User Model Extensions
```javascript
referralStats: {
  totalEarned: Number,        // Total XP earned from referrals
  directReferrals: Number,    // Count of direct referrals
  activeReferrals: Number,    // Currently active referrals
  referralTiers: {
    tier1Count: Number,       // Direct referrals count
    tier2Count: Number,       // Second-level referrals
    tier3Count: Number        // Third-level referrals
  }
}
```

#### Referral Chain Structure
```plaintext
User A (Tier 3) ← User B (Tier 2) ← User C (Tier 1) ← User D (Earner)
```

### 2.2 Reward Distribution Flow
1. User earns XP
2. System identifies referral chain
3. Calculates tiered rewards
4. Queues reward distribution
5. Processes rewards asynchronously
6. Updates user statistics

## 3. Implementation Guide

### 3.1 Prerequisites
1. Ensure MongoDB with transactions support
2. Configure Redis for queue management
3. Set up background workers
4. Implement error tracking

### 3.2 Integration Steps

#### Step 1: Database Updates
```sql
-- Add new indexes
db.users.createIndex({"referralCode": 1}, {unique: true, sparse: true});
db.referrals.createIndex({"referrer": 1, "tier": 1});
db.referralTransactions.createIndex({"status": 1, "createdAt": -1});
```

#### Step 2: Service Integration
```javascript
// In userController.js tap function
exports.tap = async (req, res) => {
  const session = await mongoose.startSession();
  try {
    session.startTransaction();
    
    // Existing tap logic...
    
    // Add referral reward processing
    await referralController.processReferralReward(user._id, xpGained, session);
    
    await session.commitTransaction();
  } catch (error) {
    await session.abortTransaction();
    throw error;
  } finally {
    session.endSession();
  }
};
```

#### Step 3: Queue Setup
```javascript
// Initialize reward queue
const rewardQueue = new Bull('referral-rewards', {
  redis: {
    host: process.env.REDIS_HOST,
    port: process.env.REDIS_PORT,
    password: process.env.REDIS_PASSWORD
  }
});
```

### 3.3 Critical Code Sections

#### Chain Management
```javascript
const validateChain = async (userId, referrerId) => {
  // Prevent circular references
  if (userId.equals(referrerId)) {
    throw new ReferralError('CIRCULAR_REFERENCE');
  }

  // Check chain depth
  const chain = await buildChain(referrerId);
  if (chain.length >= MAX_CHAIN_DEPTH) {
    throw new ReferralError('MAX_DEPTH_EXCEEDED');
  }

  return chain;
};
```

## 4. Integration Points

### 4.1 Existing Features Integration

#### XP System
```javascript
// Points where referral rewards should trigger:
1. User taps (userController.tap)
2. Quest completion (questController.complete)
3. Daily claims (userController.claimDaily)
4. Achievement unlocks (achievementController.unlock)
```

#### Achievement System
```javascript
// Add referral-based achievements:
1. First Referral (1 referral)
2. Referral Master (10 referrals)
3. Chain Builder (complete 3-tier chain)
```

### 4.2 Required Code Changes

```plaintext
1. models/User.js
   - Add referral fields
   - Add validation methods
   - Update indexes

2. controllers/userController.js
   - Add reward processing
   - Update XP calculations
   - Add referral methods

3. config/index.js
   - Add referral configuration
   - Update rate limits
   - Add queue settings
```

## 5. Migration Strategy

### 5.1 Pre-Migration Tasks
1. Backup all user data
2. Create new indexes
3. Verify database capacity
4. Test rollback procedures

### 5.2 Migration Steps
```javascript
// Step 1: Add new fields
db.users.updateMany({}, {
  $set: {
    referralStats: {
      totalEarned: 0,
      directReferrals: 0,
      activeReferrals: 0
    }
  }
});

// Step 2: Initialize referral codes
const users = await User.find({ referralCode: { $exists: false } });
for (const user of users) {
  user.referralCode = generateUniqueCode();
  await user.save();
}

// Step 3: Build existing chains
await buildExistingReferralChains();
```

### 5.3 Rollback Plan
```javascript
// Store backup collections
const backup = {
  users: 'users_backup_' + Date.now(),
  referrals: 'referrals_backup_' + Date.now()
};

// Rollback procedure
const rollback = async () => {
  await db.collection('users').drop();
  await db.collection(backup.users).rename('users');
};
```

## 6. Testing Protocol

### 6.1 Unit Tests
```javascript
describe('Referral System', () => {
  test('should generate unique codes');
  test('should prevent circular references');
  test('should maintain max chain depth');
  test('should calculate rewards correctly');
  test('should handle concurrent operations');
});
```

### 6.2 Integration Tests
1. Reward distribution
2. Chain management
3. Cache synchronization
4. Queue processing

### 6.3 Performance Tests
1. Concurrent operations
2. Large-scale reward processing
3. Chain traversal efficiency

## 7. Monitoring & Maintenance

### 7.1 Key Metrics
```javascript
// Monitor these metrics
1. Chain depths distribution
2. Reward processing times
3. Queue sizes and latency
4. Cache hit rates
5. Error rates and types
```

### 7.2 Alerts
```javascript
// Critical alerts
1. High queue backlog (>1000 items)
2. High error rate (>5%)
3. Chain validation failures
4. Reward distribution delays
```

## 8. Common Issues & Solutions

### 8.1 Known Issues
1. Race conditions in reward distribution
2. Cache inconsistency
3. Queue backlogs
4. Chain validation errors

### 8.2 Solutions
```javascript
// Race Condition Solution
const distributeRewards = async (userId, amount) => {
  const session = await mongoose.startSession();
  try {
    session.startTransaction();
    // Process rewards with transaction
    await session.commitTransaction();
  } catch (error) {
    await session.abortTransaction();
    // Add to retry queue
  }
};
```

## 9. API Documentation

### 9.1 Core Endpoints
```plaintext
POST /api/referral/generate
- Generate referral code
- Rate limit: 5/hour

POST /api/referral/apply
- Apply referral code
- Rate limit: 3/hour

GET /api/referral/stats
- Get referral statistics
- Cache duration: 5 minutes

GET /api/referral/chain
- Get referral chain
- Cache duration: 1 hour
```

### 9.2 Admin Endpoints
```plaintext
GET /api/admin/referral/metrics
- System-wide metrics
- Requires admin access

POST /api/admin/referral/repair
- Repair broken chains
- Requires admin access
```

## 10. Security Considerations

### 10.1 Rate Limiting
```javascript
const rateLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 5, // 5 requests per hour
  message: 'Too many attempts'
});
```

### 10.2 Validation
```javascript
const validateReferralCode = (code) => {
  return /^[A-Z0-9]{8}$/.test(code);
};
```

## 11. Production Checklist

### 11.1 Pre-Deployment
- [ ] All migrations tested
- [ ] Backup procedures verified
- [ ] Monitoring set up
- [ ] Load testing completed
- [ ] Security audit done

### 11.2 Deployment
- [ ] Database backups
- [ ] Staged rollout
- [ ] Performance monitoring
- [ ] Error tracking
- [ ] User communication

### 11.3 Post-Deployment
- [ ] Monitor error rates
- [ ] Check performance metrics
- [ ] Verify reward distribution
- [ ] Monitor user feedback
- [ ] Update documentation

Would you like me to provide more detailed information about any specific section or create additional documentation for specific components?
