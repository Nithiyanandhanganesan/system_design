# API Basics - Complete Guide

## What is an API?

**API (Application Programming Interface)** is a set of rules and protocols that allows different software applications to communicate with each other. Think of it as a contract between two software components defining how they should interact.

### Real-World Analogy
**Restaurant Analogy:**
- **You (Client)**: Want to order food
- **Waiter (API)**: Takes your order and communicates with kitchen
- **Kitchen (Server)**: Prepares the food
- **Menu (API Documentation)**: Shows what's available and how to order

## Types of APIs

### 1. **Web APIs (Most Common)**
```
Client Application ←→ HTTP/HTTPS ←→ Web Server
```

#### REST API Example:
```http
GET /api/users/123
Host: api.example.com
Authorization: Bearer token123

Response:
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com"
}
```

### 2. **Library/Framework APIs**
```python
# Python requests library API
import requests
response = requests.get('https://api.github.com/users/octocat')
user = response.json()
```

### 3. **Operating System APIs**
```c
// System calls in C
int file_descriptor = open("file.txt", O_RDONLY);
read(file_descriptor, buffer, 1024);
close(file_descriptor);
```

### 4. **Database APIs**
```sql
-- SQL is a database API
SELECT name, email FROM users WHERE age > 25;
```

## HTTP Methods (Verbs)

### CRUD Operations Mapping:

| Operation | HTTP Method | Purpose | Example |
|-----------|-------------|---------|---------|
| **Create** | POST | Create new resource | `POST /api/users` |
| **Read** | GET | Retrieve resource | `GET /api/users/123` |
| **Update** | PUT/PATCH | Update resource | `PUT /api/users/123` |
| **Delete** | DELETE | Remove resource | `DELETE /api/users/123` |

### Detailed HTTP Methods:

#### 1. **GET - Retrieve Data**
```http
GET /api/products?category=electronics&limit=10
```
- **Idempotent**: Multiple calls return same result
- **Safe**: No side effects on server
- **Cacheable**: Responses can be cached

#### 2. **POST - Create New Resource**
```http
POST /api/users
Content-Type: application/json

{
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "role": "developer"
}
```
- **Not idempotent**: Multiple calls create multiple resources
- **Not safe**: Changes server state

#### 3. **PUT - Update/Replace Entire Resource**
```http
PUT /api/users/123
Content-Type: application/json

{
  "id": 123,
  "name": "Alice Smith",
  "email": "alice.smith@example.com",
  "role": "senior-developer"
}
```
- **Idempotent**: Multiple calls have same effect
- **Replaces entire resource**

#### 4. **PATCH - Partial Update**
```http
PATCH /api/users/123
Content-Type: application/json

{
  "role": "team-lead"
}
```
- **May or may not be idempotent**
- **Updates only specified fields**

#### 5. **DELETE - Remove Resource**
```http
DELETE /api/users/123
```
- **Idempotent**: Deleting already deleted resource returns same result

## HTTP Status Codes

### Success (2xx)
| Code | Meaning | Use Case |
|------|---------|----------|
| **200** | OK | Successful GET, PUT, PATCH |
| **201** | Created | Successful POST |
| **204** | No Content | Successful DELETE |

### Client Errors (4xx)
| Code | Meaning | Use Case |
|------|---------|----------|
| **400** | Bad Request | Invalid request format |
| **401** | Unauthorized | Missing/invalid authentication |
| **403** | Forbidden | Valid auth but insufficient permissions |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Resource already exists |
| **422** | Unprocessable Entity | Valid format but business rule violation |
| **429** | Too Many Requests | Rate limit exceeded |

### Server Errors (5xx)
| Code | Meaning | Use Case |
|------|---------|----------|
| **500** | Internal Server Error | Unexpected server error |
| **502** | Bad Gateway | Invalid response from upstream |
| **503** | Service Unavailable | Server temporarily unavailable |
| **504** | Gateway Timeout | Upstream server timeout |

## Request/Response Structure

### Request Components:
```http
GET /api/v1/users/123?include=profile HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Content-Type: application/json
User-Agent: MyApp/1.0
Accept: application/json

{
  "additional": "data"
}
```

**Parts:**
1. **Method**: GET
2. **URL Path**: /api/v1/users/123
3. **Query Parameters**: ?include=profile
4. **Headers**: Authorization, Content-Type, etc.
5. **Body**: JSON payload (for POST/PUT/PATCH)

### Response Components:
```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=3600
X-RateLimit-Remaining: 999

{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "profile": {
    "avatar": "https://cdn.example.com/avatars/123.jpg",
    "bio": "Software Developer"
  }
}
```

## API Design Best Practices

### 1. **Resource-Based URLs**
```
✅ Good:
GET /api/users              (get all users)
GET /api/users/123          (get specific user)
POST /api/users             (create user)
GET /api/users/123/orders   (get user's orders)

❌ Bad:
GET /api/getUsers
POST /api/createUser
GET /api/getUserOrders?userId=123
```

### 2. **Use Proper HTTP Methods**
```
✅ Good:
POST /api/users             (create)
GET /api/users/123          (read)
PUT /api/users/123          (update)
DELETE /api/users/123       (delete)

❌ Bad:
GET /api/users/create
GET /api/users/123/delete
POST /api/users/123/update
```

### 3. **Consistent Response Format**
```json
{
  "success": true,
  "data": {
    "id": 123,
    "name": "John Doe"
  },
  "meta": {
    "timestamp": "2026-02-15T10:30:00Z",
    "version": "1.0"
  }
}
```

### 4. **Error Response Format**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": {
      "field": "email",
      "value": "invalid-email"
    }
  },
  "meta": {
    "timestamp": "2026-02-15T10:30:00Z",
    "request_id": "req_123456"
  }
}
```

### 5. **Versioning**
```
Option 1 - URL Path:
/api/v1/users
/api/v2/users

Option 2 - Header:
Accept: application/vnd.api+json;version=1
API-Version: 2

Option 3 - Query Parameter:
/api/users?version=1
```

## Authentication & Authorization

### 1. **API Keys**
```http
GET /api/users
X-API-Key: sk_live_51234567890abcdef
```

**Important Note about Header Names:**
`X-API-Key` is just a **common convention**, not a fixed requirement. You can use any header name:

```http
# Common conventions (all valid):
X-API-Key: your_api_key_here
API-Key: your_api_key_here
Authorization: ApiKey your_api_key_here
X-Auth-Token: your_api_key_here
X-Custom-Auth: your_api_key_here

# Company-specific examples:
X-Stripe-Key: sk_live_...
X-GitHub-Token: ghp_...
X-Twilio-Auth: AC...
```

**Why "X-" prefix?**
- Historical convention for custom headers
- Modern practice is moving away from "X-" prefix
- RFC 6648 discourages "X-" for new headers

### 2. **Bearer Tokens (JWT - JSON Web Tokens)**

#### What is "Bearer"?
**"Bearer"** is just a **token type** that means "whoever bears (carries) this token can access the resource." It's not specific to JWT - it's a general authentication scheme.

#### Bearer Token Format:
```http
Authorization: Bearer <TOKEN>
```

**"Bearer" can be used with:**
- ✅ JWT tokens: `Authorization: Bearer eyJhbGciOiJIUzI1NiIs...`
- ✅ API keys: `Authorization: Bearer sk_live_51234567890abcdef`
- ✅ Access tokens: `Authorization: Bearer abc123def456ghi789`
- ✅ Any token: `Authorization: Bearer your_secret_token_here`

#### JWT is NOT always Bearer:
```http
# JWT can be used in different ways:
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...     # Bearer scheme
Authorization: JWT eyJhbGciOiJIUzI1NiIs...        # Custom scheme
X-Auth-Token: eyJhbGciOiJIUzI1NiIs...             # Custom header
```

#### What is JWT?
**JWT (JSON Web Token)** is a compact, URL-safe token format used to securely transmit information between parties. It's self-contained, meaning it carries all the information needed within the token itself.

#### JWT Structure
A JWT consists of **3 parts** separated by dots (`.`):
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Header.Payload.Signature
```

**Parts:**
1. **Header** - Algorithm and token type
2. **Payload** - User data (ID, email, role, expiration)  
3. **Signature** - Verifies token authenticity

#### Real-World JWT Flow

**Login Process:**
```
1. User sends email/password → Server
2. Server validates credentials → Creates JWT internally → Sends to client
3. Client stores JWT (localStorage/secure storage)
```

**API Requests:**
```
1. Client includes JWT: Authorization: Bearer <token>
2. Server verifies JWT locally (no external calls)
3. Server extracts user info from JWT → Processes request
```

#### How JWT Creation & Validation Works

**Creation (Internal):**
- Server creates JWT using its own secret key
- No external JWT service needed
- Process: User info + expiration + signature = JWT

**Validation (Internal):**
- Server verifies signature using same secret key
- Checks expiration time
- Extracts user data from payload
- All happens locally (~1-2ms)

**Key Point:** Both creation and validation are **internal** processes - no external service calls required.

#### JWT vs Sessions

| JWT | Sessions |
|-----|----------|
| Stateless (info in token) | Stateful (info on server) |
| No server storage needed | Requires session store |
| Can't revoke before expiry | Can revoke immediately |
| Good for APIs/mobile | Good for web apps |

#### When to Use JWT
✅ Mobile apps, SPAs, microservices, API authentication  
❌ Traditional web apps, immediate logout needs, banking apps

```http
GET /api/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 3. **Basic Authentication**
```http
GET /api/users
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

#### Basic vs Bearer - Key Differences

| Aspect | Basic Authentication | Bearer Authentication |
|--------|---------------------|----------------------|
| **Format** | `Authorization: Basic <base64(username:password)>` | `Authorization: Bearer <token>` |
| **What it carries** | Username + Password (encoded) | Token (JWT, API key, etc.) |
| **Security** | Credentials sent with every request | Token sent with every request |
| **Expiration** | No built-in expiration | Tokens can have expiration |
| **Revocation** | Change password to revoke | Can revoke specific tokens |

#### Basic Authentication Details:
```
How it works:
1. Combine username and password: "john:password123"
2. Encode with Base64: "am9objpwYXNzd29yZDEyMw=="
3. Send: Authorization: Basic am9objpwYXNzd29yZDEyMw==
4. Server decodes and validates credentials
```

**Example:**
```http
# Original: username=john, password=secret123
# Combined: john:secret123
# Base64 encoded: am9objpzZWNyZXQxMjM=

GET /api/profile
Authorization: Basic am9objpzZWNyZXQxMjM=
```

#### Bearer Authentication Details:
```
How it works:
1. Client gets token (from login, API key registration, etc.)
2. Include token in header: Authorization: Bearer <token>
3. Server validates token
4. Server processes request if token is valid
```

**Example:**
```http
# Token could be JWT, API key, or any access token
GET /api/profile
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### When to Use Each:

**Use Basic Authentication:**
- ✅ Simple internal tools
- ✅ Development/testing environments
- ✅ Legacy systems integration
- ✅ One-time scripts
- ❌ Production web/mobile apps (security concerns)

**Use Bearer Authentication:**
- ✅ Modern web applications
- ✅ Mobile applications
- ✅ API integrations
- ✅ Microservices
- ✅ When you need token expiration/revocation

#### Security Comparison:

**Basic Authentication Risks:**
- Credentials sent with every request
- If intercepted, username/password exposed
- No built-in expiration mechanism
- Difficult to revoke specific sessions

**Bearer Authentication Benefits:**
- Tokens can expire automatically
- Can revoke specific tokens without changing passwords
- Token doesn't contain actual credentials
- Can include additional claims/permissions

### 4. **OAuth 2.0 Flow**
```
1. Client → Authorization Server: Request access
2. User → Authorization Server: Login & consent
3. Authorization Server → Client: Authorization code
4. Client → Authorization Server: Exchange code for token
5. Client → API: Use access token
```

## API Documentation

### Essential Components:

#### 1. **Endpoint Description**
```markdown
## GET /api/users/{id}

Retrieves a specific user by their ID.

**Parameters:**
- `id` (path, required): User ID (integer)
- `include` (query, optional): Additional data to include (profile, orders)

**Response:**
- 200: User found
- 404: User not found
- 401: Unauthorized
```

#### 2. **Request/Response Examples**
```markdown
**Request:**
```http
GET /api/users/123?include=profile
Authorization: Bearer token123
```

**Response:**
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "profile": {
    "avatar": "https://cdn.example.com/avatars/123.jpg"
  }
}
```

#### 3. **Error Scenarios**
```markdown
**Error Responses:**

404 Not Found:
```json
{
  "error": "User not found",
  "code": "USER_NOT_FOUND"
}
```

## Testing APIs

### 1. **Using cURL**
```bash
# GET request
curl -X GET "https://api.example.com/users/123" \
  -H "Authorization: Bearer token123" \
  -H "Accept: application/json"

# POST request
curl -X POST "https://api.example.com/users" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer token123" \
  -d '{
    "name": "Alice Johnson",
    "email": "alice@example.com"
  }'
```

## Common API Patterns

### 1. **Pagination**
```http
GET /api/users?page=2&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "current_page": 2,
    "total_pages": 10,
    "total_items": 200,
    "items_per_page": 20,
    "has_next": true,
    "has_previous": true
  }
}
```

### 2. **Filtering & Sorting**
```http
GET /api/users?role=admin&status=active&sort=created_at:desc&age_min=25
```

### 3. **Field Selection**
```http
GET /api/users/123?fields=id,name,email
```

### 4. **Nested Resources**
```http
GET /api/users/123/orders           (user's orders)
GET /api/orders/456/items           (order items)
POST /api/users/123/orders          (create order for user)
```

## API Performance Considerations

### 1. **Caching**
```http
Response Headers:
Cache-Control: public, max-age=3600
ETag: "abc123def456"
Last-Modified: Thu, 15 Feb 2026 10:30:00 GMT
```

### 2. **Compression**
```http
Request:
Accept-Encoding: gzip, deflate

Response:
Content-Encoding: gzip
```

### 3. **Rate Limiting**
```http
Response Headers:
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

### 4. **Async Processing**
```http
POST /api/reports
{
  "type": "user_analytics",
  "date_range": "2026-01-01:2026-01-31"
}

Response: 202 Accepted
{
  "job_id": "job_123456",
  "status": "processing",
  "estimated_completion": "2026-02-15T10:35:00Z"
}

Later check status:
GET /api/jobs/job_123456
```

## Interview Questions & Answers

### Q1: What's the difference between PUT and PATCH?
**Answer:**
- **PUT**: Replaces the entire resource with provided data
- **PATCH**: Updates only specified fields
- **PUT** is idempotent, **PATCH** may or may not be
- Example: `PUT /users/123` with `{name: "John"}` removes all other fields; `PATCH /users/123` with `{name: "John"}` only updates the name

### Q2: When would you use POST vs PUT for creating resources?
**Answer:**
- **POST**: When server generates the ID (`POST /users` → creates user with server-generated ID)
- **PUT**: When client provides the ID (`PUT /users/123` → creates or updates user with ID 123)
- **POST** to collections, **PUT** to specific resources

### Q3: How do you handle API versioning?
**Answer:**
Three main approaches:
1. **URL versioning**: `/api/v1/users` vs `/api/v2/users`
2. **Header versioning**: `Accept: application/vnd.api+json;version=1`
3. **Query parameter**: `/api/users?version=1`

URL versioning is most common and explicit.

### Q4: What status code for validation errors?
**Answer:**
- **400 Bad Request**: Malformed request (invalid JSON)
- **422 Unprocessable Entity**: Well-formed request but validation errors
- **409 Conflict**: Resource already exists or constraint violation

## Summary

APIs are the backbone of modern software architecture. Key takeaways:

1. **RESTful Design**: Use proper HTTP methods and resource-based URLs
2. **Consistent Structure**: Standard request/response formats
3. **Proper Status Codes**: Meaningful HTTP status codes
4. **Security**: Authentication, authorization, input validation
5. **Documentation**: Clear, comprehensive API documentation
6. **Performance**: Caching, pagination, rate limiting
7. **Error Handling**: Descriptive error messages and codes

Understanding these fundamentals will help you design robust, scalable, and maintainable APIs.
