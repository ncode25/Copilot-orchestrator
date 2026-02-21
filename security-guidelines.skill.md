---
name: SecurityGuidelines
description: Mandatory .NET Core security guidelines — input validation, auth, secrets, headers, data protection
---

# .NET Core Security Guidelines

Apply these security guidelines to ALL code you write. These are non-negotiable requirements — violations are treated as bugs.

---

## 1. Input Validation & SQL Injection Prevention

### Mandatory Rules
- **NEVER** concatenate user input into SQL queries — always use parameterized queries or EF Core LINQ
- **NEVER** trust client-side validation alone — always validate server-side
- Validate ALL inputs at the API boundary using **FluentValidation** or **Data Annotations**
- Apply `[MaxLength]`, `[Range]`, `[RegularExpression]` on DTOs and entities
- Sanitize any user input that will be rendered in responses (anti-XSS)
- Use `AllowedValues` / enum binding to restrict inputs to known values

### Patterns
- Use FluentValidation `AbstractValidator<T>` with `.NotEmpty()`, `.MaximumLength()`, `.Must()` rules
- Register validators via `AddValidatorsFromAssemblyContaining<T>()`
- Wire validation into the pipeline (MediatR `IPipelineBehavior<,>` or minimal API filters)
- Return **RFC 7807 ProblemDetails** for validation failures — never expose internal details

### Forbidden
- `FromSqlRaw()` with string interpolation of user input
- `ExecuteSqlRaw()` with concatenated parameters
- Trusting `[FromQuery]`, `[FromRoute]`, `[FromBody]` values without validation

---

## 2. Authentication & Authorization

### Mandatory Rules
- **NEVER** implement custom auth schemes — use ASP.NET Core Identity, JWT Bearer, or OAuth2/OIDC
- **ALWAYS** enforce authorization on every endpoint — no anonymous access unless explicitly intended
- Use **policy-based authorization** (`[Authorize(Policy = "...")]`) over role checks where possible
- Validate JWT tokens with proper issuer, audience, lifetime, and signing key checks
- Use `[Authorize]` at controller level, `[AllowAnonymous]` only for specific public endpoints

### Patterns
- Configure `AddAuthentication().AddJwtBearer()` with `TokenValidationParameters`
- Define authorization policies in `AddAuthorization(options => options.AddPolicy(...))`
- Use `IAuthorizationHandler` for custom authorization logic
- Propagate `ClaimsPrincipal` via `IHttpContextAccessor` — never pass tokens as strings between services
- Use resource-based authorization for object-level access control

### Forbidden
- Hard-coded roles in `[Authorize(Roles = "Admin")]` scattered across controllers — centralize in policies
- Storing JWTs in localStorage (for BFF scenarios — use HttpOnly cookies)
- Disabling token validation for "convenience" (`ValidateIssuer = false, ValidateAudience = false`)

---

## 3. Secrets Management

### Mandatory Rules
- **NEVER** hard-code secrets, API keys, connection strings, or credentials in source code
- **NEVER** commit secrets to version control — not even in `appsettings.Development.json`
- Use **User Secrets** (`dotnet user-secrets`) for local development
- Use **Azure Key Vault**, **AWS Secrets Manager**, or **HashiCorp Vault** for deployed environments
- Rotate secrets regularly — design for rotation without downtime

### Patterns
- Load secrets via `IConfiguration` from environment-appropriate providers:
  ```
  builder.Configuration.AddAzureKeyVault(...)   // Production
  builder.Configuration.AddUserSecrets<Program>() // Development
  ```
- Use `IOptionsMonitor<T>` for secrets that may rotate at runtime
- Connection strings via `builder.Configuration.GetConnectionString("Name")` — never inline

### Forbidden
- Secrets in `appsettings.json`, `launchSettings.json`, or any tracked file
- Logging secrets or connection strings — even at `Debug` level
- Passing secrets as command-line arguments (visible in process lists)

---

## 4. CORS & HTTP Security Headers

### Mandatory Rules
- **NEVER** use `AllowAnyOrigin()` with `AllowCredentials()` — this is a security vulnerability
- Define explicit CORS policies with allowed origins, methods, and headers
- Apply security headers on all responses

### Required Headers
- `Strict-Transport-Security` (HSTS) — enforce HTTPS: `max-age=31536000; includeSubDomains`
- `X-Content-Type-Options: nosniff` — prevent MIME sniffing
- `X-Frame-Options: DENY` — prevent clickjacking
- `Content-Security-Policy` — restrict resource loading origins
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy` — disable unused browser features

### Patterns
- Configure CORS in `Program.cs`:
  ```
  builder.Services.AddCors(options =>
      options.AddPolicy("AllowSpecificOrigins", policy =>
          policy.WithOrigins("https://app.example.com")
                .WithMethods("GET", "POST", "PUT", "DELETE")
                .WithHeaders("Authorization", "Content-Type")));
  ```
- Apply headers via middleware or `NWebsec` / custom middleware
- Use `app.UseHsts()` and `app.UseHttpsRedirection()` in production

### Forbidden
- `AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader()` in production
- Missing HSTS in production deployments
- Exposing `Server` header with version info

---

## 5. Data Protection & Encryption

### Mandatory Rules
- **NEVER** store passwords in plain text — use ASP.NET Core Identity's password hasher or `BCrypt`
- **NEVER** implement custom encryption algorithms — use proven libraries
- Encrypt sensitive data at rest (PII, financial data, health records)
- Enforce TLS 1.2+ for all data in transit — reject older protocols
- Use ASP.NET Core **Data Protection APIs** for encrypting application-specific data (cookies, tokens, temp data)

### Patterns
- Configure Data Protection:
  ```
  builder.Services.AddDataProtection()
      .PersistKeysToAzureBlobStorage(...)
      .ProtectKeysWithAzureKeyVault(...);
  ```
- Use `IDataProtector` for encrypting/decrypting application data
- Hash sensitive identifiers before logging (e.g., mask email: `n***@example.com`)
- Use `[PersonalData]` attribute on GDPR-relevant entity properties

### Forbidden
- `MD5` or `SHA1` for password hashing — use `PBKDF2`, `BCrypt`, or `Argon2`
- Storing encryption keys alongside encrypted data
- Disabling HTTPS in any environment
- Logging PII (emails, SSNs, credit card numbers) — even at `Debug` level

---

## 6. Additional Security Practices

### Rate Limiting
- Apply rate limiting on public endpoints using `AddRateLimiter()` (ASP.NET Core 7+)
- Use sliding window or token bucket algorithms for API endpoints
- Return `429 Too Many Requests` with `Retry-After` header

### Anti-Forgery (CSRF)
- Use `[ValidateAntiForgeryToken]` on state-changing actions in MVC/Razor Pages
- For APIs using cookie auth, validate the `X-XSRF-TOKEN` header

### Dependency Security
- Audit NuGet packages for known vulnerabilities (`dotnet list package --vulnerable`)
- Pin package versions to avoid unexpected updates
- Remove unused packages to reduce attack surface

### Error Disclosure
- **NEVER** expose stack traces, internal paths, or database errors to clients
- Use `app.UseExceptionHandler()` — not `app.UseDeveloperExceptionPage()` in production
- Return generic error messages to clients; log full details server-side
