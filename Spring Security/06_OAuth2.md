# OAuth2 and OpenID Connect

## What is OAuth2?

OAuth2 is an **authorization framework** that lets a user grant a third-party application access to their resources without sharing their password.

**Real-world example:** "Login with Google" on a website.
- You're on `myapp.com`
- You click "Login with Google"
- You're redirected to Google's login page
- You log in to Google, Google asks "Allow myapp.com to see your profile and email?"
- You click "Allow"
- Google sends `myapp.com` an **access token**
- `myapp.com` uses this token to call Google's API to get your profile

At no point did you give your Google password to `myapp.com`. That's the whole point of OAuth2.

---

## Key Roles in OAuth2

| Role | What it is | Example |
|---|---|---|
| **Resource Owner** | The end user | You, logging into myapp.com |
| **Client** | The app requesting access | myapp.com |
| **Authorization Server** | Issues tokens after user consent | Google, GitHub, Okta, your own auth server |
| **Resource Server** | The API that holds protected resources | Google's People API, your backend API |

---

## OAuth2 Flows (Grant Types)

### Authorization Code Flow (for web apps and mobile apps — the standard flow)

```
1. User clicks "Login with Google" on myapp.com
2. myapp.com redirects user to:
   https://accounts.google.com/o/oauth2/auth
     ?client_id=myapp_client_id
     &redirect_uri=https://myapp.com/callback
     &response_type=code
     &scope=openid email profile
     &state=random_csrf_token

3. User logs in at Google, grants permissions
4. Google redirects back to: https://myapp.com/callback?code=AUTHORIZATION_CODE&state=...
5. myapp.com exchanges the code for tokens (server-to-server, never visible to browser):
   POST https://oauth2.googleapis.com/token
     { code, client_id, client_secret, redirect_uri, grant_type: "authorization_code" }

6. Google responds with:
   { access_token: "...", refresh_token: "...", id_token: "..." }

7. myapp.com uses access_token to call Google APIs
```

### Client Credentials Flow (for machine-to-machine)

```
Used when there's no user involved — service-to-service communication.

Service A → POST /token { client_id, client_secret, grant_type: "client_credentials" }
Authorization Server → { access_token: "...", expires_in: 3600 }
Service A → calls Service B with access_token in Authorization header
```

---

## OpenID Connect (OIDC)

OAuth2 is about **authorization** (access to resources). It doesn't tell you WHO the user is.

OIDC is a thin layer on top of OAuth2 that adds **authentication** (identity). It introduces the `id_token` — a JWT containing the user's identity claims.

```
OIDC id_token payload:
{
  "sub": "user123",          // Subject — the unique user ID at the identity provider
  "email": "alice@gmail.com",
  "name": "Alice Smith",
  "picture": "https://...",
  "iss": "https://accounts.google.com",   // Issuer — who created this token
  "aud": "myapp_client_id",               // Audience — who this token is for
  "iat": 1698768000,
  "exp": 1698771600
}
```

When you use "Login with Google", you're using OIDC. The `id_token` is what tells your app who the user is.

---

## Spring Security as an OAuth2 Client

Your app acts as the "Client" — it uses Google/GitHub/Okta for authentication.

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

### Configuration

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            # registration — defines this OAuth2 client registration
            client-id: ${GOOGLE_CLIENT_ID}
            # client-id — your app's ID registered with Google Developer Console
            client-secret: ${GOOGLE_CLIENT_SECRET}
            # client-secret — your app's secret (keep in env var, NOT in code)
            scope: openid, email, profile
            # scope — what permissions to request from Google
            # openid = enables OIDC (id_token), email/profile = user info

          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email, read:user

        provider:
          google:
            # Spring Boot auto-configures google, github, facebook, okta
            # You only need provider config for custom/self-hosted providers
            issuer-uri: https://accounts.google.com
            # issuer-uri — the base URL of the auth server
            # Spring auto-discovers endpoints by fetching issuer-uri/.well-known/openid-configuration
```

### Security Config

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                // oauth2Login() — enables the OAuth2 / OIDC login flow
                // Spring handles all the redirects, token exchange, etc. automatically
                
                .loginPage("/login")
                // loginPage() — your custom login page (with "Login with Google" buttons)
                // If omitted, Spring generates a default login page
                
                .defaultSuccessUrl("/dashboard", true)
                // defaultSuccessUrl() — where to send the user after successful OAuth2 login
                
                .failureUrl("/login?error")
                // failureUrl() — where to send on OAuth2 failure
                
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)
                    // userService() — your custom service to process the user after OAuth2 login
                    // Use this to create/update a local user record in your DB
                )
            );

        return http.build();
    }
}
```

### Custom OAuth2 User Service (Creating Local Users from OAuth2 Login)

```java
@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {
    // DefaultOAuth2UserService — Spring's default implementation, fetches user info from provider
    // We extend it to add our own logic: create/update user in local DB

    private final AppUserRepository userRepository;

    public CustomOAuth2UserService(AppUserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        // loadUser() — called by Spring after the OAuth2 token exchange
        // userRequest — contains the access token and registration info (google, github, etc.)

        OAuth2User oAuth2User = super.loadUser(userRequest);
        // super.loadUser() — calls the provider's userinfo endpoint to get user attributes
        // For Google: calls https://openidconnect.googleapis.com/v1/userinfo

        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        // getRegistrationId() — "google", "github", etc. (matches your yml config key)

        return processOAuth2User(registrationId, oAuth2User);
    }

    private OAuth2User processOAuth2User(String registrationId, OAuth2User oAuth2User) {
        // Extract email (works for Google; GitHub might need different attribute)
        String email = oAuth2User.getAttribute("email");
        // getAttribute("email") — get the "email" claim from the provider's user info response

        if (email == null) {
            throw new OAuth2AuthenticationException("Email not found from OAuth2 provider");
        }

        // Check if user already exists in our DB
        Optional<AppUser> existingUser = userRepository.findByEmail(email);

        AppUser user;
        if (existingUser.isPresent()) {
            user = existingUser.get();
            // Update provider info in case it changed
            user.setName(oAuth2User.getAttribute("name"));
            user.setImageUrl(oAuth2User.getAttribute("picture"));
        } else {
            // First login — create a new user in our DB
            user = new AppUser();
            user.setEmail(email);
            user.setName(oAuth2User.getAttribute("name"));
            user.setImageUrl(oAuth2User.getAttribute("picture"));
            user.setProvider(AuthProvider.valueOf(registrationId.toUpperCase()));
            // AuthProvider — your enum: GOOGLE, GITHUB, LOCAL
            user.setProviderId(oAuth2User.getName());
            // oAuth2User.getName() — the unique ID from the provider (Google's "sub" claim)
            user.setEnabled(true);
            user.setRole("ROLE_USER");
            // No password for OAuth2 users — they authenticate via the provider
        }

        userRepository.save(user);

        return new AppOAuth2User(user, oAuth2User.getAttributes());
        // Return our custom OAuth2User that wraps both the provider data and our local user
    }
}

// Custom OAuth2User to expose local user data
public class AppOAuth2User implements OAuth2User {
    // OAuth2User — Spring Security's interface for users authenticated via OAuth2

    private final AppUser appUser;
    private final Map<String, Object> attributes;
    // attributes — the raw key-value pairs from the OAuth2 provider's userinfo endpoint

    public AppOAuth2User(AppUser appUser, Map<String, Object> attributes) {
        this.appUser = appUser;
        this.attributes = attributes;
    }

    @Override
    public Map<String, Object> getAttributes() {
        // getAttributes() — returns all claims from the OAuth2 provider
        return attributes;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        // getAuthorities() — what roles does this OAuth2 user have in our system?
        return List.of(new SimpleGrantedAuthority(appUser.getRole()));
    }

    @Override
    public String getName() {
        // getName() — the unique identifier for this OAuth2 user
        return appUser.getEmail();
    }

    // Custom helper — access our local user's ID
    public Long getId() {
        return appUser.getId();
    }
}
```

---

## Spring Security as an OAuth2 Resource Server

Your API acts as the "Resource Server" — it accepts JWTs issued by an Authorization Server (Auth0, Okta, Keycloak, etc.) and validates them.

This is the pattern for **microservices** — one central auth server issues tokens, all services validate them.

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### Configuration

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://your-auth-server.com
          # issuer-uri — Spring fetches the public key from the auth server automatically
          # Uses the JWKS (JSON Web Key Set) endpoint at issuer-uri/.well-known/jwks.json
          # The public key is used to verify JWT signatures without a shared secret
```

### Security Config

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            .oauth2ResourceServer(oauth2 -> oauth2
                // oauth2ResourceServer() — configures this app as a resource server
                // Automatically adds BearerTokenAuthenticationFilter to the chain
                // This filter extracts and validates JWTs on every request
                
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                    // jwtAuthenticationConverter() — converts the JWT into a Spring Authentication object
                    // Used to extract roles from JWT claims
                )
                
                .authenticationEntryPoint((request, response, exception) -> {
                    // Return JSON 401 instead of redirect for API clients
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    response.getWriter().write("{\"error\": \"Token required\"}");
                })
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        // JwtAuthenticationConverter — converts a verified Jwt object to a Spring Authentication
        
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        // JwtGrantedAuthoritiesConverter — extracts roles from JWT claims as GrantedAuthorities
        
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        // setAuthoritiesClaimName() — which JWT claim contains the roles
        // Default is "scope" — change to match your auth server's token format
        
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        // setAuthorityPrefix() — prefix added to each role from the JWT
        // If JWT has "roles": ["ADMIN"], the resulting authority is "ROLE_ADMIN"
        // Default prefix is "SCOPE_" — change to "ROLE_" to use with hasRole()

        JwtAuthenticationConverter jwtAuthenticationConverter = new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        // setJwtGrantedAuthoritiesConverter() — use our custom converter for extracting roles

        return jwtAuthenticationConverter;
    }
}
```

### Accessing the JWT in a Controller

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping
    public List<Order> getOrders(@AuthenticationPrincipal Jwt jwt) {
        // @AuthenticationPrincipal Jwt — Spring injects the validated JWT object
        // Use this when you need raw JWT claims (not just username/roles)
        
        String userId = jwt.getSubject();
        // getSubject() — the "sub" claim (user's unique ID from the auth server)
        
        String email = jwt.getClaimAsString("email");
        // getClaimAsString() — extract a specific claim as a String
        
        List<String> roles = jwt.getClaimAsStringList("roles");
        // getClaimAsStringList() — extract an array claim as a List<String>
        
        return orderService.getOrdersByUserId(userId);
    }
}
```

---

## OAuth2 Token Introspection (Opaque Tokens)

If the auth server issues opaque tokens (random strings, not JWTs), your resource server can't decode them locally. Instead it calls the auth server's introspection endpoint:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth-server.com/oauth2/introspect
          client-id: your-resource-server-id
          client-secret: your-resource-server-secret
```

---

## Production Pattern: Keycloak + Spring Boot Microservices

```
                    ┌─────────────────┐
  User Login  →     │    Keycloak      │  (Authorization Server)
                    │  (issues JWTs)   │
                    └────────┬────────┘
                             │ JWT
                             ↓
  API Requests  →   ┌────────────────┐    ┌──────────────────┐
  (with JWT)        │  API Gateway   │ →  │  Order Service   │  (Resource Server)
                    │  (validates    │    │  (validates JWT) │
                    │   JWT once)    │    └──────────────────┘
                    └────────────────┘    ┌──────────────────┐
                                      →  │  Product Service │  (Resource Server)
                                          └──────────────────┘
```

Each microservice validates the JWT independently using Keycloak's public key. No central session store needed.

---

## Common Mistakes

### Mistake 1: Storing client_secret in code

```yaml
# Bad — committed to git
spring.security.oauth2.client.registration.google.client-secret: abc123

# Good — use environment variable
spring.security.oauth2.client.registration.google.client-secret: ${GOOGLE_CLIENT_SECRET}
```

### Mistake 2: Using implicit flow (deprecated)

The implicit flow (response_type=token) sends the access token directly in the URL fragment — visible in browser history and logs. It's deprecated in OAuth2.1. Always use Authorization Code Flow.

### Mistake 3: Not validating the `state` parameter

The `state` parameter in the OAuth2 flow prevents CSRF attacks. Spring Security handles this automatically — don't disable it.
