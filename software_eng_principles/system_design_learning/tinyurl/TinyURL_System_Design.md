# TinyURL System Design

## Introduction

### What is a Short URL?
A short URL is a compressed version of a long URL that redirects users to the original destination. For example:
- **Long URL**: `https://www.example.com/products/electronics/smartphones/iphone-15-pro-max-256gb-natural-titanium?color=blue&storage=256gb&warranty=extended`
- **Short URL**: `https://tinyurl.com/abc1234`

### Why 7 Characters for Short URLs?

We chose **7 characters** for our short URLs using **Base62 encoding** (a-z, A-Z, 0-9) for the following reasons:

#### 1. Massive Capacity
```
Base62 character set: 62 characters (26 + 26 + 10)
7-character combinations: 62^7 = 3,521,614,606,208 URLs (~3.5 trillion)

Our requirement: 6 billion URLs over 5 years
Safety margin: 3.5 trillion ÷ 6 billion = 583x more capacity than needed
```

#### 2. URL Length Optimization
- **Shorter than 7**: Not enough capacity for our scale
  - 6 chars: 62^6 = 56.8 billion URLs (only 9.5x our needs - too risky)
- **Longer than 7**: Unnecessary overhead
  - 8 chars: 62^8 = 218 trillion URLs (62x more than needed)

#### 3. User Experience
- **Memorable**: 7 characters are easy to remember and type
- **Shareable**: Short enough for social media, SMS, and verbal communication
- **Professional**: Clean, consistent format like `tinyurl.com/abc1234`

#### 4. Technical Benefits
- **Fixed length**: Simplifies database indexing and caching
- **URL-safe**: Base62 avoids special characters that need encoding
- **Case-sensitive**: Doubles the character space (A ≠ a)

### Example Short URL Generation
```
Counter ID: 1,000,000
Base62 encoding: 1000000 → "4c92"
Padded to 7 chars: "aaa4c92"
Final URL: https://tinyurl.com/aaa4c92
```

This approach ensures we can handle massive scale while keeping URLs short, memorable, and user-friendly.

// ...existing content...

## Interview Preparation Guide

### Quick Reference Numbers
- **Scale**: 100M URLs/month, 100:1 read/write ratio
- **Write QPS**: 38 URLs/sec (peak: 115/sec)  
- **Read QPS**: 3,850/sec (peak: 11,550/sec)
- **Storage**: 3TB for 5 years (9TB with replication)
- **Cache**: 250MB per server for hot data
- **SLA**: 99.99% availability, <50ms latency (p99)

### Interview Flow (45-60 minutes)

#### 1. Requirements Clarification (5 minutes)
**Ask these questions:**
- What's the expected scale (URLs/day, requests/second)?
- Do we need custom aliases?
- Should URLs expire?
- Do we need analytics?
- What about user accounts and API access?

#### 2. Capacity Estimation (10 minutes)
Show your calculation process:
```
Traffic:
- 100M URLs/month ÷ 30 days ÷ 86400 seconds = 38.5 write QPS
- Peak traffic (3x): 115 write QPS
- Read QPS: 38.5 × 100 = 3,850 QPS (peak: 11,550)

Storage:
- 6B URLs over 5 years
- 500 bytes per URL = 3TB total
- With 3x replication = 9TB

Bandwidth:
- Incoming: 115 QPS × 500 bytes = 57.5 KB/sec  (data coming INTO servers from write requests)
- Outgoing: 11,550 QPS × 500 bytes = 5.78 MB/sec  (data going OUT from servers to read responses)

Note: Bandwidth = actual data transfer per second, different from traffic (requests/sec) and storage (disk space)
```

#### 3. High-Level Design (15 minutes)
Draw this architecture:
```
[Client] → [Load Balancer] → [API Gateway] → [App Servers]
                                          → [Redis Cache]
                                          → [PostgreSQL]
                                          → [Kafka Analytics]
```

**Key components to mention:**
- Load balancer for traffic distribution
- API gateway for rate limiting and auth
- Application servers for business logic
- Redis for caching hot URLs
- PostgreSQL for persistence with read replicas
- Kafka for analytics processing

#### 4. Database Design (8 minutes)
```sql
-- Show this schema
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_url VARCHAR(7) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    click_count BIGINT DEFAULT 0
);

-- Key indexes
CREATE INDEX idx_short_url ON urls(short_url);
CREATE INDEX idx_user_id ON urls(user_id);
```

**Discuss scaling:**
- **Sharding by hash(short_url)**: Distribute data across multiple PostgreSQL servers using hash of short_url as sharding key. This ensures even distribution of data and eliminates hotspots. Example: hash("abc1234") % 8 determines which of 8 database shards stores this URL.

- **Master-slave replication**: Each shard has one master node (handles writes) and multiple slave nodes (replicate data from master). Slaves provide fault tolerance and can serve read queries to reduce load on master.

- **Read replicas for scaling**: Additional read-only database copies specifically for handling the 100:1 read-heavy traffic pattern. Read requests can be load-balanced across multiple replicas while writes go to master only.

#### 5. URL Encoding Algorithm (7 minutes)
**Explain Base62 encoding:**
```python
def encode_base62(num):
    # Base62 characters: a-z (26) + A-Z (26) + 0-9 (10) = 62 total characters
    chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    
    # Handle edge case: if input is 0, return first character
    if num == 0:
        return chars[0]  # Returns 'a'
    
    result = []
    
    # Convert decimal number to base62 (like converting to binary, but base 62)
    while num > 0:
        # Get remainder when divided by 62 (this gives us the rightmost digit in base62)
        remainder = num % 62
        # Add the corresponding character to our result
        result.append(chars[remainder])
        # Integer division by 62 (move to next digit position)
        num //= 62
    
    # Reverse because we built the number backwards (least significant digit first)
    # Then pad to 7 characters with 'a' (our zero character)
    return ''.join(reversed(result)).rjust(7, 'a')

# Let's trace through a real example:
# Long URL: "https://www.amazon.com/dp/B08N5WRWNW/ref=sr_1_1?keywords=echo+dot"

# Step 1: Get unique counter ID for this URL
counter_id = 1000000  # This is our 1,000,000th URL

# HOW DOES THE COUNTER SERVICE WORK?
# The counter service ensures every URL gets a unique sequential ID:

class CounterService:
    def __init__(self, server_id, total_servers):
        # Each server gets a unique starting point to avoid collisions
        self.server_id = server_id          # e.g., Server 1, 2, 3...
        self.total_servers = total_servers  # e.g., 10 servers total
        self.current_counter = server_id    # Server 1 starts at 1, Server 2 at 2, etc.
    
    def get_next_id(self):
        """Generate next unique ID for this server"""
        current_id = self.current_counter
        # Jump by total_servers to ensure no overlap between servers
        self.current_counter += self.total_servers
        return current_id

# EXAMPLE WITH 3 SERVERS:
# Server 1: generates IDs 1, 4, 7, 10, 13, 16, 19...
# Server 2: generates IDs 2, 5, 8, 11, 14, 17, 20...  
# Server 3: generates IDs 3, 6, 9, 12, 15, 18, 21...

# So when we process the 1,000,000th URL request overall:
# - It might be the 333,334th request on Server 1
# - Server 1 would generate ID: 1 + (333,333 * 3) = 1,000,000
# - This ID is guaranteed unique across all servers

# TIMELINE EXAMPLE:
# Request 1: User submits "https://google.com" → Counter ID: 1 → Short URL: "aaaaaab"
# Request 2: User submits "https://facebook.com" → Counter ID: 2 → Short URL: "aaaaaac" 
# Request 3: User submits "https://twitter.com" → Counter ID: 3 → Short URL: "aaaaaad"
# ...
# Request 1,000,000: User submits Amazon URL → Counter ID: 1,000,000 → Short URL: "aaaemjc"

# Step 2: Convert 1000000 to base62
# 1000000 ÷ 62 = 16129, remainder = 2  → chars[2] = 'c'
# 16129 ÷ 62 = 260, remainder = 9      → chars[9] = 'j' 
# 260 ÷ 62 = 4, remainder = 12         → chars[12] = 'm'
# 4 ÷ 62 = 0, remainder = 4           → chars[4] = 'e'

# Step 3: Build result array (backwards): ['c', 'j', 'm', 'e']
# Step 4: Reverse it: ['e', 'm', 'j', 'c'] → "emjc"
# Step 5: Pad to 7 chars: "aaaemjc"

# Final result: https://tinyurl.com/aaaemjc

def create_short_url_example():
    """Complete example of URL shortening process"""
    
    # Original long URL
    long_url = "https://www.amazon.com/dp/B08N5WRWNW/ref=sr_1_1?keywords=echo+dot"
    print(f"Long URL: {long_url}")
    print(f"Length: {len(long_url)} characters")
    
    # Get next counter ID (this would come from our distributed counter service)
    counter_id = 1000000
    print(f"\nStep 1: Got counter ID: {counter_id}")
    
    # Encode to base62
    short_code = encode_base62(counter_id)
    print(f"Step 2: Encoded to base62: {short_code}")
    
    # Create final short URL
    short_url = f"https://tinyurl.com/{short_code}"
    print(f"Step 3: Final short URL: {short_url}")
    print(f"Length: {len(short_url)} characters")
    
    # Show the compression ratio
    compression_ratio = len(long_url) / len(short_url)
    print(f"\nCompression: {len(long_url)} → {len(short_url)} chars")
    print(f"Ratio: {compression_ratio:.1f}x smaller")
    
    return short_code

# Example output:
# Long URL: https://www.amazon.com/dp/B08N5WRWNW/ref=sr_1_1?keywords=echo+dot
# Length: 70 characters
# 
# Step 1: Got counter ID: 1000000
# Step 2: Encoded to base62: aaaemjc  
# Step 3: Final short URL: https://tinyurl.com/aaaemjc
# Length: 32 characters
#
# Compression: 70 → 32 chars
# Ratio: 2.2x smaller
```

**Key points:**
- **62^7 = 3.5 trillion possible URLs** (massive capacity)
- **Counter-based ensures uniqueness** (no collisions, unlike hashing)
- **Always 7 characters** (predictable length for database optimization)
- **⚠️ IMPORTANT: Same long URL can have multiple short URLs** (each request gets a new counter ID)
- **Alternative: hash-based with collision handling** (trade-off: unpredictable collisions vs coordination overhead)

**Counter-Based Behavior: Same Long URL → Multiple Short URLs**

```python
# EXAMPLE: Multiple users shorten the same Amazon URL

# User A submits the Amazon URL first:
long_url = "https://www.amazon.com/dp/B08N5WRWNW/ref=sr_1_1?keywords=echo+dot"
counter_id_1 = 1000000  # Gets counter ID 1,000,000
short_url_1 = "aaaemjc"  # Results in https://tinyurl.com/aaaemjc

# User B submits THE EXACT SAME Amazon URL later:  
long_url = "https://www.amazon.com/dp/B08N5WRWNW/ref=sr_1_1?keywords=echo+dot"
counter_id_2 = 1000001  # Gets NEXT counter ID 1,000,001
short_url_2 = "aaaemjd"  # Results in https://tinyurl.com/aaaemjd

# RESULT: Two different short URLs for the same long URL!
# https://tinyurl.com/aaaemjc → Amazon URL
# https://tinyurl.com/aaaemjd → Amazon URL (SAME destination)
```

**Database State After Both Requests:**
```sql
SELECT * FROM urls WHERE long_url LIKE '%amazon%';

| short_url | long_url                                              | user_id |
|-----------|------------------------------------------------------|---------|
| aaaemjc   | https://www.amazon.com/dp/B08N5WRWNW/ref=sr_1_1?... | 123     |
| aaaemjd   | https://www.amazon.com/dp/B08N5WRWNW/ref=sr_1_1?... | 456     |
```

**Why This Happens:**
- **Counter-based approach treats each request independently**
- **No deduplication** - every request gets a fresh counter ID
- **Stateless from long URL perspective** - doesn't check if URL was shortened before

**Pros and Cons of This Behavior:**

```
PROS:
✓ Simpler implementation (no lookup needed)
✓ Better performance (no deduplication check)
✓ User-specific tracking (different users get different short URLs)
✓ No cache pollution from lookup queries
✓ Independent analytics per short URL

CONS:
✗ Storage inefficiency (duplicate long URLs)
✗ Multiple short URLs for same content
✗ Analytics fragmentation across multiple short URLs
✗ No URL reuse benefits
```

**Why Counter-Based vs Hash-Based?**

```python
# Counter-based approach (CHOSEN):
def counter_based_encoding():
    """
    Pros:
    ✓ Guaranteed unique (no collisions)
    ✓ Predictable performance 
    ✓ Sequential generation
    ✓ Shorter URLs on average
    ✓ Simple implementation
    
    Cons:
    ✗ Requires coordination between servers
    ✗ Single point of failure for counter
    ✗ Same long URL gets multiple short URLs (no deduplication)
    ✗ Storage inefficiency for duplicate URLs
    """
    pass

# Hash-based approach (ALTERNATIVE):
def hash_based_encoding(long_url):
    """
    Pros:
    ✓ No coordination needed
    ✓ Same URL always gets same short code (DEDUPLICATION!)
    ✓ Stateless generation
    ✓ Storage efficient (no duplicates)
    
    Cons:
    ✗ Collision handling required
    ✗ Unpredictable performance
    ✗ Potentially longer codes after collision resolution
    ✗ Complex collision detection and resolution
    """
    import hashlib
    hash_value = hashlib.md5(long_url.encode()).hexdigest()
    return hash_value[:7]  # Take first 7 characters

# HASH-BASED EXAMPLE: Same Long URL → Same Short URL
# User A: "https://amazon.com/..." → hash → "a1b2c3d" 
# User B: "https://amazon.com/..." → hash → "a1b2c3d" (SAME!)
# Result: Only ONE database record, both users get same short URL
```

**Real-World Design Decision:**

Most production URL shorteners use **counter-based approach** despite the duplication because:

1. **Performance is critical** - No lookup overhead for deduplication
2. **User isolation** - Each user gets their own short URL for tracking
3. **Simplicity** - Easier to implement and maintain
4. **Analytics granularity** - Can track performance per user/campaign
5. **Storage is cheap** - The cost of duplicate storage is acceptable

**When Hash-Based Might Be Better:**
- **High storage costs** - When duplicate storage is expensive
- **Content deduplication required** - When same content should have same URL
- **URL permanence needed** - When URL should never change for same content
- **Low coordination overhead acceptable** - When performance isn't critical

#### 6. Caching Strategy (5 minutes)
**Multi-level approach:**
- L1: Application cache (100MB)
- L2: Redis distributed cache (10GB)
- L3: Database read replicas

**Policies:**
- Write-through caching
- LRU eviction
- 1-hour TTL for hot URLs

### Common Interview Questions

#### Q1: "How would you handle 10x more traffic?"
**Answer approach:**
1. Horizontal scaling of app servers
2. Database sharding (increase shards from 64 to 640)
3. More Redis cache nodes
4. CDN for popular redirects
5. Read replica scaling

#### Q2: "What if the counter service becomes a bottleneck?"
**Solutions:**

**1. Range-Based Counters (Recommended)**
```python
# Each server pre-allocates a range of IDs
class RangeBasedCounter:
    def __init__(self, server_id, range_size=100000):
        self.server_id = server_id
        self.range_size = range_size
        self.current_range_start = None
        self.current_counter = None
        self.range_end = None
    
    def get_next_id(self):
        # If we've used up current range, get a new range
        if self.current_counter is None or self.current_counter >= self.range_end:
            self.allocate_new_range()
        
        current_id = self.current_counter
        self.current_counter += 1
        return current_id
    
    def allocate_new_range(self):
        # Get next available range from central coordinator (Redis/ZooKeeper)
        self.current_range_start = redis.incr("global_range_counter") * self.range_size
        self.current_counter = self.current_range_start
        self.range_end = self.current_range_start + self.range_size

# EXAMPLE:
# Server 1 gets range: 0 - 99,999
# Server 2 gets range: 100,000 - 199,999  
# Server 3 gets range: 200,000 - 299,999
# When Server 1 uses up its range, it gets: 300,000 - 399,999
```

**2. Distributed Coordination with ZooKeeper**
```python
# Use ZooKeeper for distributed counter coordination
import kazoo

class DistributedCounter:
    def __init__(self, zk_hosts):
        self.zk = kazoo.client.KazooClient(hosts=zk_hosts)
        self.zk.start()
        self.counter_path = "/tinyurl/counter"
        
        # Ensure counter node exists
        self.zk.ensure_path(self.counter_path)
        
    def get_next_id(self):
        # Atomic increment in ZooKeeper
        current_value, stat = self.zk.get(self.counter_path)
        next_value = int(current_value) + 1
        
        # Atomic compare-and-set
        try:
            self.zk.set(self.counter_path, str(next_value), stat.version)
            return next_value
        except kazoo.exceptions.BadVersionError:
            # Someone else updated it, retry
            return self.get_next_id()
```

**3. Pre-allocation Strategy**
- Reserve ID blocks in advance during low-traffic periods
- Each server maintains a buffer of unused IDs
- Reduces coordination overhead during peak traffic

**4. Multiple Counter Services with Round-Robin**
- Deploy counter service replicas
- Route requests in round-robin fashion
- Each service handles different ID ranges

#### Q3: "SQL vs NoSQL for this system?"
**Analysis:**
```
PostgreSQL (Chosen):
✓ ACID properties for critical data
✓ Complex queries for analytics  
✓ Strong consistency
✓ Mature ecosystem

DynamoDB/Cassandra:
✓ Better horizontal scaling
✓ Higher write throughput
✗ Complex queries difficult
✗ Eventual consistency challenges
```

#### Q4: "How do you ensure high availability?"
**Strategies:**
1. Multi-region deployment
2. Circuit breakers
3. Graceful degradation (cache-only mode)
4. Health checks and auto-scaling
5. Database failover automation

#### Q5: "How to handle malicious URLs?"
**Security measures:**
1. URL validation on input
2. Domain blacklisting
3. Integration with security APIs (VirusTotal)
4. Rate limiting by IP/user
5. User reporting system

### Code Implementation Examples

#### Flask API Server
```python
from flask import Flask, request, jsonify, redirect
import time
import asyncio

app = Flask(__name__)

@app.route('/api/v1/shorten', methods=['POST'])
def create_short_url():
    # Rate limiting check
    if not rate_limiter.check(request.remote_addr):
        return jsonify({'error': 'Rate limit exceeded'}), 429
    
    # Validate and parse request
    data = request.get_json()
    long_url = data.get('long_url')
    
    if not validate_url(long_url):
        return jsonify({'error': 'Invalid URL'}), 400
    
    # Generate short URL
    short_url = url_service.create_short_url(long_url)
    
    return jsonify({
        'short_url': f'https://tinyurl.com/{short_url}',
        'long_url': long_url
    }), 201

@app.route('/<short_url>')
def redirect_to_long_url(short_url):
    # Get from cache first, then database
    long_url = cache.get(short_url) or database.get_long_url(short_url)
    
    if not long_url:
        return jsonify({'error': 'URL not found'}), 404
    
    # Track analytics asynchronously
    analytics.track_click(short_url, request.remote_addr, 
                         request.headers.get('User-Agent'))
    
    return redirect(long_url, code=302)
```

#### URL Encoding Service
```python
class URLEncoder:
    BASE62 = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    
    def __init__(self, counter_service):
        self.counter = counter_service
    
    def encode(self, url_id):
        if url_id == 0:
            return self.BASE62[0]
        
        result = []
        while url_id > 0:
            result.append(self.BASE62[url_id % 62])
            url_id //= 62
        
        return ''.join(reversed(result)).rjust(7, self.BASE62[0])
    
    def create_short_url(self, long_url):
        url_id = self.counter.get_next_id()
        short_url = self.encode(url_id)
        
        # Store in database
        database.save_url(short_url, long_url)
        
        # Cache immediately
        cache.set(short_url, long_url)
        
        return short_url
```

#### Caching Layer
```python
class CacheService:
    def __init__(self, redis_cluster):
        self.redis = redis_cluster
        self.local_cache = {}
        self.max_local_size = 10000
    
    def get(self, short_url):
        # L1: Local cache
        if short_url in self.local_cache:
            return self.local_cache[short_url]
        
        # L2: Redis
        long_url = self.redis.get(f"url:{short_url}")
        if long_url:
            self.set_local(short_url, long_url)
            return long_url
        
        return None
    
    def set(self, short_url, long_url):
        # Set in both levels
        self.redis.setex(f"url:{short_url}", 3600, long_url)
        self.set_local(short_url, long_url)
    
    def set_local(self, short_url, long_url):
        if len(self.local_cache) >= self.max_local_size:
            # Simple LRU: remove oldest
            oldest = next(iter(self.local_cache))
            del self.local_cache[oldest]
        
        self.local_cache[short_url] = long_url
```

### Advanced Topics to Discuss

#### Geographic Distribution
```
Global Architecture:
US West    US East    Europe    Asia
   |          |         |        |
   └──────────┼─────────┼────────┘
              |         |
          DNS Load Balancing
              |
        Route to closest region
```

#### Analytics Architecture
```
Click Event → Kafka → Stream Processing → Analytics DB
           ↘         ↗
            Redis (real-time counters)
```

#### Database Sharding Strategy
```python
def get_shard_id(short_url, total_shards=64):
    return hash(short_url) % total_shards

def get_database_connection(short_url):
    shard_id = get_shard_id(short_url)
    return database_connections[shard_id]
```

### Key Design Principles

1. **Scalability**: Horizontal scaling at every layer
2. **Availability**: Redundancy and failover mechanisms  
3. **Consistency**: Eventually consistent for analytics, strong for URL mappings
4. **Performance**: Multi-level caching and optimized queries
5. **Security**: Rate limiting, validation, monitoring
6. **Observability**: Comprehensive metrics and logging

### Monitoring & Alerting

```python
# Key metrics to track
metrics = {
    'requests_per_second': ['shorten', 'redirect'],
    'response_time_p99': ['shorten', 'redirect'], 
    'error_rate': ['4xx', '5xx'],
    'cache_hit_ratio': ['redis', 'local'],
    'database_connections': ['active', 'idle'],
    'queue_depth': ['analytics_events']
}

# Critical alerts
alerts = {
    'high_error_rate': 'error_rate > 1% for 2 minutes',
    'high_latency': 'response_time_p99 > 100ms for 5 minutes',
    'low_cache_hit_rate': 'cache_hit_ratio < 80% for 10 minutes',
    'database_down': 'database_connections = 0 for 1 minute'
}
```

### Cost Optimization

1. **Tiered Storage**: Hot/warm/cold data separation
2. **Reserved Instances**: For predictable baseline load
3. **Compression**: Analytics data and logs
4. **Data Lifecycle**: Automatic deletion of expired URLs
5. **Regional Optimization**: Data locality reduces costs

This comprehensive system design addresses all requirements and provides a solid foundation for handling the specified scale while maintaining high availability and low latency.
