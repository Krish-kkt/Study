# JWT Authentication

## Why JWT for APIs?

Traditional session-based auth stores login state on the **server** (in memory or DB). Every request checks this server-side session.

JWT (JSON Web Token) is **stateless** — the user's identity and roles are encoded in the token itself. The server doesn't store anything. This makes it perfect for:

- Microservices (no shared session store needed)
- Horizontal scaling (any server can verify any token)
- Mobile/SPA clients

```
Session-based (stateful):                JWT-based (stateless):

Client → Login → Server stores session   Client → Login → Server generates JWT
Client → Request + session cookie        Client → Request + JWT in header
Server → look up session in DB           Server → verify JWT signature (no DB call)
```

---

## JWT Structure

A JWT has three parts separated by dots:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbGljZUBleGFtcGxlLmNvbSIsInJvbGVzIjoiUk9MRV9VU0VSIiwiaWF0IjoxNjk4NzY4MDAwLCJleHAiOjE2OTg3NzE2MDB9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Part 1: Header (algorithm + token type)    → base64 decoded: {"alg":"HS256","typ":"JWT"}
Part 2: Payload (claims = user data)       → base64 decoded: {"sub":"alice@example.com","roles":"ROLE_USER","iat":...,"exp":...}
Part 3: Signature (verification of parts 1+2) → HMACSHA256(base64(header) + "." + base64(payload), secret)
```

The signature is what prevents tampering. If someone changes the payload, the signature won't match, and the server rejects the token.

---

## Dependencies

```xml
<!-- io.jsonwebtoken (JJWT) — the most popular Java JWT library -->

<!-- jjwt-api: the public API — contains the interfaces and classes you write code against
     (e.g. Jwts.builder(), Claims, JwtParser). Compile-time dependency. -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>

<!-- jjwt-impl: the actual implementation of the API — does the real work of signing,
     parsing, and validating tokens. You never call it directly; the API delegates to it
     via Java's ServiceLoader at runtime. Runtime-only so your code stays decoupled
     from the internals. -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>

<!-- jjwt-jackson: plugs Jackson (the JSON library Spring Boot already includes) into JJWT
     so it can serialize/deserialize the JWT payload (claims) to/from JSON.
     Without this, JJWT cannot read or write the body of the token. Runtime-only
     because it is wired up automatically, not called directly in your code. -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

---

## Step 1: Configuration Properties

```yaml
# application.yml
jwt:
  secret: "404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970"
  # A 256-bit (32-byte) hex secret — generate with: openssl rand -hex 32
  # In production: use environment variable or vault, NEVER commit to git
  
  expiration: 3600000       # 1 hour in milliseconds
  refresh-expiration: 86400000  # 24 hours for refresh token
```

---

## Step 2: JWT Token Provider

```java
@Component
// @Component — makes this a Spring bean, injectable everywhere
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    // @Value — injects value from application.yml/properties
    private String jwtSecret;

    @Value("${jwt.expiration}")
    private long jwtExpiration;

    @Value("${jwt.refresh-expiration}")
    private long refreshExpiration;

    private SecretKey getSigningKey() {
        // SecretKey — the cryptographic key used to sign and verify tokens
        byte[] keyBytes = Hex.decode(jwtSecret);
        // Hex.decode() — converts the hex string to bytes (from JJWT library)
        return Keys.hmacShaKeyFor(keyBytes);
        // Keys.hmacShaKeyFor() — creates a SecretKey suitable for HMAC-SHA algorithms
        // The key length determines which SHA variant is used (256/384/512)
    }

    public String generateToken(Authentication authentication) {
        // generateToken() — creates a JWT from an authenticated Authentication object
        // Called after successful login to give the user their token

        AppUserDetails userDetails = (AppUserDetails) authentication.getPrincipal();
        // getPrincipal() — the logged-in user's UserDetails

        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpiration);

        return Jwts.builder()
            // Jwts.builder() — entry point for building a JWT

            .subject(userDetails.getUsername())
            // subject() — the "sub" claim, typically the user's unique identifier (email)
            // This is what you'll extract later to identify the user

            .claim("userId", userDetails.getId())
            // claim(name, value) — adds a custom claim to the payload
            // Useful for putting extra user data in the token to avoid DB lookups

            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                // getAuthority() — returns the string like "ROLE_USER"
                .collect(Collectors.joining(",")))
            // Store roles as a comma-separated string in the token

            .issuedAt(now)
            // issuedAt() — sets the "iat" claim (issued at time)

            .expiration(expiryDate)
            // expiration() — sets the "exp" claim (when this token stops being valid)
            // JJWT automatically checks this during validation

            .signWith(getSigningKey())
            // signWith(key) — signs the token with your secret key
            // Algorithm is inferred from key type: SecretKey → HS256/HS384/HS512

            .compact();
            // compact() — serializes and signs the JWT, returning the final string
    }

    public String generateRefreshToken(String username) {
        // Refresh tokens have longer expiry, fewer claims
        return Jwts.builder()
            .subject(username)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + refreshExpiration))
            .signWith(getSigningKey())
            .compact();
    }

    public Claims extractAllClaims(String token) {
        // Claims — the payload of the JWT (all the key-value pairs)
        return Jwts.parser()
            // Jwts.parser() — entry point for parsing and verifying a JWT

            .verifyWith(getSigningKey())
            // verifyWith() — sets the key to use for signature verification
            // If signature doesn't match, throws SignatureException

            .build()
            // build() — creates the parser

            .parseSignedClaims(token)
            // parseSignedClaims() — parses and verifies the token
            // Throws:
            //   ExpiredJwtException if "exp" claim is in the past
            //   SignatureException if signature verification fails
            //   MalformedJwtException if token format is invalid

            .getPayload();
            // getPayload() — returns the Claims object (the JWT payload)
    }

    public String getUsernameFromToken(String token) {
        // Extract the subject (username/email) from the token
        return extractAllClaims(token).getSubject();
        // getSubject() — returns the "sub" claim
    }

    public Long getUserIdFromToken(String token) {
        return extractAllClaims(token).get("userId", Long.class);
        // get(claimName, type) — extracts a custom claim with type conversion
    }

    public boolean isTokenExpired(String token) {
        try {
            Date expiration = extractAllClaims(token).getExpiration();
            // getExpiration() — returns the "exp" claim as a Date
            return expiration.before(new Date());
            // before() — returns true if expiration date is in the past
        } catch (ExpiredJwtException e) {
            return true; // JJWT throws this instead of returning expired date in some versions
        }
    }

    public boolean validateToken(String token) {
        // validateToken() — returns true only if token is valid, not expired, and properly signed
        try {
            extractAllClaims(token); // This throws if invalid
            return true;
        } catch (ExpiredJwtException e) {
            log.warn("JWT expired: {}", e.getMessage());
            // Don't log the full token — it's a secret
        } catch (SignatureException e) {
            log.warn("Invalid JWT signature");
        } catch (MalformedJwtException e) {
            log.warn("Malformed JWT token");
        } catch (UnsupportedJwtException e) {
            log.warn("Unsupported JWT token");
        } catch (IllegalArgumentException e) {
            log.warn("JWT claims string is empty");
        }
        return false;
    }
}
```

---

## Step 3: JWT Authentication Filter

```java
// No @Component here — intentional. See SecurityConfig for why.
// Short reason: @Component would cause Spring Boot to register this filter in
// BOTH Tomcat's servlet filter chain AND Spring Security's internal FilterChainProxy,
// resulting in the filter executing twice per request.
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    // OncePerRequestFilter — runs exactly once per request, even on forwards/includes

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtTokenProvider jwtTokenProvider,
                                    UserDetailsService userDetailsService) {
        this.jwtTokenProvider = jwtTokenProvider;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
                                     throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            String username = jwtTokenProvider.getUsernameFromToken(token);
            // getUsernameFromToken() — decodes the "sub" claim to get the username

            // Only set auth if not already authenticated
            // (prevents double-processing if filters overlap)
            if (SecurityContextHolder.getContext().getAuthentication() == null) {

                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                // loadUserByUsername() — fetches fresh user data (including current roles/status)
                // This DB call ensures revoked users can't use old tokens
                // Trade-off: one DB call per request. Can cache this with Redis if needed.

                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,                    // principal
                        null,                           // credentials (null — token already verified)
                        userDetails.getAuthorities()    // authorities/roles
                    );
                // 3-arg constructor — creates an AUTHENTICATED token (isAuthenticated() = true)

                authentication.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                    // WebAuthenticationDetailsSource.buildDetails() — captures IP address, session ID
                    // Useful for security audit logs: "ROLE_ADMIN from 192.168.1.5 deleted product #42"
                );

                SecurityContextHolder.getContext().setAuthentication(authentication);
                // setAuthentication() — stores auth in thread-local storage
                // All downstream code (service layer, controllers) can call
                // SecurityContextHolder.getContext().getAuthentication() to get the user
            }
        }

        filterChain.doFilter(request, response);
        // Always call this — even if we didn't set auth
        // If no auth was set, ExceptionTranslationFilter will handle the 401 later
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        // HttpHeaders.AUTHORIZATION — constant for "Authorization" header name

        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
            // Remove "Bearer " prefix (7 characters) to get the raw token
        }
        return null; // No token present
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // shouldNotFilter() — return true to completely skip this filter for matching requests
        AntPathMatcher pathMatcher = new AntPathMatcher();
        // AntPathMatcher — utility to match URL patterns like /api/auth/**
        
        String path = request.getServletPath();
        return pathMatcher.match("/api/auth/**", path)
            || pathMatcher.match("/public/**", path)
            || pathMatcher.match("/actuator/health", path);
        // These paths don't need JWT validation — they're public endpoints
    }
}
```

---

## Step 4: Login Endpoint

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    @PostMapping("/login")
    public ResponseEntity<JwtResponse> login(@RequestBody @Valid LoginRequest request) {
        
        // Manually trigger authentication (this is what the form login filter does automatically)
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getEmail(),
                request.getPassword()
                // 2-arg constructor — unauthenticated token (request to authenticate)
            )
        );
        // authenticate() — throws AuthenticationException if credentials are wrong
        // Returns a fully populated Authentication if credentials are correct

        String accessToken = jwtTokenProvider.generateToken(authentication);
        // generateToken() — creates the JWT containing user identity and roles

        String refreshToken = jwtTokenProvider.generateRefreshToken(
            authentication.getName()
            // getName() — returns the username (from UserDetails.getUsername())
        );

        AppUserDetails userDetails = (AppUserDetails) authentication.getPrincipal();

        return ResponseEntity.ok(new JwtResponse(
            accessToken,
            refreshToken,
            userDetails.getId(),
            userDetails.getUsername()
        ));
    }

    @PostMapping("/refresh")
    public ResponseEntity<JwtResponse> refresh(@RequestBody RefreshTokenRequest request) {
        
        String refreshToken = request.getRefreshToken();

        if (!jwtTokenProvider.validateToken(refreshToken)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        String username = jwtTokenProvider.getUsernameFromToken(refreshToken);
        UserDetails userDetails = userDetailsService.loadUserByUsername(username);

        UsernamePasswordAuthenticationToken authentication =
            new UsernamePasswordAuthenticationToken(
                userDetails, null, userDetails.getAuthorities()
            );

        String newAccessToken = jwtTokenProvider.generateToken(authentication);

        return ResponseEntity.ok(new JwtResponse(newAccessToken, refreshToken,
            ((AppUserDetails) userDetails).getId(), username));
    }
}
```

---

## Step 5: Security Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // JwtTokenProvider is @Component — safe to inject here. It has no filter chain side effects.
    // We do NOT inject JwtAuthenticationFilter — we declare it as a @Bean below instead.
    private final JwtTokenProvider jwtTokenProvider;
    private final AppUserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public SecurityConfig(JwtTokenProvider jwtTokenProvider,
                          AppUserDetailsService userDetailsService,
                          PasswordEncoder passwordEncoder) {
        this.jwtTokenProvider = jwtTokenProvider;
        this.userDetailsService = userDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    // WHY @Bean here and NOT @Component on the filter class:
    //
    // Spring Boot auto-configuration scans all @Component beans that implement
    // javax.servlet.Filter and registers each one in Tomcat's servlet filter chain
    // via FilterRegistrationBean — automatically, without you asking.
    //
    // Spring Security's FilterChainProxy (the thing that runs YOUR security rules)
    // lives inside Tomcat's chain as a single DelegatingFilterProxy entry.
    // If JwtAuthenticationFilter is also registered directly in Tomcat's chain,
    // it runs BEFORE DelegatingFilterProxy — outside Spring Security's managed scope.
    // That means Spring Security's SecurityContextHolderFilter (which governs the
    // SecurityContext lifecycle) can wipe the authentication you just set.
    //
    // Declaring the filter as a @Bean here means Spring Boot never sees it as a
    // candidate for auto-registration. The ONLY place it enters a filter chain
    // is via the explicit addFilterBefore() call below — inside FilterChainProxy,
    // where exception handling, SecurityContext management, and audit logging
    // all work correctly.
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter(jwtTokenProvider, userDetailsService);
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            // CSRF disabled for REST API — stateless, no browser session to exploit

            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // cors() — enable CORS handling (details in 08_CSRF_and_CORS.md)

            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // STATELESS — Spring Security will NOT create or use HTTP sessions
            // Every request must carry a valid JWT

            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                // Login and registration are public
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                // Public catalog browsing
                .anyRequest().authenticated()
                // Everything else requires a valid JWT
            )

            .authenticationProvider(authenticationProvider())
            // Register the DaoAuthenticationProvider that uses our UserDetailsService

            .addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)
            // addFilterBefore — places JwtAuthenticationFilter INSIDE FilterChainProxy,
            // just before the form login filter. This is the ONLY place the filter
            // is wired in — Tomcat's servlet chain never sees it directly.
            // Spring intercepts the jwtAuthenticationFilter() call and returns the
            // singleton bean (not a new instance each time).

            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((request, response, exception) -> {
                    // authenticationEntryPoint — what to do when authentication fails
                    // Default for web: redirect to /login (wrong for REST APIs)
                    // We override to return JSON 401 instead
                    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.getWriter().write("{\"error\": \"Unauthorized\"}");
                })
                .accessDeniedHandler((request, response, exception) -> {
                    // accessDeniedHandler — what to do when user is authenticated but lacks permission
                    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    response.getWriter().write("{\"error\": \"Access denied\"}");
                })
            );

        return http.build();
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        // setUserDetailsService() — where to load user data from

        provider.setPasswordEncoder(passwordEncoder);
        // setPasswordEncoder() — how to compare passwords during login

        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        // AuthenticationManager bean needed for manual authentication in the login endpoint
        return config.getAuthenticationManager();
    }
}
```

---

## Token Revocation (The Stateless Problem)

JWTs are stateless — once issued, you can't invalidate them until they expire. This is a real limitation. Production strategies:

### Strategy 1: Short-lived access tokens + refresh tokens

```
Access token:  15 minutes expiry  (if stolen, expires quickly)
Refresh token: 7 days expiry      (stored in HttpOnly cookie, harder to steal)

On logout:
- Client discards access token
- Server invalidates refresh token in DB (so it can't be exchanged for new access token)
```

### Strategy 2: Token blocklist (Redis)

```java
@Service
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;
    // RedisTemplate — Spring's abstraction for Redis operations

    public void blacklistToken(String token, long expiryMs) {
        String key = "blacklist:" + token;
        // Store the token hash in Redis with expiry matching the token's own expiry
        // When the token would naturally expire, Redis auto-deletes the key
        redisTemplate.opsForValue().set(key, "revoked", Duration.ofMillis(expiryMs));
        // opsForValue().set(key, value, duration) — sets a string value with expiry
    }

    public boolean isBlacklisted(String token) {
        return Boolean.TRUE.equals(redisTemplate.hasKey("blacklist:" + token));
        // hasKey() — returns true if the key exists in Redis
        // If exists: token was revoked
    }
}

// In JwtAuthenticationFilter, add the blacklist check:
if (token != null && jwtTokenProvider.validateToken(token) 
        && !tokenBlacklistService.isBlacklisted(token)) {
    // Only authenticate if token is valid AND not blacklisted
}
```

---

## Complete Flow Diagram

```
1. POST /api/auth/login
   { email: "alice@example.com", password: "secret" }
                ↓
2. AuthController.login()
   authenticationManager.authenticate(UsernamePasswordAuthenticationToken)
                ↓
3. DaoAuthenticationProvider
   userDetailsService.loadUserByUsername("alice@example.com")
   passwordEncoder.matches("secret", "$2a$12$...")
                ↓
4. Authentication SUCCESS
   Returns Authentication with UserDetails + GrantedAuthorities
                ↓
5. jwtTokenProvider.generateToken(authentication)
   Returns: "eyJhbGciOiJIUzI1NiJ9...."
                ↓
6. Client stores token, uses on all future requests:
   GET /api/orders  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9....
                ↓
7. JwtAuthenticationFilter.doFilterInternal()
   Extracts token from Authorization header
   jwtTokenProvider.validateToken(token) → true
   jwtTokenProvider.getUsernameFromToken(token) → "alice@example.com"
   userDetailsService.loadUserByUsername("alice@example.com")
   Creates UsernamePasswordAuthenticationToken(userDetails, null, authorities)
   SecurityContextHolder.getContext().setAuthentication(authentication)
                ↓
8. Request reaches OrderController
   @AuthenticationPrincipal AppUserDetails user → alice's UserDetails
   Returns alice's orders
```

---

## Common Mistakes

### Mistake 1: Storing sensitive data in JWT payload

```java
// Bad — JWT payload is base64-encoded, NOT encrypted. Anyone can decode it.
.claim("password", user.getPassword()) // NEVER
.claim("ssn", user.getSsn())           // NEVER
.claim("creditCard", user.getCard())   // NEVER

// Good — only store non-sensitive identifiers
.subject(user.getEmail())
.claim("userId", user.getId())
.claim("roles", "ROLE_USER")
```

### Mistake 2: Weak or missing secret key

```java
// Bad — short, guessable key
@Value("${jwt.secret:secret}") // Default value is "secret" — trivially brutable
private String secret;

// Good — use a cryptographically random 256-bit key
// Generate: openssl rand -hex 32
// Store in environment variable or vault, not in code/config files
```

### Mistake 3: Not validating the token in the filter

```java
// Bad — trusting the token without verification
String username = jwtTokenProvider.getUsernameFromToken(token);
// If token is tampered, getUsernameFromToken might return attacker-controlled username

// Good — always validate before trusting any claims
if (jwtTokenProvider.validateToken(token)) {
    String username = jwtTokenProvider.getUsernameFromToken(token);
    // Only extract claims AFTER validation
}
```
