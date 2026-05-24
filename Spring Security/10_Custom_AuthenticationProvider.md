# Custom AuthenticationProvider

## Why Write a Custom AuthenticationProvider?

Spring's built-in `DaoAuthenticationProvider` handles the standard flow: username + password → check DB → verify hash. But real production systems often have non-standard requirements:

- **LDAP / Active Directory** — authenticate against a corporate directory, not your DB
- **OTP / MFA** — verify a one-time password in addition to the main password
- **API key authentication** — services authenticate with a pre-shared key, not a user+password
- **Certificate-based authentication** — authenticate using a client X.509 certificate
- **Custom token format** — authenticate with a proprietary token format (legacy system)

For any of these, you implement your own `AuthenticationProvider`.

---

## The AuthenticationProvider Interface

```java
public interface AuthenticationProvider {

    Authentication authenticate(Authentication authentication)
            throws AuthenticationException;
    // authenticate() — the core method: try to authenticate the given token
    // Input:  An Authentication object (contains credentials to verify)
    // Output: A fully authenticated Authentication object (with principal + authorities)
    // Throws: AuthenticationException if authentication fails (wrong password, locked account, etc.)
    //         Return null if this provider can't handle the given Authentication type
    //         (ProviderManager then tries the next provider)

    boolean supports(Class<?> authentication);
    // supports() — tells ProviderManager whether this provider handles the given type
    // ProviderManager calls this before calling authenticate() to avoid unnecessary processing
    // Example: return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
}
```

---

## Example 1: API Key Authentication

API key auth is common for service-to-service calls. A client includes a pre-shared key in the request header; the server validates it.

### Step 1: Custom Authentication Token

```java
// The token that represents an API key authentication request/result
public class ApiKeyAuthenticationToken extends AbstractAuthenticationToken {
    // AbstractAuthenticationToken — base class that implements most of Authentication
    // Extend this to avoid implementing all Authentication methods manually

    private final String apiKey;
    // The actual API key string

    private final Object principal;
    // The service/client identified by this API key (a ServiceDetails object or String)

    // Constructor for UNAUTHENTICATED token (before verification)
    public ApiKeyAuthenticationToken(String apiKey) {
        super(null);
        // super(null) — passing null for authorities = not yet authenticated
        this.apiKey = apiKey;
        this.principal = null;
        setAuthenticated(false);
        // setAuthenticated(false) — marks this as "not yet verified"
    }

    // Constructor for AUTHENTICATED token (after verification)
    public ApiKeyAuthenticationToken(Object principal, Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        // super(authorities) — passing authorities = authenticated token
        this.apiKey = null;
        this.principal = principal;
        super.setAuthenticated(true);
        // super.setAuthenticated(true) — marks as verified (use super, not this, to bypass the check below)
    }

    @Override
    public Object getCredentials() {
        return apiKey;
        // getCredentials() — returns the API key string (the secret used to authenticate)
    }

    @Override
    public Object getPrincipal() {
        return principal;
        // getPrincipal() — returns who this token represents after authentication
    }

    @Override
    public void setAuthenticated(boolean authenticated) {
        // Prevent external code from marking this token as authenticated
        // Only our constructor should be able to create authenticated tokens
        if (authenticated) {
            throw new IllegalArgumentException(
                "Cannot set this token to trusted - use constructor instead");
        }
        super.setAuthenticated(false);
    }
}
```

### Step 2: Custom AuthenticationProvider

```java
@Component
public class ApiKeyAuthenticationProvider implements AuthenticationProvider {

    private final ApiKeyRepository apiKeyRepository;
    // Your repository for looking up API keys in the database

    public ApiKeyAuthenticationProvider(ApiKeyRepository apiKeyRepository) {
        this.apiKeyRepository = apiKeyRepository;
    }

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {
        // authenticate() — verify the API key and return an authenticated token

        String apiKey = (String) authentication.getCredentials();
        // getCredentials() — the API key string from the request

        if (apiKey == null || apiKey.isBlank()) {
            throw new BadCredentialsException("API key is required");
            // BadCredentialsException — thrown when credentials are missing or wrong
        }

        // Look up the API key in your database
        ApiKeyDetails keyDetails = apiKeyRepository.findByKey(apiKey)
            .orElseThrow(() -> new BadCredentialsException("Invalid API key"));
            // If not found → credentials are invalid → 401

        // Check if the key is still active
        if (!keyDetails.isActive()) {
            throw new DisabledException("API key is disabled");
            // DisabledException — thrown when the account/key is disabled
        }

        // Check if the key has expired
        if (keyDetails.getExpiresAt().isBefore(Instant.now())) {
            throw new CredentialsExpiredException("API key has expired");
            // CredentialsExpiredException — thrown when credentials have expired
        }

        // Build the service principal from the key details
        ServicePrincipal principal = new ServicePrincipal(
            keyDetails.getServiceName(),
            keyDetails.getId()
        );

        // Create roles from the API key's permitted scopes
        List<GrantedAuthority> authorities = keyDetails.getScopes().stream()
            .map(scope -> new SimpleGrantedAuthority("SCOPE_" + scope))
            // SimpleGrantedAuthority — wraps a string as a GrantedAuthority
            .collect(Collectors.toList());

        // Log API key usage for audit trail
        apiKeyRepository.updateLastUsed(keyDetails.getId(), Instant.now());

        // Return authenticated token (3-arg constructor = authenticated)
        return new ApiKeyAuthenticationToken(principal, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        // supports() — only handle ApiKeyAuthenticationToken, not other types
        return ApiKeyAuthenticationToken.class.isAssignableFrom(authentication);
        // isAssignableFrom() — returns true if authentication is ApiKeyAuthenticationToken
        // or any subclass of it
    }
}
```

### Step 3: Filter to Extract API Key from Request

```java
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {
    // OncePerRequestFilter — runs exactly once per request

    private static final String API_KEY_HEADER = "X-API-Key";
    // Convention: API keys go in the X-API-Key header

    private final AuthenticationManager authenticationManager;

    public ApiKeyAuthenticationFilter(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
                                     throws ServletException, IOException {

        String apiKey = request.getHeader(API_KEY_HEADER);
        // getHeader() — extract the API key from the request header

        if (apiKey == null) {
            // No API key present — this request might use JWT or be anonymous
            filterChain.doFilter(request, response);
            // Pass to next filter without doing anything
            return;
        }

        try {
            // Create an unauthenticated token
            ApiKeyAuthenticationToken authRequest = new ApiKeyAuthenticationToken(apiKey);

            // Delegate to AuthenticationManager (which uses our ApiKeyAuthenticationProvider)
            Authentication authResult = authenticationManager.authenticate(authRequest);
            // authenticate() — calls ApiKeyAuthenticationProvider.authenticate() internally

            // Store the authenticated result in SecurityContext
            SecurityContextHolder.getContext().setAuthentication(authResult);
            // setAuthentication() — the user is now "logged in" for this request

        } catch (AuthenticationException e) {
            // Invalid API key
            SecurityContextHolder.clearContext();
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("{\"error\": \"" + e.getMessage() + "\"}");
            return;
        }

        filterChain.doFilter(request, response);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // Skip this filter for public endpoints
        return request.getServletPath().startsWith("/public/");
    }
}
```

---

## Example 2: Multi-Factor Authentication (MFA)

MFA requires two things: password + a time-based one-time password (TOTP from Google Authenticator).

The challenge: Spring Security's normal flow authenticates in one step. MFA requires two steps (or a combined check).

### Approach: Two-phase Authentication

```java
// Phase 1 token: password has been verified, awaiting OTP
public class PreAuthenticatedMfaToken extends AbstractAuthenticationToken {
    // This token represents "password verified, but OTP not yet provided"

    private final UserDetails userDetails;
    private final String tempToken;
    // tempToken — short-lived token linking this session to the password-verified state

    public PreAuthenticatedMfaToken(UserDetails userDetails, String tempToken) {
        super(null); // No authorities yet — not fully authenticated
        this.userDetails = userDetails;
        this.tempToken = tempToken;
        setAuthenticated(false);
    }

    @Override public Object getCredentials() { return tempToken; }
    @Override public Object getPrincipal() { return userDetails; }
}

// Phase 2 token: both password AND OTP have been verified
public class MfaAuthenticationToken extends AbstractAuthenticationToken {
    // This token represents "fully authenticated via MFA"

    private final UserDetails userDetails;

    public MfaAuthenticationToken(UserDetails userDetails,
                                   Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.userDetails = userDetails;
        super.setAuthenticated(true); // Fully authenticated
    }

    @Override public Object getCredentials() { return null; }
    @Override public Object getPrincipal() { return userDetails; }
}
```

```java
@RestController
@RequestMapping("/api/auth")
public class MfaAuthController {

    private final AuthenticationManager authenticationManager;
    private final TotpService totpService;
    private final TempTokenService tempTokenService;
    private final JwtTokenProvider jwtTokenProvider;

    @PostMapping("/login/step1")
    public ResponseEntity<?> loginStep1(@RequestBody LoginRequest request) {
        // Step 1: Verify username + password normally
        Authentication auth = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
        );
        // If this throws, credentials are wrong → 401

        AppUserDetails user = (AppUserDetails) auth.getPrincipal();

        if (!user.isMfaEnabled()) {
            // User doesn't have MFA — skip step 2, issue JWT immediately
            String jwt = jwtTokenProvider.generateToken(auth);
            return ResponseEntity.ok(new JwtResponse(jwt));
        }

        // Generate a short-lived temp token to link step 1 and step 2
        String tempToken = tempTokenService.generate(user.getId());
        // tempToken is stored in Redis with 5-minute TTL
        // The client sends this back with the OTP in step 2

        return ResponseEntity.ok(Map.of(
            "requiresMfa", true,
            "tempToken", tempToken
        ));
    }

    @PostMapping("/login/step2")
    public ResponseEntity<JwtResponse> loginStep2(@RequestBody MfaRequest request) {
        // Step 2: Verify the OTP using the temp token from step 1

        Long userId = tempTokenService.getUserId(request.getTempToken());
        // getUserId() — look up Redis to verify the temp token and get the user ID
        // Returns null if token is expired or invalid

        if (userId == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(null);
        }

        AppUserDetails user = (AppUserDetails) userDetailsService.loadUserByUsername(
            userRepository.findEmailById(userId)
        );

        boolean otpValid = totpService.validateTotp(
            user.getTotpSecret(),    // The user's TOTP secret (stored when they set up MFA)
            request.getOtp()         // The 6-digit code from their authenticator app
        );

        if (!otpValid) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        // Both password (step 1) and OTP (step 2) verified — issue JWT
        MfaAuthenticationToken mfaToken = new MfaAuthenticationToken(
            user, user.getAuthorities()
        );

        SecurityContextHolder.getContext().setAuthentication(mfaToken);
        // setAuthentication() — log the user in for this request

        tempTokenService.invalidate(request.getTempToken());
        // Consume the temp token — can't be used again

        String jwt = jwtTokenProvider.generateToken(mfaToken);
        return ResponseEntity.ok(new JwtResponse(jwt));
    }
}
```

---

## Example 3: Custom AuthenticationProvider for LDAP/Active Directory

Many enterprises use LDAP for user management. Here's a simplified custom provider:

```java
@Component
public class LdapAuthenticationProvider implements AuthenticationProvider {

    private final LdapContextSource ldapContextSource;
    // LdapContextSource — Spring's abstraction for connecting to LDAP server

    private final AppUserRepository userRepository;
    // Local DB: sync LDAP users with local user table for role management

    @Override
    public Authentication authenticate(Authentication authentication) 
            throws AuthenticationException {

        String username = authentication.getName();
        // getName() — returns the username from the authentication request

        String password = authentication.getCredentials().toString();
        // getCredentials().toString() — the plain text password

        // Attempt to bind (authenticate) against LDAP
        try {
            LdapTemplate ldapTemplate = new LdapTemplate(ldapContextSource);
            // LdapTemplate — Spring's LDAP client, similar to JdbcTemplate

            // Try to bind with the user's credentials
            // If bind succeeds, credentials are valid
            ldapContextSource.getContext(
                "cn=" + username + ",ou=users,dc=company,dc=com",
                // The LDAP DN (Distinguished Name) of the user
                password
            ).close();
            // close() — release the LDAP connection after verification
            // If this throws NamingException, credentials are invalid

        } catch (NamingException e) {
            throw new BadCredentialsException("LDAP authentication failed: " + e.getMessage());
        }

        // LDAP verified — now load/sync local user for roles
        AppUser localUser = userRepository.findByUsername(username)
            .orElseGet(() -> {
                // First time login from LDAP: create local record
                AppUser newUser = new AppUser();
                newUser.setUsername(username);
                newUser.setProvider("LDAP");
                newUser.setRole("ROLE_USER"); // Default role
                newUser.setEnabled(true);
                return userRepository.save(newUser);
            });

        List<GrantedAuthority> authorities = List.of(
            new SimpleGrantedAuthority(localUser.getRole())
        );

        return new UsernamePasswordAuthenticationToken(
            new AppUserDetails(localUser),  // principal
            null,                           // credentials (clear after auth)
            authorities                     // roles from local DB
        );
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
        // This provider handles the standard username/password token
        // (same as DaoAuthenticationProvider, but goes to LDAP instead)
    }
}
```

---

## Registering Multiple AuthenticationProviders

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final LdapAuthenticationProvider ldapProvider;
    private final ApiKeyAuthenticationProvider apiKeyProvider;
    private final DaoAuthenticationProvider daoProvider;

    @Bean
    public AuthenticationManager authenticationManager() {
        // ProviderManager — the AuthenticationManager that delegates to a list of providers
        ProviderManager providerManager = new ProviderManager(
            List.of(
                apiKeyProvider,    // Checked first: handles X-API-Key header requests
                ldapProvider,      // Checked second: handles username/password via LDAP
                daoProvider        // Fallback: handles username/password via DB
            )
        );
        // ProviderManager iterates through the list and calls authenticate() on each
        // A provider returns null if it doesn't support the token type
        // A provider throws AuthenticationException if it supports but fails
        // The first successful result (non-null) is returned
        // If all providers fail, ProviderManager throws the last exception

        providerManager.setEraseCredentialsAfterAuthentication(true);
        // setEraseCredentialsAfterAuthentication(true) — clears the password from
        // the Authentication token after successful auth
        // Prevents passwords lingering in memory after login
        
        return providerManager;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http,
                                                     ApiKeyAuthenticationFilter apiKeyFilter)
                                                     throws Exception {
        http
            .authenticationManager(authenticationManager())
            // Register the ProviderManager with the filter chain

            .addFilterBefore(apiKeyFilter, UsernamePasswordAuthenticationFilter.class)
            // API key filter runs before username/password filter

            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

---

## AuthenticationException Hierarchy

Know which exception to throw from your custom provider:

```java
AuthenticationException (base class)
├── BadCredentialsException         // Wrong password, invalid token, invalid API key
├── UsernameNotFoundException       // User doesn't exist (thrown from UserDetailsService)
├── AccountStatusException (base)
│   ├── DisabledException           // Account is disabled (isEnabled() = false)
│   ├── LockedException             // Account is locked (isAccountNonLocked() = false)
│   ├── AccountExpiredException     // Account has expired (isAccountNonExpired() = false)
│   └── CredentialsExpiredException // Password/credentials expired
├── AuthenticationServiceException  // Server-side error during auth (DB down, LDAP unavailable)
└── InsufficientAuthenticationException // Auth required but not present (Anonymous user)
```

---

## Common Mistakes

### Mistake 1: Returning null instead of throwing on failure

```java
// Bad — returning null from authenticate() tells ProviderManager
// "I can't handle this token type" (not "authentication failed")
// ProviderManager will then try the next provider, which may succeed unintentionally
@Override
public Authentication authenticate(Authentication auth) {
    if (!isValid(auth.getCredentials())) {
        return null; // WRONG — ProviderManager thinks you can't handle it, tries next provider
    }
}

// Good — throw an exception when validation fails
@Override
public Authentication authenticate(Authentication auth) {
    if (!isValid(auth.getCredentials())) {
        throw new BadCredentialsException("Invalid credentials"); // CORRECT
    }
}
```

### Mistake 2: Forgetting to return null when token type isn't supported

```java
// Bad — if supports() is correct but authenticate() throws when it shouldn't handle the type
// This can cause confusing error messages
@Override
public Authentication authenticate(Authentication auth) {
    // What if auth is a BearerTokenAuthenticationToken and your supports() has a bug?
    // You'd get a ClassCastException instead of a clean "can't handle this" skip
}

// Always pair supports() and authenticate() correctly:
@Override
public boolean supports(Class<?> authentication) {
    return ApiKeyAuthenticationToken.class.isAssignableFrom(authentication);
}

@Override
public Authentication authenticate(Authentication authentication) {
    // At this point, Spring guarantees supports() returned true for this type
    ApiKeyAuthenticationToken token = (ApiKeyAuthenticationToken) authentication; // Safe cast
    // ...
}
```
