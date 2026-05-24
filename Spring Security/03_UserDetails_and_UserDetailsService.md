# UserDetails and UserDetailsService

## The Problem These Interfaces Solve

Spring Security needs to answer one question during authentication:

> "Given this username, what do I know about this user?"

It needs: the stored password (to compare), the roles (for authorization), and the account status (locked? expired? enabled?).

Spring Security doesn't know whether you store users in MySQL, MongoDB, LDAP, or a hardcoded list. So it defines a contract — `UserDetailsService` — and lets YOU implement it. You tell Spring Security how to fetch user data; it handles the rest.

---

## UserDetails Interface

`UserDetails` is the object that represents a user inside Spring Security. It's what your `UserDetailsService` returns.

```java
public interface UserDetails extends Serializable {

    Collection<? extends GrantedAuthority> getAuthorities();
    // getAuthorities() — returns the roles/permissions assigned to this user
    // e.g., [ROLE_USER, ROLE_ADMIN] or [ROLE_USER, ORDER_CANCEL]

    String getPassword();
    // getPassword() — returns the STORED (hashed) password
    // Spring Security uses this to compare against what the user typed
    // Must be encoded — never store or return plain text

    String getUsername();
    // getUsername() — returns the unique identifier (usually email or username)
    // This is what gets stored in the SecurityContext

    boolean isAccountNonExpired();
    // isAccountNonExpired() — false means the account has expired (e.g., trial account ended)
    // If false, authentication fails with AccountExpiredException

    boolean isAccountNonLocked();
    // isAccountNonLocked() — false means the account is locked (e.g., too many failed logins)
    // If false, authentication fails with LockedException

    boolean isCredentialsNonExpired();
    // isCredentialsNonExpired() — false means the password has expired (e.g., 90-day policy)
    // If false, authentication fails with CredentialsExpiredException

    boolean isEnabled();
    // isEnabled() — false means the account is disabled (e.g., not yet email-verified)
    // If false, authentication fails with DisabledException
}
```

### Using User (the Built-in Implementation)

Spring provides a `User` class that implements `UserDetails`:

```java
// org.springframework.security.core.userdetails.User — built-in implementation of UserDetails

UserDetails user = User.builder()
    // builder() — fluent builder for creating User instances
    .username("alice@example.com")
    // username() — the login identifier
    .password("{bcrypt}$2a$12$...")
    // password() — the ENCODED password. The {bcrypt} prefix is for DelegatingPasswordEncoder
    .roles("USER", "MANAGER")
    // roles("USER") — convenience method that creates authorities: ROLE_USER, ROLE_MANAGER
    // Automatically adds the ROLE_ prefix
    .accountExpired(false)
    // accountExpired() — sets isAccountNonExpired() to !value
    .accountLocked(false)
    .credentialsExpired(false)
    .disabled(false)
    .build();
    // build() — creates the immutable User object
```

---

## Custom UserDetails Implementation

In production you almost always write your own. This gives you access to extra fields (user ID, profile picture, etc.) in your controllers.

```java
// Your database entity
@Entity
@Table(name = "users")
public class AppUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String password; // Stored as bcrypt hash

    @Column(nullable = false)
    private String role; // "ROLE_USER", "ROLE_ADMIN"

    private boolean enabled;
    private boolean accountLocked;

    // getters/setters...
}

// Adapter: wraps AppUser and implements UserDetails
// This is the object Spring Security works with internally
public class AppUserDetails implements UserDetails {

    private final AppUser appUser;
    // Store the real entity so you can access custom fields later

    public AppUserDetails(AppUser appUser) {
        this.appUser = appUser;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        // Convert the role string to a GrantedAuthority
        return List.of(new SimpleGrantedAuthority(appUser.getRole()));
        // SimpleGrantedAuthority — the standard implementation of GrantedAuthority
        // Takes the authority string directly: "ROLE_USER", "ROLE_ADMIN"
    }

    @Override
    public String getPassword() {
        return appUser.getPassword(); // Already encoded — Spring will compare using PasswordEncoder
    }

    @Override
    public String getUsername() {
        return appUser.getEmail(); // We use email as the login identifier
    }

    @Override
    public boolean isAccountNonExpired() {
        return true; // We don't have account expiry in this example
    }

    @Override
    public boolean isAccountNonLocked() {
        return !appUser.isAccountLocked(); // Invert: "non-locked" = "not locked"
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return appUser.isEnabled(); // User must verify email before this is true
    }

    // Custom method — NOT part of UserDetails interface
    // Use this in controllers to get the real user ID without hitting the DB again
    public Long getId() {
        return appUser.getId();
    }

    // Expose the underlying entity if you need all fields
    public AppUser getAppUser() {
        return appUser;
    }
}
```

---

## UserDetailsService Interface

```java
public interface UserDetailsService {

    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
    // loadUserByUsername() — the ONLY method you implement
    // Called during authentication to fetch the user record
    // username — the value from the login form (or Basic auth header)
    // Returns a UserDetails object, or throws UsernameNotFoundException if not found
    // NEVER return null — always throw UsernameNotFoundException
}
```

### Production Implementation: Database-Backed UserDetailsService

```java
@Service
// @Service — marks this as a Spring service bean, available for injection
public class AppUserDetailsService implements UserDetailsService {

    private final AppUserRepository userRepository;
    // Injected via constructor — your JPA repository for the users table

    public AppUserDetailsService(AppUserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // loadUserByUsername() — Spring Security calls this during every login attempt
        // "username" here is whatever the user typed in the username field
        // In our case we use email as username

        AppUser user = userRepository.findByEmail(username)
            // findByEmail() — your repository method to query DB by email
            .orElseThrow(() -> new UsernameNotFoundException(
                "No user found with email: " + username
                // UsernameNotFoundException — MUST throw this (not return null) if user not found
                // Spring Security catches this and translates it to a 401 response
                // Don't expose sensitive info: don't say "user not found" vs "wrong password"
                // (timing attacks and user enumeration are real threats)
            ));

        return new AppUserDetails(user);
        // Wrap in our custom UserDetails adapter
    }
}
```

### Registering UserDetailsService with Spring Security

Spring Boot auto-detects your `UserDetailsService` bean if there's exactly one. But it's best to be explicit:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final AppUserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public SecurityConfig(AppUserDetailsService userDetailsService,
                           PasswordEncoder passwordEncoder) {
        this.userDetailsService = userDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        // AuthenticationProvider — the component that performs the actual authentication
        
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        // DaoAuthenticationProvider — Spring's built-in provider for username/password + DB
        // "Dao" stands for Data Access Object — it uses your UserDetailsService to access data

        provider.setUserDetailsService(userDetailsService);
        // setUserDetailsService() — tells the provider WHERE to load user data from

        provider.setPasswordEncoder(passwordEncoder);
        // setPasswordEncoder() — tells the provider HOW to compare passwords
        // It will call passwordEncoder.matches(rawPassword, storedHash)

        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        // AuthenticationManager — the entry point for authentication
        // This bean is needed if you want to manually trigger auth (e.g., in a login endpoint)
        return config.getAuthenticationManager();
        // getAuthenticationManager() — builds an AuthenticationManager from your config
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authenticationProvider(authenticationProvider())
            // authenticationProvider() — registers your custom provider with the filter chain
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

---

## Getting the Current User in Controllers

Once authenticated, you'll frequently need the current user in your service/controller layer. Three ways:

### Method 1: SecurityContextHolder (works anywhere)

```java
@RestController
public class ProfileController {

    @GetMapping("/api/profile")
    public ResponseEntity<ProfileResponse> getProfile() {
        
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        // getContext() — gets the SecurityContext for the current thread
        // getAuthentication() — gets the Authentication token

        AppUserDetails userDetails = (AppUserDetails) auth.getPrincipal();
        // getPrincipal() — returns the UserDetails object (your AppUserDetails instance)
        // Cast to your custom type to access custom methods like getId()

        Long userId = userDetails.getId();
        // getId() — your custom method not on UserDetails interface
        
        return ResponseEntity.ok(profileService.getProfile(userId));
    }
}
```

### Method 2: `@AuthenticationPrincipal` (cleanest for controllers)

```java
@RestController
public class ProfileController {

    @GetMapping("/api/profile")
    public ResponseEntity<ProfileResponse> getProfile(
            @AuthenticationPrincipal AppUserDetails currentUser) {
        // @AuthenticationPrincipal — Spring MVC injects the principal from SecurityContext
        // Type: your custom UserDetails class (or UserDetails if you use the default)
        // Much cleaner than manually calling SecurityContextHolder
        
        return ResponseEntity.ok(profileService.getProfile(currentUser.getId()));
    }

    @PutMapping("/api/profile")
    public ResponseEntity<ProfileResponse> updateProfile(
            @AuthenticationPrincipal AppUserDetails currentUser,
            @RequestBody ProfileUpdateRequest request) {
        
        // You can use it alongside other parameters naturally
        return ResponseEntity.ok(profileService.updateProfile(currentUser.getId(), request));
    }
}
```

### Method 3: Custom annotation (production pattern)

```java
// Create a custom annotation to be self-documenting
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@AuthenticationPrincipal
// @AuthenticationPrincipal — this is actually what does the injection work
// Adding it to our custom annotation means our annotation has the same effect
public @interface CurrentUser {
}

// Usage — very clean, team knows exactly what @CurrentUser means
@GetMapping("/api/orders")
public List<Order> getMyOrders(@CurrentUser AppUserDetails user) {
    return orderService.getOrdersByUser(user.getId());
}
```

---

## UserDetailsPasswordService (Advanced)

If you want to automatically upgrade password encoding when users log in (e.g., migrating from MD5 to BCrypt):

```java
@Service
public class AppUserDetailsService implements UserDetailsService, UserDetailsPasswordService {
    // UserDetailsPasswordService — optional interface for password upgrades on login

    @Override
    public UserDetails loadUserByUsername(String username) {
        // ... same as before
    }

    @Override
    public UserDetails updatePassword(UserDetails user, String newPassword) {
        // updatePassword() — called by DaoAuthenticationProvider when it detects
        // the stored password uses a weaker encoding than the current default
        // newPassword — already re-encoded with the current PasswordEncoder
        
        String email = user.getUsername();
        userRepository.updatePasswordByEmail(email, newPassword);
        // updatePasswordByEmail() — your repository method to save the new hash
        
        return user;
        // Return the UserDetails object (Spring will reload it on next request)
    }
}
```

---

## In-Memory UserDetailsService (Testing Only)

For tests and demos, you can skip the database:

```java
@Bean
public UserDetailsService inMemoryUserDetailsService() {
    // UserDetailsService — the bean Spring Security will auto-detect and use
    
    UserDetails admin = User.withDefaultPasswordEncoder()
        // withDefaultPasswordEncoder() — DEPRECATED, only for demos/tests, not production
        // Encodes passwords in memory, logs a warning
        .username("admin")
        .password("admin123")
        .roles("ADMIN")
        .build();

    UserDetails alice = User.withDefaultPasswordEncoder()
        .username("alice")
        .password("password")
        .roles("USER")
        .build();

    return new InMemoryUserDetailsManager(admin, alice);
    // InMemoryUserDetailsManager — stores UserDetails in a HashMap, no DB needed
    // loadUserByUsername() looks up the HashMap
    // Perfect for tests, never use in production
}
```

---

## Common Mistakes

### Mistake 1: Returning `null` from `loadUserByUsername`

```java
// Bad — causes NullPointerException deep in Spring Security
public UserDetails loadUserByUsername(String username) {
    AppUser user = userRepository.findByEmail(username).orElse(null);
    if (user == null) return null; // NEVER DO THIS
    return new AppUserDetails(user);
}

// Good — throw the right exception
public UserDetails loadUserByUsername(String username) {
    return userRepository.findByEmail(username)
        .map(AppUserDetails::new)
        .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
}
```

### Mistake 2: Putting the raw entity directly as UserDetails

```java
// Bad — your JPA entity shouldn't implement UserDetails
@Entity
public class AppUser implements UserDetails { // Don't do this
    // Mixes persistence concerns with security concerns
    // JPA might deserialize/detach the entity and lose transient fields
    // Security context is serialized to session — entity serialization is unpredictable
}

// Good — separate adapter class
public class AppUserDetails implements UserDetails {
    private final AppUser user; // Wrap, don't extend
}
```

### Mistake 3: Not handling the enabled/locked status

```java
// If you always return true for isEnabled() and isAccountNonLocked(),
// then disabling a user in your DB has no effect on authentication.
// Make sure these map to real database fields.

@Override
public boolean isEnabled() {
    return appUser.isEmailVerified() && !appUser.isDeleted();
    // Map to your actual business logic
}
```

---

## Summary: What to Implement for Custom DB Authentication

### The 2 mandatory things you implement

#### 1. `UserDetails` (interface)
Your wrapper object that Spring Security works with internally. Create a class (e.g., `AppUserDetails`) that implements it — do NOT make your JPA entity implement it directly.

| Method | What it returns |
|---|---|
| `getUsername()` | Login identifier (email/username) |
| `getPassword()` | Stored BCrypt hash from DB |
| `getAuthorities()` | Roles like `ROLE_USER`, `ROLE_ADMIN` |
| `isEnabled()` | Is account active? |
| `isAccountNonLocked()` | Not locked (e.g., too many failed logins) |
| `isAccountNonExpired()` | Return `true` if you don't use account expiry |
| `isCredentialsNonExpired()` | Return `true` if you don't use password expiry |

#### 2. `UserDetailsService` (interface)
Your bridge between Spring Security and your database. One method only:

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

Query the DB here, wrap the result in your `UserDetails` class, and return it. Always throw `UsernameNotFoundException` if not found — never return `null`.

### The 1 class you configure (not implement)

**`DaoAuthenticationProvider`** — configure it in `SecurityConfig` and hand it your `UserDetailsService` + `PasswordEncoder`. It wires everything together.

### Flow at login time

```
Login request
    → DaoAuthenticationProvider
        → calls loadUserByUsername(email)         ← your UserDetailsService
            → queries DB, returns AppUserDetails  ← your UserDetails
        → calls passwordEncoder.matches(raw, hash)
        → checks isEnabled(), isAccountNonLocked(), etc.
    → success or exception
```

---

## Summary: Exception Handling and Provider Chain Behavior

`ProviderManager` (Spring's default `AuthenticationManager`) iterates through registered `AuthenticationProvider`s. How it handles exceptions determines whether the next provider gets a chance.

### Category 1 — Immediate hard stop (next provider is NOT tried)

Thrown when the user was found but their account has a problem. Spring treats these as definitive answers — no point asking another provider.

| Exception | Trigger |
|---|---|
| `DisabledException` | `isEnabled()` returns `false` |
| `LockedException` | `isAccountNonLocked()` returns `false` |
| `AccountExpiredException` | `isAccountNonExpired()` returns `false` |
| `CredentialsExpiredException` | `isCredentialsNonExpired()` returns `false` |
| `InternalAuthenticationServiceException` | Unexpected exception inside `loadUserByUsername()` (e.g., DB is down) |

The first four all extend `AccountStatusException`. `ProviderManager` catches that parent type and rethrows immediately.

### Category 2 — Soft failure (next provider IS tried)

| Exception | Trigger |
|---|---|
| `BadCredentialsException` | Wrong password |
| `UsernameNotFoundException` | User not found in DB |

> **Gotcha:** By default, `DaoAuthenticationProvider` converts `UsernameNotFoundException` into `BadCredentialsException` before it leaves the provider. This prevents **user enumeration attacks** — an attacker shouldn't be able to tell whether the email exists or the password was wrong. Disable with `provider.setHideUserNotFoundExceptions(false)` only in non-production environments.

### ProviderManager loop (simplified)

```
for each AuthenticationProvider:
    try:
        result = provider.authenticate(request)
        if result != null → SUCCESS, stop
    catch AccountStatusException | InternalAuthenticationServiceException:
        RETHROW IMMEDIATELY  ← next provider never runs
    catch AuthenticationException:
        save as lastException, TRY NEXT PROVIDER

if no provider succeeded:
    throw lastException  ← usually BadCredentialsException
```

### When "try next provider" actually matters

In most apps there is only one `AuthenticationProvider` (`DaoAuthenticationProvider`), so "try next" just means authentication fails with the saved exception. In apps that support multiple login methods — e.g., username+password AND OAuth — you would have multiple providers chained, and a `BadCredentialsException` from the first one lets the second one get a chance.
