# Password Encoding

## Why You Must Never Store Plain Text Passwords

If your database is compromised, and passwords are stored as plain text, every user's password is immediately exposed. Password encoding (hashing) means even if attackers get the database, they get useless hash strings, not usable passwords.

**Hashing is NOT encryption.** Encryption is reversible. A good password hash is a one-way function — you can verify a password against a hash, but you can't reverse a hash to get the password.

---

## The PasswordEncoder Interface

```java
public interface PasswordEncoder {

    String encode(CharSequence rawPassword);
    // encode() — takes a raw/plain text password and returns an encoded (hashed) string
    // Call this ONCE when a user registers or changes their password
    // Store the RESULT in your database, never the raw password

    boolean matches(CharSequence rawPassword, String encodedPassword);
    // matches() — checks if rawPassword, when encoded, matches the stored encodedPassword
    // Call this during login to verify the user's entered password
    // Returns true if they match, false otherwise
    // NEVER decode encodedPassword — it's one-way

    default boolean upgradeEncoding(String encodedPassword) {
        // upgradeEncoding() — returns true if the encoded password should be re-encoded
        // Used for automatic migration to stronger encoding
        // Most implementations return false (no upgrade needed)
        return false;
    }
}
```

---

## BCryptPasswordEncoder — The Production Standard

BCrypt is the recommended encoder for most applications. Here's why it's better than MD5/SHA:

- **Salt built-in**: Each password gets a unique random salt, so two users with the same password get different hashes
- **Slow by design**: BCrypt is intentionally slow. This makes brute-force attacks very expensive
- **Configurable cost**: You can tune the work factor as hardware gets faster

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // BCryptPasswordEncoder — implements the BCrypt hashing function
    return new BCryptPasswordEncoder(12);
    // 12 — the "strength" or "cost factor" (work factor)
    // This means 2^12 = 4096 rounds of hashing
    // Higher = more secure but slower (12 is a reasonable production default)
    // Default if you don't specify: 10 (2^10 = 1024 rounds)
    // Typical login time at 12: ~200-300ms — slow enough for attackers, fine for users
}

// Using it in a registration flow:
@Service
public class AuthService {

    private final PasswordEncoder passwordEncoder;
    private final AppUserRepository userRepository;

    public AuthService(PasswordEncoder passwordEncoder, AppUserRepository userRepository) {
        this.passwordEncoder = passwordEncoder;
        this.userRepository = userRepository;
    }

    public void register(RegisterRequest request) {
        
        // WRONG — never store this
        // String rawPassword = request.getPassword();
        
        // CORRECT — hash before storing
        String encodedPassword = passwordEncoder.encode(request.getPassword());
        // encode() — takes the raw password from the form and returns a BCrypt hash
        // Result looks like: $2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
        
        AppUser user = new AppUser();
        user.setEmail(request.getEmail());
        user.setPassword(encodedPassword); // Store the HASH
        user.setRole("ROLE_USER");
        
        userRepository.save(user);
    }

    public void changePassword(Long userId, ChangePasswordRequest request) {
        AppUser user = userRepository.findById(userId).orElseThrow();

        // Verify current password first
        if (!passwordEncoder.matches(request.getCurrentPassword(), user.getPassword())) {
            // matches() — Spring Security also calls this internally during login
            // You call it manually here to verify the current password before changing it
            throw new BadCredentialsException("Current password is incorrect");
        }

        user.setPassword(passwordEncoder.encode(request.getNewPassword()));
        userRepository.save(user);
    }
}
```

---

## DelegatingPasswordEncoder — The Modern Approach

BCrypt is great, but what happens when you need to:
- Migrate from an old system that stored MD5 hashes?
- Upgrade BCrypt strength over time without forcing all users to re-register?

`DelegatingPasswordEncoder` solves this. It stores the encoder ID **with the hash** as a prefix:

```
{bcrypt}$2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
{noop}plainTextPassword
{sha256}abc123...
   ↑
   encoder ID prefix
```

```java
@Bean
public PasswordEncoder passwordEncoder() {
    
    // PasswordEncoderFactories.createDelegatingPasswordEncoder() — creates a pre-configured
    // DelegatingPasswordEncoder that supports: bcrypt (default), noop, sha256, pbkdf2, scrypt
    // This is what Spring Security recommends for production
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}

// Or create a custom one:
@Bean
public PasswordEncoder passwordEncoder() {
    
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    // Map of encoder ID -> encoder implementation
    
    encoders.put("bcrypt", new BCryptPasswordEncoder(12));
    // "bcrypt" — the encoder ID that will appear as {bcrypt} prefix in stored hash
    
    encoders.put("sha256", new StandardPasswordEncoder());
    // Legacy encoder for migrating old SHA-256 hashes
    
    encoders.put("noop", NoOpPasswordEncoder.getInstance());
    // "noop" — plain text, ONLY for legacy migration or tests, never for new passwords
    
    DelegatingPasswordEncoder delegating = new DelegatingPasswordEncoder("bcrypt", encoders);
    // DelegatingPasswordEncoder(defaultEncoderId, encoders)
    // defaultEncoderId — ID used when encoding NEW passwords (always use bcrypt)
    // encoders — map for DECODING (checking) stored passwords by their prefix

    return delegating;
}
```

### Migration Scenario: Moving from MD5 to BCrypt

```java
// Scenario: You have 100,000 users with MD5-hashed passwords from 10 years ago.
// You can't force them all to reset passwords immediately.
// Solution: DelegatingPasswordEncoder + upgrade on login.

// Step 1: Old passwords stored as: 5f4dcc3b5aa765d61d8327deb882cf99 (MD5, no prefix)
// Step 2: Add MD5 support with prefix mapping in UserDetailsService:

@Service
public class MigratingUserDetailsService implements UserDetailsService, UserDetailsPasswordService {

    @Override
    public UserDetails loadUserByUsername(String username) {
        AppUser user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("Not found: " + username));

        // If the stored password has no prefix, it's an old MD5 hash.
        // Temporarily prefix it so DelegatingPasswordEncoder knows how to check it.
        String storedPassword = user.getPassword();
        if (!storedPassword.startsWith("{")) {
            storedPassword = "{md5}" + storedPassword;
            // Prepend the encoder ID so DelegatingPasswordEncoder routes to the MD5 checker
        }

        return User.builder()
            .username(user.getEmail())
            .password(storedPassword)
            .roles(user.getRole())
            .build();
    }

    @Override
    public UserDetails updatePassword(UserDetails user, String newEncodedPassword) {
        // updatePassword() — called automatically by DaoAuthenticationProvider
        // when upgradeEncoding() returns true for the stored password.
        // DelegatingPasswordEncoder returns true when the stored encoding != default encoding.
        
        // newEncodedPassword is already re-encoded with BCrypt by the time we get it here
        userRepository.updatePasswordByEmail(user.getUsername(), newEncodedPassword);
        // Now the DB stores: {bcrypt}$2a$12$... for this user
        // Next login: no upgrade needed, stays BCrypt
        
        return User.withUserDetails(user).password(newEncodedPassword).build();
        // withUserDetails() — copies all fields from existing UserDetails
        // Returns updated UserDetails with the new password
    }
}
```

---

## Other PasswordEncoder Implementations

```java
// Argon2 — memory-hard, the strongest modern option (requires bouncycastle dependency)
// Use when you can afford higher memory usage per hash
Argon2PasswordEncoder argon2 = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();

// SCrypt — also memory-hard, older than Argon2
SCryptPasswordEncoder scrypt = SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8();

// PBKDF2 — NIST-approved, commonly used in regulated environments
Pbkdf2PasswordEncoder pbkdf2 = Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8();

// NoOpPasswordEncoder — plain text, ONLY for tests/demos
// Using it in production is a security vulnerability
PasswordEncoder noop = NoOpPasswordEncoder.getInstance();
// getInstance() — singleton, no constructor needed
```

---

## Testing with Passwords

```java
// In tests, use BCrypt or NoOp for simplicity
@TestConfiguration
// @TestConfiguration — a @Configuration that only applies in test context
public class TestSecurityConfig {

    @Bean
    @Primary
    // @Primary — if there are multiple beans of same type, this one wins
    public PasswordEncoder testPasswordEncoder() {
        return NoOpPasswordEncoder.getInstance();
        // NoOp — plain text comparison, makes test data setup easier
        // "password" == "password" without any hashing
    }
}

// Or just use BCrypt in tests too (slightly slower but more realistic):
class AuthServiceTest {

    private final PasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
    // Instantiate directly in tests — no Spring context needed

    @Test
    void registerShouldHashPassword() {
        String raw = "myPassword123";
        String encoded = passwordEncoder.encode(raw);
        
        assertThat(encoded).doesNotContain(raw);
        // The hash should not contain the raw password
        
        assertThat(passwordEncoder.matches(raw, encoded)).isTrue();
        // matches() — verify the raw password matches the hash
        
        assertThat(passwordEncoder.matches("wrongPassword", encoded)).isFalse();
    }
}
```

---

## Complete Registration + Login Example

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<String> register(@RequestBody @Valid RegisterRequest request) {
        // @Valid — triggers Bean Validation on the request body
        authService.register(request);
        return ResponseEntity.ok("Registration successful");
    }

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request) {
        return ResponseEntity.ok(authService.login(request));
    }
}

@Service
public class AuthService {

    private final AppUserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final AuthenticationManager authenticationManager;
    private final JwtTokenProvider jwtTokenProvider;

    public void register(RegisterRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            // existsByEmail() — check for duplicate before inserting
            throw new EmailAlreadyExistsException("Email already registered");
        }

        AppUser user = new AppUser();
        user.setEmail(request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        // encode() — hash the password before saving to DB
        user.setRole("ROLE_USER");
        user.setEnabled(false); // Email verification required first

        userRepository.save(user);
        // At this point password in DB is: {bcrypt}$2a$12$...
        
        emailService.sendVerificationEmail(user.getEmail());
    }

    public LoginResponse login(LoginRequest request) {
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getEmail(),
                request.getPassword()
                // UsernamePasswordAuthenticationToken(username, password)
                // 2-arg constructor — creates an UNAUTHENTICATED token (credentials not verified yet)
                // AuthenticationManager will verify and return a new authenticated token
            )
        );
        // authenticate() — triggers the full auth flow:
        //   1. Loads user via UserDetailsService
        //   2. Calls passwordEncoder.matches(enteredPassword, storedHash)
        //   3. Returns authenticated Authentication if matched
        //   4. Throws AuthenticationException if failed

        AppUserDetails userDetails = (AppUserDetails) authentication.getPrincipal();
        // getPrincipal() — returns your AppUserDetails object after successful auth

        String token = jwtTokenProvider.generateToken(authentication);
        // generateToken() — your method to create a signed JWT

        return new LoginResponse(token, userDetails.getId(), userDetails.getUsername());
    }
}
```

---

## Common Mistakes

### Mistake 1: Encoding a password twice

```java
// Bad — encoding twice means login will always fail
// First encode: "$2a$12$abc..."
// Second encode: "$2a$12$xyz..." (encoding of the first hash string)
// During login: raw password != double-encoded hash

user.setPassword(passwordEncoder.encode(passwordEncoder.encode(rawPassword))); // WRONG

// Good — encode exactly once
user.setPassword(passwordEncoder.encode(rawPassword)); // CORRECT
```

### Mistake 2: Comparing passwords manually

```java
// Bad — comparing stored hash to raw password directly always returns false
if (user.getPassword().equals(request.getPassword())) { // WRONG
    // This compares "$2a$12$abc..." == "myPlainPassword" → always false
}

// Good — use the encoder's matches method
if (passwordEncoder.matches(request.getPassword(), user.getPassword())) { // CORRECT
    // This uses BCrypt's compare algorithm
}
```

### Mistake 3: Logging or returning passwords

```java
// Bad — logs the raw password, a serious security violation
log.debug("User {} logged in with password {}", username, rawPassword); // NEVER

// Bad — returns the hashed password in API response
return new UserResponse(user.getEmail(), user.getPassword()); // NEVER expose hashes

// Good — never log passwords, never include in responses
return new UserResponse(user.getEmail()); // Only return safe fields
```
