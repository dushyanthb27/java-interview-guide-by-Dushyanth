# Case Study 8: API Versioning Strategy Implementation

## Difficulty: Medium 🟠

---

## Scenario Context

### Company Background
**PayFlow** is a financial technology company providing payment processing services to 50,000+ merchants. Their API is used by third-party developers, enterprise clients, and internal systems for payment processing, refund management, and transaction reporting.

### Technical Stack
- **Backend**: Spring Boot microservices architecture
- **API Gateway**: Spring Cloud Gateway
- **Documentation**: OpenAPI/Swagger
- **Client SDKs**: Java, Python, JavaScript, PHP
- **Authentication**: OAuth 2.0 with JWT
- **Database**: PostgreSQL (sharded per client)
- **Message Queue**: Apache Kafka
- **Deployment**: Kubernetes with GitOps

### Business Context
The company was experiencing:
- **Breaking Changes**: New features required breaking changes that broke existing clients
- **Client Friction**: Enterprise clients refused to upgrade, stuck on old versions
- **Support Burden**: Supporting 5+ API versions simultaneously
- **Documentation Confusion**: Clients unsure which version to use
- **SDK Maintenance**: Multiple SDK versions for each API version

### Constraints
- **Backward Compatibility**: Cannot break existing integrations
- **Enterprise SLAs**: Must support old versions for minimum 12 months
- **Performance**: Version routing must not add significant latency
- **Developer Experience**: Clear upgrade path and documentation
- **Cost**: Minimize infrastructure overhead for multiple versions

---

## Problem Statement

**Primary Challenge**: Implement a comprehensive API versioning strategy that allows for evolution while maintaining backward compatibility and minimizing operational overhead.

**Specific Problems**:
1. How to version APIs without breaking existing clients?
2. How to route requests to the correct version efficiently?
3. How to deprecate old versions gracefully?
4. How to maintain multiple code versions without duplication?
5. How to communicate version changes to clients effectively?

**Success Criteria**:
- Zero breaking changes for existing clients
- Clear deprecation timeline and communication
- Minimal code duplication across versions
- Sub-10ms latency overhead for version routing
- Automated version compatibility testing
- Clear upgrade path for clients

---

## Analysis Phase

### Current API State Assessment

**Version Inventory**:
- v1 (2018): Basic payment processing, 15,000 active clients
- v2 (2020): Added webhooks, 20,000 active clients
- v3 (2021): Added recurring payments, 10,000 active clients
- v4 (2022): Added fraud detection, 5,000 active clients
- v5 (2023 - in development): Real-time notifications, PSD2 compliance

**Breaking Change Examples**:
```json
// v1 - Deprecated
POST /api/payments
{
  "amount": 100.00,
  "currency": "USD",
  "card_number": "4111111111111111",
  "expiry": "12/25",
  "cvv": "123"
}

// v2 - Breaking change (field names)
POST /api/payments
{
  "amount": 100.00,
  "currency_code": "USD",
  "payment_method": {
    "type": "card",
    "card_number": "4111111111111111",
    "expiry_month": 12,
    "expiry_year": 2025,
    "cvv": "123"
  }
}

// v3 - Breaking change (response structure)
{
  "payment_id": "pay_123",
  "status": "completed",
  "transaction": {
    "id": "txn_456",
    "amount": 100.00,
    "currency": "USD"
  }
}
```

### Client Usage Analysis

**Client Distribution**:
| Version | Active Clients | Enterprise Clients | Monthly Calls | Last Updated |
|---------|----------------|-------------------|---------------|--------------|
| v1 | 15,000 | 200 | 50M | 2019 |
| v2 | 20,000 | 350 | 80M | 2021 |
| v3 | 10,000 | 150 | 30M | 2022 |
| v4 | 5,000 | 100 | 20M | 2023 |

**Key Findings**:
- 65% of clients on v1/v2 (oldest versions)
- Enterprise clients slower to upgrade (SLA requirements)
- High volume on v2 (most popular)
- v1 clients haven't updated in 4+ years

### Versioning Strategy Evaluation

| Strategy | Pros | Cons | Score |
|----------|------|------|-------|
| URL Path Versioning | Clear, cacheable | URL pollution, version in URL | 7/10 |
| Header Versioning | Clean URLs, flexible | Harder to test, not cacheable | 6/10 |
| Query Parameter Versioning | Simple, cacheable | Easy to miss, not RESTful | 5/10 |
| Content Negotiation | RESTful standard | Complex implementation | 8/10 |
| Hybrid Approach | Flexibility | Complexity | 9/10 |

**Decision**: Hybrid approach using URL path versioning for public APIs and header versioning for internal services.

---

## Solution Approach

### Strategy: Semantic Versioning with Deprecation Timeline

**Versioning Rules**:
- **Major Version**: Breaking changes (v1 → v2)
- **Minor Version**: Backward-compatible additions (v1.1 → v1.2)
- **Patch Version**: Bug fixes (v1.1.1 → v1.1.2)

**Deprecation Policy**:
- Announce deprecation 6 months before removal
- Show deprecation warnings in API responses
- Provide migration guides and tools
- Support old versions for minimum 12 months
- Remove only after <1% traffic

### Architecture Decisions

**Version Routing Architecture**:
```
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway                               │
├─────────────────────────────────────────────────────────────┤
│  Version Extraction → Routing → Rate Limiting → Auth        │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  v1 Service  │   │  v2 Service  │   │  v3 Service  │
│  (Legacy)    │   │  (Stable)    │   │  (Current)   │
└──────────────┘   └──────────────┘   └──────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                    ┌───────▼────────┐
                    │  Shared Core   │
                    │  (Business     │
                    │   Logic)       │
                    └────────────────┘
```

**Code Organization Strategy**:
```java
// Shared business logic
@Service
public class PaymentService {
    public Payment processPayment(PaymentRequest request) {
        // Version-agnostic business logic
    }
}

// Version-specific controllers
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentV1Controller {
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private PaymentV1Mapper mapper;
    
    @PostMapping
    public ResponseEntity<PaymentV1Response> createPayment(
            @Valid @RequestBody PaymentV1Request request) {
        Payment payment = paymentService.processPayment(
            mapper.toDomain(request));
        return ResponseEntity.ok(mapper.toV1Response(payment));
    }
}

@RestController
@RequestMapping("/api/v2/payments")
public class PaymentV2Controller {
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private PaymentV2Mapper mapper;
    
    @PostMapping
    public ResponseEntity<PaymentV2Response> createPayment(
            @Valid @RequestBody PaymentV2Request request) {
        Payment payment = paymentService.processPayment(
            mapper.toDomain(request));
        return ResponseEntity.ok(mapper.toV2Response(payment));
    }
}
```

---

## Implementation Details

### Phase 1: Version Routing Implementation

#### API Gateway Configuration

```java
@Configuration
public class ApiGatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // v1 routes (legacy)
            .route("payment-v1", r -> r
                .path("/api/v1/payments/**")
                .filters(f -> f
                    .filter(new VersionLoggingFilter("v1"))
                    .filter(new DeprecationWarningFilter("v1", "2024-06-30"))
                )
                .uri("lb://payment-service-v1"))
            
            // v2 routes (stable)
            .route("payment-v2", r -> r
                .path("/api/v2/payments/**")
                .filters(f -> f
                    .filter(new VersionLoggingFilter("v2"))
                )
                .uri("lb://payment-service-v2"))
            
            // v3 routes (current)
            .route("payment-v3", r -> r
                .path("/api/v3/payments/**")
                .filters(f -> f
                    .filter(new VersionLoggingFilter("v3"))
                )
                .uri("lb://payment-service-v3"))
            
            // Default to latest version
            .route("payment-latest", r -> r
                .path("/api/payments/**")
                .filters(f -> f
                    .rewritePath("/api/(?<segment>.*)", "/api/v3/${segment}")
                )
                .uri("lb://payment-service-v3"))
            
            .build();
    }
}
```

#### Deprecation Warning Filter

```java
@Component
public class DeprecationWarningFilter implements GatewayFilter {
    
    private final String version;
    private final LocalDate deprecationDate;
    
    public DeprecationWarningFilter(String version, String deprecationDate) {
        this.version = version;
        this.deprecationDate = LocalDate.parse(deprecationDate);
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpResponse response = exchange.getResponse();
        
        // Add deprecation headers
        response.getHeaders().add("X-API-Deprecated", "true");
        response.getHeaders().add("X-API-Version", version);
        response.getHeaders().add("X-API-Sunset", deprecationDate.toString());
        
        // Add warning to response body for JSON responses
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            if (response.getHeaders().getContentType() != null &&
                response.getHeaders().getContentType().includes(MediaType.APPLICATION_JSON)) {
                // Modify response to include deprecation warning
                // (implementation depends on response modifier)
            }
        }));
    }
}
```

### Phase 2: Version-Specific Controllers with Shared Logic

#### Base Controller with Common Functionality

```java
public abstract class BasePaymentController<V, R> {
    
    @Autowired
    protected PaymentService paymentService;
    
    @Autowired
    protected PaymentValidator validator;
    
    @Autowired
    protected RateLimiter rateLimiter;
    
    protected abstract V toDomain(R request);
    
    protected abstract R toResponse(V domain);
    
    @PostMapping
    public ResponseEntity<R> createPayment(@Valid @RequestBody R request) {
        // Rate limiting
        String clientId = extractClientId();
        if (!rateLimiter.tryAcquire(clientId, "create_payment")) {
            throw new RateLimitExceededException();
        }
        
        // Validation
        validator.validate(request);
        
        // Business logic
        V domain = toDomain(request);
        V result = paymentService.processPayment(domain);
        
        // Response
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(toResponse(result));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<R> getPayment(@PathVariable String id) {
        V payment = paymentService.getPayment(id);
        return ResponseEntity.ok(toResponse(payment));
    }
    
    protected String extractClientId() {
        // Extract from JWT or API key
        return SecurityContextHolder.getContext()
            .getAuthentication()
            .getName();
    }
}
```

#### V1 Controller Implementation

```java
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentV1Controller extends BasePaymentController<Payment, PaymentV1Response> {
    
    @Autowired
    private PaymentV1Mapper mapper;
    
    @Override
    protected Payment toDomain(PaymentV1Request request) {
        return mapper.toDomain(request);
    }
    
    @Override
    protected PaymentV1Response toResponse(Payment domain) {
        PaymentV1Response response = mapper.toV1Response(domain);
        response.setApiVersion("v1");
        response.setDeprecationWarning(
            "This API version is deprecated. Please migrate to v3 by 2024-06-30.");
        return response;
    }
    
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        return ResponseEntity.ok(Map.of(
            "status", "healthy",
            "version", "v1",
            "deprecated", "true",
            "sunset_date", "2024-06-30"
        ));
    }
}
```

#### V2 Controller Implementation

```java
@RestController
@RequestMapping("/api/v2/payments")
public class PaymentV2Controller extends BasePaymentController<Payment, PaymentV2Response> {
    
    @Autowired
    private PaymentV2Mapper mapper;
    
    @Autowired
    private WebhookService webhookService;
    
    @Override
    protected Payment toDomain(PaymentV2Request request) {
        return mapper.toDomain(request);
    }
    
    @Override
    protected PaymentV2Response toResponse(Payment domain) {
        PaymentV2Response response = mapper.toV2Response(domain);
        response.setApiVersion("v2");
        return response;
    }
    
    @PostMapping("/{id}/webhooks")
    public ResponseEntity<WebhookResponse> registerWebhook(
            @PathVariable String id,
            @Valid @RequestBody WebhookRequest request) {
        // V2-specific feature: webhooks
        Webhook webhook = webhookService.register(id, request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(mapper.toWebhookResponse(webhook));
    }
}
```

#### V3 Controller Implementation

```java
@RestController
@RequestMapping("/api/v3/payments")
public class PaymentV3Controller extends BasePaymentController<Payment, PaymentV3Response> {
    
    @Autowired
    private PaymentV3Mapper mapper;
    
    @Autowired
    private FraudDetectionService fraudService;
    
    @Autowired
    private RecurringPaymentService recurringService;
    
    @Override
    protected Payment toDomain(PaymentV3Request request) {
        Payment payment = mapper.toDomain(request);
        
        // V3-specific: fraud detection
        FraudAssessment assessment = fraudService.assess(payment);
        payment.setFraudScore(assessment.getScore());
        payment.setFraudStatus(assessment.getStatus());
        
        return payment;
    }
    
    @Override
    protected PaymentV3Response toResponse(Payment domain) {
        PaymentV3Response response = mapper.toV3Response(domain);
        response.setApiVersion("v3");
        response.setLatest(true);
        return response;
    }
    
    @PostMapping("/recurring")
    public ResponseEntity<RecurringPaymentResponse> createRecurringPayment(
            @Valid @RequestBody RecurringPaymentRequest request) {
        // V3-specific feature: recurring payments
        RecurringPayment recurring = recurringService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(mapper.toRecurringResponse(recurring));
    }
}
```

### Phase 3: Mapper Implementation for Version Compatibility

#### V1 Mapper (Legacy Format)

```java
@Component
public class PaymentV1Mapper {
    
    public Payment toDomain(PaymentV1Request request) {
        Payment payment = new Payment();
        payment.setAmount(request.getAmount());
        payment.setCurrency(request.getCurrency());
        
        // V1 uses flat card structure
        PaymentMethod method = new PaymentMethod();
        method.setType(PaymentMethod.Type.CARD);
        CardDetails card = new CardDetails();
        card.setCardNumber(request.getCardNumber());
        
        // Parse expiry "MM/YY"
        String[] expiryParts = request.getExpiry().split("/");
        card.setExpiryMonth(Integer.parseInt(expiryParts[0]));
        card.setExpiryYear(2000 + Integer.parseInt(expiryParts[1]));
        card.setCvv(request.getCvv());
        
        method.setCardDetails(card);
        payment.setPaymentMethod(method);
        
        return payment;
    }
    
    public PaymentV1Response toV1Response(Payment payment) {
        PaymentV1Response response = new PaymentV1Response();
        response.setPaymentId(payment.getId());
        response.setStatus(payment.getStatus().name().toLowerCase());
        response.setAmount(payment.getAmount());
        response.setCurrency(payment.getCurrency());
        
        // V1 response format
        if (payment.getPaymentMethod() != null && 
            payment.getPaymentMethod().getCardDetails() != null) {
            CardDetails card = payment.getPaymentMethod().getCardDetails();
            response.setCardLast4(card.getCardNumber().substring(
                card.getCardNumber().length() - 4));
        }
        
        response.setCreatedAt(payment.getCreatedAt().toString());
        return response;
    }
}
```

#### V2 Mapper (Improved Format)

```java
@Component
public class PaymentV2Mapper {
    
    public Payment toDomain(PaymentV2Request request) {
        Payment payment = new Payment();
        payment.setAmount(request.getAmount());
        payment.setCurrency(request.getCurrencyCode());
        
        // V2 uses nested payment method structure
        PaymentMethod method = new PaymentMethod();
        method.setType(PaymentMethod.Type.valueOf(
            request.getPaymentMethod().getType().toUpperCase()));
        
        if (request.getPaymentMethod().getCard() != null) {
            CardDetails card = new CardDetails();
            card.setCardNumber(request.getPaymentMethod().getCard().getCardNumber());
            card.setExpiryMonth(request.getPaymentMethod().getCard().getExpiryMonth());
            card.setExpiryYear(request.getPaymentMethod().getCard().getExpiryYear());
            card.setCvv(request.getPaymentMethod().getCard().getCvv());
            method.setCardDetails(card);
        }
        
        payment.setPaymentMethod(method);
        payment.setMetadata(request.getMetadata());
        
        return payment;
    }
    
    public PaymentV2Response toV2Response(Payment payment) {
        PaymentV2Response response = new PaymentV2Response();
        response.setPaymentId(payment.getId());
        response.setStatus(payment.getStatus());
        
        // V2 nested transaction structure
        TransactionV2 transaction = new TransactionV2();
        transaction.setId(payment.getTransactionId());
        transaction.setAmount(payment.getAmount());
        transaction.setCurrency(payment.getCurrency());
        transaction.setProcessedAt(payment.getProcessedAt());
        response.setTransaction(transaction);
        
        // Payment method summary
        if (payment.getPaymentMethod() != null) {
            PaymentMethodSummary summary = new PaymentMethodSummary();
            summary.setType(payment.getPaymentMethod().getType().name().toLowerCase());
            if (payment.getPaymentMethod().getCardDetails() != null) {
                summary.setLast4(payment.getPaymentMethod()
                    .getCardDetails().getCardNumber().substring(
                        payment.getPaymentMethod().getCardDetails()
                            .getCardNumber().length() - 4));
            }
            response.setPaymentMethod(summary);
        }
        
        response.setCreatedAt(payment.getCreatedAt());
        response.setUpdatedAt(payment.getUpdatedAt());
        
        return response;
    }
}
```

### Phase 4: Deprecation and Migration Tools

#### Migration Assistant Service

```java
@Service
public class MigrationAssistantService {
    
    @Autowired
    private PaymentV1Mapper v1Mapper;
    
    @Autowired
    private PaymentV2Mapper v2Mapper;
    
    @Autowired
    private PaymentV3Mapper v3Mapper;
    
    /**
     * Convert V1 request to V3 format for migration testing
     */
    public PaymentV3Request migrateV1ToV3(PaymentV1Request v1Request) {
        // Convert V1 → Domain → V3
        Payment domain = v1Mapper.toDomain(v1Request);
        return v3Mapper.toV3Request(domain);
    }
    
    /**
     * Compare responses between versions to identify breaking changes
     */
    public MigrationReport compareVersions(
            PaymentV1Request v1Request,
            PaymentV2Request v2Request,
            PaymentV3Request v3Request) {
        
        MigrationReport report = new MigrationReport();
        
        // Process with each version
        Payment v1Result = v1Mapper.toDomain(v1Request);
        Payment v2Result = v2Mapper.toDomain(v2Request);
        Payment v3Result = v3Mapper.toDomain(v3Request);
        
        // Compare key fields
        if (!v1Result.getAmount().equals(v2Result.getAmount())) {
            report.addDiscrepancy("amount", v1Result.getAmount(), 
                v2Result.getAmount());
        }
        
        if (!v1Result.getCurrency().equals(v2Result.getCurrencyCode())) {
            report.addDiscrepancy("currency", v1Result.getCurrency(), 
                v2Result.getCurrencyCode());
        }
        
        // Check for missing fields
        report.setV1OnlyFields(getV1SpecificFields());
        report.setV2OnlyFields(getV2SpecificFields());
        report.setV3OnlyFields(getV3SpecificFields());
        
        return report;
    }
    
    /**
     * Generate migration guide for specific client
     */
    public MigrationGuide generateMigrationGuide(String clientId, 
            String fromVersion, String toVersion) {
        
        MigrationGuide guide = new MigrationGuide();
        guide.setClientId(clientId);
        guide.setFromVersion(fromVersion);
        guide.setToVersion(toVersion);
        
        // Analyze client's API usage
        ApiUsageStats stats = analyzeClientUsage(clientId);
        
        // Identify breaking changes based on usage
        List<BreakingChange> breakingChanges = 
            identifyBreakingChanges(stats, fromVersion, toVersion);
        guide.setBreakingChanges(breakingChanges);
        
        // Generate code examples
        guide.setCodeExamples(generateCodeExamples(breakingChanges));
        
        // Estimate migration effort
        guide.setEstimatedEffort(estimateMigrationEffort(breakingChanges));
        
        return guide;
    }
    
    private ApiUsageStats analyzeClientUsage(String clientId) {
        // Query API logs for client usage patterns
        return apiUsageRepository.findByClientId(clientId);
    }
}
```

#### Version Compatibility Test

```java
@SpringBootTest
@AutoConfigureMockMvc
public class ApiVersionCompatibilityTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private MigrationAssistantService migrationService;
    
    @Test
    public void testV1ToV2Migration() throws Exception {
        // V1 request
        String v1Request = """
            {
              "amount": 100.00,
              "currency": "USD",
              "card_number": "4111111111111111",
              "expiry": "12/25",
              "cvv": "123"
            }
            """;
        
        // Call V1 API
        MvcResult v1Result = mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(v1Request))
            .andExpect(status().isCreated())
            .andReturn();
        
        // Call V2 API with equivalent request
        String v2Request = """
            {
              "amount": 100.00,
              "currency_code": "USD",
              "payment_method": {
                "type": "card",
                "card": {
                  "card_number": "4111111111111111",
                  "expiry_month": 12,
                  "expiry_year": 2025,
                  "cvv": "123"
                }
              }
            }
            """;
        
        MvcResult v2Result = mockMvc.perform(post("/api/v2/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(v2Request))
            .andExpect(status().isCreated())
            .andReturn();
        
        // Verify both produce same business result
        JsonNode v1Response = objectMapper.readTree(
            v1Result.getResponse().getContentAsString());
        JsonNode v2Response = objectMapper.readTree(
            v2Result.getResponse().getContentAsString());
        
        assertEquals(v1Response.get("amount").asDouble(), 
            v2Response.get("transaction").get("amount").asDouble(), 0.01);
    }
    
    @Test
    public void testDeprecationHeaderPresent() throws Exception {
        mockMvc.perform(get("/api/v1/payments/health"))
            .andExpect(status().isOk())
            .andExpect(header().exists("X-API-Deprecated"))
            .andExpect(header().string("X-API-Version", "v1"))
            .andExpect(header().exists("X-API-Sunset"));
    }
}
```

### Phase 5: Client Communication and Documentation

#### API Documentation with Version Information

```java
@Configuration
public class OpenApiConfig {
    
    @Bean
    public GroupedOpenApi v1Api() {
        return GroupedOpenApi.builder()
            .group("v1")
            .pathsToMatch("/api/v1/**")
            .addOpenApiMethodFilter(method -> 
                method.isAnnotationPresent(Deprecated.class))
            .build();
    }
    
    @Bean
    public GroupedOpenApi v2Api() {
        return GroupedOpenApi.builder()
            .group("v2")
            .pathsToMatch("/api/v2/**")
            .build();
    }
    
    @Bean
    public GroupedOpenApi v3Api() {
        return GroupedOpenApi.builder()
            .group("v3")
            .pathsToMatch("/api/v3/**")
            .build();
    }
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("PayFlow API")
                .version("3.0.0")
                .description("Payment processing API with versioning support")
                .contact(new Contact()
                    .name("API Support")
                    .email("api-support@payflow.com")))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")))
            .externalDocs(new ExternalDocumentation()
                .description("Migration Guide")
                .url("https://docs.payflow.com/migration"));
    }
}
```

#### Version-Specific Documentation

```java
@RestController
@RequestMapping("/api/docs")
public class ApiDocumentationController {
    
    @GetMapping("/versions")
    public ResponseEntity<List<ApiVersionInfo>> getVersions() {
        List<ApiVersionInfo> versions = List.of(
            ApiVersionInfo.builder()
                .version("v1")
                .status("deprecated")
                .deprecationDate(LocalDate.of(2023, 1, 1))
                .sunsetDate(LocalDate.of(2024, 6, 30))
                .features(List.of("Basic payments", "Refunds"))
                .migrationPath("v3")
                .build(),
            
            ApiVersionInfo.builder()
                .version("v2")
                .status("stable")
                .features(List.of("Basic payments", "Refunds", "Webhooks"))
                .migrationPath("v3")
                .build(),
            
            ApiVersionInfo.builder()
                .version("v3")
                .status("current")
                .features(List.of(
                    "Basic payments", 
                    "Refunds", 
                    "Webhooks",
                    "Fraud detection",
                    "Recurring payments",
                    "Real-time notifications"
                ))
                .latest(true)
                .build()
        );
        
        return ResponseEntity.ok(versions);
    }
    
    @GetMapping("/migration/{from}/{to}")
    public ResponseEntity<MigrationGuide> getMigrationGuide(
            @PathVariable String from,
            @PathVariable String to) {
        
        MigrationGuide guide = migrationService.generateMigrationGuide(
            getCurrentClientId(), from, to);
        
        return ResponseEntity.ok(guide);
    }
}
```

---

## Results & Metrics

### Quantitative Outcomes

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Breaking Changes | 12/year | 0/year | 100% reduction |
| Client Support Tickets | 500/month | 50/month | 90% reduction |
| Version Migration Time | 6 months | 2 months | 67% faster |
| API Documentation Accuracy | 70% | 95% | 25% improvement |
| Version Routing Latency | N/A | 3ms | Minimal overhead |
| Active API Versions | 5 | 3 | 40% reduction |
| Client Satisfaction | 6.2/10 | 8.5/10 | 37% improvement |

### Qualitative Improvements

1. **Zero Breaking Changes**: New features deployed without breaking existing clients
2. **Clear Migration Path**: Clients understand how and when to upgrade
3. **Reduced Support Burden**: Automated deprecation warnings reduced support tickets
4. **Better Developer Experience**: Clear documentation and migration tools
5. **Operational Efficiency**: Reduced number of supported versions from 5 to 3

### Lessons Learned

#### What Worked Well

1. **Hybrid Versioning Strategy**: URL versioning for public APIs, header versioning for internal
2. **Shared Business Logic**: Minimized code duplication across versions
3. **Deprecation Warnings**: Proactive communication reduced client friction
4. **Migration Tools**: Automated comparison and guide generation helped clients
5. **Gradual Rollout**: Feature flags allowed safe version introduction

#### What Didn't Work

1. **Underestimated Client Inertia**: Enterprise clients slower to upgrade than expected
2. **SDK Complexity**: Maintaining multiple SDK versions was challenging
3. **Documentation Maintenance**: Keeping all versions in sync was difficult
4. **Testing Overhead**: Compatibility testing between versions was time-consuming
5. **Deprecation Timeline**: 12 months was too short for some enterprise clients

---

## Interview Takeaways

### Key Technical Concepts to Discuss

1. **API Versioning Strategies**: URL path, header, query parameter, content negotiation
2. **Semantic Versioning**: Major, minor, patch versioning rules
3. **Deprecation Patterns**: Sunset headers, warnings, migration timelines
4. **Backward Compatibility**: How to design APIs that don't break clients
5. **Mapper Pattern**: Converting between version-specific formats
6. **Shared Business Logic**: Separating version-specific from core logic
7. **API Gateway Routing**: Version-based routing and filtering
8. **Migration Testing**: Compatibility testing between versions

### What to Emphasize in Interviews

1. **Trade-off Analysis**: Why choose one versioning strategy over another
2. **Client Communication**: How to handle deprecation and migration
3. **Code Organization**: How to avoid duplication across versions
4. **Operational Overhead**: Cost of maintaining multiple versions
5. **Migration Strategy**: How to help clients upgrade smoothly
6. **Testing Strategy**: How to ensure compatibility between versions

### Sample Interview Answer Structure

"When implementing API versioning for our payment platform, we chose a hybrid approach using URL path versioning for public APIs and header versioning for internal services. We implemented a shared business logic layer with version-specific controllers and mappers to minimize code duplication. For deprecation, we added warning headers and provided automated migration tools. This approach allowed us to introduce new features without breaking existing clients, reducing support tickets by 90% while maintaining clear upgrade paths for our 50,000+ API consumers."

---

## Follow-up Questions

### Technical Questions

1. **How did you handle database schema changes across API versions?**
   - Discuss backward-compatible schema changes, migration scripts, data transformation

2. **What happens when a client sends a request to a deprecated version?**
   - Deprecation headers, warnings, redirect to newer version

3. **How did you test compatibility between versions?**
   - Automated comparison tests, migration testing, client integration tests

4. **How did you handle authentication across versions?**
   - Consistent auth strategy, token validation, version-independent auth

5. **What about SDK versioning?**
   - Version alignment with API, deprecation, migration tools

### Architectural Questions

6. **How did you decide when to create a new major version?**
   - Breaking changes criteria, backward compatibility rules

7. **How do you handle shared libraries across versions?**
   - Library versioning, dependency management, breaking changes

8. **What's your strategy for sunset dates?**
   - Usage metrics, client communication, forced migration

9. **How do you document multiple versions?**
   - Version-specific docs, migration guides, changelogs

### Leadership Questions

10. **How did you communicate deprecation to clients?**
    - Email campaigns, in-app warnings, documentation, support outreach

11. **How did you handle clients who refused to upgrade?**
    - Extended support, custom contracts, migration assistance

12. **What would you do differently next time?**
    - Longer deprecation timeline, better SDK strategy, more automated testing

---

## Related Topics

- [System Design: API Design](../../sections/14-system-design/README.md)
- [RESTful API Design](../../sections/13-spring-boot-advanced/README.md)
- [Behavioral: Handling Breaking Changes](../../sections/15-behavioral-scenarios/README.md)
- [Performance Optimization](../../sections/12-performance-optimization/README.md)
