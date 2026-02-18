# API Design - Complete Guide

## API Design Principles

### 1. **RESTful Design**
```javascript
// Good RESTful URLs
GET    /api/users              // Get all users
GET    /api/users/123          // Get specific user
POST   /api/users              // Create user
PUT    /api/users/123          // Update entire user
PATCH  /api/users/123          // Partial update
DELETE /api/users/123          // Delete user

// Nested resources
GET    /api/users/123/orders   // Get user's orders
POST   /api/users/123/orders   // Create order for user

// Bad examples
GET    /api/getUsers
POST   /api/createUser
GET    /api/user?action=delete&id=123
```

### 2. **Consistent Naming**
```javascript
// Use nouns, not verbs
✅ /api/orders
❌ /api/getOrders

// Use plural nouns for collections  
✅ /api/products
❌ /api/product

// Use kebab-case for multi-word resources
✅ /api/shopping-carts
❌ /api/shoppingCarts
❌ /api/shopping_carts

// Be consistent with parameters
✅ user_id everywhere
❌ user_id, userId, uid mixed
```

### 3. **Proper HTTP Status Codes**
```javascript
// Success codes
200 OK           // Successful GET, PUT, PATCH
201 Created      // Successful POST
204 No Content   // Successful DELETE

// Client error codes
400 Bad Request     // Invalid syntax or missing required fields
401 Unauthorized    // Authentication required
403 Forbidden      // Valid auth but insufficient permissions
404 Not Found      // Resource doesn't exist
409 Conflict       // Resource already exists or constraint violation
422 Unprocessable  // Valid syntax but business logic error
429 Too Many Requests // Rate limit exceeded

// Server error codes
500 Internal Server Error  // Unexpected server error
502 Bad Gateway           // Invalid response from upstream
503 Service Unavailable   // Temporarily unavailable
504 Gateway Timeout       // Upstream timeout
```

## API Structure Design

### 1. **Request/Response Format**
```javascript
// Consistent request structure
{
  "data": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "meta": {
    "client_version": "1.2.0",
    "request_id": "req_123456"
  }
}

// Consistent response structure
{
  "success": true,
  "data": {
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com"
  },
  "meta": {
    "timestamp": "2026-02-15T10:30:00Z",
    "version": "1.0",
    "request_id": "req_123456"
  }
}

// Error response structure
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

### 2. **Pagination**
```javascript
// Offset-based pagination
GET /api/users?page=2&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "current_page": 2,
    "per_page": 20,
    "total_items": 150,
    "total_pages": 8,
    "has_next": true,
    "has_previous": true,
    "links": {
      "first": "/api/users?page=1&limit=20",
      "previous": "/api/users?page=1&limit=20", 
      "next": "/api/users?page=3&limit=20",
      "last": "/api/users?page=8&limit=20"
    }
  }
}

// Cursor-based pagination (better for large datasets)
GET /api/users?cursor=eyJpZCI6MTIzfQ&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTQzfQ",
    "has_more": true,
    "limit": 20
  }
}
```

### 3. **Filtering and Searching**
```javascript
// Simple filters
GET /api/products?category=electronics&status=active&price_min=100

// Complex filters with operators
GET /api/products?filter[price][gte]=100&filter[price][lte]=500

// Search
GET /api/products?search=laptop&search_fields=name,description

// Sorting
GET /api/products?sort=price:asc,created_at:desc

// Field selection
GET /api/users?fields=id,name,email
```

### 4. **Versioning Strategies**

#### URL Versioning (Recommended)
```javascript
// Version in URL path
GET /api/v1/users
GET /api/v2/users

// Implementation
app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);

// Subdomain versioning
// v1.api.example.com/users
// v2.api.example.com/users
```

#### Header Versioning
```javascript
GET /api/users
Accept: application/vnd.api+json;version=1

// Or custom header
API-Version: 2
```

#### Query Parameter Versioning
```javascript
GET /api/users?version=1
```

## Advanced API Design Patterns

### 1. **HATEOAS (Hypermedia as the Engine of Application State)**
```javascript
// Response includes navigation links
{
  "id": 123,
  "name": "John Doe",
  "status": "active",
  "_links": {
    "self": {
      "href": "/api/users/123"
    },
    "orders": {
      "href": "/api/users/123/orders"
    },
    "edit": {
      "href": "/api/users/123",
      "method": "PUT"
    },
    "delete": {
      "href": "/api/users/123", 
      "method": "DELETE"
    }
  }
}
```

### 2. **Async Operations**
```javascript
// For long-running operations
POST /api/reports
{
  "type": "user_analytics",
  "date_range": "2026-01-01:2026-01-31"
}

Response: 202 Accepted
{
  "job_id": "job_123456",
  "status": "processing",
  "status_url": "/api/jobs/job_123456",
  "estimated_completion": "2026-02-15T10:35:00Z"
}

// Check status
GET /api/jobs/job_123456
{
  "id": "job_123456", 
  "status": "completed",
  "result": {
    "report_url": "/api/reports/download/report_789"
  },
  "completed_at": "2026-02-15T10:33:22Z"
}
```

### 3. **Bulk Operations**
```javascript
// Bulk create
POST /api/users/bulk
{
  "users": [
    {"name": "User 1", "email": "user1@example.com"},
    {"name": "User 2", "email": "user2@example.com"}
  ]
}

Response:
{
  "created": [
    {"id": 101, "name": "User 1", "email": "user1@example.com"},
    {"id": 102, "name": "User 2", "email": "user2@example.com"}
  ],
  "errors": []
}

// Bulk update with partial success
PATCH /api/users/bulk
{
  "updates": [
    {"id": 101, "name": "Updated User 1"},
    {"id": 999, "name": "Non-existent User"}
  ]
}

Response:
{
  "updated": [
    {"id": 101, "name": "Updated User 1"}
  ],
  "errors": [
    {"id": 999, "error": "User not found"}
  ]
}
```

### 4. **GraphQL-Style Field Selection**
```javascript
// Allow clients to specify needed fields
GET /api/users?include=profile,orders&exclude=internal_notes

// Or JSON in query parameter
GET /api/users?fields={"user":["id","name"],"profile":["avatar"]}

Response:
{
  "data": [
    {
      "id": 123,
      "name": "John Doe",
      "profile": {
        "avatar": "avatar.jpg"
      }
    }
  ]
}
```

## Error Handling Design

### 1. **Comprehensive Error Codes**
```javascript
// Application-specific error codes
const ErrorCodes = {
  // Validation errors (4000-4099)
  VALIDATION_ERROR: 4001,
  MISSING_REQUIRED_FIELD: 4002,
  INVALID_EMAIL_FORMAT: 4003,
  PASSWORD_TOO_WEAK: 4004,
  
  // Business logic errors (4100-4199)
  INSUFFICIENT_FUNDS: 4101,
  PRODUCT_OUT_OF_STOCK: 4102,
  ORDER_ALREADY_SHIPPED: 4103,
  
  // Authentication/Authorization (4200-4299)
  INVALID_CREDENTIALS: 4201,
  TOKEN_EXPIRED: 4202,
  INSUFFICIENT_PERMISSIONS: 4203,
  
  // Rate limiting (4300-4399)
  RATE_LIMIT_EXCEEDED: 4301,
  QUOTA_EXCEEDED: 4302,
  
  // System errors (5000-5099)
  DATABASE_ERROR: 5001,
  EXTERNAL_SERVICE_ERROR: 5002,
  CONFIGURATION_ERROR: 5003
};

// Error response
{
  "success": false,
  "error": {
    "code": 4003,
    "type": "INVALID_EMAIL_FORMAT",
    "message": "The email address format is invalid",
    "details": {
      "field": "email",
      "value": "invalid-email",
      "expected_format": "user@domain.com"
    },
    "help_url": "https://docs.api.com/errors#invalid-email-format"
  },
  "meta": {
    "request_id": "req_123456",
    "timestamp": "2026-02-15T10:30:00Z"
  }
}
```

### 2. **Field-Level Validation Errors**
```javascript
// Multiple validation errors
{
  "success": false,
  "error": {
    "code": 4001,
    "type": "VALIDATION_ERROR", 
    "message": "Multiple validation errors occurred",
    "errors": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Email format is invalid"
      },
      {
        "field": "age",
        "code": "OUT_OF_RANGE",
        "message": "Age must be between 18 and 120"
      },
      {
        "field": "password",
        "code": "TOO_WEAK",
        "message": "Password must contain at least 8 characters"
      }
    ]
  }
}
```

## Performance Optimization

### 1. **Caching Headers**
```javascript
app.get('/api/users/:id', (req, res) => {
  const user = getUserById(req.params.id);
  
  // Set caching headers
  res.set({
    'Cache-Control': 'public, max-age=300', // 5 minutes
    'ETag': generateETag(user),
    'Last-Modified': user.updated_at.toUTCString()
  });
  
  // Handle conditional requests
  if (req.headers['if-none-match'] === generateETag(user)) {
    return res.status(304).end();
  }
  
  res.json(user);
});
```

### 2. **Compression**
```javascript
const compression = require('compression');

// Enable gzip compression
app.use(compression({
  // Only compress responses larger than 1KB
  threshold: 1024,
  
  // Compress these MIME types
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  }
}));
```

### 3. **Response Size Optimization**
```javascript
// Implement response size limits
app.use((req, res, next) => {
  const originalSend = res.send;
  
  res.send = function(data) {
    const size = Buffer.byteLength(data);
    const maxSize = 10 * 1024 * 1024; // 10MB
    
    if (size > maxSize) {
      return res.status(413).json({
        error: 'Response too large',
        size: size,
        max_size: maxSize
      });
    }
    
    originalSend.call(this, data);
  };
  
  next();
});
```

## Documentation Design

### 1. **OpenAPI/Swagger Specification**
```yaml
# openapi.yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
  description: User management API

paths:
  /api/users:
    get:
      summary: List users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: List of users
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

    post:
      summary: Create user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
          format: email
        created_at:
          type: string
          format: date-time
```

### 2. **Interactive Documentation**
```javascript
const swaggerUI = require('swagger-ui-express');
const YAML = require('yamljs');

const swaggerDocument = YAML.load('./openapi.yaml');

// Serve interactive documentation
app.use('/api-docs', swaggerUI.serve, swaggerUI.setup(swaggerDocument, {
  explorer: true,
  customCss: '.swagger-ui .topbar { display: none }',
  customSiteTitle: 'API Documentation'
}));
```

## Testing API Design

### 1. **Contract Testing**
```javascript
// API contract tests
describe('User API Contract', () => {
  test('GET /api/users should return valid structure', async () => {
    const response = await request(app).get('/api/users');
    
    expect(response.status).toBe(200);
    expect(response.body).toMatchObject({
      success: true,
      data: expect.any(Array),
      pagination: {
        current_page: expect.any(Number),
        total_items: expect.any(Number)
      }
    });
  });
  
  test('POST /api/users should validate required fields', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'John' }); // Missing email
      
    expect(response.status).toBe(400);
    expect(response.body.error.code).toBe('VALIDATION_ERROR');
    expect(response.body.error.errors).toContainEqual({
      field: 'email',
      code: 'REQUIRED_FIELD_MISSING'
    });
  });
});
```

### 2. **Performance Testing**
```javascript
// Load testing example
const autocannon = require('autocannon');

async function loadTest() {
  const result = await autocannon({
    url: 'http://localhost:3000/api/users',
    connections: 10,
    duration: 30, // 30 seconds
    headers: {
      'Authorization': 'Bearer test-token'
    }
  });
  
  console.log('Average latency:', result.latency.average);
  console.log('Requests per second:', result.requests.average);
  
  // Assert performance requirements
  expect(result.latency.p99).toBeLessThan(500); // p99 < 500ms
  expect(result.requests.average).toBeGreaterThan(100); // > 100 RPS
}
```

## API Design Checklist

### Pre-Development
- [ ] Define clear API purpose and scope
- [ ] Identify target consumers
- [ ] Choose appropriate architectural style (REST, GraphQL, etc.)
- [ ] Plan versioning strategy
- [ ] Design consistent URL structure
- [ ] Define data models and schemas

### During Development  
- [ ] Implement proper HTTP status codes
- [ ] Add comprehensive error handling
- [ ] Include pagination for collections
- [ ] Implement filtering and sorting
- [ ] Add caching headers
- [ ] Include rate limiting
- [ ] Validate all inputs
- [ ] Add request/response logging

### Post-Development
- [ ] Write comprehensive documentation
- [ ] Create interactive API explorer
- [ ] Add usage examples
- [ ] Implement monitoring and analytics
- [ ] Set up automated testing
- [ ] Plan deprecation strategy
- [ ] Gather user feedback

## Best Practices Summary

### URL Design
✅ Use nouns, not verbs
✅ Use plural forms for collections  
✅ Keep URLs predictable and consistent
✅ Use query parameters for filtering/pagination

### Response Design
✅ Use standard HTTP status codes
✅ Include consistent response structure
✅ Provide meaningful error messages
✅ Add metadata (timestamps, request IDs)

### Performance
✅ Implement caching strategies
✅ Use compression for large responses
✅ Add pagination for large datasets
✅ Optimize database queries

### Security
✅ Validate all inputs
✅ Implement proper authentication
✅ Use HTTPS everywhere
✅ Add rate limiting

### Documentation
✅ Use OpenAPI/Swagger specification
✅ Provide interactive documentation
✅ Include code examples
✅ Keep documentation up to date

Good API design makes integration easy, reduces support burden, and creates positive developer experience.
