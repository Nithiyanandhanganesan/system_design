# Rate Limiting - Complete Guide

## What is Rate Limiting?

Rate limiting controls the number of requests a client can make to an API within a specific time window. It protects servers from abuse, ensures fair usage, and maintains service quality.

## Why Rate Limiting Matters

```javascript
// Without Rate Limiting:
// Malicious user sends 10,000 requests/second
// Server crashes, legitimate users affected

// With Rate Limiting:
// User limited to 100 requests/minute
// Server remains stable
// Fair access for all users
```

## Common Rate Limiting Algorithms

### 1. Fixed Window Counter
```javascript
class FixedWindowLimiter {
  constructor(limit, windowMs) {
    this.limit = limit;
    this.windowMs = windowMs;
    this.windows = new Map();
  }
  
  isAllowed(key) {
    const now = Date.now();
    const windowStart = Math.floor(now / this.windowMs) * this.windowMs;
    const windowKey = `${key}:${windowStart}`;
    
    const currentCount = this.windows.get(windowKey) || 0;
    
    if (currentCount >= this.limit) {
      return false;
    }
    
    this.windows.set(windowKey, currentCount + 1);
    return true;
  }
}

// Usage
const limiter = new FixedWindowLimiter(100, 60000); // 100 req/min

app.use('/api/', (req, res, next) => {
  const clientIP = req.ip;
  
  if (!limiter.isAllowed(clientIP)) {
    return res.status(429).json({
      error: 'Too many requests',
      retryAfter: 60
    });
  }
  
  next();
});
```

### 2. Sliding Window Log
```javascript
class SlidingWindowLimiter {
  constructor(limit, windowMs) {
    this.limit = limit;
    this.windowMs = windowMs;
    this.logs = new Map();
  }
  
  isAllowed(key) {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    // Get existing log
    let log = this.logs.get(key) || [];
    
    // Remove old entries
    log = log.filter(timestamp => timestamp > windowStart);
    
    if (log.length >= this.limit) {
      return false;
    }
    
    // Add current request
    log.push(now);
    this.logs.set(key, log);
    
    return true;
  }
}
```

### 3. Token Bucket Algorithm
```javascript
class TokenBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillRate; // tokens per second
    this.lastRefill = Date.now();
  }
  
  consume(tokens = 1) {
    this.refill();
    
    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }
    
    return false;
  }
  
  refill() {
    const now = Date.now();
    const timePassed = (now - this.lastRefill) / 1000;
    const tokensToAdd = timePassed * this.refillRate;
    
    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
}

// Usage with Redis
class DistributedTokenBucket {
  constructor(redis, capacity, refillRate) {
    this.redis = redis;
    this.capacity = capacity;
    this.refillRate = refillRate;
  }
  
  async consume(key, tokens = 1) {
    const lua = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refill_rate = tonumber(ARGV[2])
      local requested = tonumber(ARGV[3])
      local now = tonumber(ARGV[4])
      
      local bucket = redis.call('hmget', key, 'tokens', 'last_refill')
      local tokens = tonumber(bucket[1]) or capacity
      local last_refill = tonumber(bucket[2]) or now
      
      -- Refill tokens
      local time_passed = (now - last_refill) / 1000
      local tokens_to_add = time_passed * refill_rate
      tokens = math.min(capacity, tokens + tokens_to_add)
      
      if tokens >= requested then
        tokens = tokens - requested
        redis.call('hmset', key, 'tokens', tokens, 'last_refill', now)
        redis.call('expire', key, 3600)
        return 1
      else
        redis.call('hmset', key, 'tokens', tokens, 'last_refill', now)
        redis.call('expire', key, 3600)
        return 0
      end
    `;
    
    const result = await this.redis.eval(
      lua, 
      1, 
      key, 
      this.capacity, 
      this.refillRate, 
      tokens, 
      Date.now()
    );
    
    return result === 1;
  }
}
```

### 4. Sliding Window Counter
```javascript
class SlidingWindowCounter {
  constructor(limit, windowMs) {
    this.limit = limit;
    this.windowMs = windowMs;
    this.buckets = new Map();
  }
  
  isAllowed(key) {
    const now = Date.now();
    const currentWindow = Math.floor(now / this.windowMs);
    const previousWindow = currentWindow - 1;
    
    const currentCount = this.getBucketCount(key, currentWindow);
    const previousCount = this.getBucketCount(key, previousWindow);
    
    // Calculate weight for previous window
    const timeInCurrentWindow = now % this.windowMs;
    const weight = 1 - (timeInCurrentWindow / this.windowMs);
    
    const estimatedCount = (previousCount * weight) + currentCount;
    
    if (estimatedCount >= this.limit) {
      return false;
    }
    
    this.incrementBucket(key, currentWindow);
    return true;
  }
  
  getBucketCount(key, window) {
    const bucketKey = `${key}:${window}`;
    return this.buckets.get(bucketKey) || 0;
  }
  
  incrementBucket(key, window) {
    const bucketKey = `${key}:${window}`;
    const current = this.buckets.get(bucketKey) || 0;
    this.buckets.set(bucketKey, current + 1);
  }
}
```

## Implementation Examples

### Express.js Middleware
```javascript
const rateLimit = require('express-rate-limit');

// Basic rate limiting
const basicLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/api/', basicLimiter);

// Advanced rate limiting with Redis
const RedisStore = require('rate-limit-redis');
const redis = require('redis');
const client = redis.createClient();

const advancedLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args) => client.sendCommand(args),
  }),
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,
});

// Different limits for different endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // Stricter limit for authentication
  skipSuccessfulRequests: true,
});

app.use('/api/auth/', authLimiter);

// API key based limiting
const apiKeyLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: (req) => {
    // Different limits based on API key tier
    const apiKey = req.headers['x-api-key'];
    const tier = getApiKeyTier(apiKey);
    
    switch (tier) {
      case 'premium': return 1000;
      case 'standard': return 100;
      case 'basic': return 10;
      default: return 5;
    }
  },
  keyGenerator: (req) => req.headers['x-api-key'] || req.ip,
});
```

### Custom Implementation with Redis
```javascript
class RedisRateLimiter {
  constructor(redis) {
    this.redis = redis;
  }
  
  async isAllowed(key, limit, windowSeconds) {
    const pipeline = this.redis.pipeline();
    const now = Math.floor(Date.now() / 1000);
    const window = Math.floor(now / windowSeconds);
    const redisKey = `rate_limit:${key}:${window}`;
    
    // Increment counter
    pipeline.incr(redisKey);
    pipeline.expire(redisKey, windowSeconds);
    
    const results = await pipeline.exec();
    const count = results[0][1];
    
    return count <= limit;
  }
  
  async getRemainingRequests(key, limit, windowSeconds) {
    const now = Math.floor(Date.now() / 1000);
    const window = Math.floor(now / windowSeconds);
    const redisKey = `rate_limit:${key}:${window}`;
    
    const count = await this.redis.get(redisKey) || 0;
    return Math.max(0, limit - count);
  }
  
  async getResetTime(windowSeconds) {
    const now = Math.floor(Date.now() / 1000);
    const window = Math.floor(now / windowSeconds);
    return (window + 1) * windowSeconds;
  }
}

// Usage
const limiter = new RedisRateLimiter(redis);

app.use(async (req, res, next) => {
  const key = req.ip;
  const limit = 100;
  const windowSeconds = 3600; // 1 hour
  
  const allowed = await limiter.isAllowed(key, limit, windowSeconds);
  
  if (!allowed) {
    const resetTime = await limiter.getResetTime(windowSeconds);
    
    return res.status(429).json({
      error: 'Rate limit exceeded',
      resetTime: resetTime,
      retryAfter: resetTime - Math.floor(Date.now() / 1000)
    });
  }
  
  // Add rate limit headers
  const remaining = await limiter.getRemainingRequests(key, limit, windowSeconds);
  res.set({
    'X-RateLimit-Limit': limit,
    'X-RateLimit-Remaining': remaining,
    'X-RateLimit-Reset': await limiter.getResetTime(windowSeconds)
  });
  
  next();
});
```

## Advanced Patterns

### Hierarchical Rate Limiting
```javascript
class HierarchicalRateLimiter {
  constructor() {
    this.limiters = {
      global: new TokenBucket(10000, 100), // Global: 10k capacity, 100/sec
      perUser: new Map(),
      perIP: new Map()
    };
  }
  
  async isAllowed(userId, ip, tokens = 1) {
    // Check global limit first
    if (!this.limiters.global.consume(tokens)) {
      return { allowed: false, reason: 'global_limit' };
    }
    
    // Check per-user limit
    if (!this.limiters.perUser.has(userId)) {
      this.limiters.perUser.set(userId, new TokenBucket(1000, 10)); // 1k capacity, 10/sec
    }
    
    if (!this.limiters.perUser.get(userId).consume(tokens)) {
      return { allowed: false, reason: 'user_limit' };
    }
    
    // Check per-IP limit
    if (!this.limiters.perIP.has(ip)) {
      this.limiters.perIP.set(ip, new TokenBucket(100, 1)); // 100 capacity, 1/sec
    }
    
    if (!this.limiters.perIP.get(ip).consume(tokens)) {
      return { allowed: false, reason: 'ip_limit' };
    }
    
    return { allowed: true };
  }
}
```

### Adaptive Rate Limiting
```javascript
class AdaptiveRateLimiter {
  constructor() {
    this.baseLimit = 100;
    this.currentLimit = 100;
    this.errorRate = 0;
    this.responseTime = 0;
  }
  
  updateMetrics(isError, responseTime) {
    // Update error rate (exponential moving average)
    this.errorRate = (this.errorRate * 0.9) + (isError ? 0.1 : 0);
    
    // Update response time
    this.responseTime = (this.responseTime * 0.9) + (responseTime * 0.1);
    
    // Adjust rate limit based on metrics
    if (this.errorRate > 0.05 || this.responseTime > 1000) {
      // High error rate or slow responses: decrease limit
      this.currentLimit = Math.max(10, this.currentLimit * 0.8);
    } else if (this.errorRate < 0.01 && this.responseTime < 500) {
      // Low error rate and fast responses: increase limit
      this.currentLimit = Math.min(this.baseLimit * 2, this.currentLimit * 1.1);
    }
  }
  
  getCurrentLimit() {
    return Math.floor(this.currentLimit);
  }
}
```

### Distributed Rate Limiting
```javascript
class DistributedRateLimiter {
  constructor(redis, nodeId) {
    this.redis = redis;
    this.nodeId = nodeId;
    this.localCounts = new Map();
  }
  
  async isAllowed(key, globalLimit, windowMs) {
    const now = Date.now();
    const window = Math.floor(now / windowMs);
    
    // Local rate limiting (approximation)
    const localKey = `${key}:${window}`;
    const localCount = this.localCounts.get(localKey) || 0;
    const localLimit = Math.floor(globalLimit / 4); // Assume 4 nodes
    
    if (localCount >= localLimit) {
      // Check global count in Redis
      const globalKey = `global:${key}:${window}`;
      const globalCount = await this.redis.incr(globalKey);
      await this.redis.expire(globalKey, Math.ceil(windowMs / 1000));
      
      if (globalCount > globalLimit) {
        return false;
      }
    }
    
    // Update local count
    this.localCounts.set(localKey, localCount + 1);
    
    // Cleanup old entries
    this.cleanupLocalCounts();
    
    return true;
  }
  
  cleanupLocalCounts() {
    const now = Date.now();
    for (const [key] of this.localCounts) {
      const parts = key.split(':');
      const window = parseInt(parts[parts.length - 1]);
      if (now > (window + 1) * 60000) { // Cleanup entries older than 1 minute
        this.localCounts.delete(key);
      }
    }
  }
}
```

## Response Headers

### Standard Rate Limit Headers
```javascript
app.use((req, res, next) => {
  // Standard headers
  res.set({
    'X-RateLimit-Limit': '100',
    'X-RateLimit-Remaining': '95',
    'X-RateLimit-Reset': '1640995200',
    'X-RateLimit-Reset-After': '3600'
  });
  
  // When limit exceeded
  if (rateLimited) {
    res.set({
      'Retry-After': '3600'
    });
    return res.status(429).json({
      error: 'Too Many Requests',
      message: 'Rate limit exceeded. Try again in 1 hour.'
    });
  }
  
  next();
});
```

### Draft Standard Headers
```javascript
// New draft standard (RFC 6585 update)
res.set({
  'RateLimit-Limit': '100',
  'RateLimit-Remaining': '95',
  'RateLimit-Reset': '1640995200'
});
```

## Testing Rate Limits

```javascript
describe('Rate Limiting', () => {
  test('should allow requests within limit', async () => {
    for (let i = 0; i < 100; i++) {
      const response = await request(app)
        .get('/api/data')
        .set('X-Forwarded-For', '192.168.1.1');
        
      expect(response.status).toBe(200);
    }
  });
  
  test('should block requests exceeding limit', async () => {
    // Make 100 requests (at limit)
    for (let i = 0; i < 100; i++) {
      await request(app).get('/api/data');
    }
    
    // 101st request should be blocked
    const response = await request(app).get('/api/data');
    expect(response.status).toBe(429);
    expect(response.body.error).toMatch(/rate limit/i);
  });
  
  test('should reset limit after window expires', async () => {
    // Fill up the limit
    for (let i = 0; i < 100; i++) {
      await request(app).get('/api/data');
    }
    
    // Wait for window to reset
    await new Promise(resolve => setTimeout(resolve, 61000));
    
    // Should allow requests again
    const response = await request(app).get('/api/data');
    expect(response.status).toBe(200);
  });
});
```

## Best Practices

### 1. Choose Right Algorithm
- **Fixed Window**: Simple, memory efficient
- **Sliding Window**: More accurate, higher memory usage
- **Token Bucket**: Allows bursts, good for APIs
- **Leaky Bucket**: Smooth rate, prevents bursts

### 2. Set Appropriate Limits
```javascript
const limits = {
  authentication: { requests: 5, window: '15m' },
  api_free: { requests: 100, window: '1h' },
  api_premium: { requests: 10000, window: '1h' },
  file_upload: { requests: 10, window: '1h' },
  password_reset: { requests: 3, window: '1h' }
};
```

### 3. Graceful Degradation
```javascript
app.use(async (req, res, next) => {
  const allowed = await checkRateLimit(req);
  
  if (!allowed.allowed) {
    // Instead of blocking, reduce response quality
    if (req.path.startsWith('/api/search')) {
      req.reducedResults = true; // Return fewer results
    } else if (req.path.startsWith('/api/data')) {
      req.cached = true; // Return cached data
    } else {
      // Block critical operations
      return res.status(429).json({ error: 'Rate limit exceeded' });
    }
  }
  
  next();
});
```

### 4. Monitor and Alert
```javascript
class RateLimitMonitor {
  constructor() {
    this.metrics = {
      blocked: 0,
      allowed: 0,
      lastReset: Date.now()
    };
  }
  
  record(allowed) {
    if (allowed) {
      this.metrics.allowed++;
    } else {
      this.metrics.blocked++;
    }
    
    // Alert if block rate is high
    const total = this.metrics.allowed + this.metrics.blocked;
    const blockRate = this.metrics.blocked / total;
    
    if (blockRate > 0.1) { // More than 10% blocked
      this.alert(`High rate limit block rate: ${(blockRate * 100).toFixed(1)}%`);
    }
  }
  
  alert(message) {
    console.error(`RATE LIMIT ALERT: ${message}`);
    // Send to monitoring system
  }
}
```

## Summary

**Rate limiting protects against:**
✅ DDoS attacks
✅ API abuse
✅ Resource exhaustion
✅ Unfair usage
✅ Cost overruns

**Key considerations:**
- Choose appropriate algorithm for use case
- Set limits based on capacity and user needs
- Provide clear error messages and retry guidance
- Monitor and adjust limits based on usage patterns
- Consider different limits for different user tiers
