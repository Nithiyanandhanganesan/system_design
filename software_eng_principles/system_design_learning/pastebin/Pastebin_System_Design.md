# Pastebin System Design

## Introduction

### What is Pastebin?
Pastebin is a web application that allows users to store and share text online. Think of it as a "temporary clipboard" on the internet where you can:
- **Paste text**: Code snippets, logs, configuration files, notes
- **Share instantly**: Get a unique URL to share with others
- **Set expiration**: Control how long the paste stays available
- **Syntax highlighting**: Format code with language-specific coloring
- **Privacy controls**: Public, unlisted, or private pastes

### Key Differences from TinyURL
| Aspect | TinyURL | Pastebin |
|--------|---------|----------|
| **Primary Data** | URL mappings | Text content |
| **Storage Size** | Small (500 bytes) | Variable (1KB to 10MB) |
| **Read Pattern** | Redirect (302) | Display content |
| **Caching Strategy** | Cache mappings | Cache popular pastes |
| **Content Type** | URLs only | Rich text, code, logs |

## 1. Requirements Analysis

### 1.1 Functional Requirements

#### Core Features
1. **Create Paste**: Users can paste text and get a unique URL
2. **Read Paste**: Access paste content via the unique URL  
3. **Expiration Support**: Default expiration (never, 10min, 1hour, 1day, 1week, 1month)
4. **Privacy Levels**:
   - **Public**: Listed in recent pastes, indexed by search engines
   - **Unlisted**: Accessible via URL but not publicly listed
   - **Private**: Only accessible by creator (requires login)
5. **Syntax Highlighting**: Support 50+ programming languages
6. **Raw Text View**: Plain text access for API consumers
7. **Clone/Fork**: Create a copy of existing paste

#### Advanced Features (Optional)
- **User Accounts**: Register, login, manage pastes
- **Custom URLs**: Let users set custom paste IDs
- **Edit History**: Track paste modifications
- **Comments**: Allow comments on public pastes
- **Search**: Full-text search across public pastes
- **API Access**: RESTful API for programmatic access

### 1.2 Non-Functional Requirements

#### Performance
- **Read Latency**: < 100ms (p99) for paste retrieval
- **Write Latency**: < 200ms (p99) for paste creation
- **Availability**: 99.9% uptime (8.76 hours downtime/year)
- **Consistency**: Eventually consistent (acceptable for this use case)

#### Scalability
- **Traffic**: 5M pastes/day (58 writes/sec, peak: 175/sec)
- **Read:Write Ratio**: 10:1 (moderate read-heavy)
- **Paste Size**: 1KB average, 10MB maximum
- **Retention**: 80% of pastes expire within 1 day

#### Security
- **Rate Limiting**: Prevent spam (max 20 pastes/hour per IP)
- **Content Filtering**: Block malicious content, spam
- **Data Privacy**: Encrypt private pastes at rest
- **DDoS Protection**: Handle traffic spikes and attacks

## 2. Capacity Planning

### 2.1 Traffic Estimation

```bash
Write Traffic:
- 5M pastes/day
- Daily: 5,000,000 / 86,400 = 58 writes/sec
- Peak (3x): 175 writes/sec

Read Traffic:
- Read:Write = 10:1
- Daily: 58 × 10 = 580 reads/sec  
- Peak: 175 × 10 = 1,750 reads/sec
```

### 2.2 Storage Estimation

```bash
Storage per paste:
- Content: 1KB average (up to 10MB max)
- Metadata: 200 bytes (ID, timestamp, expiration, language, etc.)
- Total per paste: ~1.2KB average

Daily storage:
- New pastes: 5M × 1.2KB = 6GB/day
- Annual: 6GB × 365 = 2.19TB/year

5-year storage (considering expiration):
- Active pastes: ~70% expire within 1 month
- Long-term storage: 2.19TB × 5 × 0.3 = 3.3TB
- With replication (3x): ~10TB total
```

### 2.3 Bandwidth Estimation

```bash
Incoming (Write):
- Peak: 175 requests/sec × 1.2KB = 210KB/sec
- Average: 58 requests/sec × 1.2KB = 70KB/sec

Outgoing (Read):
- Peak: 1,750 requests/sec × 1.2KB = 2.1MB/sec  
- Average: 580 requests/sec × 1.2KB = 696KB/sec

Note: Bandwidth = data transferred per second
- Incoming = data coming into your servers (writes)
- Outgoing = data going out from your servers (reads)
- This doesn't include overhead like TCP headers, HTTP headers, etc.
```

### 2.4 Memory Requirements (Caching)

```bash
Cache Strategy:
- Cache 20% hottest pastes (by access frequency)
- Cache size per server: 1.2KB × 100,000 pastes = 120MB
- Cache hit ratio target: 80%
- TTL: 24 hours (refresh based on access patterns)
```

## 3. High-Level System Architecture

```
                                    ┌─────────────────┐
                                    │   CDN / Cloudflare │
                                    └─────────┬───────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
         ┌──────────▼──────────┐   ┌─────────▼──────────┐   ┌─────────▼──────────┐
         │   Load Balancer     │   │   Load Balancer    │   │   Load Balancer    │
         │    (Web Tier)       │   │   (API Tier)       │   │   (Static Assets)  │
         └──────────┬──────────┘   └─────────┬──────────┘   └─────────┬──────────┘
                    │                        │                        │
         ┌──────────▼──────────┐   ┌─────────▼──────────┐   ┌─────────▼──────────┐
         │   Web Servers       │   │   API Gateway      │   │   Static Files     │
         │   (Frontend)        │   │                    │   │   (CSS, JS, etc.)  │
         └─────────────────────┘   └─────────┬──────────┘   └────────────────────┘
                                             │
                              ┌──────────────▼──────────────┐
                              │     Application Servers     │
                              │     (Business Logic)        │
                              └──────────────┬──────────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
         ┌──────────▼──────────┐   ┌─────────▼──────────┐   ┌─────────▼──────────┐
         │   Redis Cluster     │   │   PostgreSQL       │   │   Object Storage   │
         │     (Cache)         │   │   (Metadata)       │   │   (Large Files)    │
         └─────────────────────┘   └─────────┬──────────┘   └────────────────────┘
                                             │
                                   ┌─────────▼──────────┐
                                   │   Read Replicas    │
                                   │   (Scale Reads)    │
                                   └────────────────────┘
```

## 4. Detailed Component Design

### 4.1 Database Schema

#### PostgreSQL Tables

```sql
-- Main pastes table
CREATE TABLE pastes (
    id VARCHAR(8) PRIMARY KEY,                    -- e.g., "a7b9c2d1"
    content TEXT,                                 -- For small pastes < 1MB
    content_url VARCHAR(255),                     -- S3 URL for large files
    title VARCHAR(100),                          -- Optional paste title
    language VARCHAR(20) DEFAULT 'text',         -- Programming language
    privacy_level SMALLINT DEFAULT 0,            -- 0=public, 1=unlisted, 2=private
    user_id INTEGER REFERENCES users(id),        -- NULL for anonymous
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,                        -- NULL = never expires
    view_count INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,              -- Soft delete flag
    content_hash VARCHAR(64),                    -- SHA256 for duplicate detection
    ip_address INET,                             -- For rate limiting
    user_agent VARCHAR(255)                      -- For analytics
);

-- Users table (for registered users)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(30) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE,
    paste_count INTEGER DEFAULT 0,
    last_login TIMESTAMP
);

-- Analytics table
CREATE TABLE paste_analytics (
    id SERIAL PRIMARY KEY,
    paste_id VARCHAR(8) REFERENCES pastes(id),
    accessed_at TIMESTAMP DEFAULT NOW(),
    ip_address INET,
    user_agent VARCHAR(255),
    referer VARCHAR(255)
);

-- Indexes for performance
CREATE INDEX idx_pastes_created_at ON pastes(created_at DESC);
CREATE INDEX idx_pastes_expires_at ON pastes(expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_pastes_user_id ON pastes(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_pastes_hash ON pastes(content_hash);
CREATE UNIQUE INDEX idx_pastes_active ON pastes(id) WHERE is_active = TRUE;
```

### 4.2 ID Generation Strategy

We'll use a **similar approach to TinyURL** but optimized for Pastebin:

```python
class PasteIDGenerator:
    """
    Generate unique 8-character paste IDs using Base62 encoding
    
    Why 8 characters?
    - Base62^8 = 218 trillion possible combinations
    - For 5M pastes/day × 365 days × 5 years = 9.1B pastes
    - Safety margin: 218T ÷ 9.1B = 24,000x more than needed
    """
    
    BASE62_CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    
    def __init__(self):
        self.counter_service = CounterService()  # Redis-based counter
    
    def generate_id(self) -> str:
        """
        Generate unique paste ID using counter-based approach
        
        Example:
        1. Get counter: 15,234,567
        2. Convert to Base62: "a7b9c2d1"  
        3. Return ID: "a7b9c2d1"
        """
        # Get unique counter from Redis
        counter = self.counter_service.get_next_counter()
        
        # Convert counter to Base62
        paste_id = self._to_base62(counter)
        
        # Pad to 8 characters if needed
        return paste_id.rjust(8, 'a')
    
    def _to_base62(self, num: int) -> str:
        """Convert integer to Base62 string"""
        if num == 0:
            return self.BASE62_CHARS[0]
        
        result = ""
        while num > 0:
            result = self.BASE62_CHARS[num % 62] + result
            num //= 62
        return result

# Example usage:
# Counter: 1000000 → Base62: "4c92" → Padded: "aaaa4c92"
# Counter: 5000000 → Base62: "1c8Q8" → Padded: "aaa1c8Q8"
```

### 4.3 Application Service Layer

```python
from typing import Optional, Dict
from datetime import datetime, timedelta

class PasteService:
    def __init__(self):
        self.id_generator = PasteIDGenerator()
        self.db = DatabaseService()
        self.cache = RedisService()
        self.object_store = S3Service()
        self.rate_limiter = RateLimiter()
    
    def create_paste(self, content: str, **options) -> Dict:
        """
        Create a new paste
        
        Args:
            content: Paste content (1KB to 10MB)
            **options: title, language, privacy_level, expires_in, user_id
        
        Returns:
            {"paste_id": "abc12345", "url": "https://pastebin.com/abc12345"}
        """
        # Rate limiting check
        if not self.rate_limiter.is_allowed(request.ip):
            raise RateLimitExceeded("Too many pastes created")
        
        # Generate unique ID
        paste_id = self.id_generator.generate_id()
        
        # Handle large content (> 1MB) - store in object storage
        content_url = None
        if len(content.encode('utf-8')) > 1024 * 1024:  # 1MB threshold
            content_url = self.object_store.upload(
                key=f"pastes/{paste_id}",
                content=content.encode('utf-8')
            )
            content = None  # Don't store in DB if using object storage
        
        # Calculate expiration
        expires_at = None
        if expires_in := options.get('expires_in'):
            expires_at = datetime.utcnow() + timedelta(seconds=expires_in)
        
        # Create paste record
        paste_data = {
            'id': paste_id,
            'content': content,
            'content_url': content_url,
            'title': options.get('title', ''),
            'language': options.get('language', 'text'),
            'privacy_level': options.get('privacy_level', 0),
            'user_id': options.get('user_id'),
            'expires_at': expires_at,
            'content_hash': hashlib.sha256(content.encode()).hexdigest(),
            'ip_address': request.ip,
            'user_agent': request.user_agent
        }
        
        # Save to database
        self.db.create_paste(paste_data)
        
        # Cache hot paste for faster access
        if options.get('privacy_level', 0) == 0:  # Only cache public pastes
            self.cache.set(f"paste:{paste_id}", paste_data, ttl=3600)
        
        return {
            'paste_id': paste_id,
            'url': f"https://pastebin.com/{paste_id}",
            'expires_at': expires_at.isoformat() if expires_at else None
        }
    
    def get_paste(self, paste_id: str, user_id: Optional[int] = None) -> Dict:
        """
        Retrieve paste by ID
        
        Args:
            paste_id: The paste identifier
            user_id: Current user ID (for private paste access)
        
        Returns:
            Paste data or raises PasteNotFound
        """
        # Check cache first
        cached_paste = self.cache.get(f"paste:{paste_id}")
        if cached_paste:
            return self._format_paste_response(cached_paste)
        
        # Get from database
        paste = self.db.get_paste(paste_id)
        if not paste:
            raise PasteNotFound("Paste not found")
        
        # Check expiration
        if paste.expires_at and paste.expires_at < datetime.utcnow():
            raise PasteExpired("Paste has expired")
        
        # Check privacy permissions
        if paste.privacy_level == 2 and paste.user_id != user_id:
            raise PasteAccessDenied("Private paste - access denied")
        
        # Get content from object storage if needed
        if paste.content_url:
            paste.content = self.object_store.download(paste.content_url)
        
        # Update view count asynchronously
        self._increment_view_count(paste_id)
        
        # Cache public pastes
        if paste.privacy_level == 0:
            self.cache.set(f"paste:{paste_id}", paste, ttl=3600)
        
        return self._format_paste_response(paste)
    
    def _increment_view_count(self, paste_id: str):
        """Increment view count asynchronously"""
        # This could be done via message queue for better performance
        self.db.increment_view_count(paste_id)
        
        # Log analytics data
        analytics_data = {
            'paste_id': paste_id,
            'ip_address': request.ip,
            'user_agent': request.user_agent,
            'referer': request.referer
        }
        self.db.log_paste_view(analytics_data)
```

### 4.4 Caching Strategy

```python
class CacheService:
    """
    Multi-level caching strategy for optimal performance
    """
    
    def __init__(self):
        self.redis = RedisCluster()
        self.local_cache = LRUCache(max_size=1000)  # In-memory cache
    
    def get_paste(self, paste_id: str) -> Optional[Dict]:
        """
        L1: Local in-memory cache (fastest, ~1ms)
        L2: Redis cluster cache (~5ms)  
        L3: Database fallback (~20-50ms)
        """
        # L1: Check local cache first
        if paste := self.local_cache.get(paste_id):
            return paste
        
        # L2: Check Redis cluster
        if paste := self.redis.get(f"paste:{paste_id}"):
            # Populate L1 cache for next request
            self.local_cache.set(paste_id, paste, ttl=300)  # 5 min TTL
            return paste
        
        return None  # Cache miss - will hit database
    
    def set_paste(self, paste_id: str, paste_data: Dict, ttl: int = 3600):
        """Store in both cache levels"""
        # Store in Redis with longer TTL
        self.redis.setex(f"paste:{paste_id}", ttl, paste_data)
        
        # Store in local cache with shorter TTL
        self.local_cache.set(paste_id, paste_data, ttl=min(ttl, 300))
    
    def invalidate_paste(self, paste_id: str):
        """Remove from all cache levels"""
        self.local_cache.delete(paste_id)
        self.redis.delete(f"paste:{paste_id}")
```

### 4.5 Data Partitioning (Sharding)

For horizontal scaling, we'll use **hash-based sharding**:

```python
class ShardingService:
    """
    Shard pastes across multiple PostgreSQL databases
    
    Sharding Strategy: Hash-based on paste_id
    - Ensures even distribution
    - Simple routing logic  
    - No hotspots
    """
    
    def __init__(self, num_shards: int = 8):
        self.num_shards = num_shards
        self.db_connections = {
            i: PostgreSQLConnection(f"paste_db_shard_{i}")
            for i in range(num_shards)
        }
    
    def get_shard(self, paste_id: str) -> int:
        """Determine which shard contains this paste"""
        return hash(paste_id) % self.num_shards
    
    def get_connection(self, paste_id: str):
        """Get database connection for this paste"""
        shard_id = self.get_shard(paste_id)
        return self.db_connections[shard_id]
    
    def create_paste(self, paste_data: Dict):
        """Create paste in appropriate shard"""
        paste_id = paste_data['id']
        conn = self.get_connection(paste_id)
        return conn.create_paste(paste_data)
    
    def get_paste(self, paste_id: str):
        """Get paste from appropriate shard"""
        conn = self.get_connection(paste_id)
        return conn.get_paste(paste_id)
```

**Additional Database Optimizations:**

1. **Master-Slave Replication**: Each shard has 1 master + 2 read replicas
2. **Read Replicas for Scaling**: Route read queries to replicas  
3. **Connection Pooling**: Use PgBouncer for connection management
4. **Partition by Time**: Archive old expired pastes to separate tables

```sql
-- Example: Partition pastes table by creation month
CREATE TABLE pastes_2026_02 PARTITION OF pastes
FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Archive expired pastes
CREATE TABLE archived_pastes (LIKE pastes INCLUDING ALL);
```

## 5. API Design

### 5.1 REST API Endpoints

```python
# POST /api/pastes - Create new paste
{
    "content": "print('Hello World')",
    "title": "My Python Script",
    "language": "python", 
    "privacy_level": 0,     # 0=public, 1=unlisted, 2=private
    "expires_in": 3600      # seconds (null = never expires)
}

# Response:
{
    "paste_id": "a7b9c2d1",
    "url": "https://pastebin.com/a7b9c2d1",
    "created_at": "2026-02-15T10:30:00Z",
    "expires_at": "2026-02-15T11:30:00Z"
}

# GET /api/pastes/{paste_id} - Get paste
# Response:
{
    "paste_id": "a7b9c2d1",
    "content": "print('Hello World')",
    "title": "My Python Script", 
    "language": "python",
    "privacy_level": 0,
    "created_at": "2026-02-15T10:30:00Z",
    "expires_at": "2026-02-15T11:30:00Z",
    "view_count": 42
}

# GET /api/pastes/{paste_id}/raw - Get raw content
# Returns: plain text content with proper Content-Type

# GET /api/user/pastes - Get user's pastes (requires auth)
# DELETE /api/pastes/{paste_id} - Delete paste (owner only)
# PUT /api/pastes/{paste_id} - Update paste (owner only)
```

### 5.2 Web Interface Routes

```python
# Main routes
GET  /                          # Homepage with create form
POST /                          # Create paste (form submission)  
GET  /{paste_id}               # View paste with syntax highlighting
GET  /{paste_id}/raw           # Raw text view
GET  /{paste_id}/download      # Download as file
GET  /{paste_id}/clone         # Clone/fork paste

# User routes (requires authentication)
GET  /login                    # Login page
GET  /register                 # Registration page  
GET  /user/dashboard          # User's paste dashboard
GET  /user/settings           # Account settings

# Additional features
GET  /recent                   # Recent public pastes
GET  /trending                 # Popular pastes
GET  /search?q=python         # Search public pastes
```

## 6. Performance Optimizations

### 6.1 Caching Layers

```
┌─────────────────┐    1ms     ┌─────────────────┐
│  Local Cache    │ ◄────────► │  Application    │
│  (LRU, 1000)    │            │  Server         │
└─────────────────┘            └─────────────────┘
          │                              │
          │ 5ms                         │
          ▼                              ▼
┌─────────────────┐            ┌─────────────────┐
│  Redis Cluster  │ ◄────────► │  Load Balancer  │  
│  (Distributed)  │    20ms    │                 │
└─────────────────┘            └─────────────────┘
          │                              │
          │ 50ms                        │
          ▼                              ▼
┌─────────────────┐            ┌─────────────────┐
│  PostgreSQL     │ ◄────────► │  Database       │
│  (Sharded)      │            │  Cluster        │
└─────────────────┘            └─────────────────┘
```

### 6.2 Database Optimizations

```sql
-- Hot data partitioning
CREATE INDEX CONCURRENTLY idx_pastes_hot 
ON pastes(created_at DESC) 
WHERE created_at > NOW() - INTERVAL '7 days';

-- Partial indexes for active pastes only
CREATE INDEX idx_pastes_active_public 
ON pastes(created_at DESC) 
WHERE is_active = TRUE AND privacy_level = 0;

-- Covering index for paste retrieval
CREATE INDEX idx_paste_lookup 
ON pastes(id) 
INCLUDE (content, title, language, created_at, expires_at);
```

### 6.3 Content Delivery Network (CDN)

```
User Request Flow:

1. Static Assets (CSS, JS, Images):
   User → CDN → Origin Server (only if cache miss)

2. Dynamic Content (Paste Pages):  
   User → CDN → Load Balancer → App Server → Database
   
3. Raw/API Endpoints:
   User → Load Balancer → App Server → Cache/DB
   (Skip CDN for frequently changing content)
```

## 7. Security Considerations

### 7.1 Input Validation & Sanitization

```python
class ContentValidator:
    MAX_CONTENT_SIZE = 10 * 1024 * 1024  # 10MB
    BLOCKED_PATTERNS = [
        r'<script.*?>.*?</script>',  # XSS prevention
        r'javascript:',              # Prevent JS URLs
        r'data:.*?base64',          # Block data URIs
    ]
    
    def validate_paste(self, content: str) -> bool:
        # Size check
        if len(content.encode('utf-8')) > self.MAX_CONTENT_SIZE:
            raise ContentTooLarge("Content exceeds 10MB limit")
        
        # Malicious content detection
        for pattern in self.BLOCKED_PATTERNS:
            if re.search(pattern, content, re.IGNORECASE):
                raise MaliciousContentDetected("Potentially harmful content")
        
        # Spam detection (simple example)
        if self._is_spam(content):
            raise SpamDetected("Content flagged as spam")
        
        return True
    
    def _is_spam(self, content: str) -> bool:
        # Check for excessive links, repeated text, etc.
        url_count = len(re.findall(r'http[s]?://\S+', content))
        return url_count > 10  # More than 10 URLs might be spam
```

### 7.2 Rate Limiting

```python
class RateLimiter:
    """
    Multi-tier rate limiting strategy
    """
    
    def __init__(self):
        self.redis = RedisService()
    
    def is_allowed(self, identifier: str, action: str) -> bool:
        """
        Rate limiting rules:
        - Anonymous users: 20 pastes/hour, 100 views/hour
        - Registered users: 100 pastes/hour, 500 views/hour  
        - API users: Based on API key tier
        """
        limits = {
            'create_anonymous': (20, 3600),    # 20 per hour
            'create_registered': (100, 3600),  # 100 per hour  
            'view_anonymous': (100, 3600),     # 100 per hour
            'view_registered': (500, 3600),    # 500 per hour
        }
        
        max_requests, window = limits.get(action, (10, 3600))
        
        # Use sliding window counter
        current_time = int(time.time())
        window_start = current_time - window
        
        # Redis pipeline for atomic operations
        pipe = self.redis.pipeline()
        key = f"rate_limit:{identifier}:{action}"
        
        # Remove expired entries
        pipe.zremrangebyscore(key, 0, window_start)
        
        # Count current requests
        pipe.zcard(key)
        
        # Add current request
        pipe.zadd(key, {str(current_time): current_time})
        
        # Set expiration
        pipe.expire(key, window)
        
        results = pipe.execute()
        current_requests = results[1]
        
        return current_requests < max_requests
```

### 7.3 Data Privacy & Encryption

```python
class EncryptionService:
    """
    Encrypt private pastes at rest
    """
    
    def __init__(self):
        self.key = os.environ['ENCRYPTION_KEY']  # 32-byte AES key
        self.cipher_suite = Fernet(self.key)
    
    def encrypt_content(self, content: str, privacy_level: int) -> str:
        """Encrypt private paste content"""
        if privacy_level == 2:  # Private pastes only
            encrypted_content = self.cipher_suite.encrypt(content.encode())
            return base64.b64encode(encrypted_content).decode()
        return content
    
    def decrypt_content(self, content: str, privacy_level: int) -> str:
        """Decrypt private paste content"""
        if privacy_level == 2:  # Private pastes only
            encrypted_content = base64.b64decode(content.encode())
            decrypted_content = self.cipher_suite.decrypt(encrypted_content)
            return decrypted_content.decode()
        return content
```

## 8. Monitoring & Analytics

### 8.1 Key Metrics to Track

```python
class MetricsCollector:
    """
    Collect and emit key business and technical metrics
    """
    
    def __init__(self):
        self.metrics = MetricsClient()  # Prometheus/StatsD client
    
    def track_paste_created(self, paste_data: Dict):
        """Track paste creation metrics"""
        self.metrics.increment('pastes.created.total')
        self.metrics.increment(f'pastes.created.language.{paste_data["language"]}')
        self.metrics.increment(f'pastes.created.privacy.{paste_data["privacy_level"]}')
        
        # Content size distribution
        content_size = len(paste_data.get('content', ''))
        self.metrics.histogram('pastes.content_size.bytes', content_size)
        
        # Track expiration settings
        if paste_data.get('expires_at'):
            self.metrics.increment('pastes.with_expiration')
        else:
            self.metrics.increment('pastes.permanent')
    
    def track_paste_viewed(self, paste_id: str, response_time_ms: float):
        """Track paste view metrics"""
        self.metrics.increment('pastes.viewed.total')
        self.metrics.histogram('pastes.view.response_time.ms', response_time_ms)
        
        # Track cache hit/miss
        if self._was_cache_hit(paste_id):
            self.metrics.increment('pastes.cache.hit')
        else:
            self.metrics.increment('pastes.cache.miss')
```

### 8.2 Health Checks & Alerting

```python
class HealthChecker:
    """
    Comprehensive health monitoring
    """
    
    def get_health_status(self) -> Dict:
        """Return overall system health"""
        checks = {
            'database': self._check_database(),
            'cache': self._check_redis(),
            'storage': self._check_s3(),
            'rate_limiter': self._check_rate_limiter(),
        }
        
        overall_status = 'healthy' if all(
            check['status'] == 'healthy' for check in checks.values()
        ) else 'unhealthy'
        
        return {
            'status': overall_status,
            'timestamp': datetime.utcnow().isoformat(),
            'checks': checks
        }
    
    def _check_database(self) -> Dict:
        """Check database connectivity and performance"""
        try:
            start_time = time.time()
            # Simple query to test connectivity
            result = db.execute("SELECT 1").fetchone()
            response_time = (time.time() - start_time) * 1000
            
            if response_time > 100:  # > 100ms is concerning
                return {'status': 'degraded', 'response_time_ms': response_time}
            
            return {'status': 'healthy', 'response_time_ms': response_time}
        except Exception as e:
            return {'status': 'unhealthy', 'error': str(e)}
```

## 9. Interview Questions & Answers

### 9.1 Common Interview Questions

**Q: How would you handle duplicate content detection?**

A: I'd use content hashing (SHA-256) to detect duplicate pastes:
```python
content_hash = hashlib.sha256(content.encode('utf-8')).hexdigest()

# Check for existing paste with same hash
existing_paste = db.get_paste_by_hash(content_hash)
if existing_paste and not existing_paste.expired:
    return {"paste_id": existing_paste.id, "duplicate": True}
```

**Q: What if a paste goes viral and gets millions of views?**

A: Multi-layered approach:
1. **CDN Caching**: Cache static content at edge locations
2. **Application Caching**: Store hot pastes in Redis with high TTL
3. **Read Replicas**: Route read traffic to database replicas
4. **Circuit Breaker**: Gracefully degrade service if database is overwhelmed
5. **Auto-scaling**: Automatically add more application servers

**Q: How do you prevent abuse and spam?**

A: Comprehensive anti-abuse system:
1. **Rate Limiting**: IP-based and user-based limits
2. **Content Filtering**: Block malicious patterns, excessive links
3. **Reputation System**: Track user behavior over time
4. **CAPTCHA**: For suspicious activity patterns
5. **Community Reporting**: Allow users to report spam
6. **ML-based Detection**: Train models on spam patterns

**Q: How would you implement search functionality?**

A: Full-text search using Elasticsearch:
```python
# Index public pastes in Elasticsearch
{
    "paste_id": "a7b9c2d1",
    "title": "Python Hello World",
    "content": "print('Hello World')",
    "language": "python",
    "created_at": "2026-02-15T10:30:00Z",
    "view_count": 42
}

# Search query
GET /pastes/_search
{
    "query": {
        "multi_match": {
            "query": "python hello world",
            "fields": ["title^3", "content"]  # Boost title matches
        }
    },
    "filter": {
        "range": {
            "created_at": {"gte": "2026-01-01"}
        }
    }
}
```

**Q: How do you handle large file uploads (10MB)?**

A: Stream processing and object storage:
1. **Stream Upload**: Don't load entire file into memory
2. **Object Storage**: Store large files in S3/GCS, not database
3. **Pre-signed URLs**: Let clients upload directly to S3
4. **Background Processing**: Extract metadata, generate thumbnails asynchronously
5. **Content Compression**: Gzip compress text content

## 10. Scaling Considerations

### 10.1 Handling 10x Traffic Growth

**Current Scale**: 5M pastes/day, 580 reads/sec
**Future Scale**: 50M pastes/day, 5800 reads/sec

**Scaling Strategy:**

1. **Database Scaling**:
   ```
   Current: 8 shards, each handling ~625K pastes/day
   Future: 32 shards, each handling ~1.56M pastes/day
   
   Read Replicas: 2 → 4 per shard
   Connection Pooling: Increase pool size
   ```

2. **Cache Scaling**:
   ```
   Redis Cluster: 6 nodes → 12 nodes
   Memory per node: 8GB → 16GB  
   Cache hit ratio: Maintain 80%+
   ```

3. **Application Scaling**:
   ```
   App servers: 10 → 30 instances
   Load balancing: Add regional load balancers
   Auto-scaling: Based on CPU/memory/queue length
   ```

### 10.2 Global Distribution

For worldwide users, implement multi-region deployment:

```
Primary Regions:
- US-East (Virginia): Main traffic
- EU-West (Ireland): European users  
- AP-Southeast (Singapore): Asian users

Data Synchronization:
- Master-master replication between regions
- Eventual consistency acceptable for paste reads
- Strong consistency required for paste creation

CDN Strategy:
- CloudFlare/AWS CloudFront for static assets
- Regional edge caches for popular pastes
- Geolocation-based routing
```

## 11. Summary

Pastebin system design focuses on:

### Key Technical Decisions:
1. **8-character Base62 IDs**: 218 trillion capacity, user-friendly
2. **PostgreSQL + Redis**: Reliable data storage + fast caching
3. **Hash-based sharding**: Even distribution, simple routing
4. **Object storage for large files**: Cost-effective, scalable
5. **Multi-tier rate limiting**: Prevent abuse, ensure fairness

### Performance Targets:
- **Write latency**: < 200ms (p99)
- **Read latency**: < 100ms (p99)  
- **Availability**: 99.9% uptime
- **Scale**: 50M pastes/day, 5800 reads/sec

### Scalability Features:
- **Horizontal scaling**: Add more shards and app servers
- **Caching layers**: Local + Redis + CDN
- **Read replicas**: Scale read traffic independently
- **Auto-expiration**: Automatic cleanup of old pastes

This design can handle massive scale while maintaining simplicity and reliability. The system is optimized for the 10:1 read-heavy traffic pattern typical of paste sharing services.
