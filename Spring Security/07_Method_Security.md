# Method-Level Security

## What is Method Security?

URL-level security (in `SecurityFilterChain`) protects endpoints based on URL patterns. But sometimes you need finer control:

- A user can GET `/api/documents/{id}` — but only if it's their own document
- An admin can call `deleteUser()` — but not a regular manager
- Certain return values should be filtered based on who's asking

Method security lets you put access rules **directly on your service methods**, using annotations.

---

## Enabling Method Security

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
// @EnableMethodSecurity — activates @PreAuthorize, @PostAuthorize, @PreFilter, @PostFilter
// Must be on a @Configuration class
// Replace old @EnableGlobalMethodSecurity(prePostEnabled = true) — that's deprecated

// Optional parameters:
// @EnableMethodSecurity(securedEnabled = true)  — enables @Secured annotation
// @EnableMethodSecurity(jsr250Enabled = true)   — enables @RolesAllowed from javax.annotation
public class SecurityConfig {
    // ... rest of config
}
```

---

## @PreAuthorize — Check Before the Method Runs

The most commonly used annotation. Blocks the method from executing if the condition is false.

```java
@Service
public class UserService {

    // --- Basic role checks ---

    @PreAuthorize("hasRole('ADMIN')")
    // @PreAuthorize — evaluates SpEL expression BEFORE the method body runs
    // hasRole('ADMIN') — checks if SecurityContext has ROLE_ADMIN authority
    // If false → throws AccessDeniedException → returns 403
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'SUPPORT')")
    // hasAnyRole() — user must have at least one of the listed roles
    public UserDetails getUserDetails(Long userId) {
        return userRepository.findById(userId).orElseThrow();
    }

    @PreAuthorize("hasAuthority('USER_WRITE')")
    // hasAuthority() — checks for exact authority string (no ROLE_ prefix added)
    // Use this for fine-grained permissions, not broad roles
    public void updateUserProfile(Long userId, ProfileUpdateRequest request) {
        // ...
    }

    // --- Combining conditions ---

    @PreAuthorize("hasRole('ADMIN') and hasAuthority('SYSTEM_ACCESS')")
    // and/or — standard boolean logic in SpEL
    public void performSystemOperation() {
        // Requires BOTH ROLE_ADMIN AND SYSTEM_ACCESS
    }

    // --- Using method parameters ---

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    // #userId — refers to the method parameter named 'userId'
    // authentication — the current Authentication object in SecurityContext
    // authentication.principal — the UserDetails (your AppUserDetails object)
    // authentication.principal.id — calls getId() on your AppUserDetails
    // Meaning: admins can access any user's data, users can only access their own
    public UserProfile getProfile(Long userId) {
        return userRepository.findProfileById(userId).orElseThrow();
    }

    @PreAuthorize("#email == authentication.name")
    // authentication.name — calls getName() on Authentication = the username/email
    // Users can only look up their own email
    public AppUser findByEmail(String email) {
        return userRepository.findByEmail(email).orElseThrow();
    }

    // --- Using custom security expressions ---

    @PreAuthorize("@orderSecurity.canModify(authentication, #orderId)")
    // @beanName — calls a method on a Spring bean as part of the SpEL expression
    // This is the most powerful pattern — delegate complex logic to a dedicated bean
    public void updateOrder(Long orderId, OrderUpdateRequest request) {
        // ...
    }
}

// The bean called by @orderSecurity above:
@Component("orderSecurity")
// The bean name matches what we used in @PreAuthorize("@orderSecurity...")
public class OrderSecurityService {

    private final OrderRepository orderRepository;

    public boolean canModify(Authentication authentication, Long orderId) {
        // Any logic you need — DB calls, business rules, etc.
        AppUserDetails user = (AppUserDetails) authentication.getPrincipal();
        
        // Admins can modify any order
        if (user.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            return true;
        }
        
        // Customers can only modify their own pending orders
        return orderRepository.findById(orderId)
            .map(order -> order.getCustomerId().equals(user.getId())
                       && order.getStatus() == OrderStatus.PENDING)
            .orElse(false);
    }
}
```

---

## @PostAuthorize — Check After the Method Runs

Use when the authorization decision depends on what the method **returned**.

```java
@Service
public class DocumentService {

    @PostAuthorize("returnObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
    // @PostAuthorize — the method RUNS FIRST, then the expression is evaluated
    // returnObject — the value returned by the method
    // If the expression is false, AccessDeniedException is thrown AFTER the method ran
    // Warning: side effects (DB writes) from the method can't be rolled back by @PostAuthorize
    public Document getDocument(Long documentId) {
        // This fetches the document from DB (runs regardless of who calls it)
        // THEN Spring checks if the caller owns it
        return documentRepository.findById(documentId).orElseThrow();
    }

    @PostAuthorize("returnObject != null ? returnObject.department == authentication.principal.department : true")
    // Complex SpEL — null-safe check
    // returnObject != null ? ... : true — if no record found (null), allow it (let caller handle null)
    // returnObject.department — custom field on the returned object
    // authentication.principal.department — custom field on your UserDetails
    public Employee findEmployee(Long id) {
        return employeeRepository.findById(id).orElse(null);
    }
}
```

**Caution:** If the method has side effects (writes to DB, sends email), `@PostAuthorize` can't undo them. Prefer `@PreAuthorize` when possible.

---

## @PreFilter — Filter Input Collections

Use to filter a collection argument **before** the method runs.

```java
@Service
public class DocumentService {

    @PreFilter("filterObject.ownerId == authentication.principal.id")
    // @PreFilter — filters elements from a Collection/Array parameter before method runs
    // filterObject — each element in the collection
    // Only elements that pass the expression are passed to the method
    // Elements failing the expression are silently removed from the collection
    public void processDocuments(List<Document> documents) {
        // By the time this runs, 'documents' only contains docs owned by the current user
        // Even if the caller tried to sneak in someone else's documents
        documents.forEach(doc -> doc.setProcessed(true));
        documentRepository.saveAll(documents);
    }
    
    // If your method has multiple params, specify which to filter:
    @PreFilter(value = "filterObject.status == 'PENDING'", filterTarget = "orders")
    // filterTarget = "orders" — explicitly names which parameter to filter
    public void bulkApprove(List<Order> orders, String approvalNote) {
        // orders is pre-filtered; only PENDING orders remain
        orders.forEach(o -> o.setStatus("APPROVED"));
    }
}
```

---

## @PostFilter — Filter Return Collections

Use to filter a collection **returned** by the method.

```java
@Service
public class DocumentService {

    @PostFilter("filterObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
    // @PostFilter — removes elements from the returned Collection that don't pass the expression
    // filterObject — each element in the returned collection
    // WARNING: The method fetches ALL records from DB, then filters in memory
    //          For large datasets, use @PreAuthorize with a custom query instead
    public List<Document> getAllDocuments() {
        return documentRepository.findAll(); // Returns all from DB
        // Spring then filters this list, returning only the caller's own documents
        // (or all documents if caller is ADMIN)
    }
}
```

**Production warning:** `@PostFilter` loads everything into memory and then discards. For large collections, write a filtered query instead:

```java
// Better for large data — filter at DB level, not in memory
@PreAuthorize("isAuthenticated()")
public List<Document> getMyDocuments() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    AppUserDetails user = (AppUserDetails) auth.getPrincipal();
    
    if (user.getAuthorities().stream().anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
        return documentRepository.findAll();
    }
    return documentRepository.findByOwnerId(user.getId());
    // findByOwnerId() — query at SQL level, much more efficient
}
```

---

## @Secured (Simpler, Less Powerful)

```java
@Secured("ROLE_ADMIN")
// @Secured — simpler than @PreAuthorize, only supports exact role names
// Unlike @PreAuthorize, no SpEL support (can't use #params or method calls)
// Requires @EnableMethodSecurity(securedEnabled = true)
public void adminOnlyMethod() { }

@Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
// Multiple roles — user must have AT LEAST ONE (not all)
public void adminOrManagerMethod() { }
```

Use `@PreAuthorize` over `@Secured` unless you're on an older codebase.

---

## @RolesAllowed (JSR-250, Standard Java)

```java
@RolesAllowed("ADMIN")
// @RolesAllowed — standard Java annotation (javax.annotation.security)
// Behaves like @Secured but is framework-neutral (works without Spring)
// Requires @EnableMethodSecurity(jsr250Enabled = true)
// Note: NO "ROLE_" prefix in @RolesAllowed — different from @PreAuthorize and @Secured
public void adminMethod() { }

@RolesAllowed({"ADMIN", "MANAGER"})
public void adminOrManagerMethod() { }

@DenyAll
// @DenyAll — nobody can call this method, regardless of role
// Useful to explicitly block access and communicate intent to readers
public void blockedMethod() { }

@PermitAll
// @PermitAll — anyone can call this, even unauthenticated
// Overrides any URL-level security for this method
public void publicMethod() { }
```

---

## SpEL (Spring Expression Language) Cheat Sheet

SpEL is the expression language used in `@PreAuthorize` and `@PostAuthorize`.

```java
// Authentication object
authentication                          // The full Authentication object
authentication.name                     // Calls getName() → username/email
authentication.principal                // The UserDetails object
authentication.principal.id             // Custom field on your UserDetails
authentication.authorities              // Collection of GrantedAuthority

// Role/authority checks
hasRole('ADMIN')                        // Has ROLE_ADMIN authority
hasAnyRole('ADMIN', 'MANAGER')          // Has ROLE_ADMIN or ROLE_MANAGER
hasAuthority('USER_WRITE')              // Has exact authority string
hasAnyAuthority('USER_WRITE', 'ADMIN')  // Has any of these exact authority strings

// Authentication state
isAuthenticated()                       // Not anonymous
isAnonymous()                           // Not authenticated
isFullyAuthenticated()                  // Not anonymous AND not remember-me
isRememberMe()                          // Authenticated via remember-me cookie

// Method parameters (with #)
#paramName                              // Value of the parameter named 'paramName'
#entity.field                           // Field access on a parameter object

// Return value (in @PostAuthorize/@PostFilter)
returnObject                            // The value returned by the method
returnObject.field                      // Field access on the returned object

// Bean access
@beanName.method(args)                  // Call a method on a Spring bean

// Combining
hasRole('ADMIN') and #userId > 0
hasRole('ADMIN') or #email == authentication.name
!isAnonymous() and hasAuthority('READ')
```

---

## Complete Production Example: Document Management System

```java
@Service
public class DocumentService {

    private final DocumentRepository documentRepository;
    private final DocumentSharingRepository sharingRepository;

    // Anyone can list public documents, but authenticated users also see their private ones
    @PreAuthorize("isAuthenticated()")
    public List<Document> getAccessibleDocuments() {
        AppUserDetails user = (AppUserDetails)
            SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        
        if (hasRole("ROLE_ADMIN")) {
            return documentRepository.findAll();
        }
        return documentRepository.findAccessibleByUserId(user.getId());
    }

    // Users can view their own documents OR documents shared with them
    // Admins can view all
    @PreAuthorize("@documentSecurity.canView(authentication, #documentId)")
    public Document getDocument(Long documentId) {
        return documentRepository.findById(documentId).orElseThrow();
    }

    // Only the document owner or admin can edit
    @PreAuthorize("@documentSecurity.isOwner(authentication, #documentId) or hasRole('ADMIN')")
    public Document updateDocument(Long documentId, DocumentUpdateRequest request) {
        Document doc = documentRepository.findById(documentId).orElseThrow();
        doc.setTitle(request.getTitle());
        doc.setContent(request.getContent());
        return documentRepository.save(doc);
    }

    // Only the document owner or admin can delete
    @PreAuthorize("@documentSecurity.isOwner(authentication, #documentId) or hasRole('ADMIN')")
    public void deleteDocument(Long documentId) {
        documentRepository.deleteById(documentId);
    }

    // Owner can share with others; admin can share any document
    @PreAuthorize("@documentSecurity.isOwner(authentication, #documentId) or hasRole('ADMIN')")
    public void shareDocument(Long documentId, Long targetUserId) {
        DocumentSharing sharing = new DocumentSharing(documentId, targetUserId);
        sharingRepository.save(sharing);
    }
    
    private boolean hasRole(String role) {
        return SecurityContextHolder.getContext().getAuthentication()
            .getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals(role));
    }
}

@Component("documentSecurity")
// Bean name matches @PreAuthorize("@documentSecurity...")
public class DocumentSecurityService {

    private final DocumentRepository documentRepository;
    private final DocumentSharingRepository sharingRepository;

    public boolean canView(Authentication authentication, Long documentId) {
        AppUserDetails user = (AppUserDetails) authentication.getPrincipal();
        Long userId = user.getId();
        
        // Admins can view all
        if (authentication.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            return true;
        }
        
        return documentRepository.findById(documentId)
            .map(doc -> 
                doc.getOwnerId().equals(userId) ||   // Owner
                doc.isPublic() ||                     // Public document
                sharingRepository.existsByDocumentIdAndSharedWithUserId(documentId, userId)
                // Shared with this user
            )
            .orElse(false);
    }

    public boolean isOwner(Authentication authentication, Long documentId) {
        AppUserDetails user = (AppUserDetails) authentication.getPrincipal();
        return documentRepository.findById(documentId)
            .map(doc -> doc.getOwnerId().equals(user.getId()))
            .orElse(false);
    }
}
```

---

## Common Mistakes

### Mistake 1: Putting @PreAuthorize on a private or package-private method

```java
// Bad — Spring Security uses AOP proxies. Proxies intercept calls from OUTSIDE the class.
// Calling a private method from within the same class bypasses the proxy entirely.
@PreAuthorize("hasRole('ADMIN')")
private void internalMethod() { // WRONG — proxy can't intercept private methods
    // This annotation has NO EFFECT
}

// Good — only put security annotations on public methods
@PreAuthorize("hasRole('ADMIN')")
public void publicMethod() { // Correct — AOP proxy can intercept this
    internalHelper(); // Private helper — no annotation needed
}
```

### Mistake 2: Calling a secured method from the same bean (self-invocation)

```java
@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN')")
    public void adminOperation() { }

    public void normalOperation() {
        this.adminOperation(); // BYPASSES SECURITY — calls the real method, not the proxy
        // AOP proxy only intercepts calls from OTHER beans
    }
}

// Fix: inject the bean into itself (Spring 4.3+) or use ApplicationContext
@Service
public class OrderService {

    @Autowired
    private OrderService self; // Spring injects the PROXIED version
    // @Autowired — injects the Spring-managed (proxied) bean, not 'this'

    public void normalOperation() {
        self.adminOperation(); // Goes through the proxy — security is enforced
    }
}
```

### Mistake 3: Forgetting @EnableMethodSecurity

```java
// If you add @PreAuthorize to your methods but forget @EnableMethodSecurity,
// ALL CALLS SUCCEED regardless of the annotation. No error, no warning.
// The annotation is silently ignored.

// Always put this on your SecurityConfig:
@EnableMethodSecurity // Don't forget!
public class SecurityConfig { }
```
