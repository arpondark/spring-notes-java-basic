# Spring Security for Beginners — A Complete, Practical Guide

This guide is designed as a concise “book-length” set of notes to take a beginner from zero to building secure Spring Boot applications with Spring Security. It focuses on fundamentals, the modern Spring Security model (5.7+), and hands-on patterns used by top engineers. Examples use Spring Boot 3.x, Java 17+, and the new component-based security configuration (no WebSecurityConfigurerAdapter).

Contents:

- Part I — Foundations
- Part II — Core Concepts Deep Dive
- Part III — Authentication, Authorization, and Customization
- Part IV — Modern App Scenarios (JWT, OAuth2, Sessions)
- Part V — Method Security and Domain-Driven Permissions
- Part VI — CSRF, CORS, and Common Web Concerns
- Part VII — Testing Security
- Part VIII — Production Hardening Checklist
- Part IX — Troubleshooting, Patterns, and Anti-Patterns
- Part X — Quick Reference

Note: Code blocks favor clarity; in real projects, split into files and add proper error handling.

***

## Part I — Foundations

### What Is Spring Security?

Spring Security is Spring’s framework for authentication (who are you) and authorization (what can you do), plus secure defaults for modern web apps and APIs. It works via servlet filters (on the web stack) and method interceptors (for method security). It is highly extensible and follows a “compose with beans” model.

### Key Terms

- Authentication: Verifying identity; results in Authentication object in SecurityContext.
- Authorization: Deciding access to resources based on roles/authorities, rules, or policies.
- Principal: The authenticated user or system identity.
- GrantedAuthority: A permission or role (e.g., ROLE_ADMIN, SCOPE_read).
- SecurityContext: Thread-bound storage holding Authentication.
- FilterChain: Ordered filters handling security concerns per request.
- CSRF: Protection against cross-site request forgery.
- CORS: Cross-origin resource sharing rules.


### Spring Security From 10,000 ft

- Defaults to secure: All endpoints require authentication unless explicitly permitted.
- Declarative configuration using HttpSecurity and SecurityFilterChain beans.
- Supports form login, HTTP Basic, sessions, stateless JWT, OAuth2 login, OAuth2 resource server, SAML2, and more.
- Method security enables fine-grained protection with annotations like @PreAuthorize.

***

## Part II — Core Concepts Deep Dive

### The Security Filter Chain

Spring Security builds a chain of servlet filters. The most important:

- SecurityContextHolderFilter: Binds SecurityContext to the thread.
- Authentication filters: e.g., UsernamePasswordAuthenticationFilter, BearerTokenAuthenticationFilter.
- AuthorizationFilter: Applies authorization rules (formerly FilterSecurityInterceptor path role checks).
- CsrfFilter, LogoutFilter, ExceptionTranslationFilter: CSRF handling, logout, access denials.

You configure these via a SecurityFilterChain bean.

### SecurityContext and Authentication

- After successful authentication, an Authentication object is placed in SecurityContextHolder.
- Access it in controllers via Authentication or @AuthenticationPrincipal.

Example:

```java
@GetMapping("/me")
public Map<String, Object> me(Authentication auth) {
  return Map.of("name", auth.getName(), "authorities", auth.getAuthorities());
}
```


### Roles vs Authorities

- By convention, roles start with ROLE_ (e.g., ROLE_ADMIN).
- Authorities can be any string (e.g., SCOPE_read for OAuth scopes).
- hasRole("ADMIN") is sugar for hasAuthority("ROLE_ADMIN").

***

## Part III — Authentication, Authorization, and Customization

### Minimal Spring Boot Security Setup

Dependencies (build.gradle.kts example):

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-security")
  implementation("org.springframework.boot:spring-boot-starter-validation")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.springframework.security:spring-security-test")
}
```

Security configuration:

```java
@Configuration
@EnableMethodSecurity // enables @PreAuthorize, @PostAuthorize, etc.
public class SecurityConfig {

  @Bean
  SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
      .csrf(csrf -> csrf.disable()) // For APIs without browser session; enable for browser apps.
      .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // if JWT API
      .authorizeHttpRequests(auth -> auth
         .requestMatchers("/public/**", "/actuator/health").permitAll()
         .requestMatchers(HttpMethod.GET, "/docs/**").permitAll()
         .requestMatchers("/admin/**").hasRole("ADMIN")
         .anyRequest().authenticated()
      )
      .httpBasic(Customizer.withDefaults()); // or .formLogin(Customizer.withDefaults())
    return http.build();
  }

  // For demo only: in-memory users
  @Bean
  UserDetailsService userDetailsService(PasswordEncoder encoder) {
    UserDetails user = User.withUsername("user")
      .password(encoder.encode("password"))
      .roles("USER")
      .build();
    UserDetails admin = User.withUsername("admin")
      .password(encoder.encode("password"))
      .roles("ADMIN")
      .build();
    return new InMemoryUserDetailsManager(user, admin);
  }

  @Bean
  PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
  }
}
```

Controller example:

```java
@RestController
@RequestMapping("/api")
public class DemoController {

  @GetMapping("/public/hello")
  public String publicHello() { return "Hello, world"; }

  @GetMapping("/user/hello")
  public String userHello(Authentication auth) {
    return "Hello, " + auth.getName();
  }

  @GetMapping("/admin/hello")
  public String adminHello() { return "Hello, admin"; }
}
```

- With HTTP Basic: call /api/user/hello using Authorization: Basic header.
- With formLogin(): Spring generates a login page for browser flows.


### Custom Authentication

Most apps authenticate via:

- DAO authentication (UserDetailsService + PasswordEncoder) — form login.
- Token auth (JWT) — stateless APIs.
- OAuth2/OIDC — delegate to identity providers (Google, Okta, Keycloak).
- LDAP, SAML2 — enterprise environments.

Custom AuthenticationProvider:

```java
@Component
public class ApiKeyAuthProvider implements AuthenticationProvider {
  @Override
  public Authentication authenticate(Authentication auth) {
    String apiKey = (String) auth.getCredentials();
    if ("secret-key-123".equals(apiKey)) {
      return new UsernamePasswordAuthenticationToken("service-client", null,
        List.of(new SimpleGrantedAuthority("ROLE_SERVICE")));
    }
    throw new BadCredentialsException("Invalid API key");
  }

  @Override
  public boolean supports(Class<?> authentication) {
    return ApiKeyAuthentication.class.isAssignableFrom(authentication);
  }
}
```

Register a custom filter to read the API key and call AuthenticationManager.

### Authorization Rules

- Path-based: configure in HttpSecurity.authorizeHttpRequests.
- Method-based: @PreAuthorize("hasRole('ADMIN')") on service/controller methods.
- Post-filtering: @PostAuthorize for returns, @PostFilter for collections.
- SpEL allows referencing method args, return values, beans: @PreAuthorize("@perm.canView(\#id, authentication)")

***

## Part IV — Modern App Scenarios

### A. Stateless JWT Resource Server

Use this for REST APIs with tokens issued by your auth server.

Dependencies:

- spring-boot-starter-oauth2-resource-server
- spring-security-oauth2-jose

Configuration:

```java
@Configuration
@EnableMethodSecurity
public class ResourceServerSecurity {

  @Bean
  SecurityFilterChain api(HttpSecurity http) throws Exception {
    http
      .csrf(csrf -> csrf.disable())
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/public/**").permitAll()
        .anyRequest().authenticated()
      )
      .oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
      );
    return http.build();
  }

  @Bean
  Converter<Jwt, ? extends AbstractAuthenticationToken> jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter scopes = new JwtGrantedAuthoritiesConverter();
    scopes.setAuthorityPrefix("SCOPE_"); // default
    scopes.setAuthoritiesClaimName("scope"); // or "scp" or "authorities"

    return jwt -> {
      Collection<GrantedAuthority> authorities = scopes.convert(jwt);
      // Optionally map custom roles claim to ROLE_
      List<String> roles = jwt.getClaimAsStringList("roles");
      if (roles != null) {
        authorities = Stream.concat(
          authorities.stream(),
          roles.stream().map(r -> new SimpleGrantedAuthority("ROLE_" + r))
        ).collect(Collectors.toSet());
      }
      return new JwtAuthenticationToken(jwt, authorities, jwt.getSubject());
    };
  }
}
```

application.yml (point to JWKS of your IdP):

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://idp.example.com/oauth2/jwks
```

Use @PreAuthorize("hasAuthority('SCOPE_read')") or hasRole('ADMIN') depending on claims mapping.

Key notes:

- Do not store sessions for JWT APIs; set stateless policy.
- Validate audience (aud) and other claims if needed via custom JwtValidator.


### B. Form Login + Session (Traditional MVC)

- Enable CSRF (default) and use formLogin(). Ensure logout POST with CSRF token or configure logout for GET only if justified.
- Use remember-me only with secure cookies and short lifetimes.


### C. OAuth2 Login (Client) + OIDC

For Single Sign-On in web apps:

- Add spring-boot-starter-oauth2-client.
- Configure client registration (client-id/secret, scopes, redirect-uri).
- http.oauth2Login(Customizer.withDefaults()).
- Use @AuthenticationPrincipal OidcUser to read profile, email, roles.


### D. API Keys

- Implement custom filter extracting header (e.g., X-API-KEY).
- Authenticate to a ROLE_SERVICE authority.
- Rate-limit and scope keys; rotate regularly.

***

## Part V — Method Security and Domain-Driven Permissions

Enable:

```java
@EnableMethodSecurity(prePostEnabled = true)
```

Common annotations:

- @PreAuthorize("hasRole('ADMIN')")
- @PreAuthorize("\#id == authentication.name")
- @PreAuthorize("@perm.canEdit(\#docId, authentication)")
- @PostAuthorize("returnObject.owner == authentication.name")
- @PreFilter("@perm.filterOwned(authentication, filterObject)")
- @PostFilter("filterObject.visibility == 'PUBLIC'")

Permission service:

```java
@Component("perm")
public class PermissionService {
  private final DocumentRepository repo;

  public boolean canEdit(UUID docId, Authentication auth) {
    var doc = repo.findById(docId).orElse(null);
    if (doc == null) return false;
    return doc.getOwner().equals(auth.getName()) ||
           auth.getAuthorities().stream().anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
  }
}
```

Use method security primarily at service layer to protect business invariants even if a controller is misconfigured.

***

## Part VI — CSRF, CORS, and Common Web Concerns

### CSRF

- Keep CSRF enabled for browser-based session apps (form login). Disable only for stateless APIs or non-browser clients.
- When enabled, include token in forms and X-CSRF-TOKEN header for AJAX.
- For split frontend/backends with cookies, either disable CSRF for API endpoints or configure CookieCsrfTokenRepository and send header.


### CORS

- Configure allowed origins, methods, headers precisely.
- Avoid global “*” in production if credentials are used.
Example:

```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
  var config = new CorsConfiguration();
  config.setAllowedOrigins(List.of("https://app.example.com"));
  config.setAllowedMethods(List.of("GET","POST","PUT","DELETE"));
  config.setAllowedHeaders(List.of("Authorization","Content-Type"));
  config.setAllowCredentials(true);
  var source = new UrlBasedCorsConfigurationSource();
  source.registerCorsConfiguration("/**", config);
  return source;
}
```

Enable via http.cors(Customizer.withDefaults()).

### Headers, HTTPS, Cookies

- Force HTTPS at the reverse proxy; set server.forward-headers-strategy=framework.
- Use Strict-Transport-Security (HSTS).
- Set SameSite, HttpOnly, Secure flags for cookies.

***

## Part VII — Testing Security

Add:

- spring-security-test

Patterns:

```java
@WebMvcTest(controllers = DemoController.class)
@Import(SecurityConfig.class)
class DemoControllerTest {

  @Autowired MockMvc mvc;

  @Test
  void publicAccessible() throws Exception {
    mvc.perform(get("/api/public/hello"))
       .andExpect(status().isOk());
  }

  @Test
  @WithMockUser(username="user", roles={"USER"})
  void userEndpointAuthenticated() throws Exception {
    mvc.perform(get("/api/user/hello"))
       .andExpect(status().isOk());
  }

  @Test
  void adminDeniedWithoutRole() throws Exception {
    mvc.perform(get("/api/admin/hello"))
       .andExpect(status().isUnauthorized()); // if no auth at all
  }

  @Test
  @WithMockUser(roles={"USER"})
  void adminDeniedForUserRole() throws Exception {
    mvc.perform(get("/api/admin/hello"))
       .andExpect(status().isForbidden());
  }
}
```

JWT testing:

```java
@Test
@WithMockJwtAuth(authorities = {"SCOPE_read","ROLE_ADMIN"})
void jwtProtected() { /* custom annotation or use SecurityMockMvcRequestPostProcessors.jwt() */ }
```

Or:

```java
mvc.perform(get("/secure").with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_read"))))
   .andExpect(status().isOk());
```

Method security tests:

- Use @WithMockUser and call service methods directly.
- Verify access denied via assertThrows(AccessDeniedException.class, ...).

***

## Part VIII — Production Hardening Checklist

Authentication:

- Use strong password encoding (bcrypt default via DelegatingPasswordEncoder).
- Enforce password policies, MFA (via IdP), and lockout/backoff.

Tokens:

- Prefer short-lived access tokens; use refresh token rotation.
- Validate audience, issuer, expiry; reject none algorithms.
- Store secrets safely; rotate keys regularly.

Sessions:

- Set stateless for APIs; for sessions, set session fixation protection (default), concurrency control limits, short idle timeouts.

Transport:

- Enforce TLS 1.2+; HSTS at edge; secure cookies.

Authorization:

- Principle of least privilege; design permissions explicitly.
- Centralize authorization decisions or policies (method security or ABAC).

Input/Output:

- Validate inputs; sanitize outputs; avoid exposing internal IDs (use UUIDs).
- Disable default error whitelabels leaking stack traces in prod.

Headers:

- Content-Security-Policy (CSP) where applicable.
- X-Content-Type-Options, X-Frame-Options (or frame-ancestors via CSP), Referrer-Policy.

Operational:

- Monitor authentication failures, suspicious patterns.
- Log security events with correlation IDs; avoid logging secrets.
- Run dependency checks; keep Spring Boot/Security updated.

***

## Part IX — Troubleshooting, Patterns, and Anti-Patterns

Common pitfalls and fixes:

- 403 vs 401 confusion:
    - 401 Unauthorized = not authenticated, 403 Forbidden = authenticated but not allowed.
    - Ensure endpoints are properly matched in order; most specific matchers first.
- CSRF 403 on APIs:
    - Disable CSRF for stateless APIs or configure token/header in client.
- hasRole not working:
    - Ensure authorities are prefixed with ROLE_. Use hasAuthority if not using role prefix.
- Multiple filter chains:
    - Use ordered SecurityFilterChain beans targeted by securityMatcher("/api/**") vs default; ensures API and MVC have distinct configs (JWT vs form login).

Example multiple chains:

```java
@Bean
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
  http.securityMatcher("/api/**")
    .csrf(csrf -> csrf.disable())
    .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
    .oauth2ResourceServer(oauth2 -> oauth2.jwt());
  return http.build();
}

@Bean
@Order(Ordered.LOWEST_PRECEDENCE)
SecurityFilterChain mvcChain(HttpSecurity http) throws Exception {
  http.authorizeHttpRequests(auth -> auth
      .requestMatchers("/login","/public/**").permitAll()
      .anyRequest().authenticated())
    .formLogin(Customizer.withDefaults());
  return http.build();
}
```

- Custom JWT claim mapping:
    - Verify which claim carries roles (realm_access/resource_access in Keycloak, cognito:groups in Cognito). Map consistently.
- CORS not applied:
    - Define CorsConfigurationSource bean and http.cors(); ensure proxy/front-end origin matches.

Anti-patterns:

- Disabling security globally for convenience.
- Using plaintext passwords or NoOp encoder.
- Accepting all origins with credentials in CORS.
- Packing business authorization in controllers only; enforce in services too.

***

## Part X — Quick Reference

Authentication options:

- Form login + session: http.formLogin()
- HTTP Basic (development/testing, or service-to-service over TLS): http.httpBasic()
- OAuth2 login (OIDC): http.oauth2Login()
- Resource server JWT: http.oauth2ResourceServer().jwt()
- Custom token/api key: custom filter + AuthenticationProvider

Authorization expressions:

- Path rules: authorizeHttpRequests().requestMatchers("...").hasRole("ADMIN").anyRequest().authenticated()
- Method rules: @PreAuthorize("hasAnyAuthority('SCOPE_read','ROLE_ADMIN')")
- SpEL variables: authentication, principal, \#paramName, returnObject

CSRF:

- Enable for browser sessions, disable for stateless APIs.

CORS:

- Configure specific origins, headers, methods; enable with http.cors().

Password encoding:

- PasswordEncoderFactories.createDelegatingPasswordEncoder() gives bcrypt by default.

Testing helpers:

- @WithMockUser, SecurityMockMvcRequestPostProcessors.{user, jwt, oauth2Login}

***

## Appendix — End-to-End JWT Example

Dependencies:

- spring-boot-starter-web
- spring-boot-starter-security
- spring-boot-starter-validation
- spring-boot-starter-oauth2-resource-server
- spring-security-oauth2-jose

Config:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://issuer.example.com/ # Preferred over jwk-set-uri
server:
  forward-headers-strategy: framework
```

Security:

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

  @Bean
  SecurityFilterChain api(HttpSecurity http) throws Exception {
    http
      .csrf(csrf -> csrf.disable())
      .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/public/**").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/docs/**").permitAll()
        .requestMatchers("/api/admin/**").hasRole("ADMIN")
        .anyRequest().authenticated()
      )
      .oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
      );
    return http.build();
  }

  @Bean
  Converter<Jwt, AbstractAuthenticationToken> jwtAuthenticationConverter() {
    var scopes = new JwtGrantedAuthoritiesConverter();
    scopes.setAuthorityPrefix("SCOPE_");
    scopes.setAuthoritiesClaimName("scope");
    return jwt -> {
      Set<GrantedAuthority> authorities = new HashSet<>(scopes.convert(jwt));
      List<String> roles = Optional.ofNullable(jwt.getClaimAsStringList("roles")).orElse(List.of());
      roles.forEach(r -> authorities.add(new SimpleGrantedAuthority("ROLE_" + r)));
      return new JwtAuthenticationToken(jwt, authorities, jwt.getSubject());
    };
  }
}
```

Controller:

```java
@RestController
@RequestMapping("/api")
public class ApiController {
  @GetMapping("/public/ping")
  public Map<String,String> ping() { return Map.of("status", "ok"); }

  @PreAuthorize("hasAuthority('SCOPE_read')")
  @GetMapping("/data")
  public Map<String,String> data() { return Map.of("data", "secure"); }

  @PreAuthorize("hasRole('ADMIN')")
  @DeleteMapping("/admin/item/{id}")
  public ResponseEntity<Void> delete(@PathVariable String id) { return ResponseEntity.noContent().build(); }
}
```


***

## Learning Path and Practice

1) Start with HTTP Basic + in-memory users to understand the flow.
2) Switch to formLogin and learn CSRF flows for browser apps.
3) Build a stateless API with JWT; protect endpoints by scopes and roles.
4) Add method security to enforce business rules in services.
5) Integrate OAuth2 login with an external IdP for SSO.
6) Write security tests for each scenario.
7) Harden configuration and deploy behind TLS with proper headers.

Hands-on exercises:

- Exercise A: Create endpoints /public, /user, /admin with different access rules.
- Exercise B: Implement a custom filter to read X-API-KEY and grant ROLE_SERVICE.
- Exercise C: Map custom roles from JWT into ROLE_ authorities and secure endpoints.
- Exercise D: Add @PreAuthorize with a PermissionService checking ownership.
- Exercise E: Configure CORS for a SPA at https://app.example.com and verify preflight.

***

If a specific stack (e.g., Keycloak, Okta, Cognito) or app type (SPA + backend, mobile + backend, server-rendered MVC) is in mind, the configuration can be tailored further with precise mappings, claim names, and flows.

