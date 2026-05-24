# Session Management

## What is a Session?

When a user logs into a web application, the server creates an **HTTP session** — a server-side storage area identified by a session ID. The session ID is sent to the browser as a cookie (`JSESSIONID`).

On each subsequent request, the browser sends this cookie, and the server looks up the session to find out who the user is.

```
User logs in → Server creates session:
  SessionID: abc123
  SecurityContext: { user: "alice", roles: ["ROLE_USER"] }
  
Browser stores cookie: JSESSIONID=abc123

Later request:
  Browser sends: Cookie: JSESSIONID=abc123
  Server looks up session abc123 → finds Alice's SecurityContext → knows who Alice is
```

---

## Session Creation Policies

Spring Security gives you control over when and how sessions are created.

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.sessionManagement(session -> session
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        // SessionCreationPolicy — an enum that controls session creation behavior
    );
    return http.build();
}
```

| Policy | Meaning | Use When |
|---|---|---|
| `ALWAYS` | Always create a session | Rarely needed |
| `IF_REQUIRED` | Create session if needed (default) | Traditional web apps |
| `NEVER` | Never CREATE a session, but USE one if it exists | Rare |
| `STATELESS` | Never create or use sessions | REST APIs with JWT |

```java
// SessionCreationPolicy.STATELESS — the correct choice for JWT REST APIs
// Spring Security won't save SecurityContext to session
// Each request must authenticate itself (via JWT)
// No server-side state = horizontally scalable (any server handles any request)

http.sessionManagement(session ->
    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
);

// SessionCreationPolicy.IF_REQUIRED — the default for form-login web apps
// Session is created on first login and persists until logout/expiry
http.sessionManagement(session ->
    session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
);
```

---

## Session Fixation Protection

**The attack:**
```
1. Attacker visits your site → gets session ID: XYZ
2. Attacker tricks victim into using that session ID:
   "Click this link: https://bank.com/login;jsessionid=XYZ"
3. Victim logs in with session ID XYZ
4. Server binds Alice's authentication to session XYZ
5. Attacker already knows XYZ → attacker is now authenticated as Alice
```

**Spring Security's defense:** After successful login, create a **new** session ID (and discard the old one). The attacker's pre-known session ID becomes invalid.

```java
http.sessionManagement(session -> session
    .sessionFixation(fixation -> fixation
        .changeSessionId()
        // changeSessionId() — after login, generate a new session ID but keep same session data
        // This is the DEFAULT and RECOMMENDED option
        // Invalidates the pre-login session ID (defeats session fixation attack)
        
        // Other options (less common):
        // .newSession()          — creates a brand new session, loses pre-auth data
        // .migrateSession()      — old behavior: creates new session, copies attributes
        // .none()                — DANGEROUS: no session fixation protection (don't use)
    )
);
```

---

## Concurrent Session Control

**The problem:** A user logs in from work, then from home. Now there are two active sessions. You might want to:
- Limit users to one active session (like Netflix basic plan)
- Let users know they were logged in elsewhere

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .maximumSessions(1)
            // maximumSessions(n) — maximum number of concurrent sessions per user
            // n=1: only one active session allowed
            // n=-1: unlimited (default)
            
            .maxSessionsPreventsLogin(false)
            // maxSessionsPreventsLogin(false) — when true, NEW login is rejected if limit reached
            //   → The new login attempt fails: "Maximum sessions exceeded"
            // maxSessionsPreventsLogin(false) — the DEFAULT: OLD session is invalidated
            //   → The previous session gets kicked out, new login succeeds
            // Netflix "Someone else logged in" behavior = false
            // "You're already logged in" behavior = true
            
            .expiredSessionStrategy(event -> {
                // expiredSessionStrategy() — what happens when a session is expired by new login
                HttpServletResponse response = event.getResponse();
                response.setContentType("application/json");
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.getWriter().write("{\"error\": \"Session expired: logged in from another device\"}");
                // Custom response when a session is terminated due to concurrent login limit
            })
        );
    return http.build();
}

// REQUIRED for concurrent session control: register the HttpSessionEventPublisher
// Without this, Spring Security doesn't know when sessions are destroyed
@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    // HttpSessionEventPublisher — publishes session lifecycle events to Spring context
    // When a session is destroyed (logout, timeout), this publishes SessionDestroyedEvent
    // SessionManagementFilter listens to this to update the active session count
    return new HttpSessionEventPublisher();
}
```

---

## Session Timeout Configuration

```yaml
# application.yml
server:
  servlet:
    session:
      timeout: 30m
      # Session expires after 30 minutes of inactivity
      # Format: 30m, 1h, 3600s
      # Default: 30 minutes for embedded Tomcat
      
      cookie:
        name: JSESSIONID    # Session cookie name (default)
        http-only: true     # JavaScript can't read the cookie (prevents XSS session theft)
        secure: true        # Cookie only sent over HTTPS (set to true in production)
        same-site: strict   # Cookie not sent on cross-site requests (extra CSRF protection)
        # same-site options: strict, lax, none
        # strict — only same-site requests (recommended for most apps)
        # lax — same-site + top-level navigation from other sites (GET only)
        # none — all requests (requires secure=true, used for cross-site embedding)
```

---

## Session Security Best Practices

### Setting Secure Session Cookie in Code

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
        )
        .headers(headers -> headers
            .frameOptions(frameOptions -> frameOptions.deny())
            // deny() — prevents the page from being displayed in frames (clickjacking protection)
        );
    
    return http.build();
}

// For programmatic session cookie configuration:
@Bean
public ServletContextInitializer servletContextInitializer() {
    return servletContext -> {
        SessionCookieConfig sessionCookieConfig = servletContext.getSessionCookieConfig();
        // SessionCookieConfig — configures the session cookie properties
        
        sessionCookieConfig.setHttpOnly(true);
        // setHttpOnly(true) — prevents JavaScript from reading the session cookie
        // Mitigates XSS-based session hijacking

        sessionCookieConfig.setSecure(true);
        // setSecure(true) — cookie only sent over HTTPS connections
        // Prevents session hijacking over HTTP (man-in-the-middle)
        
        sessionCookieConfig.setName("SESSIONID");
        // setName() — rename from JSESSIONID to make fingerprinting harder
        // An attacker seeing JSESSIONID knows exactly what server you're using
        
        sessionCookieConfig.setAttribute("SameSite", "Strict");
        // SameSite Strict — cookie not sent on any cross-site request
        // Best CSRF protection for session cookies (complements Spring's CSRF token)
    };
}
```

---

## Remember-Me (Persistent Login)

"Remember me" lets users stay logged in across browser restarts (beyond session lifetime).

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .rememberMe(remember -> remember
            // rememberMe() — configures the Remember-Me feature

            .rememberMeParameter("rememberMe")
            // rememberMeParameter() — the form field name that activates remember-me
            // Default: "remember-me". Your HTML: <input type="checkbox" name="rememberMe">

            .tokenValiditySeconds(7 * 24 * 60 * 60)
            // tokenValiditySeconds() — how long the remember-me token is valid (in seconds)
            // 7 * 24 * 60 * 60 = 1 week

            .key("your-remember-me-secret-key")
            // key() — a secret used to sign the remember-me token
            // Without this, tokens are randomly generated per server restart (breaks remember-me on redeploy)
            // In production: use a stable, secret, unique value from environment variables

            .userDetailsService(userDetailsService)
            // userDetailsService() — how to load the user when the remember-me token is presented
        );
    
    return http.build();
}
```

### Persistent Remember-Me (Stored in Database — Safer)

The simple remember-me token contains username + expiry in the cookie. If stolen, an attacker can use it until expiry. The persistent approach stores tokens in DB and can invalidate them:

```java
@Bean
public PersistentTokenRepository persistentTokenRepository(DataSource dataSource) {
    JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
    // JdbcTokenRepositoryImpl — stores remember-me tokens in a DB table
    // Table schema:
    //   CREATE TABLE persistent_logins (
    //     username VARCHAR(64) NOT NULL,
    //     series VARCHAR(64) PRIMARY KEY,
    //     token VARCHAR(64) NOT NULL,
    //     last_used TIMESTAMP NOT NULL
    //   );
    
    tokenRepository.setDataSource(dataSource);
    // setDataSource() — the DB to use for token storage
    
    tokenRepository.setCreateTableOnStartup(true);
    // setCreateTableOnStartup(true) — auto-creates the table if it doesn't exist
    // Set to false in production (use Flyway/Liquibase migrations instead)
    
    return tokenRepository;
}

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http, 
                                                PersistentTokenRepository tokenRepository) throws Exception {
    http
        .rememberMe(remember -> remember
            .tokenRepository(tokenRepository)
            // tokenRepository() — use the DB-backed persistent token repository
            // The token in the cookie is a random series + token pair
            // Server looks it up in DB; can be revoked at any time (e.g., on logout from all devices)
            
            .tokenValiditySeconds(7 * 24 * 60 * 60)
            .userDetailsService(userDetailsService)
        );
    
    return http.build();
}
```

---

## Full Session-Based Web App Configuration

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {

    private final AppUserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;
    private final DataSource dataSource;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/login", "/register").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            
            .formLogin(form -> form
                // formLogin() — enables the standard HTML form-based login
                .loginPage("/login")
                // loginPage() — URL of your login page (GET renders form, POST submits it)
                .loginProcessingUrl("/login")
                // loginProcessingUrl() — the URL where the form POSTs to (processed by Spring, not your controller)
                .usernameParameter("email")
                // usernameParameter() — name of the form field for username (default: "username")
                .passwordParameter("password")
                // passwordParameter() — name of the form field for password (default: "password")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error")
                .permitAll()
            )
            
            .logout(logout -> logout
                .logoutUrl("/logout")
                // logoutUrl() — URL to trigger logout (POST by default)
                .logoutSuccessUrl("/login?logout")
                .invalidateHttpSession(true)
                // invalidateHttpSession(true) — destroys the server-side session on logout
                .deleteCookies("SESSIONID")
                // deleteCookies() — instructs browser to delete these cookies on logout
                .clearAuthentication(true)
                // clearAuthentication(true) — removes Authentication from SecurityContext on logout
                .permitAll()
            )
            
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .sessionFixation(fixation -> fixation.changeSessionId())
                // changeSessionId() — new session ID on login (session fixation protection)
                .maximumSessions(1)
                // One active session per user
                .expiredSessionStrategy(event -> {
                    event.getResponse().sendRedirect("/login?expired");
                    // Redirect to login page when session is expired by a new login from elsewhere
                })
            )
            
            .rememberMe(remember -> remember
                .tokenRepository(persistentTokenRepository())
                .tokenValiditySeconds(7 * 24 * 60 * 60)
                .key("stable-remember-me-key-from-env")
                .userDetailsService(userDetailsService)
            )
            
            .csrf(csrf -> csrf
                // Keep CSRF enabled for form-based apps
                .ignoringRequestMatchers("/api/**")
                // ignoringRequestMatchers() — disable CSRF for specific paths
                // If you have a mix of form-based pages AND a REST API, you can disable CSRF for API only
            );
        
        return http.build();
    }

    @Bean
    public HttpSessionEventPublisher httpSessionEventPublisher() {
        return new HttpSessionEventPublisher();
        // Required for concurrent session control
    }

    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl repo = new JdbcTokenRepositoryImpl();
        repo.setDataSource(dataSource);
        return repo;
    }
}
```

---

## Common Mistakes

### Mistake 1: Using STATELESS session with form-based login

```java
// Bad — form login requires a session to complete the auth flow
http
    .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    .formLogin(Customizer.withDefaults()); // Won't work correctly — loses auth state

// Fix: use STATELESS only with JWT
// Use IF_REQUIRED with form login
```

### Mistake 2: Forgetting HttpSessionEventPublisher for concurrent sessions

```java
// If you configure maximumSessions() but DON'T register HttpSessionEventPublisher,
// session counts never decrement when sessions expire or users log out.
// Eventually, all users appear to have the maximum sessions active.

// Always add this bean when using concurrent session control:
@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}
```

### Mistake 3: Not setting secure cookie in production

```java
// In development: secure=false is OK (localhost is HTTP)
// In production: secure=true is MANDATORY
// If secure=false in production, session ID can be sniffed on HTTP traffic

// Use profiles:
// application.yml:         server.servlet.session.cookie.secure=false  (dev default)
// application-prod.yml:   server.servlet.session.cookie.secure=true   (production)
```
