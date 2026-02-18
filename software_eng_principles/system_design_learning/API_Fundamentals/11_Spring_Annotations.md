# Spring Boot Annotations - Complete Guide

## Introduction

Spring Boot provides a rich set of annotations that simplify application development by reducing boilerplate code and providing declarative configuration. This guide covers the most commonly used annotations in Spring Boot applications.

## Core Spring Annotations

### 1. **Component Scanning Annotations**

#### @Component
Base annotation for any Spring-managed component.

```java
@Component
public class EmailService {
    public void sendEmail(String to, String subject, String body) {
        // Email sending logic
    }
}
```

#### @Service
Specialization of @Component for service layer classes.

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    public Optional<User> findById(Long id) {
        return userRepository.findById(id);
    }
}
```

#### @Repository
Specialization of @Component for data access layer classes.

```java
@Repository
public class CustomUserRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    public List<User> findActiveUsers() {
        return entityManager.createQuery(
            "SELECT u FROM User u WHERE u.status = 'ACTIVE'", User.class
        ).getResultList();
    }
}
```

#### @Controller
Specialization of @Component for MVC controllers.

```java
@Controller
public class UserController {
    
    @GetMapping("/users")
    public String listUsers(Model model) {
        model.addAttribute("users", userService.getAllUsers());
        return "users/list"; // Returns view name
    }
}
```

#### @RestController
Combines @Controller and @ResponseBody for REST APIs.

```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping
    public List<User> getAllUsers() {
        return userService.getAllUsers();
    }
}
```

### 2. **Dependency Injection Annotations**

#### @Autowired
Automatic dependency injection by type.

```java
@Service
public class OrderService {
    
    // Field injection (not recommended)
    @Autowired
    private PaymentService paymentService;
    
    // Constructor injection (recommended)
    private final UserService userService;
    private final EmailService emailService;
    
    @Autowired
    public OrderService(UserService userService, EmailService emailService) {
        this.userService = userService;
        this.emailService = emailService;
    }
    
    // Setter injection
    @Autowired
    public void setNotificationService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

#### @Qualifier
Used with @Autowired when multiple beans of same type exist.

```java
public interface PaymentProcessor {
    void processPayment(Payment payment);
}

@Component
@Qualifier("creditCard")
public class CreditCardProcessor implements PaymentProcessor {
    public void processPayment(Payment payment) {
        // Credit card processing logic
    }
}

@Component
@Qualifier("paypal")
public class PayPalProcessor implements PaymentProcessor {
    public void processPayment(Payment payment) {
        // PayPal processing logic
    }
}

@Service
public class PaymentService {
    
    @Autowired
    @Qualifier("creditCard")
    private PaymentProcessor creditCardProcessor;
    
    @Autowired
    @Qualifier("paypal")
    private PaymentProcessor paypalProcessor;
}
```

#### @Value
Inject values from properties files.

```java
@Component
public class DatabaseConfig {
    
    @Value("${app.database.url}")
    private String databaseUrl;
    
    @Value("${app.database.username}")
    private String username;
    
    @Value("${app.database.password}")
    private String password;
    
    @Value("${app.feature.enabled:false}") // Default value: false
    private boolean featureEnabled;
    
    @Value("#{'${app.allowed.origins}'.split(',')}")
    private List<String> allowedOrigins;
}
```

## Web Layer Annotations

### 1. **Request Mapping Annotations**

#### @RequestMapping
Base annotation for mapping web requests.

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    // Maps to GET /api/v1/users
    @RequestMapping(method = RequestMethod.GET)
    public List<User> getUsers() {
        return userService.getAllUsers();
    }
    
    // Maps to POST /api/v1/users
    @RequestMapping(method = RequestMethod.POST, consumes = "application/json")
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User savedUser = userService.save(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(savedUser);
    }
}
```

#### HTTP Method Specific Annotations

```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    
    @GetMapping // Equivalent to @RequestMapping(method = RequestMethod.GET)
    public List<User> getAllUsers() {
        return userService.getAllUsers();
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        return userService.findById(id)
            .map(user -> ResponseEntity.ok(user))
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody @Valid User user) {
        User savedUser = userService.save(user);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(savedUser.getId())
            .toUri();
        return ResponseEntity.created(location).body(savedUser);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, @RequestBody @Valid User user) {
        return userService.update(id, user)
            .map(updatedUser -> ResponseEntity.ok(updatedUser))
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PatchMapping("/{id}")
    public ResponseEntity<User> partialUpdateUser(@PathVariable Long id, @RequestBody Map<String, Object> updates) {
        return userService.partialUpdate(id, updates)
            .map(updatedUser -> ResponseEntity.ok(updatedUser))
            .orElse(ResponseEntity.notFound().build());
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        if (userService.deleteById(id)) {
            return ResponseEntity.noContent().build();
        }
        return ResponseEntity.notFound().build();
    }
}
```

### 2. **Parameter Binding Annotations**

#### @PathVariable
Extracts values from URI path.

```java
@RestController
@RequestMapping("/api")
public class UserController {
    
    // Single path variable
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // Multiple path variables
    @GetMapping("/users/{userId}/orders/{orderId}")
    public ResponseEntity<Order> getUserOrder(
        @PathVariable Long userId,
        @PathVariable Long orderId) {
        
        return orderService.findByUserIdAndOrderId(userId, orderId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // Path variable with different name
    @GetMapping("/users/{user-id}")
    public ResponseEntity<User> getUserByCustomId(@PathVariable("user-id") Long userId) {
        return userService.findById(userId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // Optional path variable
    @GetMapping({"/users/{id}", "/users"})
    public ResponseEntity<?> getUsers(@PathVariable(required = false) Long id) {
        if (id != null) {
            return userService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
        }
        return ResponseEntity.ok(userService.getAllUsers());
    }
    
    // Path variable validation
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getValidatedUser(
        @PathVariable @Min(1) @Max(999999) Long id) {
        
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

#### @RequestParam
Extracts query parameters from URL.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    // Basic query parameters
    @GetMapping
    public ResponseEntity<List<User>> getUsers(
        @RequestParam(defaultValue = "1") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id") String sortBy,
        @RequestParam(defaultValue = "asc") String sortDir) {
        
        Pageable pageable = PageRequest.of(page - 1, size, 
            Sort.by(Sort.Direction.fromString(sortDir), sortBy));
        
        Page<User> userPage = userService.findAll(pageable);
        return ResponseEntity.ok(userPage.getContent());
    }
    
    // Optional query parameters
    @GetMapping("/search")
    public ResponseEntity<List<User>> searchUsers(
        @RequestParam(required = false) String name,
        @RequestParam(required = false) String email,
        @RequestParam(required = false) String status,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate createdAfter,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate createdBefore) {
        
        UserSearchCriteria criteria = UserSearchCriteria.builder()
            .name(name)
            .email(email)
            .status(status)
            .createdAfter(createdAfter)
            .createdBefore(createdBefore)
            .build();
            
        List<User> users = userService.searchUsers(criteria);
        return ResponseEntity.ok(users);
    }
    
    // Query parameter with different name
    @GetMapping("/filter")
    public ResponseEntity<List<User>> filterUsers(
        @RequestParam("user-status") String userStatus,
        @RequestParam("created-date") @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate createdDate) {
        
        List<User> users = userService.findByStatusAndCreatedDate(userStatus, createdDate);
        return ResponseEntity.ok(users);
    }
    
    // Multiple values for same parameter
    @GetMapping("/by-ids")
    public ResponseEntity<List<User>> getUsersByIds(@RequestParam List<Long> ids) {
        List<User> users = userService.findByIds(ids);
        return ResponseEntity.ok(users);
    }
    
    // Map all query parameters
    @GetMapping("/dynamic")
    public ResponseEntity<List<User>> getDynamicUsers(@RequestParam Map<String, String> allParams) {
        // Process dynamic parameters
        UserFilter filter = UserFilter.fromMap(allParams);
        List<User> users = userService.findWithDynamicFilter(filter);
        return ResponseEntity.ok(users);
    }
    
    // Validation on query parameters
    @GetMapping("/validated")
    public ResponseEntity<List<User>> getValidatedUsers(
        @RequestParam @Min(1) @Max(100) int size,
        @RequestParam @Pattern(regexp = "^(id|name|email|createdAt)$") String sortBy,
        @RequestParam @Email String email) {
        
        // ...existing code...
        return ResponseEntity.ok(users);
    }
}
```

#### @RequestHeader
Extracts HTTP headers.

```java
@RestController
@RequestMapping("/api")
public class ApiController {
    
    @GetMapping("/secure")
    public ResponseEntity<String> secureEndpoint(
        @RequestHeader("Authorization") String authorization,
        @RequestHeader(value = "X-API-Version", defaultValue = "1.0") String apiVersion,
        @RequestHeader(value = "X-Client-Id", required = false) String clientId) {
        
        // Validate authorization header
        if (!authorization.startsWith("Bearer ")) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        
        String token = authorization.substring(7);
        if (!jwtService.validateToken(token)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        
        return ResponseEntity.ok("Access granted for API version: " + apiVersion);
    }
    
    @GetMapping("/headers")
    public ResponseEntity<Map<String, String>> getAllHeaders(@RequestHeader Map<String, String> headers) {
        // Filter out sensitive headers
        Map<String, String> safeHeaders = headers.entrySet().stream()
            .filter(entry -> !entry.getKey().toLowerCase().contains("authorization"))
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
            
        return ResponseEntity.ok(safeHeaders);
    }
}
```

#### @RequestBody
Maps HTTP request body to Java object.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody @Valid User user) {
        User savedUser = userService.save(user);
        
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(savedUser.getId())
            .toUri();
            
        return ResponseEntity.created(location).body(savedUser);
    }
    
    @PostMapping("/batch")
    public ResponseEntity<List<User>> createUsers(@RequestBody @Valid List<@Valid User> users) {
        List<User> savedUsers = userService.saveAll(users);
        return ResponseEntity.status(HttpStatus.CREATED).body(savedUsers);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
        @PathVariable Long id,
        @RequestBody @Valid UserUpdateRequest updateRequest) {
        
        return userService.update(id, updateRequest)
            .map(updatedUser -> ResponseEntity.ok(updatedUser))
            .orElse(ResponseEntity.notFound().build());
    }
    
    // Partial update with Map
    @PatchMapping("/{id}")
    public ResponseEntity<User> partialUpdate(
        @PathVariable Long id,
        @RequestBody Map<String, Object> updates) {
        
        return userService.partialUpdate(id, updates)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

#### @ModelAttribute
Binds form data or query parameters to object.

```java
@Controller
@RequestMapping("/users")
public class UserFormController {
    
    @GetMapping("/search")
    public String searchForm(Model model) {
        model.addAttribute("searchCriteria", new UserSearchCriteria());
        return "users/search";
    }
    
    @PostMapping("/search")
    public String searchUsers(@ModelAttribute @Valid UserSearchCriteria searchCriteria,
                             BindingResult bindingResult,
                             Model model) {
        
        if (bindingResult.hasErrors()) {
            return "users/search";
        }
        
        List<User> users = userService.searchUsers(searchCriteria);
        model.addAttribute("users", users);
        return "users/results";
    }
    
    @GetMapping("/{id}/edit")
    public String editForm(@PathVariable Long id, Model model) {
        User user = userService.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        model.addAttribute("user", user);
        return "users/edit";
    }
    
    @PostMapping("/{id}")
    public String updateUser(@PathVariable Long id,
                            @ModelAttribute @Valid User user,
                            BindingResult bindingResult) {
        
        if (bindingResult.hasErrors()) {
            return "users/edit";
        }
        
        userService.update(id, user);
        return "redirect:/users/" + id;
    }
}

// DTO for search criteria
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserSearchCriteria {
    
    @Size(min = 2, max = 50)
    private String name;
    
    @Email
    private String email;
    
    @Pattern(regexp = "^(ACTIVE|INACTIVE|SUSPENDED)$")
    private String status;
    
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdAfter;
    
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdBefore;
}
```

## Data Layer Annotations

### 1. **JPA Annotations**

#### Entity and Table Mapping

```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_users_email", columnList = "email"),
    @Index(name = "idx_users_status", columnList = "status")
})
@EntityListeners(AuditingEntityListener.class)
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
    @SequenceGenerator(name = "user_seq", sequenceName = "user_sequence", allocationSize = 1)
    private Long id;
    
    @Column(name = "full_name", nullable = false, length = 100)
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;
    
    @Column(unique = true, nullable = false)
    @Email(message = "Email should be valid")
    private String email;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private UserStatus status = UserStatus.ACTIVE;
    
    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Version
    private Long version;
    
    // Relationships
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private UserProfile profile;
    
    // ...existing code...
}

public enum UserStatus {
    ACTIVE, INACTIVE, SUSPENDED
}
```

#### Repository Annotations

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
    
    // Simple query methods
    Optional<User> findByEmail(String email);
    
    List<User> findByStatus(UserStatus status);
    
    List<User> findByNameContainingIgnoreCase(String name);
    
    // Custom queries with @Query
    @Query("SELECT u FROM User u WHERE u.createdAt >= :date")
    List<User> findUsersCreatedAfter(@Param("date") LocalDateTime date);
    
    @Query("SELECT u FROM User u WHERE u.status = :status AND u.department.name = :departmentName")
    Page<User> findByStatusAndDepartmentName(@Param("status") UserStatus status,
                                           @Param("departmentName") String departmentName,
                                           Pageable pageable);
    
    // Native query
    @Query(value = "SELECT * FROM users u WHERE u.email LIKE %:domain%", nativeQuery = true)
    List<User> findByEmailDomain(@Param("domain") String domain);
    
    // Modifying queries
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateUserStatus(@Param("id") Long id, @Param("status") UserStatus status);
    
    @Modifying
    @Query("DELETE FROM User u WHERE u.status = :status AND u.createdAt < :date")
    int deleteOldInactiveUsers(@Param("status") UserStatus status, @Param("date") LocalDateTime date);
    
    // Projections
    @Query("SELECT new com.example.dto.UserSummaryDTO(u.id, u.name, u.email, u.status) FROM User u")
    List<UserSummaryDTO> findAllUserSummaries();
    
    // Count queries
    @Query("SELECT COUNT(u) FROM User u WHERE u.status = :status")
    long countByStatus(@Param("status") UserStatus status);
}
```

### 2. **Transaction Annotations**

#### @Transactional

```java
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private AuditService auditService;
    
    // Read-only transaction
    @Transactional(readOnly = true)
    public Optional<User> findById(Long id) {
        return userRepository.findById(id);
    }
    
    @Transactional(readOnly = true)
    public List<User> findAll(Pageable pageable) {
        return userRepository.findAll(pageable).getContent();
    }
    
    // Write transaction with rollback rules
    @Transactional(rollbackFor = {Exception.class})
    public User createUser(User user) {
        // Save user
        User savedUser = userRepository.save(user);
        
        // Send welcome email (if this fails, rollback user creation)
        emailService.sendWelcomeEmail(savedUser.getEmail());
        
        // Audit log
        auditService.logUserCreation(savedUser.getId());
        
        return savedUser;
    }
    
    // Transaction with specific isolation level
    @Transactional(isolation = Isolation.READ_COMMITTED, 
                   propagation = Propagation.REQUIRED)
    public User updateUserWithLocking(Long id, User updateData) {
        User existingUser = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        existingUser.setName(updateData.getName());
        existingUser.setEmail(updateData.getEmail());
        
        return userRepository.save(existingUser);
    }
    
    // Transaction that should not rollback on specific exceptions
    @Transactional(noRollbackFor = {EmailSendingException.class})
    public User createUserWithOptionalEmail(User user) {
        User savedUser = userRepository.save(user);
        
        try {
            emailService.sendWelcomeEmail(savedUser.getEmail());
        } catch (EmailSendingException e) {
            // Log but don't rollback user creation
            logger.warn("Failed to send welcome email to {}", savedUser.getEmail(), e);
        }
        
        return savedUser;
    }
    
    // New transaction (independent of calling transaction)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logUserActivity(Long userId, String activity) {
        UserActivityLog log = new UserActivityLog();
        log.setUserId(userId);
        log.setActivity(activity);
        log.setTimestamp(LocalDateTime.now());
        
        activityLogRepository.save(log);
    }
}
```

## Validation Annotations

### 1. **Bean Validation (JSR-303)**

```java
@Entity
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotNull(message = "Name cannot be null")
    @NotBlank(message = "Name cannot be blank")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;
    
    @NotNull(message = "Email cannot be null")
    @Email(message = "Email should be valid")
    @Column(unique = true)
    private String email;
    
    @NotNull(message = "Age cannot be null")
    @Min(value = 18, message = "User must be at least 18 years old")
    @Max(value = 120, message = "User age cannot exceed 120")
    private Integer age;
    
    @Pattern(regexp = "^\\+?[1-9]\\d{1,14}$", message = "Phone number should be valid")
    private String phoneNumber;
    
    @Past(message = "Date of birth must be in the past")
    @NotNull(message = "Date of birth is required")
    private LocalDate dateOfBirth;
    
    @DecimalMin(value = "0.0", inclusive = false, message = "Salary must be greater than 0")
    @DecimalMax(value = "999999.99", message = "Salary cannot exceed 999999.99")
    @Digits(integer = 6, fraction = 2, message = "Salary must have at most 6 digits and 2 decimal places")
    private BigDecimal salary;
    
    @AssertTrue(message = "User must accept terms and conditions")
    private Boolean acceptedTerms;
    
    // Custom validation
    @ValidPassword
    private String password;
    
    // ...existing code...
}

// Custom validation annotation
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordValidator.class)
public @interface ValidPassword {
    String message() default "Password must contain at least one uppercase letter, one lowercase letter, one number, and one special character";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Custom validator
public class PasswordValidator implements ConstraintValidator<ValidPassword, String> {
    
    private static final String PASSWORD_PATTERN = 
        "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$";
    
    private Pattern pattern;
    
    @Override
    public void initialize(ValidPassword constraintAnnotation) {
        pattern = Pattern.compile(PASSWORD_PATTERN);
    }
    
    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null) {
            return false;
        }
        return pattern.matcher(password).matches();
    }
}
```

### 2. **Validation in Controllers**

```java
@RestController
@RequestMapping("/api/users")
@Validated
public class UserController {
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody @Valid User user) {
        User savedUser = userService.save(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(savedUser);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
        @PathVariable @Min(1) Long id,
        @RequestBody @Valid User user) {
        
        return userService.update(id, user)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @GetMapping
    public ResponseEntity<List<User>> getUsers(
        @RequestParam(defaultValue = "1") @Min(1) Integer page,
        @RequestParam(defaultValue = "20") @Min(1) @Max(100) Integer size,
        @RequestParam(required = false) @Email String email) {
        
        // ...existing code...
    }
    
    // Validation groups
    @PostMapping("/admin")
    public ResponseEntity<User> createAdminUser(@RequestBody @Validated(AdminUserGroup.class) User user) {
        User savedUser = userService.saveAdminUser(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(savedUser);
    }
}

// Validation groups
public interface BasicUserGroup {}
public interface AdminUserGroup {}

// Entity with validation groups
@Entity
public class User {
    
    @NotNull(groups = {BasicUserGroup.class, AdminUserGroup.class})
    @Size(min = 2, max = 50, groups = {BasicUserGroup.class, AdminUserGroup.class})
    private String name;
    
    @NotNull(groups = {AdminUserGroup.class})
    @Size(min = 5, max = 20, groups = {AdminUserGroup.class})
    private String adminCode;
    
    // ...existing code...
}
```

## Configuration Annotations

### 1. **Configuration Classes**

#### @Configuration

```java
@Configuration
@EnableWebMvc
@EnableJpaRepositories(basePackages = "com.example.repository")
@EnableTransactionManagement
@EnableCaching
@EnableAsync
public class ApplicationConfig {
    
    @Value("${app.cors.allowed-origins}")
    private String[] allowedOrigins;
    
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
    
    @Bean
    @ConditionalOnMissingBean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("users", "orders");
    }
    
    @Bean
    @Profile("!test")
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-executor-");
        executor.initialize();
        return executor;
    }
    
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins(allowedOrigins)
                    .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                    .allowedHeaders("*")
                    .allowCredentials(true)
                    .maxAge(3600);
            }
        };
    }
}
```

#### @Profile

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:h2:mem:devdb");
        dataSource.setUsername("sa");
        dataSource.setPassword("");
        return dataSource;
    }
    
    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:postgresql://prod-server:5432/proddb");
        dataSource.setUsername("${db.username}");
        dataSource.setPassword("${db.password}");
        dataSource.setMaximumPoolSize(20);
        return dataSource;
    }
    
    @Bean
    @Profile("test")
    public DataSource testDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:schema.sql")
            .addScript("classpath:test-data.sql")
            .build();
    }
}
```

#### @Conditional Annotations

```java
@Configuration
public class ConditionalConfig {
    
    @Bean
    @ConditionalOnProperty(value = "app.feature.email.enabled", havingValue = "true")
    public EmailService emailService() {
        return new SmtpEmailService();
    }
    
    @Bean
    @ConditionalOnMissingBean(EmailService.class)
    public EmailService mockEmailService() {
        return new MockEmailService();
    }
    
    @Bean
    @ConditionalOnClass(name = "org.springframework.data.redis.core.RedisTemplate")
    @ConditionalOnProperty(value = "app.cache.type", havingValue = "redis")
    public CacheManager redisCacheManager() {
        return new RedisCacheManager(redisConnectionFactory());
    }
    
    @Bean
    @ConditionalOnWebApplication
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .build();
    }
}
```

## Security Annotations

### 1. **Method Level Security**

```java
@Service
@PreAuthorize("hasRole('USER')")
public class UserService {
    
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public User findById(Long userId) {
        return userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
    }
    
    @PreAuthorize("hasRole('ADMIN')")
    @PostAuthorize("hasPermission(returnObject, 'READ')")
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    @PreAuthorize("#user.email == authentication.principal.username")
    public User updateProfile(User user) {
        return userRepository.save(user);
    }
    
    @PostAuthorize("@securityService.canAccess(authentication.principal, returnObject)")
    public List<User> findUsersByDepartment(String department) {
        return userRepository.findByDepartment(department);
    }
    
    @Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
    
    @RolesAllowed({"ADMIN"})
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
}
```

### 2. **Controller Level Security**

```java
@RestController
@RequestMapping("/api/admin")
@PreAuthorize("hasRole('ADMIN')")
public class AdminController {
    
    @GetMapping("/users")
    public ResponseEntity<List<User>> getAllUsers() {
        return ResponseEntity.ok(userService.getAllUsers());
    }
    
    @PostMapping("/users/{id}/suspend")
    @PreAuthorize("hasAuthority('USER_MANAGEMENT')")
    public ResponseEntity<Void> suspendUser(@PathVariable Long id) {
        userService.suspendUser(id);
        return ResponseEntity.ok().build();
    }
}
```

## Testing Annotations

### 1. **Spring Boot Test Annotations**

```java
// Full integration test
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(locations = "classpath:application-test.properties")
class UserControllerIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldCreateUser() {
        User user = new User("John Doe", "john@example.com");
        
        ResponseEntity<User> response = restTemplate.postForEntity("/api/users", user, User.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getName()).isEqualTo("John Doe");
    }
}

// Web layer test
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void shouldCreateUser() throws Exception {
        User user = new User("John Doe", "john@example.com");
        User savedUser = new User(1L, "John Doe", "john@example.com");
        
        when(userService.save(any(User.class))).thenReturn(savedUser);
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(user)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("John Doe"));
    }
}

// JPA layer test
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldFindByEmail() {
        User user = new User("John Doe", "john@example.com");
        entityManager.persistAndFlush(user);
        
        Optional<User> found = userRepository.findByEmail("john@example.com");
        
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John Doe");
    }
}

// JSON serialization test
@JsonTest
class UserJsonTest {
    
    @Autowired
    private JacksonTester<User> json;
    
    @Test
    void shouldSerializeUser() throws Exception {
        User user = new User(1L, "John Doe", "john@example.com");
        
        JsonContent<User> result = json.write(user);
        
        assertThat(result).hasJsonPathNumberValue("$.id", 1);
        assertThat(result).hasJsonPathStringValue("$.name", "John Doe");
        assertThat(result).hasJsonPathStringValue("$.email", "john@example.com");
    }
    
    @Test
    void shouldDeserializeUser() throws Exception {
        String content = "{\"id\":1,\"name\":\"John Doe\",\"email\":\"john@example.com\"}";
        
        User user = json.parse(content).getObject();
        
        assertThat(user.getId()).isEqualTo(1L);
        assertThat(user.getName()).isEqualTo("John Doe");
        assertThat(user.getEmail()).isEqualTo("john@example.com");
    }
}
```

## Caching Annotations

### 1. **Cache Operations**

```java
@Service
@CacheConfig(cacheNames = "users")
public class UserService {
    
    @Cacheable(key = "#id")
    public Optional<User> findById(Long id) {
        logger.info("Fetching user from database: {}", id);
        return userRepository.findById(id);
    }
    
    @Cacheable(key = "#email")
    public Optional<User> findByEmail(String email) {
        return userRepository.findByEmail(email);
    }
    
    @Cacheable(condition = "#pageable.pageSize <= 100")
    public Page<User> findAll(Pageable pageable) {
        return userRepository.findAll(pageable);
    }
    
    @CachePut(key = "#result.id", condition = "#result != null")
    public User save(User user) {
        return userRepository.save(user);
    }
    
    @CacheEvict(key = "#id")
    public void deleteById(Long id) {
        userRepository.deleteById(id);
    }
    
    @CacheEvict(allEntries = true)
    public void deleteAll() {
        userRepository.deleteAll();
    }
    
    @Caching(
        cacheable = @Cacheable(key = "#id"),
        evict = @CacheEvict(cacheNames = "userSummaries", allEntries = true)
    )
    public User updateUser(Long id, User user) {
        User existingUser = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        existingUser.setName(user.getName());
        existingUser.setEmail(user.getEmail());
        
        return userRepository.save(existingUser);
    }
}
```

## Best Practices and Common Patterns

### 1. **Exception Handling**

```java
@ControllerAdvice
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationException(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        
        return new ErrorResponse("Validation failed", errors);
    }
    
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleUserNotFoundException(UserNotFoundException ex) {
        return new ErrorResponse("User not found", ex.getMessage());
    }
    
    @ExceptionHandler(DataIntegrityViolationException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleDataIntegrityViolation(DataIntegrityViolationException ex) {
        return new ErrorResponse("Data conflict", "Resource already exists or violates constraints");
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericException(Exception ex) {
        logger.error("Unexpected error", ex);
        return new ErrorResponse("Internal server error", "An unexpected error occurred");
    }
}

@Data
@AllArgsConstructor
public class ErrorResponse {
    private String message;
    private Object details;
}
```

### 2. **Async Processing**

```java
@Service
public class NotificationService {
    
    @Async
    public CompletableFuture<Void> sendEmailNotification(String email, String message) {
        // Simulate email sending
        try {
            Thread.sleep(2000);
            logger.info("Email sent to: {}", email);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return CompletableFuture.completedFuture(null);
    }
    
    @Async("taskExecutor")
    @Retryable(value = {Exception.class}, maxAttempts = 3, backoff = @Backoff(delay = 1000))
    public CompletableFuture<String> processLongRunningTask(String data) {
        // Simulate processing
        logger.info("Processing data: {}", data);
        
        if (Math.random() > 0.7) {
            throw new RuntimeException("Random processing error");
        }
        
        return CompletableFuture.completedFuture("Processed: " + data);
    }
    
    @Recover
    public CompletableFuture<String> recoverFromProcessingError(Exception ex, String data) {
        logger.error("Failed to process data after retries: {}", data, ex);
        return CompletableFuture.completedFuture("Failed: " + data);
    }
}
```

## Summary

Spring Boot annotations provide a powerful way to:

1. **Reduce boilerplate code** through declarative programming
2. **Configure applications** without XML configuration
3. **Handle web requests** with parameter binding and validation
4. **Manage data access** with JPA and transaction support
5. **Implement security** at method and class levels
6. **Enable caching** for performance optimization
7. **Support testing** with specialized test annotations
8. **Handle cross-cutting concerns** like validation and exception handling

Understanding these annotations is crucial for building robust, maintainable Spring Boot applications. They form the foundation of modern Spring development and enable clean, readable, and efficient code.
