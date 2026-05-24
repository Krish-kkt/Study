# Spring Security — Overview & Architecture

## What is Spring Security?

Spring Security is a **security framework for Spring-based applications** that handles two core concerns:

- **Authentication** — Who are you? (login, identity verification)
- **Authorization** — What are you allowed to do? (access control)

Think of it like a building's security system. Authentication is the ID badge scanner at the entrance. Authorization is the door access level each badge grants (floor 1 vs floor 3 vs server room).

---

## How Spring Security Plugs Into Your App

Spring Boot auto-configures Spring Security the moment you add the dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

As soon as this is on the classpath, **every endpoint in your app is locked down by default**. This is intentional — "secure by default" is a design principle. You must explicitly open up what you want public.

---

## The Big Picture: How a Request Flows Through Spring Security

```
Incoming HTTP Request
        |
        v
+---------------------------+
|   Servlet Container       |
|   (Tomcat/Jetty)          |
+---------------------------+
        |
        v
+---------------------------+
|   DelegatingFilterProxy   |  <-- Bridge between Servlet world and Spring world
+---------------------------+
        |
        v
+---------------------------+
|   FilterChainProxy        |  <-- The real Spring Security entry point
+---------------------------+
        |
        v
+------------------------------------------+
|         Security Filter Chain            |
|  ┌────────────────────────────────────┐  |
|  │ DisableEncodeUrlFilter             │  |
|  │ SecurityContextPersistenceFilter   │  |
|  │ UsernamePasswordAuthenticationFilter│ |
|  │ BasicAuthenticationFilter          │  |
|  │ BearerTokenAuthenticationFilter    │  |
|  │ ExceptionTranslationFilter         │  |
|  │ FilterSecurityInterceptor          │  |
|  └────────────────────────────────────┘  |
+------------------------------------------+
        |
        v
+---------------------------+
|   DispatcherServlet       |
|   (Your Controllers)      |
+---------------------------+
```

Each filter does one specific job. Requests pass through all relevant filters before reaching your controller. If any filter rejects the request, it short-circuits and sends a 401/403 response.

---

## Core Components You Must Know

### 1. SecurityContext & SecurityContextHolder

```java
// SecurityContextHolder is the global store for the currently authenticated user.
// Think of it like a thread-local variable that holds "who is logged in right now."

SecurityContext context = SecurityContextHolder.getContext();
// getContext() — returns the SecurityContext for the current thread

Authentication authentication = context.getAuthentication();
// getAuthentication() — returns the Authentication object for the current user
// Returns null if the user is not authenticated

if (authentication != null && authentication.isAuthenticated()) {
    String username = authentication.getName();
    // getName() — returns the username/principal identifier
    
    Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
    // getAuthorities() — returns the roles/permissions granted to this user
}
```

**Real-world use:** In your service layer, you often need to know which user is making a request:

```java
@Service
public class OrderService {

    public List<Order> getMyOrders() {
        // Grab the currently logged-in user from the security context
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth.getName();
        
        return orderRepository.findByUsername(username);
    }
}
```

---

### 2. Authentication Object

The `Authentication` interface represents both:
- An **authentication request** (before verification) — carries credentials
- An **authenticated principal** (after verification) — carries the user details

```java
public interface Authentication extends Principal, Serializable {
    
    Collection<? extends GrantedAuthority> getAuthorities();
    // getAuthorities() — roles/permissions like ROLE_ADMIN, ROLE_USER
    
    Object getCredentials();
    // getCredentials() — the secret (password, token) used to authenticate
    // After authentication, this is usually cleared for security
    
    Object getPrincipal();
    // getPrincipal() — the user identity, usually a UserDetails object
    
    boolean isAuthenticated();
    // isAuthenticated() — true if the authentication has been verified
    
    void setAuthenticated(boolean isAuthenticated);
    // setAuthenticated() — mark as authenticated/unauthenticated
}
```

---

### 3. GrantedAuthority

Represents a permission granted to a user:

```java
// GrantedAuthority is a simple interface with one method
GrantedAuthority authority = new SimpleGrantedAuthority("ROLE_ADMIN");
// SimpleGrantedAuthority — the most common implementation, wraps a string role name

String permission = authority.getAuthority();
// getAuthority() — returns the string representation like "ROLE_ADMIN"
```

**Convention:** roles are prefixed with `ROLE_`. So `hasRole("ADMIN")` checks for authority `ROLE_ADMIN`.

---

### 4. AuthenticationManager & AuthenticationProvider

```
AuthenticationManager
        |
        +-- ProviderManager (default implementation)
                    |
                    +-- AuthenticationProvider 1 (handles username/password)
                    +-- AuthenticationProvider 2 (handles JWT tokens)
                    +-- AuthenticationProvider 3 (handles LDAP)
```

- `AuthenticationManager` — the entry point for authentication. Has one method: `authenticate(Authentication)`
- `ProviderManager` — iterates through a list of `AuthenticationProvider`s until one can handle the request
- `AuthenticationProvider` — performs the actual authentication logic for a specific type

```java
// AuthenticationManager — the main interface
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
    // authenticate() — tries to authenticate the given token
    // Returns a fully populated Authentication if successful
    // Throws AuthenticationException if authentication fails
}

// AuthenticationProvider — handles a specific type of authentication
public interface AuthenticationProvider {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
    // authenticate() — same as AuthenticationManager but this is for a specific auth type
    
    boolean supports(Class<?> authentication);
    // supports() — tells ProviderManager whether this provider can handle the given auth type
    // e.g., return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication)
}
```

---

## Minimal Working Configuration

This is the minimum `SecurityConfig` you'll write in almost every project:

```java
@Configuration
// @Configuration — marks this class as a Spring configuration class (bean factory)

@EnableWebSecurity
// @EnableWebSecurity — activates Spring Security's web security support
// Without this, your SecurityFilterChain bean won't be picked up

public class SecurityConfig {

    @Bean
    // @Bean — tells Spring to manage the returned object as a Spring bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        // HttpSecurity — the main builder for configuring web-based security
        // Think of it as a fluent API for "what should be secured and how"

        http
            .authorizeHttpRequests(auth -> auth
                // authorizeHttpRequests() — configure URL-based access rules
                
                .requestMatchers("/public/**", "/auth/**").permitAll()
                // requestMatchers() — matches specific URL patterns
                // permitAll() — anyone can access, no authentication needed
                
                .requestMatchers("/admin/**").hasRole("ADMIN")
                // hasRole("ADMIN") — only users with ROLE_ADMIN can access
                
                .anyRequest().authenticated()
                // anyRequest() — catches all remaining URLs
                // authenticated() — user must be logged in, any role allowed
            )
            .httpBasic(Customizer.withDefaults());
            // httpBasic() — enables HTTP Basic authentication (username:password in header)
            // Good for testing, not for production UIs

        return http.build();
        // build() — constructs the SecurityFilterChain from the configuration
    }
}
```

---

## What Happens When You Hit a Secured Endpoint Without Auth?

1. Request arrives → passes through filters
2. `ExceptionTranslationFilter` wraps the chain
3. `FilterSecurityInterceptor` checks if the user is authenticated
4. No auth found → throws `AuthenticationException`
5. `ExceptionTranslationFilter` catches it → calls `AuthenticationEntryPoint`
6. Default entry point for web: redirects to `/login`
7. Default entry point for API: returns HTTP 401

---

## Spring Security Auto-Configuration (What You Get for Free)

When you add the starter without any config, Spring Boot gives you:

| Feature | Default Behavior |
|---|---|
| Login page | Auto-generated at `/login` |
| Logout | Enabled at `/logout` |
| Password | Random UUID printed to console at startup |
| Username | `user` |
| All endpoints | Require authentication |
| CSRF | Enabled for state-changing requests |
| Session | Created on demand |

---

## Files in This Study Set

| File | Topic |
|---|---|
| `01_Authentication_vs_Authorization.md` | Core concepts, difference, flow |
| `02_SecurityFilterChain.md` | Filter chain, key filters, configuration |
| `03_UserDetails_and_UserDetailsService.md` | Loading users, custom user store |
| `04_PasswordEncoding.md` | Hashing, BCrypt, encoding strategies |
| `05_JWT_Authentication.md` | Stateless auth, token filter, validation |
| `06_OAuth2.md` | OAuth2, resource server, JWT integration |
| `07_Method_Security.md` | @PreAuthorize, @PostAuthorize, SpEL |
| `08_CSRF_and_CORS.md` | CSRF protection, CORS configuration |
| `09_Session_Management.md` | Session fixation, concurrent sessions |
| `10_Custom_AuthenticationProvider.md` | Custom auth flows, multi-step auth |
