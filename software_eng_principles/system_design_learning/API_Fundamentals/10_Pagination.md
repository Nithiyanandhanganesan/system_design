# API Pagination - Complete Guide

## What is Pagination?

**Pagination** is a technique used to divide large datasets into smaller, manageable chunks (pages). Instead of returning all records at once, the API returns a subset of data along with information on how to retrieve the next set.

### Real-World Analogy
**Book Example:**
- **Large Dataset**: A 1000-page book
- **Pagination**: Reading one chapter (page) at a time
- **Navigation**: Page numbers, "Next Chapter", "Previous Chapter"
- **Benefits**: Easier to read, faster to find specific content

### Why Pagination is Needed

#### Problem Without Pagination:
```javascript
// API returns ALL users at once
GET /api/users
Response: [
  { "id": 1, "name": "User 1" },
  { "id": 2, "name": "User 2" },
  ...
  { "id": 1000000, "name": "User 1000000" }
]
// Problems:
// - Slow response time (30+ seconds)
// - High memory usage (100MB+ response)
// - Network timeout
// - Poor user experience
```

#### Solution With Pagination:
```javascript
// API returns 20 users per page
GET /api/users?page=1&limit=20
Response: {
  "data": [
    { "id": 1, "name": "User 1" },
    { "id": 2, "name": "User 2" },
    ...
    { "id": 20, "name": "User 20" }
  ],
  "pagination": {
    "current_page": 1,
    "total_pages": 50000,
    "total_items": 1000000,
    "has_next": true
  }
}
// Benefits:
// - Fast response (100ms)
// - Small payload (2KB)
// - Better user experience
// - Reduced server load
```

## Types of Pagination

### 1. **Offset-Based Pagination (Most Common)**

#### How it Works:
```
Page 1: Records 1-20    (OFFSET 0, LIMIT 20)
Page 2: Records 21-40   (OFFSET 20, LIMIT 20)
Page 3: Records 41-60   (OFFSET 40, LIMIT 20)
```

#### Implementation:
```javascript
// Request
GET /api/users?page=2&limit=20

// Server logic
const page = parseInt(req.query.page) || 1;
const limit = parseInt(req.query.limit) || 20;
const offset = (page - 1) * limit;

// SQL query
SELECT * FROM users 
ORDER BY id 
LIMIT 20 OFFSET 20;

// Response
{
  "data": [...],
  "pagination": {
    "current_page": 2,
    "per_page": 20,
    "total_items": 1000,
    "total_pages": 50,
    "has_next": true,
    "has_previous": true,
    "links": {
      "first": "/api/users?page=1&limit=20",
      "previous": "/api/users?page=1&limit=20",
      "next": "/api/users?page=3&limit=20",
      "last": "/api/users?page=50&limit=20"
    }
  }
}
```

#### Pros & Cons:
✅ **Pros:**
- Easy to implement
- User-friendly (page numbers)
- Can jump to any page
- Works with most databases

❌ **Cons:**
- Performance degrades with large offsets
- Data inconsistency (if records added/deleted)
- Deep pagination is expensive

### 2. **Cursor-Based Pagination (Better for Large Datasets)**

#### How it Works:
```
Page 1: Records where id > 0      (get first 20)
Page 2: Records where id > 20     (get next 20)
Page 3: Records where id > 40     (get next 20)
```

#### Implementation:
```javascript
// Request
GET /api/users?cursor=eyJpZCI6MjB9&limit=20

// Server logic
const limit = parseInt(req.query.limit) || 20;
const cursor = req.query.cursor ? JSON.parse(atob(req.query.cursor)) : {};

// SQL query
SELECT * FROM users 
WHERE id > ${cursor.id || 0}
ORDER BY id 
LIMIT 21; // Get one extra to check if there's a next page

// Response
{
  "data": [...], // 20 records
  "pagination": {
    "next_cursor": "eyJpZCI6NDB9", // base64 encoded {"id": 40}
    "has_more": true,
    "limit": 20
  }
}
```

#### Pros & Cons:
✅ **Pros:**
- Consistent performance (even with millions of records)
- Real-time data consistency
- No duplicate/missing records
- Efficient for large datasets

❌ **Cons:**
- Can't jump to specific page
- More complex to implement
- Harder to show total count
- Less user-friendly for browsing

### 3. **Time-Based Pagination**

#### Use Case:
Perfect for time-series data (logs, messages, events)

```javascript
// Request
GET /api/messages?since=2026-02-15T10:00:00Z&limit=50

// Response
{
  "data": [
    {
      "id": 123,
      "message": "Hello world",
      "created_at": "2026-02-15T10:30:00Z"
    }
  ],
  "pagination": {
    "next_since": "2026-02-15T10:30:00Z",
    "has_more": true
  }
}
```

## Pagination Frameworks & Libraries

### 1. **Backend Frameworks**

#### Spring Boot (Java) - Complete Implementation
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserRepository userRepository;
    
    @GetMapping
    public ResponseEntity<PaginatedResponse<User>> getUsers(
        @RequestParam(defaultValue = "1") int page,
        @RequestParam(defaultValue = "20") int limit,
        @RequestParam(required = false) String search,
        @RequestParam(defaultValue = "id") String sortBy,
        @RequestParam(defaultValue = "asc") String sortDir) {
        
        // Validate parameters
        if (page < 1) page = 1;
        if (limit < 1 || limit > 100) limit = 20;
        
        // Create Pageable object
        Sort sort = sortDir.equalsIgnoreCase("desc") ? 
            Sort.by(sortBy).descending() : Sort.by(sortBy).ascending();
        Pageable pageable = PageRequest.of(page - 1, limit, sort);
        
        // Get data
        Page<User> userPage;
        if (search != null && !search.isEmpty()) {
            userPage = userRepository.findByNameContainingOrEmailContaining(
                search, search, pageable);
        } else {
            userPage = userRepository.findAll(pageable);
        }
        
        // Build response
        PaginatedResponse<User> response = new PaginatedResponse<>();
        response.setData(userPage.getContent());
        response.setPagination(new PaginationInfo(
            page,
            limit,
            userPage.getTotalElements(),
            userPage.getTotalPages(),
            userPage.hasNext(),
            userPage.hasPrevious(),
            search,
            sortBy,
            sortDir
        ));
        
        return ResponseEntity.ok(response);
    }
}

// Response DTOs
@Data
public class PaginatedResponse<T> {
    private List<T> data;
    private PaginationInfo pagination;
}

@Data
@AllArgsConstructor
public class PaginationInfo {
    private int currentPage;
    private int perPage;
    private long totalItems;
    private int totalPages;
    private boolean hasNext;
    private boolean hasPrevious;
    private String searchQuery;
    private String sortBy;
    private String sortDirection;
    private Map<String, String> links;
    
    public PaginationInfo(int currentPage, int perPage, long totalItems, 
                         int totalPages, boolean hasNext, boolean hasPrevious,
                         String searchQuery, String sortBy, String sortDirection) {
        this.currentPage = currentPage;
        this.perPage = perPage;
        this.totalItems = totalItems;
        this.totalPages = totalPages;
        this.hasNext = hasNext;
        this.hasPrevious = hasPrevious;
        this.searchQuery = searchQuery;
        this.sortBy = sortBy;
        this.sortDirection = sortDirection;
        this.links = generateLinks();
    }
    
    private Map<String, String> generateLinks() {
        Map<String, String> links = new HashMap<>();
        String baseUrl = "/api/users";
        String params = String.format("?limit=%d&sortBy=%s&sortDir=%s", 
            perPage, sortBy, sortDirection);
        if (searchQuery != null) {
            params += "&search=" + searchQuery;
        }
        
        links.put("first", baseUrl + params + "&page=1");
        links.put("last", baseUrl + params + "&page=" + totalPages);
        
        if (hasPrevious) {
            links.put("previous", baseUrl + params + "&page=" + (currentPage - 1));
        }
        if (hasNext) {
            links.put("next", baseUrl + params + "&page=" + (currentPage + 1));
        }
        
        return links;
    }
}
```

#### Spring Data JPA Repository
```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Built-in pagination methods
    Page<User> findAll(Pageable pageable);
    
    // Custom query with pagination
    @Query("SELECT u FROM User u WHERE u.status = :status")
    Page<User> findByStatus(@Param("status") String status, Pageable pageable);
    
    // Search with pagination
    Page<User> findByNameContainingOrEmailContaining(
        String nameSearch, String emailSearch, Pageable pageable);
    
    // Complex query with pagination
    @Query("SELECT u FROM User u WHERE " +
           "(:search IS NULL OR u.name LIKE %:search% OR u.email LIKE %:search%) AND " +
           "(:status IS NULL OR u.status = :status)")
    Page<User> findUsersWithFilters(@Param("search") String search, 
                                   @Param("status") String status, 
                                   Pageable pageable);
    
    // Cursor-based pagination
    List<User> findTop20ByIdGreaterThanOrderByIdAsc(Long cursor);
    
    // Count queries for performance
    @Query("SELECT COUNT(u) FROM User u WHERE u.status = 'ACTIVE'")
    long countActiveUsers();
}
```

#### Express.js + Sequelize (Node.js)
```javascript
const express = require('express');
const { User } = require('./models');
const { Op } = require('sequelize');

const router = express.Router();

router.get('/users', async (req, res) => {
  try {
    const page = parseInt(req.query.page) || 1;
    const limit = Math.min(parseInt(req.query.limit) || 20, 100);
    const offset = (page - 1) * limit;
    const search = req.query.search;
    const sortBy = req.query.sortBy || 'id';
    const sortDir = req.query.sortDir === 'desc' ? 'DESC' : 'ASC';
    
    // Build where clause
    let whereClause = {};
    if (search) {
      whereClause = {
        [Op.or]: [
          { name: { [Op.iLike]: `%${search}%` } },
          { email: { [Op.iLike]: `%${search}%` } }
        ]
      };
    }
    
    // Execute query
    const { count, rows } = await User.findAndCountAll({
      where: whereClause,
      limit: limit,
      offset: offset,
      order: [[sortBy, sortDir]],
      attributes: ['id', 'name', 'email', 'status', 'createdAt']
    });
    
    const totalPages = Math.ceil(count / limit);
    
    res.json({
      data: rows,
      pagination: {
        currentPage: page,
        perPage: limit,
        totalItems: count,
        totalPages: totalPages,
        hasNext: page < totalPages,
        hasPrevious: page > 1,
        searchQuery: search,
        sortBy: sortBy,
        sortDirection: sortDir.toLowerCase(),
        links: {
          first: `/api/users?page=1&limit=${limit}`,
          last: `/api/users?page=${totalPages}&limit=${limit}`,
          ...(page > 1 && { previous: `/api/users?page=${page-1}&limit=${limit}` }),
          ...(page < totalPages && { next: `/api/users?page=${page+1}&limit=${limit}` })
        }
      }
    });
    
  } catch (error) {
    res.status(500).json({ 
      error: 'Internal server error', 
      message: error.message 
    });
  }
});

module.exports = router;
```

#### Django Rest Framework (Python)
```python
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response
from rest_framework import generics, filters
from django_filters.rest_framework import DjangoFilterBackend

class CustomPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'limit'
    max_page_size = 100
    page_query_param = 'page'
    
    def get_paginated_response(self, data):
        return Response({
            'data': data,
            'pagination': {
                'currentPage': self.page.number,
                'perPage': self.get_page_size(self.request),
                'totalItems': self.page.paginator.count,
                'totalPages': self.page.paginator.num_pages,
                'hasNext': self.page.has_next(),
                'hasPrevious': self.page.has_previous(),
                'links': {
                    'first': self.get_first_link(),
                    'last': self.get_last_link(),
                    'next': self.get_next_link(),
                    'previous': self.get_previous_link()
                }
            }
        })

class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    pagination_class = CustomPagination
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    
    # Search fields
    search_fields = ['name', 'email']
    
    # Filterable fields
    filterset_fields = ['status', 'role']
    
    # Ordering fields
    ordering_fields = ['id', 'name', 'email', 'created_at']
    ordering = ['id']
    
    def get_queryset(self):
        queryset = User.objects.select_related('profile').prefetch_related('roles')
        
        # Custom filtering logic
        status = self.request.query_params.get('status')
        if (status):
            queryset = queryset.filter(status=status)
            
        return queryset
```

### 2. **Database-Specific Pagination**

#### PostgreSQL
```sql
-- Offset-based pagination
SELECT * FROM users 
ORDER BY id 
LIMIT 20 OFFSET 40;

-- Cursor-based pagination
SELECT * FROM users 
WHERE id > 123 
ORDER BY id 
LIMIT 20;

-- Get total count (expensive for large tables)
SELECT COUNT(*) FROM users;

-- Optimized count for large tables
SELECT reltuples::bigint AS estimate 
FROM pg_class 
WHERE relname = 'users';
```

#### MongoDB
```javascript
// Offset-based
db.users.find()
  .sort({ _id: 1 })
  .skip(40)
  .limit(20);

// Cursor-based
db.users.find({ _id: { $gt: ObjectId("507f1f77bcf86cd799439011") } })
  .sort({ _id: 1 })
  .limit(20);

// Count
db.users.countDocuments(); // Exact count
db.users.estimatedDocumentCount(); // Fast estimate
```

#### MySQL
```sql
-- Offset-based
SELECT * FROM users 
ORDER BY id 
LIMIT 20 OFFSET 40;

-- Cursor-based
SELECT * FROM users 
WHERE id > 123 
ORDER BY id 
LIMIT 20;

-- Optimized count for large tables
SELECT TABLE_ROWS 
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_NAME = 'users';
```

### 2. **Spring Boot Advanced Pagination Configurations**

#### Configuration Class
```java
@Configuration
@EnableWebMvc
public class PaginationConfig implements WebMvcConfigurer {
    
    @Bean
    public PageableHandlerMethodArgumentResolver pageableResolver() {
        PageableHandlerMethodArgumentResolver resolver = new PageableHandlerMethodArgumentResolver();
        resolver.setPageParameterName("page");
        resolver.setSizeParameterName("limit");
        resolver.setOneIndexedParameters(true); // Start from page 1 instead of 0
        resolver.setMaxPageSize(100); // Maximum allowed page size
        resolver.setFallbackPageable(PageRequest.of(0, 20)); // Default pagination
        return resolver;
    }
    
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(pageableResolver());
    }
}
```

#### Global Exception Handler
```java
@ControllerAdvice
public class PaginationExceptionHandler {
    
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleInvalidPagination(IllegalArgumentException ex) {
        ErrorResponse error = new ErrorResponse(
            "INVALID_PAGINATION_PARAMS",
            "Invalid pagination parameters: " + ex.getMessage()
        );
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(PropertyReferenceException.class)
    public ResponseEntity<ErrorResponse> handleInvalidSortProperty(PropertyReferenceException ex) {
        ErrorResponse error = new ErrorResponse(
            "INVALID_SORT_PROPERTY",
            "Invalid sort property: " + ex.getPropertyName()
        );
        return ResponseEntity.badRequest().body(error);
    }
}
```

#### Service Layer Implementation
```java
@Service
@Transactional(readOnly = true)
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public PaginatedResponse<UserDTO> getUsers(PaginationRequest request) {
        // Validate and sanitize inputs
        validatePaginationRequest(request);
        
        // Build Pageable
        Pageable pageable = buildPageable(request);
        
        // Check cache first
        String cacheKey = generateCacheKey(request);
        PaginatedResponse<UserDTO> cachedResult = getCachedResult(cacheKey);
        if (cachedResult != null) {
            return cachedResult;
        }
        
        // Execute query
        Page<User> userPage = executeQuery(request, pageable);
        
        // Convert to DTOs
        List<UserDTO> userDTOs = userPage.getContent()
            .stream()
            .map(this::convertToDTO)
            .collect(Collectors.toList());
        
        // Build response
        PaginatedResponse<UserDTO> response = new PaginatedResponse<>();
        response.setData(userDTOs);
        response.setPagination(buildPaginationInfo(userPage, request));
        
        // Cache result
        cacheResult(cacheKey, response);
        
        return response;
    }
    
    private void validatePaginationRequest(PaginationRequest request) {
        if (request.getPage() < 1) {
            throw new IllegalArgumentException("Page must be greater than 0");
        }
        if (request.getLimit() < 1 || request.getLimit() > 100) {
            throw new IllegalArgumentException("Limit must be between 1 and 100");
        }
        
        // Validate sort properties
        if (request.getSortBy() != null) {
            Set<String> allowedSortFields = Set.of("id", "name", "email", "createdAt", "status");
            if (!allowedSortFields.contains(request.getSortBy())) {
                throw new IllegalArgumentException("Invalid sort field: " + request.getSortBy());
            }
        }
    }
    
    private Pageable buildPageable(PaginationRequest request) {
        Sort sort = Sort.unsorted();
        
        if (request.getSortBy() != null) {
            Sort.Direction direction = "desc".equalsIgnoreCase(request.getSortDir()) 
                ? Sort.Direction.DESC : Sort.Direction.ASC;
            sort = Sort.by(direction, request.getSortBy());
        }
        
        return PageRequest.of(request.getPage() - 1, request.getLimit(), sort);
    }
    
    private Page<User> executeQuery(PaginationRequest request, Pageable pageable) {
        if (request.getSearch() != null && !request.getSearch().isEmpty()) {
            return userRepository.findUsersWithFilters(
                request.getSearch(), 
                request.getStatus(), 
                pageable
            );
        } else {
            return userRepository.findAll(pageable);
        }
    }
    
    private String generateCacheKey(PaginationRequest request) {
        return String.format("users:page:%d:limit:%d:search:%s:sort:%s:%s",
            request.getPage(),
            request.getLimit(),
            request.getSearch() != null ? request.getSearch() : "null",
            request.getSortBy() != null ? request.getSortBy() : "id",
            request.getSortDir() != null ? request.getSortDir() : "asc"
        );
    }
    
    @SuppressWarnings("unchecked")
    private PaginatedResponse<UserDTO> getCachedResult(String cacheKey) {
        try {
            return (PaginatedResponse<UserDTO>) redisTemplate.opsForValue().get(cacheKey);
        } catch (Exception e) {
            // Log and continue without cache
            return null;
        }
    }
    
    private void cacheResult(String cacheKey, PaginatedResponse<UserDTO> response) {
        try {
            redisTemplate.opsForValue().set(cacheKey, response, Duration.ofMinutes(5));
        } catch (Exception e) {
            // Log but don't fail the request
        }
    }
}

// Request DTO
@Data
public class PaginationRequest {
    private int page = 1;
    private int limit = 20;
    private String search;
    private String status;
    private String sortBy;
    private String sortDir = "asc";
}
```

## Advanced Backend Pagination Patterns

### 1. **Cursor-Based Pagination with Spring Boot**

#### Important Concept: Cursor Fields Must Be Unique
**The key point**: You should **never use non-unique fields** as cursors. Here's why and how to handle it:

#### Problem Scenario You Asked About:
```java
// ❌ WRONG: Using non-unique field as cursor
// If you have 2000 records with id=20, this creates problems:

// First request: GET /api/users?limit=20
// Returns 20 records, all with id=20
// Next cursor would be: id=20

// Second request: GET /api/users?cursor=eyJpZCI6MjB9&limit=20  
// WHERE id > 20 
// This SKIPS all remaining 1980 records with id=20!
```

#### Solution 1: Use Unique Fields (Primary Key)
```java
@RestController
@RequestMapping("/api/users")
public class UserCursorController {
    
    @Autowired
    private UserRepository userRepository;
    
    @GetMapping("/cursor")
    public ResponseEntity<CursorPaginatedResponse<User>> getUsersCursor(
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") int limit) {
        
        limit = Math.min(limit, 100);
        
        List<User> users;
        if (cursor != null) {
            Long cursorId = decodeCursor(cursor);
            // ✅ CORRECT: Using unique primary key
            users = userRepository.findTop21ByIdGreaterThanOrderByIdAsc(cursorId);
        } else {
            users = userRepository.findTop21ByOrderByIdAsc();
        }
        
        // Get one extra to check if there are more records
        boolean hasMore = users.size() > limit;
        List<User> limitedUsers = hasMore ? users.subList(0, limit) : users;
        
        String nextCursor = hasMore && !limitedUsers.isEmpty() 
            ? encodeCursor(limitedUsers.get(limitedUsers.size() - 1).getId())
            : null;
        
        return ResponseEntity.ok(CursorPaginatedResponse.<User>builder()
            .data(limitedUsers)
            .hasMore(hasMore)
            .nextCursor(nextCursor)
            .limit(limit)
            .build());
    }
    
    // Repository method using unique primary key
    @Repository
    public interface UserRepository extends JpaRepository<User, Long> {
        List<User> findTop21ByIdGreaterThanOrderByIdAsc(Long id);
        List<User> findTop21ByOrderByIdAsc();
    }
}
```

#### Solution 2: Compound Cursor (Multiple Fields)
```java
// For cases where you need to sort by non-unique fields
@GetMapping("/cursor-compound")
public ResponseEntity<CursorPaginatedResponse<User>> getUsersWithCompoundCursor(
    @RequestParam(required = false) String cursor,
    @RequestParam(defaultValue = "20") int limit) {
    
    CompoundCursor decodedCursor = cursor != null ? decodeCompoundCursor(cursor) : null;
    
    List<User> users;
    if (decodedCursor != null) {
        // Use compound cursor: created_date + unique_id
        users = userRepository.findUsersAfterCursor(
            decodedCursor.getCreatedAt(),
            decodedCursor.getId(),
            limit + 1
        );
    } else {
        users = userRepository.findUsersFromStart(limit + 1);
    }
    
    boolean hasMore = users.size() > limit;
    List<User> limitedUsers = hasMore ? users.subList(0, limit) : users;
    
    String nextCursor = hasMore && !limitedUsers.isEmpty() 
        ? encodeCompoundCursor(
            limitedUsers.get(limitedUsers.size() - 1).getCreatedAt(),
            limitedUsers.get(limitedUsers.size() - 1).getId()
          )
        : null;
    
    return ResponseEntity.ok(/* response */);
}

// Compound cursor implementation
@Data
@AllArgsConstructor
public class CompoundCursor {
    private LocalDateTime createdAt;
    private Long id;
}

// Repository with compound cursor query
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE " +
           "(u.createdAt > :createdAt) OR " +
           "(u.createdAt = :createdAt AND u.id > :id) " +
           "ORDER BY u.createdAt ASC, u.id ASC")
    List<User> findUsersAfterCursor(@Param("createdAt") LocalDateTime createdAt,
                                   @Param("id") Long id,
                                   Pageable pageable);
    
    @Query("SELECT u FROM User u ORDER BY u.createdAt ASC, u.id ASC")
    List<User> findUsersFromStart(Pageable pageable);
}

// Encoding/Decoding compound cursor
private String encodeCompoundCursor(LocalDateTime createdAt, Long id) {
    CompoundCursor cursor = new CompoundCursor(createdAt, id);
    ObjectMapper mapper = new ObjectMapper();
    mapper.registerModule(new JavaTimeModule());
    
    try {
        String jsonCursor = mapper.writeValueAsString(cursor);
        return Base64.getEncoder().encodeToString(jsonCursor.getBytes());
    } catch (Exception e) {
        throw new RuntimeException("Failed to encode cursor", e);
    }
}

private CompoundCursor decodeCompoundCursor(String cursor) {
    try {
        String decoded = new String(Base64.getDecoder().decode(cursor));
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        return mapper.readValue(decoded, CompoundCursor.class);
    } catch (Exception e) {
        throw new IllegalArgumentException("Invalid cursor format");
    }
}
```

#### Solution 3: Time-Based Cursor with Tie-Breaker
```java
// Perfect for time-series data where timestamps might be identical
@GetMapping("/cursor-time")
public ResponseEntity<CursorPaginatedResponse<User>> getTimeBasedCursor(
    @RequestParam(required = false) String cursor,
    @RequestParam(defaultValue = "20") int limit) {
    
    TimeCursor decodedCursor = cursor != null ? decodeTimeCursor(cursor) : null;
    
    List<User> users;
    if (decodedCursor != null) {
        // Use timestamp + id as tie-breaker
        users = userRepository.findUsersAfterTimestamp(
            decodedCursor.getTimestamp(),
            decodedCursor.getId(),
            limit + 1
        );
    } else {
        users = userRepository.findUsersFromStartTime(limit + 1);
    }
    
    // ...existing pagination logic...
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE " +
           "(u.createdAt > :timestamp) OR " +
           "(u.createdAt = :timestamp AND u.id > :id) " +
           "ORDER BY u.createdAt ASC, u.id ASC")
    List<User> findUsersAfterTimestamp(@Param("timestamp") LocalDateTime timestamp,
                                      @Param("id") Long id,
                                      Pageable pageable);
}

@Data
@AllArgsConstructor
public class TimeCursor {
    private LocalDateTime timestamp;
    private Long id; // Tie-breaker for records with same timestamp
}
```

#### Why This Matters - Performance Comparison:
```java
// Scenario: 1 million users, many created at the same timestamp

// ❌ BAD: Using non-unique created_at as cursor
// Query: WHERE created_at > '2026-02-16 10:30:00'
// Problem: Skips records with exact same timestamp
// Result: Missing data, inconsistent pagination

// ✅ GOOD: Using compound cursor (created_at + id)
// Query: WHERE (created_at > '2026-02-16 10:30:00') OR 
//              (created_at = '2026-02-16 10:30:00' AND id > 12345)
// Result: No missing data, consistent pagination
// Performance: O(log n) with proper indexing

// Database index for optimal performance:
// CREATE INDEX idx_users_created_id ON users(created_at, id);
```

#### Real-World Example: Chat Messages
```java
// Chat messages often have identical timestamps
@Entity
public class ChatMessage {
    @Id
    private Long id;
    
    private String content;
    private Long userId;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    // ...existing code...
}

@Repository
public interface ChatMessageRepository extends JpaRepository<ChatMessage, Long> {
    
    // ✅ CORRECT: Compound cursor for chat messages
    @Query("SELECT m FROM ChatMessage m WHERE m.roomId = :roomId AND " +
           "((m.createdAt > :createdAt) OR " +
           " (m.createdAt = :createdAt AND m.id > :id)) " +
           "ORDER BY m.createdAt ASC, m.id ASC")
    List<ChatMessage> findMessagesAfterCursor(@Param("roomId") Long roomId,
                                            @Param("createdAt") LocalDateTime createdAt,
                                            @Param("id") Long id,
                                            Pageable pageable);
    
    // First page query
    @Query("SELECT m FROM ChatMessage m WHERE m.roomId = :roomId " +
           "ORDER BY m.createdAt ASC, m.id ASC")
    List<ChatMessage> findFirstMessages(@Param("roomId") Long roomId, 
                                       Pageable pageable);
}

// Usage in chat API
@RestController
@RequestMapping("/api/chat")
public class ChatController {
    
    @GetMapping("/{roomId}/messages")
    public ResponseEntity<CursorPaginatedResponse<ChatMessage>> getMessages(
        @PathVariable Long roomId,
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "50") int limit) {
        
        // ...existing code...
        
        // Cursor example for chat:
        // eyJ0aW1lc3RhbXAiOiIyMDI2LTAyLTE2VDEwOjMwOjAwWiIsImlkIjoxMjM0NX0=
        // Decoded: {"timestamp":"2026-02-16T10:30:00Z","id":12345}
    }
}
```
```

### 2. **Search + Pagination with Elasticsearch**
```java
@Service
public class UserSearchService {
    
    @Autowired
    private ElasticsearchRestTemplate elasticsearchTemplate;
    
    public PaginatedResponse<UserDTO> searchUsers(UserSearchRequest request) {
        // Build search query
        BoolQuery.Builder boolQuery = QueryBuilders.bool();
        
        if (request.getSearch() != null && !request.getSearch().isEmpty()) {
            boolQuery.should(QueryBuilders.match(m -> m
                .field("name")
                .query(request.getSearch())
                .boost(2.0f)
            ));
            boolQuery.should(QueryBuilders.match(m -> m
                .field("email")
                .query(request.getSearch())
            ));
            boolQuery.minimumShouldMatch("1");
        }
        
        if (request.getStatus() != null) {
            boolQuery.filter(QueryBuilders.term(t -> t
                .field("status")
                .value(request.getStatus())
            ));
        }
        
        // Build sort
        List<SortOptions> sortOptions = new ArrayList<>();
        if (request.getSortBy() != null) {
            SortOrder order = "desc".equalsIgnoreCase(request.getSortDir()) 
                ? SortOrder.Desc : SortOrder.Asc;
            sortOptions.add(SortOptions.of(s -> s
                .field(f -> f.field(request.getSortBy()).order(order))
            ));
        }
        sortOptions.add(SortOptions.of(s -> s
            .field(f -> f.field("_score").order(SortOrder.Desc))
        ));
        
        // Execute search
        SearchRequest searchRequest = SearchRequest.of(s -> s
            .index("users")
            .query(boolQuery.build()._toQuery())
            .sort(sortOptions)
            .from((request.getPage() - 1) * request.getLimit())
            .size(request.getLimit())
            .trackTotalHits(TrackHits.of(t -> t.enabled(true)))
        );
        
        SearchResponse<UserDocument> response = elasticsearchTemplate
            .execute(client -> client.search(searchRequest, UserDocument.class));
        
        // Convert results
        List<UserDTO> users = response.hits().hits()
            .stream()
            .map(hit -> convertToDTO(hit.source()))
            .collect(Collectors.toList());
        
        long totalHits = response.hits().total().value();
        int totalPages = (int) Math.ceil((double) totalHits / request.getLimit());
        
        return buildPaginatedResponse(users, request, totalHits, totalPages);
    }
}

@Data
public class UserSearchRequest extends PaginationRequest {
    private String search;
    private String status;
    private List<String> tags;
    private LocalDate createdAfter;
    private LocalDate createdBefore;
}
```

### 3. **Multi-Sort Pagination with JPA Specifications**
```java
@Service
public class UserSpecificationService {
    
    public PaginatedResponse<User> findUsers(UserFilterRequest request) {
        // Build dynamic specification
        Specification<User> spec = Specification.where(null);
        
        if (request.getSearch() != null && !request.getSearch().isEmpty()) {
            spec = spec.and(UserSpecifications.nameOrEmailContains(request.getSearch()));
        }
        
        if (request.getStatus() != null) {
            spec = spec.and(UserSpecifications.hasStatus(request.getStatus()));
        }
        
        if (request.getCreatedAfter() != null) {
            spec = spec.and(UserSpecifications.createdAfter(request.getCreatedAfter()));
        }
        
        if (request.getCreatedBefore() != null) {
            spec = spec.and(UserSpecifications.createdBefore(request.getCreatedBefore()));
        }
        
        // Build complex sorting
        Sort sort = buildComplexSort(request.getSortCriteria());
        Pageable pageable = PageRequest.of(
            request.getPage() - 1, 
            request.getLimit(), 
            sort
        );
        
        Page<User> userPage = userRepository.findAll(spec, pageable);
        
        return buildPaginatedResponse(userPage, request);
    }
    
    private Sort buildComplexSort(List<SortCriterion> sortCriteria) {
        if (sortCriteria == null || sortCriteria.isEmpty()) {
            return Sort.by("id").ascending();
        }
        
        List<Sort.Order> orders = sortCriteria.stream()
            .map(criterion -> {
                Sort.Direction direction = "desc".equalsIgnoreCase(criterion.getDirection())
                    ? Sort.Direction.DESC : Sort.Direction.ASC;
                
                Sort.Order order = new Sort.Order(direction, criterion.getField());
                
                if (criterion.getNullHandling() != null) {
                    order = "first".equalsIgnoreCase(criterion.getNullHandling())
                        ? order.nullsFirst() : order.nullsLast();
                }
                
                return order;
            })
            .collect(Collectors.toList());
        
        return Sort.by(orders);
    }
}

public class UserSpecifications {
    
    public static Specification<User> nameOrEmailContains(String search) {
        return (root, query, criteriaBuilder) -> {
            String pattern = "%" + search.toLowerCase() + "%";
            return criteriaBuilder.or(
                criteriaBuilder.like(
                    criteriaBuilder.lower(root.get("name")), 
                    pattern
                ),
                criteriaBuilder.like(
                    criteriaBuilder.lower(root.get("email")), 
                    pattern
                )
            );
        };
    }
    
    public static Specification<User> hasStatus(String status) {
        return (root, query, criteriaBuilder) -> 
            criteriaBuilder.equal(root.get("status"), status);
    }
    
    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, criteriaBuilder) -> 
            criteriaBuilder.greaterThanOrEqualTo(root.get("createdAt"), date);
    }
    
    public static Specification<User> createdBefore(LocalDateTime date) {
        return (root, query, criteriaBuilder) -> 
            criteriaBuilder.lessThanOrEqualTo(root.get("createdAt"), date);
    }
}

@Data
public class SortCriterion {
    private String field;
    private String direction = "asc"; // asc or desc
    private String nullHandling; // first or last
}

@Data
public class UserFilterRequest extends PaginationRequest {
    private String search;
    private String status;
    private LocalDateTime createdAfter;
    private LocalDateTime createdBefore;
    private List<SortCriterion> sortCriteria;
}

## Performance Optimization

### 1. **Database Indexing**
```sql
-- Index for pagination queries
CREATE INDEX idx_users_id_created_at ON users(id, created_at);

-- Index for filtered pagination
CREATE INDEX idx_users_status_created_at ON users(status, created_at) 
WHERE status IN ('active', 'pending');

-- Composite index for search + pagination
CREATE INDEX idx_users_search ON users USING GIN(to_tsvector('english', name || ' ' || email));
```

### 2. **Caching Strategies with Spring Boot**
```java
@Service
@CacheConfig(cacheNames = "users")
public class UserCachingService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private CacheManager cacheManager;
    
    // Cache total count (expensive operation)
    @Cacheable(key = "'user_count'", unless = "#result == null")
    public long getCachedUserCount() {
        return userRepository.count();
    }
    
    // Cache page results
    @Cacheable(key = "#page + '_' + #limit + '_' + (#search ?: 'null') + '_' + (#status ?: 'null')")
    public Page<User> getCachedUserPage(int page, int limit, String search, String status) {
        Pageable pageable = PageRequest.of(page - 1, limit, Sort.by("id").ascending());
        
        if (search != null && !search.isEmpty()) {
            return userRepository.findUsersWithFilters(search, status, pageable);
        }
        return userRepository.findAll(pageable);
    }
    
    // Evict cache when user data changes
    @CacheEvict(allEntries = true)
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    @CacheEvict(allEntries = true)
    public User updateUser(User user) {
        return userRepository.save(user);
    }
    
    @CacheEvict(allEntries = true)
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
}

@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        RedisCacheManager.Builder builder = RedisCacheManager
            .RedisCacheManagerBuilder
            .fromConnectionFactory(redisConnectionFactory())
            .cacheDefaults(cacheConfiguration());
        
        return builder.build();
    }
    
    private RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(
            new RedisStandaloneConfiguration("localhost", 6379)
        );
    }
}
```

### 3. **Optimized Counting with Spring Boot**
```java
@Repository
public class OptimizedUserRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    private static final int LARGE_TABLE_THRESHOLD = 100000;
    
    public PaginationMetadata getOptimizedPaginationMetadata(int page, int limit, String search) {
        // First, get an estimate
        long estimatedCount = getEstimatedRowCount();
        
        if (estimatedCount < LARGE_TABLE_THRESHOLD) {
            // For smaller tables, get exact count
            long exactCount = getExactCount(search);
            return new PaginationMetadata(exactCount, page, limit, true);
        } else {
            // For large tables, use approximation
            return new PaginationMetadata(estimatedCount, page, limit, false);
        }
    }
    
    private long getEstimatedRowCount() {
        Query query = entityManager.createNativeQuery(
            "SELECT reltuples::bigint AS estimate " +
            "FROM pg_class " + 
            "WHERE relname = 'users'"
        );
        
        BigInteger result = (BigInteger) query.getSingleResult();
        return result != null ? result.longValue() : 0L;
    }
    
    private long getExactCount(String search) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Long> query = cb.createQuery(Long.class);
        Root<User> root = query.from(User.class);
        
        query.select(cb.count(root));
        
        if (search != null && !search.isEmpty()) {
            String pattern = "%" + search.toLowerCase() + "%";
            Predicate searchPredicate = cb.or(
                cb.like(cb.lower(root.get("name")), pattern),
                cb.like(cb.lower(root.get("email")), pattern)
            );
            query.where(searchPredicate);
        }
        
        return entityManager.createQuery(query).getSingleResult();
    }
    
    // Show ranges instead of exact counts for large datasets
    public String getDisplayCount(long totalItems, boolean isExact) {
        if (!isExact && totalItems > 10000) {
            return formatLargeNumber(totalItems) + "+";
        }
        return String.valueOf(totalItems);
    }
    
    private String formatLargeNumber(long number) {
        if (number >= 1000000) {
            return (number / 1000000) + "M";
        } else if (number >= 1000) {
            return (number / 1000) + "K";
        }
        return String.valueOf(number);
    }
}

@Data
@AllArgsConstructor
public class PaginationMetadata {
    private long totalItems;
    private int currentPage;
    private int itemsPerPage;
    private boolean isExactCount;
    
    public int getTotalPages() {
        return (int) Math.ceil((double) totalItems / itemsPerPage);
    }
    
    public String getDisplayRange() {
        int startItem = (currentPage - 1) * itemsPerPage + 1;
        int endItem = (int) Math.min(currentPage * itemsPerPage, totalItems);
        return startItem + "-" + endItem;
    }
}
```

## Common Pitfalls & Solutions

### 1. **Deep Pagination Problem**
```sql
-- Problem: OFFSET 100000 is very slow
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 100000;

-- Solution: Use cursor-based pagination
SELECT * FROM users WHERE id > 100000 ORDER BY id LIMIT 20;
```

### 2. **Inconsistent Results (Spring Boot Solution)**
```java
@Service
public class ConsistentPaginationService {
    
    @Autowired
    private UserRepository userRepository;
    
    public PaginatedResponse<User> getConsistentUsers(PaginationRequest request) {
        // Use stable timestamp to prevent inconsistent results
        LocalDateTime snapshotTime = request.getSnapshot() != null 
            ? request.getSnapshot() 
            : LocalDateTime.now();
        
        // Build specification for stable pagination
        Specification<User> spec = (root, query, criteriaBuilder) -> {
            List<Predicate> predicates = new ArrayList<>();
            
            // Only include records created before snapshot time
            predicates.add(criteriaBuilder.lessThanOrEqualTo(
                root.get("createdAt"), snapshotTime
            ));
            
            // Add search criteria if provided
            if (request.getSearch() != null && !request.getSearch().isEmpty()) {
                String pattern = "%" + request.getSearch().toLowerCase() + "%";
                predicates.add(criteriaBuilder.or(
                    criteriaBuilder.like(
                        criteriaBuilder.lower(root.get("name")), 
                        pattern
                    ),
                    criteriaBuilder.like(
                        criteriaBuilder.lower(root.get("email")), 
                        pattern
                    )
                ));
            }
            
            return criteriaBuilder.and(predicates.toArray(new Predicate[0]));
        };
        
        // Use compound sorting for stability
        Sort sort = Sort.by(
            Sort.Order.desc("createdAt"),
            Sort.Order.desc("id")  // Tie-breaker for records with same timestamp
        );
        
        Pageable pageable = PageRequest.of(
            request.getPage() - 1, 
            request.getLimit(), 
            sort
        );
        
        Page<User> userPage = userRepository.findAll(spec, pageable);
        
        PaginatedResponse<User> response = new PaginatedResponse<>();
        response.setData(userPage.getContent());
        response.setPagination(buildPaginationInfo(userPage, request, snapshotTime));
        
        return response;
    }
    
    private PaginationInfo buildPaginationInfo(Page<User> page, 
                                             PaginationRequest request, 
                                             LocalDateTime snapshot) {
        PaginationInfo info = new PaginationInfo();
        info.setCurrentPage(request.getPage());
        info.setPerPage(request.getLimit());
        info.setTotalItems(page.getTotalElements());
        info.setTotalPages(page.getTotalPages());
        info.setHasNext(page.hasNext());
        info.setHasPrevious(page.hasPrevious());
        info.setSnapshot(snapshot); // Include snapshot for subsequent requests
        
        return info;
    }
}

@Data
public class PaginationRequest {
    private int page = 1;
    private int limit = 20;
    private String search;
    private LocalDateTime snapshot; // For consistent pagination
}
```

### 3. **Expensive COUNT Queries (Spring Boot Solutions)**
```java
@Service
public class EfficientCountService {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    private static final int LARGE_TABLE_THRESHOLD = 100000;
    
    public PaginationResponse<User> getUsersWithEfficientCount(PaginationRequest request) {
        long estimatedCount = getTableRowEstimate();
        
        if (estimatedCount > LARGE_TABLE_THRESHOLD) {
            // Solution 1: Use "Load More" pattern instead of page numbers
            return getLoadMorePagination(request);
        } else {
            // Solution 2: Use exact count for smaller tables
            return getExactPagination(request);
        }
    }
    
    private PaginationResponse<User> getLoadMorePagination(PaginationRequest request) {
        // Get one extra record to check if there are more
        int actualLimit = request.getLimit() + 1;
        
        List<User> users = userRepository.findUsers(
            request.getSearch(),
            request.getStatus(),
            request.getLastUserId(),
            actualLimit
        );
        
        boolean hasMore = users.size() > request.getLimit();
        if (hasMore) {
            users = users.subList(0, request.getLimit());
        }
        
        Long nextCursor = hasMore && !users.isEmpty() 
            ? users.get(users.size() - 1).getId() 
            : null;
        
        return PaginationResponse.<User>builder()
            .data(users)
            .hasMore(hasMore)
            .nextCursor(nextCursor)
            .totalCountEstimate(formatLargeNumber(getTableRowEstimate()))
            .build();
    }
    
    private PaginationResponse<User> getExactPagination(PaginationRequest request) {
        Pageable pageable = PageRequest.of(
            request.getPage() - 1, 
            request.getLimit(),
            Sort.by("id").ascending()
        );
        
        Page<User> userPage = userRepository.findAll(pageable);
        
        return PaginationResponse.<User>builder()
            .data(userPage.getContent())
            .currentPage(userPage.getNumber() + 1)
            .totalPages(userPage.getTotalPages())
            .totalItems(userPage.getTotalElements())
            .hasNext(userPage.hasNext())
            .hasPrevious(userPage.hasPrevious())
            .isExactCount(true)
            .build();
    }
    
    private long getTableRowEstimate() {
        try {
            Query query = entityManager.createNativeQuery(
                "SELECT reltuples::bigint FROM pg_class WHERE relname = 'users'"
            );
            BigInteger result = (BigInteger) query.getSingleResult();
            return result != null ? result.longValue() : 0L;
        } catch (Exception e) {
            // Fallback to actual count for non-PostgreSQL databases
            return userRepository.count();
        }
    }
    
    private String formatLargeNumber(long number) {
        if (number >= 1000000) {
            return (number / 1000000) + "M+";
        } else if (number >= 1000) {
            return (number / 1000) + "K+";
        }
        return String.valueOf(number);
    }
}

@Data
@Builder
public class PaginationResponse<T> {
    private List<T> data;
    
    // For exact pagination
    private Integer currentPage;
    private Integer totalPages;
    private Long totalItems;
    private Boolean hasNext;
    private Boolean hasPrevious;
    private Boolean isExactCount;
    
    // For cursor/load-more pagination
    private Boolean hasMore;
    private Long nextCursor;
    private String totalCountEstimate;
}

// Repository method for cursor-based queries
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE " +
           "(:search IS NULL OR u.name LIKE %:search% OR u.email LIKE %:search%) AND " +
           "(:status IS NULL OR u.status = :status) AND " +
           "(:lastUserId IS NULL OR u.id > :lastUserId) " +
           "ORDER BY u.id ASC")
    List<User> findUsers(@Param("search") String search,
                        @Param("status") String status,
                        @Param("lastUserId") Long lastUserId,
                        Pageable pageable);
}
```

## Testing Pagination

### 1. **Spring Boot Unit Tests**
```java
@SpringBootTest
@AutoConfigureTestDatabase
@TestPropertySource(properties = {"spring.jpa.hibernate.ddl-auto=create-drop"})
class UserPaginationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    @BeforeEach
    void setUp() {
        // Create 100 test users
        List<User> users = new ArrayList<>();
        for (int i = 1; i <= 100; i++) {
            User user = new User();
            user.setName("User " + i);
            user.setEmail("user" + i + "@example.com");
            user.setStatus(i % 2 == 0 ? "ACTIVE" : "INACTIVE");
            user.setCreatedAt(LocalDateTime.now().minusDays(i));
            users.add(user);
        }
        userRepository.saveAll(users);
    }
    
    @Test
    void shouldReturnFirstPageOfUsers() {
        ResponseEntity<PaginatedResponse> response = restTemplate.getForEntity(
            "/api/users?page=1&limit=20", 
            PaginatedResponse.class
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        
        PaginatedResponse body = response.getBody();
        assertThat(body.getData()).hasSize(20);
        assertThat(body.getPagination().getCurrentPage()).isEqualTo(1);
        assertThat(body.getPagination().getTotalItems()).isEqualTo(100);
        assertThat(body.getPagination().getTotalPages()).isEqualTo(5);
        assertThat(body.getPagination().isHasNext()).isTrue();
        assertThat(body.getPagination().isHasPrevious()).isFalse();
    }
    
    @Test
    void shouldReturnLastPageOfUsers() {
        ResponseEntity<PaginatedResponse> response = restTemplate.getForEntity(
            "/api/users?page=5&limit=20", 
            PaginatedResponse.class
        );
        
        PaginatedResponse body = response.getBody();
        assertThat(body.getData()).hasSize(20);
        assertThat(body.getPagination().getCurrentPage()).isEqualTo(5);
        assertThat(body.getPagination().isHasNext()).isFalse();
        assertThat(body.getPagination().isHasPrevious()).isTrue();
    }
    
    @Test
    void shouldReturnEmptyResultsForInvalidPage() {
        ResponseEntity<PaginatedResponse> response = restTemplate.getForEntity(
            "/api/users?page=10&limit=20", 
            PaginatedResponse.class
        );
        
        PaginatedResponse body = response.getBody();
        assertThat(body.getData()).isEmpty();
        assertThat(body.getPagination().isHasNext()).isFalse();
    }
    
    @Test
    void shouldValidatePaginationParameters() {
        // Test negative page
        ResponseEntity<String> response1 = restTemplate.getForEntity(
            "/api/users?page=-1&limit=20", 
            String.class
        );
        assertThat(response1.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        
        // Test excessive limit
        ResponseEntity<String> response2 = restTemplate.getForEntity(
            "/api/users?page=1&limit=1000", 
            String.class
        );
        assertThat(response2.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
    }
    
    @Test
    void shouldSupportSearchPagination() {
        ResponseEntity<PaginatedResponse> response = restTemplate.getForEntity(
            "/api/users?search=User 1&page=1&limit=10", 
            PaginatedResponse.class
        );
        
        PaginatedResponse body = response.getBody();
        assertThat(body.getData()).hasSizeLessThanOrEqualTo(10);
        assertThat(body.getPagination().getSearchQuery()).isEqualTo("User 1");
    }
    
    @Test
    void shouldSupportSortingPagination() {
        ResponseEntity<PaginatedResponse> response = restTemplate.getForEntity(
            "/api/users?sortBy=name&sortDir=desc&page=1&limit=10", 
            PaginatedResponse.class
        );
        
        PaginatedResponse body = response.getBody();
        assertThat(body.getPagination().getSortBy()).isEqualTo("name");
        assertThat(body.getPagination().getSortDirection()).isEqualTo("desc");
    }
}

// Repository layer tests
@DataJpaTest
class UserRepositoryPaginationTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @BeforeEach
    void setUp() {
        for (int i = 1; i <= 50; i++) {
            User user = new User();
            user.setName("User " + i);
            user.setEmail("user" + i + "@example.com");
            user.setStatus(i % 3 == 0 ? "ACTIVE" : "INACTIVE");
            entityManager.persistAndFlush(user);
        }
    }
    
    @Test
    void shouldSupportPagination() {
        Pageable pageable = PageRequest.of(0, 10, Sort.by("id"));
        Page<User> userPage = userRepository.findAll(pageable);
        
        assertThat(userPage.getContent()).hasSize(10);
        assertThat(userPage.getTotalElements()).isEqualTo(50);
        assertThat(userPage.getTotalPages()).isEqualTo(5);
    }
    
    @Test
    void shouldSupportSearchWithPagination() {
        Pageable pageable = PageRequest.of(0, 5);
        Page<User> userPage = userRepository.findByNameContainingOrEmailContaining(
            "User 1", "User 1", pageable
        );
        
        assertThat(userPage.getContent()).hasSizeGreaterThan(0);
        assertThat(userPage.getContent()).allSatisfy(user -> 
            assertThat(user.getName()).contains("User 1")
        );
    }
    
    @Test
    void shouldSupportCustomQueryPagination() {
        Pageable pageable = PageRequest.of(0, 10);
        Page<User> activePage = userRepository.findByStatus("ACTIVE", pageable);
        
        assertThat(activePage.getContent()).allSatisfy(user -> 
            assertThat(user.getStatus()).isEqualTo("ACTIVE")
        );
    }
}
```

### 2. **Integration Tests with TestContainers**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:tc:postgresql:13:///testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
class PaginationIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldHandleLargePaginationEfficiently() {
        // Create 10,000 test users
        List<User> users = IntStream.rangeClosed(1, 10000)
            .mapToObj(i -> {
                User user = new User();
                user.setName("User " + i);
                user.setEmail("user" + i + "@example.com");
                user.setCreatedAt(LocalDateTime.now().minusSeconds(i));
                return user;
            })
            .collect(Collectors.toList());
        
        userRepository.saveAll(users);
        
        // Test first page performance
        long startTime = System.currentTimeMillis();
        ResponseEntity<PaginatedResponse> firstPage = restTemplate.getForEntity(
            "/api/users?page=1&limit=20", 
            PaginatedResponse.class
        );
        long firstPageTime = System.currentTimeMillis() - startTime;
        
        assertThat(firstPageTime).isLessThan(100); // Should be under 100ms
        assertThat(firstPage.getBody().getData()).hasSize(20);
        
        // Test deep pagination performance
        startTime = System.currentTimeMillis();
        ResponseEntity<PaginatedResponse> deepPage = restTemplate.getForEntity(
            "/api/users?page=500&limit=20", 
            PaginatedResponse.class
        );
        long deepPageTime = System.currentTimeMillis() - startTime;
        
        assertThat(deepPageTime).isLessThan(500); // Should be under 500ms
        assertThat(deepPage.getBody().getData()).hasSize(20);
    }
    
    @Test
    void shouldHandleConcurrentPaginationRequests() throws InterruptedException {
        // Create test data
        createTestUsers(1000);
        
        int numberOfThreads = 10;
        ExecutorService executor = Executors.newFixedThreadPool(numberOfThreads);
        CountDownLatch latch = new CountDownLatch(numberOfThreads);
        List<CompletableFuture<ResponseEntity<PaginatedResponse>>> futures = new ArrayList<>();
        
        // Make concurrent requests
        for (int i = 0; i < numberOfThreads; i++) {
            final int page = i + 1;
            CompletableFuture<ResponseEntity<PaginatedResponse>> future = 
                CompletableFuture.supplyAsync(() -> {
                    try {
                        return restTemplate.getForEntity(
                            "/api/users?page=" + page + "&limit=20", 
                            PaginatedResponse.class
                        );
                    } finally {
                        latch.countDown();
                    }
                }, executor);
            futures.add(future);
        }
        
        latch.await(10, TimeUnit.SECONDS);
        
        // Verify all requests completed successfully
        for (CompletableFuture<ResponseEntity<PaginatedResponse>> future : futures) {
            ResponseEntity<PaginatedResponse> response = future.get();
            assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
            assertThat(response.getBody().getData()).isNotEmpty();
        }
        
        executor.shutdown();
    }
    
    private void createTestUsers(int count) {
        List<User> users = IntStream.rangeClosed(1, count)
            .mapToObj(i -> {
                User user = new User();
                user.setName("User " + i);
                user.setEmail("user" + i + "@example.com");
                return user;
            })
            .collect(Collectors.toList());
        
        userRepository.saveAll(users);
    }
}
```

### 3. **Performance Tests with Spring Boot**
```java
@SpringBootTest
class PaginationPerformanceTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testPaginationPerformanceWithLargeDataset() {
        // Create large dataset
        createLargeDataset(100000);
        
        // Test first page performance
        long startTime = System.nanoTime();
        PaginatedResponse<UserDTO> firstPage = userService.getUsers(
            PaginationRequest.builder()
                .page(1)
                .limit(20)
                .build()
        );
        long firstPageDuration = (System.nanoTime() - startTime) / 1_000_000; // Convert to ms
        
        assertThat(firstPageDuration).isLessThan(100);
        assertThat(firstPage.getData()).hasSize(20);
        
        // Test middle page performance
        startTime = System.nanoTime();
        PaginatedResponse<UserDTO> middlePage = userService.getUsers(
            PaginationRequest.builder()
                .page(2500) // Middle of 100k records
                .limit(20)
                .build()
        );
        long middlePageDuration = (System.nanoTime() - startTime) / 1_000_000;
        
        assertThat(middlePageDuration).isLessThan(200);
        assertThat(middlePage.getData()).hasSize(20);
    }
    
    @Test
    void testSearchPaginationPerformance() {
        createLargeDataset(50000);
        
        long startTime = System.nanoTime();
        PaginatedResponse<UserDTO> searchResults = userService.getUsers(
            PaginationRequest.builder()
                .page(1)
                .limit(20)
                .search("User 123")
                .build()
        );
        long searchDuration = (System.nanoTime() - startTime) / 1_000_000;
        
        assertThat(searchDuration).isLessThan(150);
        assertThat(searchResults.getData()).isNotEmpty();
    }
    
    private void createLargeDataset(int size) {
        List<User> users = new ArrayList<>();
        for (int i = 1; i <= size; i++) {
            User user = new User();
            user.setName("User " + i);
            user.setEmail("user" + i + "@example.com");
            user.setStatus(i % 2 == 0 ? "ACTIVE" : "INACTIVE");
            user.setCreatedAt(LocalDateTime.now().minusSeconds(i));
            users.add(user);
            
            // Batch insert every 1000 records
            if (i % 1000 == 0) {
                userRepository.saveAll(users);
                users.clear();
            }
        }
        
        if (!users.isEmpty()) {
            userRepository.saveAll(users);
        }
    }
}
```

## Best Practices Summary

### ✅ **Do's:**
1. **Use consistent response format** across all paginated endpoints
2. **Set reasonable default limits** (10-50 items per page)
3. **Validate pagination parameters** (page >= 1, limit <= max)
4. **Include metadata** (total count, page info, navigation links)
5. **Use database indexes** for sort columns
6. **Consider cursor pagination** for real-time data
7. **Cache expensive counts** when possible
8. **Provide navigation helpers** (first, last, prev, next links)

### ❌ **Don'ts:**
1. **Don't return unlimited results** without pagination
2. **Don't allow arbitrary large limits** (set max limit: 100)
3. **Don't use deep offset pagination** for large datasets
4. **Don't ignore performance implications** of COUNT queries
5. **Don't forget to handle edge cases** (empty results, invalid pages)
6. **Don't expose internal implementation** details in response
7. **Don't make pagination parameters required** (provide defaults)

## Summary

Pagination is essential for handling large datasets efficiently. Key takeaways:

1. **Choose the right type**: Offset for user browsing, cursor for performance
2. **Optimize database queries**: Use indexes and avoid expensive operations
3. **Cache when possible**: Total counts and frequently accessed pages
4. **Provide good UX**: Clear navigation and loading states
5. **Test thoroughly**: Both functionality and performance
6. **Monitor performance**: Track slow queries and optimize bottlenecks

Understanding pagination will help you build scalable APIs that can handle millions of records while maintaining fast response times and good user experience.
