# BizCart — Authentication Module Technical Approach

## Document Information

| Field               | Value                                           |
| ------------------- | ----------------------------------------------- |
| Project             | BizCart — Small Business Commerce Platform      |
| Module              | Authentication                                  |
| Backend             | Spring Boot Monolithic Application              |
| Database            | MySQL                                           |
| Document Type       | Technical Approach                              |
| Status              | Ready for Development                           |
| Version             | 1.2                                             |
| Related Requirement | `bizcart-authentication-module-requirements.md` |
| API Base Path       | `/api/v1/auth`                                  |

---

## 1. Overview / Introduction

The BizCart Authentication module will securely authenticate users and manage their sessions across the platform.

The module will use:

* JWT access tokens for protected API access
* Database-backed refresh tokens for long-lived sessions
* BCrypt for password encoding
* HMAC-SHA-256 for refresh, reset and verification-token hashing
* Spring Security for authentication and authorisation
* MySQL for persistent session and token storage
* Spring Security OAuth2 Client for Google OAuth 2.0 social login
* Redis for local phone OTP verification storage and expiry management

The module will support:

* User login
* JWT access-token generation
* Refresh-token generation and rotation
* Current-device logout
* Logout from all devices
* Forgot-password flow
* Password reset
* Email verification
* Verification-email resend
* Password change
* Current authenticated-user retrieval
* Account-status validation
* Login-attempt tracking
* Roles and permissions in the authenticated session
* Google OAuth 2.0 social login
* Phone OTP verification using Redis in the local Spring profile
* Google OAuth 2.0 for social login
* Redis for local phone OTP verification

This document explains how the Authentication requirements will be implemented in the BizCart Spring Boot monolithic backend.

---

## 2. Objective

The objective of the Authentication module is to provide a secure, scalable and maintainable authentication mechanism for all BizCart users.

The module must:

* Authenticate registered users using email and password.
* Prevent unauthorised access to protected APIs.
* Issue short-lived JWT access tokens.
* Support database-backed refresh-token sessions.
* Support multiple active devices.
* Allow current-device and all-device logout.
* Prevent refresh-token reuse.
* Protect passwords and sensitive tokens.
* Validate user status before login and token refresh.
* Support password recovery and email verification.
* Provide roles and permissions to downstream modules.
* Track successful and failed login attempts.
* Follow common BizCart validation and exception formats.

* Support Google OAuth 2.0 social login.
* Support Redis-backed phone OTP verification in the local profile.
* Keep two-factor authentication available for future enhancement.


---

## 3. Final Architecture Decisions

### 3.1 Application Architecture

BizCart uses a modular monolithic Spring Boot architecture.

Authentication will remain a dedicated module inside the same backend application.

The module will use:

* One primary authentication controller
* One main authentication service
* Limited focused helper services
* Separate request and response DTO packages
* Separate entities and repositories for authentication-specific data

The module will not create unnecessary controllers or services for every individual endpoint.

### 3.2 User Type and Store Staff Decision

The primary user types are:

```text
ADMIN
SELLER
CUSTOMER
```

`STORE_STAFF` will not be stored as a separate primary user type.

Store Staff will be represented through:

* A user account
* A store membership
* A `STORE_STAFF` role
* Store-specific permissions

This avoids expanding the main user-type enum and supports assigning different roles to staff members.

### 3.3 Token Transport Decision

For the BizCart browser-based frontend:

#### Access Token

* Returned in the login or refresh response body.
* Stored in frontend application memory.
* Sent through the `Authorization` header.
* Must not be stored in localStorage.

```http
Authorization: Bearer <access-token>
```

#### Refresh Token

* Sent as a `Secure`, `HttpOnly` cookie.
* Not accessible through frontend JavaScript.
* Automatically sent to refresh and logout endpoints.
* Not returned inside the JSON response body.

Recommended cookie settings:

```text
HttpOnly: true
Secure: true in deployed environments
SameSite: Lax
Path: /api/v1/auth
Max-Age: 7 days
```

For local development, the `Secure` flag may be disabled through environment-specific configuration.

### 3.4 CSRF Decision

Because the refresh token is stored in a cookie:

* Login, refresh-token and logout endpoints must apply controlled CSRF protection.
* The access token remains protected through the Bearer header.
* SameSite cookie protection will be enabled.
* Approved frontend origins will be explicitly configured.
* Cross-origin credentials will be allowed only for trusted frontend domains.

For the first version, CSRF protection will use a cookie-to-header token approach for cookie-authenticated endpoints.

The frontend must send the CSRF value through a custom header such as:

```http
X-CSRF-TOKEN: <csrf-token>
```

### 3.5 JWT Algorithm Decision

The initial implementation will use:

```text
HS256
```

Requirements:

* Minimum 256-bit random secret
* Secret supplied through environment configuration
* Different secret for each environment
* Secret never committed to Git
* Future support for RS256 when independent services require token verification

### 3.6 Token Expiry Decisions

| Token                    |     Expiry |
| ------------------------ | ---------: |
| Access token             | 15 minutes |
| Refresh token            |     7 days |
| Password-reset token     | 15 minutes |
| Email-verification token |   24 hours |

All expiry durations must be configurable through environment properties.


### 3.7 Google OAuth 2.0 Decision

BizCart will support Google OAuth 2.0 login as a social-login option.

Google OAuth will be used only for external identity verification.

BizCart will still generate its own JWT access token and refresh-token session after successful Google login.

The backend will receive user information from Google and map it to the internal BizCart user account.

The initial OAuth provider will be:

```text
GOOGLE
```

Future providers such as Facebook, GitHub or Apple may be added later.

Google OAuth credentials must be supplied through environment variables and must never be committed to Git.

### 3.8 Phone OTP Verification Decision

BizCart will support phone OTP verification.

For the local Spring profile, the OTP implementation will use the following approach:

```text
Spring profile: local
OTP: Random 6 digit
SMS: Console log only
Storage: Redis
Expiry: 5 minutes
Max attempts: 3
```

Real SMS gateway integration is not part of the local implementation.

In local development, the generated OTP will be printed in the backend console for testing through Postman or frontend.

---

## 4. Proposed Approach / Solution

## 4.1 Authentication Strategy

The Authentication module will use a hybrid session model:

* JWT access tokens provide stateless API authentication.
* Refresh tokens provide persistent session management.
* Refresh tokens are stored in MySQL as hashes.
* Each successful login creates a separate refresh-token session.
* Multiple devices can maintain independent sessions.

The application will not store active sessions in server memory.

---

## 4.2 Login Flow

1. The client submits email and password.
2. The request DTO validates the input.
3. The email is trimmed and converted to lowercase.
4. The system retrieves the user by email.
5. The account status is validated.
6. Email-verification and approval rules are validated.
7. The password is verified using BCrypt.
8. Roles and permissions are loaded.
9. A JWT access token is generated.
10. A cryptographically secure refresh token is generated.
11. The refresh token is hashed using HMAC-SHA-256.
12. The hashed token is stored with device and IP information.
13. `last_login_at` is updated.
14. A successful login-attempt event is stored.
15. The access token and safe user details are returned.
16. The raw refresh token is set as an HttpOnly cookie.

For invalid credentials:

* Unknown email and incorrect password return the same response.
* Failed attempts are recorded.
* Passwords and tokens are never logged.

---

## 4.3 Account Status Behaviour

### Customer

| Status   | Email Verified |   Login |                    Business APIs |
| -------- | -------------: | ------: | -------------------------------: |
| PENDING  |             No |  Denied |                           Denied |
| ACTIVE   |            Yes | Allowed | Allowed according to permissions |
| INACTIVE |            Any |  Denied |                           Denied |
| BLOCKED  |            Any |  Denied |                           Denied |

After successful email verification, a Customer may be changed from `PENDING` to `ACTIVE`.

### Seller

| Status   | Email Verified | Admin Approved |             Login |                    Business APIs |
| -------- | -------------: | -------------: | ----------------: | -------------------------------: |
| PENDING  |             No |             No |            Denied |                           Denied |
| PENDING  |            Yes |             No | Limited or denied |                           Denied |
| ACTIVE   |            Yes |            Yes |           Allowed | Allowed according to permissions |
| INACTIVE |            Any |            Any |            Denied |                           Denied |
| BLOCKED  |            Any |            Any |            Denied |                           Denied |

A Seller requires:

* Email verification
* Admin approval
* `ACTIVE` account status

The final approval workflow belongs to the Seller or User Management module.

### Admin

Admin accounts must be:

* Created or approved through a controlled administrative process
* Active
* Assigned valid admin roles

---

## 4.4 Access Token Flow

The access token will contain:

* `sub` — User ID
* `email`
* `userType`
* `roles`
* `tokenVersion`
* `iat`
* `exp`
* `jti`

Permissions will not be placed directly into the JWT in the initial version.

Permissions will be loaded through the user’s roles when the authenticated principal is created.

This avoids:

* Oversized tokens
* Stale permission lists
* Exposing excessive internal permission details

The access token will be valid for 15 minutes.

---

## 4.5 Refresh Token Design

Each refresh-token record will contain:

* User ID
* Token hash
* Token family ID
* Parent-token ID
* Replaced-by-token ID
* Device information
* IP address
* Expiry
* Revocation timestamp
* Revocation reason
* Creation timestamp

### Refresh Flow

1. The browser sends the refresh-token cookie.
2. The backend hashes the submitted token.
3. The token record is retrieved.
4. The token is checked for:

   * Existence
   * Expiry
   * Revocation
   * User ownership
5. The associated user is loaded.
6. The user status is validated.
7. The old token is atomically revoked.
8. A new access token is generated.
9. A new refresh token is generated.
10. The new refresh token is stored in the same token family.
11. The old record references the replacement token.
12. The new access token is returned.
13. The new refresh token replaces the old HttpOnly cookie.

The old token and new token creation will happen in one database transaction.

---

## 4.6 Refresh-Token Reuse Detection

If an already rotated refresh token is used again:

1. The request will be rejected.
2. The event will be treated as possible token theft.
3. All active tokens in the same token family will be revoked.
4. A security event will be recorded.
5. The user will be required to log in again.

Suggested error code:

```text
AUTH_REFRESH_TOKEN_REUSE_DETECTED
```

---

## 4.7 Concurrent Refresh Handling

Two simultaneous refresh requests must not produce two valid replacement tokens.

The token record will be updated atomically.

Example condition:

```sql
UPDATE refresh_tokens
SET revoked_at = CURRENT_TIMESTAMP,
    revocation_reason = 'ROTATED'
WHERE id = ?
  AND revoked_at IS NULL
  AND expires_at > CURRENT_TIMESTAMP;
```

The refresh operation proceeds only when exactly one row is updated.

A pessimistic lock or atomic update may be used depending on repository implementation.

---

## 4.8 Logout Flow

### Current-Device Logout

1. The backend reads the refresh-token cookie.
2. The token is hashed and located.
3. The current token is revoked.
4. The refresh-token cookie is cleared.
5. A successful response is returned.

Calling logout with an already revoked token may still return a safe success response.

### Logout From All Devices

1. The authenticated user is identified from the access token.
2. All active refresh tokens belonging to that user are revoked.
3. The current refresh-token cookie is cleared.
4. A successful response is returned.

The existing access token may remain valid for up to 15 minutes.

---

## 4.9 Forgot-Password Flow

1. The user submits an email address.
2. The email is normalised.
3. The system returns a generic response.
4. If the user exists:

   * Previous active reset tokens are invalidated.
   * A secure token is generated.
   * The token is hashed with HMAC-SHA-256.
   * The token expires after 15 minutes.
   * A password-reset event is created.
   * The email delivery process is triggered.

Generic response:

```text
If the email is registered, a password reset link has been sent.
```

---

## 4.10 Reset-Password Flow

1. The user submits the token, new password and confirmation password.
2. The token is hashed.
3. The token record is loaded.
4. The system verifies that the token:

   * Exists
   * Has not expired
   * Has not been used
5. The new password is validated.
6. The password and confirmation are compared.
7. The password is encoded using BCrypt.
8. The user password is updated.
9. The reset token is marked as used.
10. All active refresh tokens are revoked.
11. The user must log in again.

Concurrent use of the same reset token will be prevented using an atomic database update.

---

## 4.11 Email-Verification Flow

1. A secure verification token is generated.
2. The token is hashed.
3. Previous active verification tokens are invalidated.
4. The token is stored with a 24-hour expiry.
5. A verification email is triggered.
6. The user opens the verification link.
7. The token is validated.
8. `email_verified` is updated to `true`.
9. The token is marked as verified.
10. The account status is updated according to the user type.

For Customers:

```text
PENDING → ACTIVE
```

For Sellers:

Email verification does not automatically activate the account when admin approval is still pending.

---

## 4.12 Change-Password Flow

1. The user is identified through the access token.
2. The current password is verified.
3. The new password is validated.
4. The new password must differ from the current password.
5. The confirmation password must match.
6. The new password is BCrypt encoded.
7. The password is updated.
8. The user’s token version is incremented.
9. All active refresh tokens are revoked.
10. The user must log in again.

---

## 4.13 Current-User Flow

The `/api/v1/auth/me` endpoint will return:

* User ID
* First name
* Last name
* Email
* Phone
* Profile image
* User type
* Account status
* Email-verification status
* Roles
* Permissions
* Store memberships where applicable

The endpoint must not return:

* Password
* Token hashes
* JWT secret information
* Internal security fields

---


### 4.14 Google OAuth 2.0 Login Flow

1. The user clicks "Login with Google" on the frontend.
2. The frontend redirects the user to the backend Google OAuth start endpoint.
3. The backend redirects the user to Google OAuth consent screen.
4. The user selects a Google account and grants permission.
5. Google redirects back to the backend callback endpoint with an authorization code.
6. The backend exchanges the authorization code for Google user information.
7. The backend validates that the Google email is verified.
8. The backend searches the user by email.
9. If the user exists, the system logs in the existing user.
10. If the user does not exist, the system creates a new user with provider `GOOGLE`.
11. The backend generates a BizCart JWT access token.
12. The backend generates a BizCart refresh token.
13. The refresh token is stored as a hash.
14. The raw refresh token is sent through an HTTP-only cookie.
15. The access token and safe user details are returned or redirected to the frontend.

Google OAuth must not replace BizCart JWT logic.  
OAuth is used only to verify the external identity.




### 4.15 Phone OTP Verification Flow

1. The user submits a phone number and OTP purpose.
2. The backend validates the phone number.
3. The backend generates a random 6 digit OTP.
4. The OTP is hashed before storage.
5. The OTP data is stored in Redis.
6. Redis TTL is set to 5 minutes.
7. In the local profile, the OTP is printed in the backend console.
8. The user submits the OTP for verification.
9. The backend reads the OTP data from Redis.
10. The backend checks OTP expiry and attempt count.
11. If the OTP is wrong, attempt count is increased.
12. After 3 wrong attempts, the OTP becomes invalid.
13. If the OTP is correct, the phone number is marked as verified.
14. After successful verification, the OTP key is deleted from Redis.

Redis key format:

```text
otp:phone:{phoneNumber}:{purpose}
```

Example:

```text
otp:phone:9876543210:PHONE_VERIFICATION
```


## 5. Technology Stack and Justification

| Technology                   | Usage                            | Reason                                                       |
| ---------------------------- | -------------------------------- | ------------------------------------------------------------ |
| Java 21                      | Backend language                 | LTS release and modern Java features                         |
| Spring Boot                  | Application framework            | Simplifies REST, configuration and dependency injection      |
| Spring Security              | Authentication and authorisation | Standard security filter chain and access control            |
| Spring Security OAuth2 Client | Google OAuth 2.0 login           | Standard OAuth2 client support for Google login              |
| Spring Data JPA              | Persistence                      | Repository abstraction and parameterised queries             |
| Hibernate                    | ORM                              | Entity-to-table mapping                                      |
| MySQL 8                      | Primary database                 | Reliable relational storage                                  |
| Redis                        | Phone OTP storage                | Fast temporary OTP storage with TTL support                  |
| JWT library                  | Access tokens                    | Stateless authentication                                     |
| BCrypt                       | Password hashing                 | Adaptive and salted password protection                      |
| HMAC-SHA-256                 | Token hashing                    | Deterministic secure token lookup using a server-side pepper |
| Bean Validation              | Request validation               | Reusable DTO validation                                      |
| Flyway                       | Database migration               | Version-controlled schema changes                            |
| Spring Mail/provider adapter | Email delivery                   | Reset and verification emails                                |
| Springdoc OpenAPI            | Swagger documentation            | API collaboration and testing                                |
| JUnit 5                      | Testing                          | Standard Spring Boot testing framework                       |
| Mockito                      | Unit tests                       | Mocking dependencies                                         |
| Testcontainers               | Integration tests                | Real MySQL test environment                                  |
| Maven                        | Build management                 | Stable dependency and build process                          |
| Docker                       | Local environment                | Consistent application and MySQL setup                       |

---

## 6. Module Breakdown and File Structure

The module will use a practical layered structure suitable for the BizCart monolithic backend.

```text
authentication/
├── AuthController.java
├── OAuthController.java
├── PhoneOtpController.java
├── AuthService.java
├── JwtService.java
├── RefreshTokenService.java
├── PasswordService.java
├── VerificationService.java
├── OAuthService.java
├── PhoneOtpService.java
├── SmsService.java
├── ConsoleSmsService.java
│
├── dto/
│   ├── request/
│   │   ├── LoginRequest.java
│   │   ├── SendPhoneOtpRequest.java
│   │   ├── VerifyPhoneOtpRequest.java
│   │   ├── ForgotPasswordRequest.java
│   │   ├── ResetPasswordRequest.java
│   │   ├── VerifyEmailRequest.java
│   │   ├── ResendVerificationRequest.java
│   │   └── ChangePasswordRequest.java
│   │
│   └── response/
│       ├── LoginResponse.java
│       ├── OAuthLoginResponse.java
│       ├── PhoneOtpResponse.java
│       ├── TokenResponse.java
│       ├── CurrentUserResponse.java
│       └── MessageResponse.java
│
├── entity/
│   ├── RefreshToken.java
│   ├── PasswordResetToken.java
│   ├── VerificationToken.java
│   └── LoginAttempt.java
│
├── repository/
│   ├── RefreshTokenRepository.java
│   ├── PasswordResetTokenRepository.java
│   ├── VerificationTokenRepository.java
│   └── LoginAttemptRepository.java
│
├── mapper/
│   └── AuthMapper.java
│
├── exception/
│   ├── InvalidCredentialsException.java
│   ├── InvalidTokenException.java
│   ├── TokenExpiredException.java
│   ├── TokenRevokedException.java
│   ├── TokenReuseDetectedException.java
│   ├── AccountBlockedException.java
│   └── AccountInactiveException.java
│
└── config/
    ├── AuthenticationProperties.java
    ├── OAuth2Config.java
    └── RedisConfig.java
```

Shared security classes:

```text
security/
├── SecurityConfig.java
├── JwtAuthenticationFilter.java
├── AuthenticatedUserPrincipal.java
├── CustomUserDetailsService.java
├── RestAuthenticationEntryPoint.java
└── RestAccessDeniedHandler.java
```

### Structure Rules

* Only one `AuthController` will be used initially.
* Each small endpoint will not receive a separate controller.
* Services will be separated only by clear responsibility.
* Authentication will reuse the existing User module.
* The Authentication module must not create a duplicate User entity.
* Email-provider implementation may remain in a shared notification or infrastructure package.

---

## 7. Database Design Approach

## 7.1 Users Table

Authentication will reuse the existing `users` table.

Required authentication fields:

```text
id
email
password
user_type
status
email_verified
token_version
last_login_at
created_at
updated_at
```

Required indexes:

* Unique index on `email`
* Index on `status`
* Index on `user_type`

Emails must be stored in lowercase.

Additional fields required for Google OAuth 2.0 and phone OTP verification:

```text
phone_verified
auth_provider
provider_id
provider_email
```

`auth_provider` will initially support:

```text
LOCAL
GOOGLE
```

`provider_id` must store the unique Google user ID received from Google OAuth.

`phone_verified` will be updated only after successful phone OTP verification.

---

## 7.2 Redis Phone OTP Data

Phone OTP data will be stored in Redis because OTP is temporary security data.

Redis key format:

```text
otp:phone:{phoneNumber}:{purpose}
```

Example:

```text
otp:phone:9876543210:PHONE_VERIFICATION
```

Redis value must contain:

```text
otp_hash
phone_number
purpose
attempt_count
expires_at
```

Redis TTL must be 5 minutes.

OTP must be deleted from Redis after successful verification.

---

## 7.3 Refresh Tokens Table

```text
refresh_tokens
├── id
├── user_id
├── token_hash
├── token_family_id
├── parent_token_id
├── replaced_by_token_id
├── device_info
├── ip_address
├── expires_at
├── revoked_at
├── revocation_reason
├── created_at
└── updated_at
```

Required indexes:

* Unique index on `token_hash`
* Index on `user_id`
* Index on `token_family_id`
* Index on `expires_at`
* Composite index on `user_id`, `revoked_at`, `expires_at`

---

## 7.4 Password Reset Tokens Table

```text
password_reset_tokens
├── id
├── user_id
├── token_hash
├── expires_at
├── used_at
├── invalidated_at
└── created_at
```

Required indexes:

* Unique index on `token_hash`
* Index on `user_id`
* Index on `expires_at`
* Composite index on `user_id`, `used_at`, `expires_at`

---

## 7.5 Verification Tokens Table

```text
verification_tokens
├── id
├── user_id
├── token_hash
├── verification_type
├── expires_at
├── verified_at
├── invalidated_at
└── created_at
```

Required indexes:

* Unique index on `token_hash`
* Index on `user_id`
* Index on `expires_at`
* Composite index on `user_id`, `verified_at`, `expires_at`

---

## 7.6 Login Attempts Table

```text
login_attempts
├── id
├── user_id
├── email
├── ip_address
├── user_agent
├── was_successful
├── failure_reason
└── attempted_at
```

Required indexes:

* Index on `email`
* Index on `ip_address`
* Index on `attempted_at`
* Composite index on `email`, `attempted_at`

Login-attempt records will be retained for 90 days by default.

The retention period must be configurable.

---

## 7.7 Token Hashing

Refresh, reset and verification tokens will use:

```text
HMAC-SHA-256(rawToken, TOKEN_HASH_SECRET)
```

The token-hash secret:

* Must remain outside the database.
* Must be supplied through environment configuration.
* Must be different from the JWT signing secret.
* Must never be committed to Git.

Passwords will continue to use BCrypt.

---

## 7.8 Transaction Management

The following operations must be transactional:

* Successful login
* Refresh-token rotation
* Token-family revocation
* Password reset
* Password change
* Email verification
* Logout-all
* Google OAuth callback success
* Google OAuth existing-user login
* Google OAuth new-user creation
* Phone OTP generation
* Phone OTP verification success
* Phone OTP invalid attempt count
* Phone OTP expiry
* Phone OTP max-attempt failure
* Verification-token replacement

Failed login-attempt tracking must use a separate transaction so that the record is not rolled back with the authentication failure.

---

## 7.9 Cleanup Jobs

A scheduled cleanup process will remove or archive:

* Expired refresh tokens
* Expired reset tokens
* Used reset tokens
* Expired verification tokens
* Old login attempts

Initial schedule:

```text
Once per day
```

Retention values must remain configurable.

---

## 8. API Design Approach

## 8.1 Endpoint Summary

| Method | Endpoint                           | Access                | Purpose                   |
| ------ | ---------------------------------- | --------------------- | ------------------------- |
| POST   | `/api/v1/auth/login`               | Public                | Authenticate user         |
| POST   | `/api/v1/auth/refresh-token`       | Refresh cookie + CSRF | Rotate tokens             |
| POST   | `/api/v1/auth/logout`              | Refresh cookie + CSRF | Logout current device     |
| POST   | `/api/v1/auth/logout-all`          | Bearer token          | Logout all devices        |
| POST   | `/api/v1/auth/forgot-password`     | Public                | Request reset email       |
| POST   | `/api/v1/auth/reset-password`      | Reset token           | Reset password            |
| POST   | `/api/v1/auth/verify-email`        | Verification token    | Verify email              |
| POST   | `/api/v1/auth/resend-verification` | Public                | Resend verification email |
| PUT    | `/api/v1/auth/change-password`     | Bearer token          | Change password           |
| GET    | `/api/v1/auth/me`                  | Bearer token          | Get current user          |
| GET    | `/api/v1/auth/oauth2/google`       | Public                | Start Google OAuth login  |
| GET    | `/api/v1/auth/oauth2/callback/google` | Public             | Handle Google OAuth callback |
| POST   | `/api/v1/auth/phone/send-otp`      | Public                | Generate local phone OTP  |
| POST   | `/api/v1/auth/phone/verify-otp`    | Public                | Verify local phone OTP    |

---

## 8.2 Login Response

The refresh token will not be returned in JSON.

```json
{
  "message": "Login successful",
  "data": {
    "accessToken": "jwt-access-token",
    "accessTokenExpiresIn": 900,
    "user": {
      "id": 1,
      "firstName": "Mahendra",
      "lastName": "Singh",
      "email": "customer@example.com",
      "userType": "CUSTOMER",
      "roles": ["CUSTOMER"],
      "permissions": []
    }
  }
}
```

The refresh token will be set through an HttpOnly cookie.

---

## 8.3 Google OAuth 2.0 API Flow

### Start Google OAuth Login

```http
GET /api/v1/auth/oauth2/google
```

Purpose:

```text
Redirect the user to the Google OAuth consent screen.
```

### Google OAuth Callback

```http
GET /api/v1/auth/oauth2/callback/google
```

Purpose:

```text
Receive Google authorization code, fetch Google user information, create or find the BizCart user, and issue BizCart tokens.
```

After successful Google login, the backend must generate the same BizCart access token and refresh-token cookie used by normal login.

---

## 8.4 Phone OTP API Flow

### Send Phone OTP

```http
POST /api/v1/auth/phone/send-otp
```

Request:

```json
{
  "phoneNumber": "9876543210",
  "purpose": "PHONE_VERIFICATION"
}
```

Local behaviour:

```text
The backend generates a random 6 digit OTP, stores the OTP hash in Redis and prints the OTP in the backend console.
```

### Verify Phone OTP

```http
POST /api/v1/auth/phone/verify-otp
```

Request:

```json
{
  "phoneNumber": "9876543210",
  "otp": "482915",
  "purpose": "PHONE_VERIFICATION"
}
```

Verification rules:

* OTP must match the Redis OTP hash.
* OTP must not be expired.
* OTP must not exceed 3 failed attempts.
* OTP must be deleted after successful verification.
* User phone verification status must be updated after successful verification.

---

## 8.5 Error Response

```json
{
  "timestamp": "2026-07-09T10:30:00Z",
  "status": 401,
  "error": "Unauthorized",
  "code": "AUTH_INVALID_CREDENTIALS",
  "message": "Invalid email or password",
  "path": "/api/v1/auth/login",
  "correlationId": "4e08a42a"
}
```

### Important Error Codes

| HTTP Status | Code                                |
| ----------: | ----------------------------------- |
|         400 | `AUTH_VALIDATION_FAILED`            |
|         400 | `AUTH_PASSWORD_MISMATCH`            |
|         400 | `AUTH_NEW_PASSWORD_SAME`            |
|         401 | `AUTH_INVALID_CREDENTIALS`          |
|         401 | `AUTH_INVALID_TOKEN`                |
|         401 | `AUTH_REFRESH_TOKEN_EXPIRED`        |
|         401 | `AUTH_REFRESH_TOKEN_REVOKED`        |
|         401 | `AUTH_REFRESH_TOKEN_REUSE_DETECTED` |
|         401 | `AUTH_RESET_TOKEN_INVALID`          |
|         410 | `AUTH_RESET_TOKEN_EXPIRED`          |
|         410 | `AUTH_RESET_TOKEN_USED`             |
|         401 | `AUTH_VERIFICATION_TOKEN_INVALID`   |
|         410 | `AUTH_VERIFICATION_TOKEN_EXPIRED`   |
|         403 | `AUTH_ACCOUNT_INACTIVE`             |
|         403 | `AUTH_ACCOUNT_BLOCKED`              |
|         403 | `AUTH_EMAIL_NOT_VERIFIED`           |
|         403 | `AUTH_ACCOUNT_APPROVAL_PENDING`     |
|         409 | `AUTH_EMAIL_ALREADY_VERIFIED`       |
|         429 | `AUTH_TOO_MANY_ATTEMPTS`            |
|         400 | `AUTH_INVALID_OTP`                  |
|         410 | `AUTH_OTP_EXPIRED`                  |
|         429 | `AUTH_OTP_MAX_ATTEMPTS_EXCEEDED`    |
|         429 | `AUTH_OTP_RESEND_LIMIT_EXCEEDED`    |
|         401 | `AUTH_OAUTH_LOGIN_FAILED`           |
|         409 | `AUTH_PHONE_ALREADY_VERIFIED`       |
|         500 | `AUTH_INTERNAL_ERROR`               |

---

## 9. Security Approach

### 9.1 Password Security

* BCrypt must be used.
* Passwords must never be logged.
* Passwords must never be returned.
* BCrypt strength must be configurable.
* New passwords must follow the requirement document rules.
* Password reset and password change must revoke active refresh tokens.

### 9.2 Token Security

* Access tokens expire after 15 minutes.
* Refresh tokens expire after 7 days.
* Refresh tokens are stored only as hashes.
* Refresh-token rotation is mandatory.
* Refresh-token reuse revokes the token family.
* Reset and verification tokens are single-use.
* Token values and hashes must never appear in logs.

### 9.3 Rate Limiting

Initial configurable limits:

| Endpoint            |                                      Limit |
| ------------------- | -----------------------------------------: |
| Login               | 5 attempts per 15 minutes per email and IP |
| Forgot password     |       3 requests per hour per email and IP |
| Resend verification |              3 requests per hour per email |
| Refresh token       |         20 requests per minute per session |
| Reset password      | 5 attempts per 15 minutes per token and IP |
| Verify email        |                10 attempts per hour per IP |

The initial implementation may use application-level rate limiting.

Redis-based distributed rate limiting will be introduced when multiple backend instances are deployed.

### 9.4 Existing Access Tokens After Account Blocking

Because access tokens are stateless, a blocked user may retain an existing token until it expires.

Mitigation:

* Access token expiry remains 15 minutes.
* Highly sensitive APIs may recheck account status.
* Password change, block and major security actions increment `token_version`.
* JWT validation may compare the token version where required.

### 9.5 Logging

Allowed:

* User ID
* Normalised email where permitted
* IP address
* User agent
* Authentication result
* Failure category
* Correlation ID

Not allowed:

* Password
* Access token
* Refresh token
* Reset token
* Verification token
* Token hash
* JWT secret
* Token-hash secret

### 9.6 Client IP Handling

Client IP must be derived only from trusted proxy headers.

Supported headers:

* `Forwarded`
* `X-Forwarded-For`

These headers must not be trusted when requests bypass the configured proxy or load balancer.

### 9.7 Security Headers

Deployed environments should configure:

* `Strict-Transport-Security`
* `X-Content-Type-Options`
* Cache-control headers for authentication responses
* Frontend-level Content Security Policy
* Restricted CORS origins

### 9.8 Google OAuth Security

* Google Client ID and Client Secret must be loaded from environment variables.
* Google Client Secret must never be exposed to the frontend.
* Google access token must not be returned in the BizCart API response.
* Only verified Google email addresses will be accepted.
* BizCart will generate its own JWT after successful OAuth login.
* Duplicate user creation must be prevented using email uniqueness.
* Google OAuth must not bypass account status validation.

### 9.9 Phone OTP Security

* OTP must be a random 6 digit numeric value.
* OTP must be stored as hash in Redis.
* OTP must expire after 5 minutes.
* Maximum failed verification attempts must be 3.
* OTP must be deleted after successful verification.
* OTP must not be stored in MySQL for the local implementation.
* OTP must not be returned in the API response.
* In the local profile only, OTP may be printed in the backend console.
* Real SMS gateway integration will be added later for deployed environments.

---

## 10. Testing Approach

## 10.1 Unit Tests

Primary targets:

* `AuthService`
* `JwtService`
* `RefreshTokenService`
* `PasswordService`
* `VerificationService`
* `OAuthService`
* `PhoneOtpService`
* `SmsService`
* `AuthMapper`

Important scenarios:

* Successful login
* Invalid credentials
* Inactive account
* Blocked account
* Unverified customer
* Unapproved seller
* Access-token creation
* Access-token expiry
* Refresh rotation
* Concurrent refresh requests
* Refresh-token reuse detection
* Token-family revocation
* Password reset
* Used reset token
* Email verification
* Password change
* Logout
* Logout-all

---

## 10.2 Controller and Security Tests

Tests must verify:

* Public endpoint access
* Protected endpoint rejection
* Missing Bearer token
* Invalid Bearer token
* Expired access token
* Missing CSRF token
* Invalid CSRF token
* Missing refresh cookie
* CORS rejection
* Correct status codes
* Standard error response

---

## 10.3 Repository Tests

Tests will use MySQL Testcontainers.

Coverage:

* User lookup by lowercase email
* Token lookup by hash
* Active refresh-token queries
* Atomic token rotation
* Family revocation
* Reset-token single use
* Verification-token invalidation
* Composite index-supported queries
* Login-attempt persistence

---

## 10.4 Integration Tests

### Login

```text
Create active user
→ Login
→ Verify access token
→ Verify refresh cookie
→ Verify refresh-token database record
→ Verify login attempt
```

### Refresh

```text
Login
→ Refresh
→ Verify previous token revoked
→ Verify new cookie
→ Reuse old token
→ Verify token family revoked
```

### Password Reset

```text
Request reset
→ Reset password
→ Verify token used
→ Verify active sessions revoked
→ Login with new password
```

### Email Verification

```text
Create pending customer
→ Generate token
→ Verify email
→ Confirm account active
→ Confirm token cannot be reused
```

### Google OAuth 2.0

```text
Start OAuth login
→ Mock Google callback
→ Fetch Google user info
→ Create or find BizCart user
→ Generate BizCart access token
→ Verify refresh-token cookie
```

### Phone OTP Verification

```text
Send OTP
→ Verify Redis key and TTL
→ Read OTP from local console or mocked SMS service
→ Verify OTP
→ Confirm phone_verified=true
→ Confirm Redis OTP key deleted
```

---

## 10.5 Performance Targets

| Operation     |           Target |
| ------------- | ---------------: |
| Login         | p95 below 500 ms |
| Refresh token | p95 below 300 ms |
| Current user  | p95 below 300 ms |

Targets exclude external email delivery.

Performance tests must use an agreed test load and production-like BCrypt configuration.

---

## 11. Risks and Mitigation

| Risk                                    | Mitigation                                                    |
| --------------------------------------- | ------------------------------------------------------------- |
| JWT secret exposure                     | Environment-managed secret and controlled rotation            |
| Refresh-token theft                     | HttpOnly cookie, hashing, rotation and family reuse detection |
| Concurrent refresh                      | Atomic database update and transaction                        |
| User enumeration                        | Generic authentication and recovery responses                 |
| Access token remains valid after logout | Short 15-minute expiry                                        |
| Email delivery failure                  | Retry support and resend endpoint                             |
| Application stops after token creation  | Transactional outbox recommended for production               |
| Login-attempt growth                    | 90-day retention and scheduled cleanup                        |
| Stale permissions in sessions           | Permissions loaded through roles; short access-token lifetime |
| Proxy IP spoofing                       | Trust proxy headers only from approved proxies                |
| Brute-force attack                      | Rate limiting and login-attempt tracking                      |
| Cookie-based CSRF                       | SameSite cookie and CSRF header                               |
| Role changes after login                | New access token on refresh and short token expiry            |
| Database outage                         | Health checks, connection pooling and database monitoring     |
| Misconfigured environment URL           | Separate frontend URLs per environment                        |
| Google OAuth credential exposure         | Store OAuth client secret only in environment configuration    |
| OAuth duplicate account creation         | Match users by unique email and provider ID                    |
| OTP brute-force attempts                 | Redis TTL, max attempts and resend cooldown                    |
| OTP leakage in deployed logs             | Console OTP printing allowed only in local profile             |

---

## 12. Assumptions and Dependencies

### Assumptions

* The User module exists.
* The Role and Permission module exists or will be implemented before complete authorisation.
* Email is the primary login identifier.
* User emails are unique.
* Store Staff is represented through role and store membership.
* MySQL is the primary database.
* The application is deployed over HTTPS.
* Frontend supports credentials and CSRF headers.
* Access tokens are stored in memory.
* Refresh tokens are stored only in HttpOnly cookies.
* Server clocks are synchronised.
* Redis is available for local phone OTP verification.
* Local profile uses console-based SMS simulation.
* Google OAuth credentials are available through environment variables.

### Internal Dependencies

* User module
* Role and Permission module
* Store membership module
* Global exception handler
* Common API-response model
* Spring Security configuration
* Flyway configuration
* Email or notification abstraction
* Google OAuth provider configuration
* Redis configuration for local OTP storage
* Audit and logging configuration

### External Dependencies

* MySQL
* Email provider
* Frontend reset-password page
* Frontend email-verification page
* Environment secret management
* Redis in future scaled deployments

---

## 13. Required Configuration

```text
JWT_SECRET
JWT_ACCESS_TOKEN_EXPIRY
JWT_REFRESH_TOKEN_EXPIRY

TOKEN_HASH_SECRET
PASSWORD_RESET_TOKEN_EXPIRY
EMAIL_VERIFICATION_TOKEN_EXPIRY

BCRYPT_STRENGTH

FRONTEND_BASE_URL
PASSWORD_RESET_URL
EMAIL_VERIFICATION_URL

MAIL_HOST
MAIL_PORT
MAIL_USERNAME
MAIL_PASSWORD
MAIL_FROM

ALLOWED_CORS_ORIGINS
REFRESH_COOKIE_NAME
REFRESH_COOKIE_SECURE
REFRESH_COOKIE_SAME_SITE

LOGIN_RATE_LIMIT
FORGOT_PASSWORD_RATE_LIMIT
REFRESH_RATE_LIMIT
VERIFICATION_RATE_LIMIT

LOGIN_ATTEMPT_RETENTION_DAYS

GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
GOOGLE_OAUTH_REDIRECT_URI

REDIS_HOST
REDIS_PORT
OTP_LENGTH
OTP_EXPIRY_MINUTES
OTP_MAX_ATTEMPTS
OTP_SMS_MODE
```

---

## 14. Out of Scope for Version One

* Facebook, Apple and other social-login providers except Google OAuth 2.0
* Facebook login
* Apple login
* Real SMS gateway integration for Phone OTP in deployed environments
* Two-factor authentication
* Biometric authentication
* CAPTCHA
* Single Sign-On
* Passwordless login
* Active-session list UI
* Manual selective device revocation UI
* Access-token blacklist
* Redis-based distributed rate limiting beyond local OTP storage
* Security-notification dashboard
* Password-compromise database integration

---

## 15. Implementation Sequence

1. Verify User and Role data models.
2. Create Flyway migrations.
3. Add Authentication properties.
4. Configure BCrypt.
5. Configure Spring Security and CORS.
6. Add CSRF handling.
7. Implement JWT service.
8. Implement refresh-token storage.
9. Implement login and current-user APIs.
10. Implement refresh-token rotation.
11. Implement token-family reuse detection.
12. Implement logout and logout-all.
13. Implement password recovery.
14. Implement email verification.
15. Implement password change.
16. Add login-attempt tracking.
17. Add rate limiting.
18. Implement Google OAuth 2.0 login.
19. Implement Redis-backed local phone OTP verification.
20. Add token cleanup job.
21. Add Swagger documentation.
22. Add unit and integration tests.
23. Perform security and QA validation.

---

## 16. Final Development Readiness

The Authentication module will be considered ready for release when:

* Login works for valid active users.
* Account-status rules are enforced.
* Access tokens are correctly validated.
* Refresh tokens are stored only as hashes.
* Refresh tokens are transported through HttpOnly cookies.
* Refresh-token rotation works.
* Reused refresh tokens revoke their token family.
* Logout and logout-all revoke sessions.
* Password-reset tokens are single-use.
* Verification tokens expire after 24 hours.
* Password change revokes active sessions.
* Rate limiting is active.
* Sensitive values are absent from logs.
* Swagger documentation is complete.
* Unit and integration tests pass.
* Flyway migrations work on a clean MySQL database.
* Google OAuth 2.0 login works for existing and new users.
* Local phone OTP verification works with Redis, 5-minute TTL and 3 max attempts.
* QA checklist is completed.