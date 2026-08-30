# Advanced Security Patterns Deep-Dive

## 📋 Question
Explain advanced security patterns including OAuth2/OpenID Connect implementation, JWT (JSON Web Token) best practices, Spring Security internals, and security architecture for microservices. How do you handle token refresh, token revocation, and cross-service authentication? Provide real-world implementation examples.

## Difficulty: 🔴 Expert

---

## 💡 Answer

Security in modern applications requires understanding authentication protocols, token management, and secure communication patterns. Advanced security patterns address complex scenarios like microservices authentication, token lifecycle management, and defense against sophisticated attacks.

### OAuth2 and OpenID Connect

OAuth2 is an authorization framework, while OpenID Connect (OIDC) adds authentication on top of OAuth2.

**Authorization Code Flow (PKCE):**

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .oauth2Login()
                .authorizationEndpoint()
                    .baseUri("/oauth2/authorize")
                    .authorizationRequestRepository(
                        cookieAuthorizationRequestRepository()
                    )
                .and()
                .redirectionEndpoint()
                    .baseUri("/oauth2/callback/*")
                .and()
                .userInfoEndpoint()
                    .userService(customOAuth2UserService())
            .and()
            .logout()
                .logoutSuccessUrl("/")
            .and()
            .authorizeRequests()
                .antMatchers("/", "/login", "/oauth2/**").permitAll()
                .anyRequest().authenticated();
    }
    
    @Bean
    public AuthorizationRequestRepository<OAuth2AuthorizationRequest>
            cookieAuthorizationRequestRepository() {
        return new HttpCookieOAuth2AuthorizationRequestRepository();
    }
    
    @Bean
    public CustomOAuth2UserService customOAuth2UserService() {
        return new CustomOAuth2UserService();
    }
}
```

**Custom OAuth2 User Service:**

```java
@Service
public class CustomOAuth2UserService implements 
    OAuth2UserService<OAuth2UserRequest, OAuth2User> {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) 
            throws OAuth2AuthenticationException {
        
        OAuth2User oauth2User = delegate.loadUser(userRequest);
        
        String registrationId = userRequest.getClientRegistration()
            .getRegistrationId();
        
        String email = oauth2User.getAttribute("email");
        String name = oauth2User.getAttribute("name");
        
        // Find or create user
        User user = userRepository.findByEmail(email)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setEmail(email);
                newUser.setName(name);
                newUser.setProvider(registrationId);
                newUser.setProviderId(oauth2User.getName());
                return userRepository.save(newUser);
            });
        
        return new CustomOAuth2User(oauth2User, user);
    }
}

public class CustomOAuth2User implements OAuth2User {
    
    private final OAuth2User delegate;
    private final User user;
    
    public CustomOAuth2User(OAuth2User delegate, User user) {
        this.delegate = delegate;
        this.user = user;
    }
    
    @Override
    public Map<String, Object> getAttributes() {
        return delegate.getAttributes();
    }
    
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return AuthorityUtils.createAuthorityList("ROLE_USER");
    }
    
    @Override
    public String getName() {
        return user.getId().toString();
    }
    
    public User getUser() {
        return user;
    }
}
```

### JWT Implementation and Best Practices

JWTs are stateless tokens that contain claims. They must be properly validated and secured.

**JWT Utility Class:**

```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.*;
import java.security.*;
import java.util.*;

@Component
public class JwtTokenProvider {
    
    private final Key secretKey;
    private final long jwtExpirationMs;
    
    public JwtTokenProvider(
        @Value("${app.jwt.secret}") String secret,
        @Value("${app.jwt.expiration:86400000}") long jwtExpirationMs
    ) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes());
        this.jwtExpirationMs = jwtExpirationMs;
    }
    
    public String generateToken(Authentication authentication) {
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpirationMs);
        
        return Jwts.builder()
            .setSubject(Long.toString(userPrincipal.getId()))
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(secretKey, SignatureAlgorithm.HS512)
            .compact();
    }
    
    public Long getUserIdFromJWT(String token) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(secretKey)
            .build()
            .parseClaimsJws(token)
            .getBody();
        
        return Long.parseLong(claims.getSubject());
    }
    
    public boolean validateToken(String authToken) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJws(authToken);
            return true;
        } catch (SecurityException ex) {
            // Invalid signature
        } catch (MalformedJwtException ex) {
            // Invalid token
        } catch (ExpiredJwtException ex) {
            // Expired token
        } catch (UnsupportedJwtException ex) {
            // Unsupported token
        } catch (IllegalArgumentException ex) {
            // Empty token
        }
        return false;
    }
}
```

**JWT Authentication Filter:**

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private CustomUserDetailsService customUserDetailsService;
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        try {
            String jwt = getJwtFromRequest(request);
            
            if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)) {
                Long userId = tokenProvider.getUserIdFromJWT(jwt);
                
                UserDetails userDetails = customUserDetailsService
                    .loadUserById(userId);
                
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );
                
                authentication.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                );
                
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            logger.error("Could not set user authentication in security context", ex);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

**JWT Best Practices:**

```java
// ✅ Use strong secret keys (256-bit or more)
// ❌ Don't use weak secrets like "secret" or "password"
private final Key secretKey = Keys.hmacShaKeyFor(
    SecureRandom.getInstanceStrong().generateSeed(32)
);

// ✅ Use short expiration times
private final long jwtExpirationMs = 15 * 60 * 1000;  // 15 minutes

// ✅ Implement refresh tokens
public String generateRefreshToken(User user) {
    Date now = new Date();
    Date expiryDate = new Date(now.getTime() + 7 * 24 * 60 * 60 * 1000);  // 7 days
    
    return Jwts.builder()
        .setSubject(user.getId().toString())
        .setIssuedAt(now)
        .setExpiration(expiryDate)
        .signWith(secretKey, SignatureAlgorithm.HS512)
        .compact();
}

// ✅ Include relevant claims
public String generateTokenWithClaims(User user) {
    Map<String, Object> claims = new HashMap<>();
    claims.put("email", user.getEmail());
    claims.put("role", user.getRole());
    claims.put("tenantId", user.getTenantId());
    
    return Jwts.builder()
        .setClaims(claims)
        .setSubject(user.getId().toString())
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
        .signWith(secretKey, SignatureAlgorithm.HS512)
        .compact();
}

// ✅ Validate issuer and audience
public boolean validateToken(String token) {
    try {
        Jwts.parserBuilder()
            .setSigningKey(secretKey)
            .requireIssuer("https://myapp.com")
            .requireAudience("myapp-api")
            .build()
            .parseClaimsJws(token);
        return true;
    } catch (Exception e) {
        return false;
    }
}
```

### Token Refresh and Revocation

**Refresh Token Implementation:**

```java
@Service
@Transactional
public class TokenRefreshService {
    
    @Autowired
    private RefreshTokenRepository refreshTokenRepository;
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    public RefreshToken createRefreshToken(Long userId) {
        RefreshToken refreshToken = new RefreshToken();
        refreshToken.setUser(userRepository.findById(userId).get());
        refreshToken.setExpiryDate(Instant.now()
            .plus(7, ChronoUnit.DAYS));
        refreshToken.setToken(UUID.randomUUID().toString());
        
        refreshToken = refreshTokenRepository.save(refreshToken);
        return refreshToken;
    }
    
    public RefreshToken verifyExpiration(RefreshToken token) {
        if (token.getExpiryDate().compareTo(Instant.now()) < 0) {
            refreshTokenRepository.delete(token);
            throw new TokenRefreshException(
                token.getToken(), "Refresh token was expired. Please make a new signin request"
            );
        }
        return token;
    }
    
    @Transactional
    public String refreshAccessToken(RefreshTokenRequest request) {
        String requestRefreshToken = request.getRefreshToken();
        
        return refreshTokenRepository.findByToken(requestRefreshToken)
            .map(this::verifyExpiration)
            .map(RefreshToken::getUser)
            .map(user -> {
                String token = tokenProvider.generateTokenFromUserId(user.getId());
                return token;
            })
            .orElseThrow(() -> new TokenRefreshException(
                requestRefreshToken, "Refresh token is not in database!"
            ));
    }
}

@Entity
public class RefreshToken {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String token;
    
    @ManyToOne
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    private Instant expiryDate;
    
    // Getters and setters
}
```

**Token Revocation:**

```java
@Service
public class TokenRevocationService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private static final String REVOKED_TOKEN_PREFIX = "revoked:";
    private static final long REVOCATION_TTL = 7 * 24 * 60 * 60;  // 7 days
    
    public void revokeToken(String jti) {
        redisTemplate.opsForValue().set(
            REVOKED_TOKEN_PREFIX + jti,
            "revoked",
            REVOCATION_TTL,
            TimeUnit.SECONDS
        );
    }
    
    public boolean isTokenRevoked(String jti) {
        return Boolean.TRUE.equals(
            redisTemplate.hasKey(REVOKED_TOKEN_PREFIX + jti)
        );
    }
    
    public void revokeAllUserTokens(Long userId) {
        // Find all active tokens for user and revoke them
        // This requires tracking issued tokens
    }
}

// Include in JWT validation
public boolean validateToken(String token) {
    try {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(secretKey)
            .build()
            .parseClaimsJws(token)
            .getBody();
        
        String jti = claims.getId();
        
        // Check if token is revoked
        if (tokenRevocationService.isTokenRevoked(jti)) {
            return false;
        }
        
        return true;
    } catch (Exception e) {
        return false;
    }
}
```

### Microservices Security Architecture

**Service-to-Service Authentication with mTLS:**

```yaml
# Kubernetes mTLS configuration with Istio
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT  # Enforce mTLS for all services

---
apiVersion: networking.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
spec:
  selector:
    matchLabels:
      app: order-service
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/default/sa/payment-service  # Only payment service
  - to:
    - operation:
        methods: ["POST"]
        paths: ["/api/orders/*"]
```

**JWT Propagation Across Services:**

```java
@Component
public class JwtPropagationInterceptor implements ClientHttpRequestInterceptor {
    
    @Override
    public ClientHttpResponse intercept(
        HttpRequest request,
        byte[] body,
        ClientHttpRequestExecution execution
    ) throws IOException {
        
        Authentication authentication = SecurityContextHolder.getContext()
            .getAuthentication();
        
        if (authentication != null && authentication.getCredentials() instanceof String) {
            String token = (String) authentication.getCredentials();
            request.getHeaders().add("Authorization", "Bearer " + token);
        }
        
        return execution.execute(request, body);
    }
}

@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setInterceptors(
            Collections.singletonList(new JwtPropagationInterceptor())
        );
        return restTemplate;
    }
}
```

**API Gateway Security:**

```java
@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {
    
    @Bean
    public SecurityWebFilterChain securityWebFilterChain(
        ServerHttpSecurity http
    ) {
        http
            .csrf().disable()
            .authorizeExchange()
                .pathMatchers("/auth/**").permitAll()
                .pathMatchers("/public/**").permitAll()
                .anyExchange().authenticated()
            .and()
            .oauth2ResourceServer()
                .jwt()
                .jwtAuthenticationConverter(jwtAuthenticationConverter());
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        
        JwtAuthenticationConverter jwtAuthenticationConverter = 
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter
            .setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        
        return jwtAuthenticationConverter;
    }
}
```

### Security Best Practices

**Password Hashing:**

```java
@Service
public class PasswordService {
    
    private final PasswordEncoder passwordEncoder;
    
    public PasswordService() {
        // Use Argon2id (recommended) or BCrypt
        int memory = 16384;
        int parallelism = 1;
        int iterations = 2;
        int length = 16;
        int saltLength = 16;
        
        this.passwordEncoder = new Argon2PasswordEncoder(
            memory, parallelism, iterations, length, saltLength
        );
    }
    
    public String hashPassword(String rawPassword) {
        return passwordEncoder.encode(rawPassword);
    }
    
    public boolean matches(String rawPassword, String encodedPassword) {
        return passwordEncoder.matches(rawPassword, encodedPassword);
    }
}
```

**CSRF Protection:**

```java
@Configuration
@EnableWebSecurity
public class CsrfConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf()
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .and()
            // For stateless APIs, CSRF can be disabled
            // .csrf().disable()
            // But only if using JWT or similar
    }
}
```

**Rate Limiting:**

```java
@Component
public class RateLimitingFilter extends OncePerRequestFilter {
    
    private final RateLimiter rateLimiter;
    
    public RateLimitingFilter() {
        // 100 requests per minute
        this.rateLimiter = RateLimiter.create(100.0 / 60.0);
    }
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        if (!rateLimiter.tryAcquire()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Rate limit exceeded");
            return;
        }
        
        filterChain.doFilter(request, response);
    }
}
```

### Code Example: Complete Security Demo

```java
package com.interview.advanced.security;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.web.builders.*;
import org.springframework.security.config.annotation.web.configuration.*;
import org.springframework.security.core.*;
import org.springframework.security.core.authority.*;
import org.springframework.security.core.userdetails.*;
import org.springframework.security.crypto.password.*;
import org.springframework.security.oauth2.server.resource.authentication.*;
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.*;
import java.security.*;
import java.util.*;

// === JWT TOKEN PROVIDER ===
@Component
public class JwtTokenProvider {
    
    private final Key secretKey;
    private final long jwtExpirationMs = 900000;  // 15 minutes
    
    public JwtTokenProvider(@Value("${app.jwt.secret}") String secret) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes());
    }
    
    public String generateToken(UserDetails userDetails) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpirationMs);
        
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities());
        
        return Jwts.builder()
            .setClaims(claims)
            .setSubject(userDetails.getUsername())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .setId(UUID.randomUUID().toString())  // JWT ID for revocation
            .signWith(secretKey, SignatureAlgorithm.HS512)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .requireIssuer("https://myapp.com")
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
    
    public String getUsernameFromToken(String token) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(secretKey)
            .build()
            .parseClaimsJws(token)
            .getBody();
        
        return claims.getSubject();
    }
}

// === SECURITY CONFIGURATION ===
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new Argon2PasswordEncoder();
    }
    
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter();
    }
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder());
    }
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeRequests()
                .antMatchers("/api/auth/**").permitAll()
                .antMatchers("/api/public/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .addFilterBefore(jwtAuthenticationFilter(), 
                UsernamePasswordAuthenticationFilter.class);
    }
}

// === JWT AUTHENTICATION FILTER ===
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(
        javax.servlet.http.HttpServletRequest request,
        javax.servlet.http.HttpServletResponse response,
        javax.servlet.FilterChain filterChain
    ) throws java.io.IOException, javax.servlet.ServletException {
        
        try {
            String jwt = getJwtFromRequest(request);
            
            if (jwt != null && tokenProvider.validateToken(jwt)) {
                String username = tokenProvider.getUsernameFromToken(jwt);
                
                UserDetails userDetails = userDetailsService
                    .loadUserByUsername(username);
                
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );
                
                authentication.setDetails(
                    new org.springframework.security.web.authentication
                        .WebAuthenticationDetailsSource()
                        .buildDetails(request)
                );
                
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            logger.error("Could not set user authentication", ex);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String getJwtFromRequest(javax.servlet.http.HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}

// === CUSTOM USER DETAILS SERVICE ===
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) 
            throws UsernameNotFoundException {
        
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found with username: " + username
            ));
        
        return UserPrincipal.create(user);
    }
    
    public UserDetails loadUserById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
        
        return UserPrincipal.create(user);
    }
}

// === USER PRINCIPAL ===
public class UserPrincipal implements UserDetails {
    
    private Long id;
    private String username;
    private String email;
    private String password;
    private Collection<? extends GrantedAuthority> authorities;
    
    // Constructor, getters, and UserDetails implementation methods
}

// === SECURED SERVICE ===
@Service
public class SecureService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public void adminOnlyMethod() {
        // Only accessible by users with ADMIN role
    }
    
    @PreAuthorize("hasAuthority('ORDER_WRITE')")
    public void orderWriteMethod() {
        // Only accessible by users with ORDER_WRITE permission
    }
    
    @PreAuthorize("#userId == authentication.principal.id")
    public void getUserData(Long userId) {
        // Only accessible if userId matches authenticated user
    }
}
```

### ⚠️ Common Mistakes

**1. Storing sensitive data in JWT**

```java
// ❌ Don't store sensitive data in JWT
claims.put("ssn", user.getSsn());
claims.put("creditCard", user.getCreditCard());

// ✅ Store only non-sensitive, reference data
claims.put("userId", user.getId());
claims.put("email", user.getEmail());
```

**2. Using weak secrets**

```java
// ❌ Weak secret
private final String secret = "secret";

// ✅ Strong secret (256-bit)
private final Key secretKey = Keys.hmacShaKeyFor(
    SecureRandom.getInstanceStrong().generateSeed(32)
);
```

**3. Not validating JWT properly**

```java
// ❌ Not validating signature or expiration
Claims claims = Jwts.parser().parseClaimsJws(token).getBody();

// ✅ Full validation
Jwts.parserBuilder()
    .setSigningKey(secretKey)
    .requireIssuer("https://myapp.com")
    .requireAudience("myapp-api")
    .setAllowedClockSkewSeconds(30)
    .build()
    .parseClaimsJws(token);
```

### ⚡ Performance Tip

**Use asymmetric keys for service-to-service communication** to avoid shared secrets. RS256 (RSA Signature with SHA-256) is more secure than HS256 for distributed systems.

**Implement token caching** to reduce validation overhead for frequently used tokens.

### 🔧 Security Internals

JWTs consist of three parts: header, payload, and signature, all base64url encoded. The signature is created by signing the header and payload with a secret (HS256) or private key (RS256). JWTs are stateless but can be revoked by maintaining a blacklist of revoked token IDs (jti).

OAuth2 defines several grant types: Authorization Code (for server-side apps), Implicit (deprecated), Client Credentials (for service-to-service), and Resource Owner Password Credentials (legacy). PKCE (Proof Key for Code Exchange) adds security to the Authorization Code flow for public clients.

---

## 🎯 Interviewer Insight

Security questions test your understanding of authentication/authorization protocols and secure coding practices. Interviewers want to see that you:
- Understand OAuth2 flows and when to use each
- Know JWT best practices (expiration, revocation, validation)
- Can implement secure microservices communication
- Understand the difference between authentication and authorization
- Know common security vulnerabilities and how to prevent them

Strong candidates can explain the difference between OAuth2 and OIDC, and demonstrate proper JWT validation with issuer and audience checks.

## ⚡ Follow-up Questions

1. What is the difference between OAuth2 and OpenID Connect?
2. How do you handle JWT token revocation in a stateless system?
3. What are the security implications of storing JWTs in localStorage vs cookies?

## 📚 Related Topics

- [Java Security](../../sections/11-java-security/README.md)
- [Spring Security](../../sections/13-framework-integration/README.md)
- [System Design](../../sections/14-system-design/README.md)
