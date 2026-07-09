# BizCart — Authentication Module Requirements

## 1. Document Overview

**Project:** BizCart — Small Business Commerce Platform  
**Module:** Authentication  
**Backend:** Spring Boot  
**Database:** MySQL  
**Authentication Method:** JWT Access Token + Refresh Token  
**Password Encoding:** BCrypt

This document defines the complete requirements for implementing the Authentication module in the BizCart backend.

---

## 2. Purpose

The Authentication module securely identifies users and allows them to access BizCart according to their account status, role and permissions.

The module provides:

- User login and logout
- JWT access-token generation
- Refresh-token generation and rotation
- Forgot-password and reset-password flows
- Email verification
- Password change
- Current authenticated-user retrieval
- Login-attempt tracking

---

## 3. Scope

### 3.1 In Scope

- Login using email and password
- JWT access-token generation
- Refresh-token generation and storage
- Refreshing an expired access token
- Logout from the current device
- Logout from all devices
- Forgot-password request
- Password reset using a secure token
- Email verification
- Resending a verification email
- Changing password for authenticated users
- Retrieving current authenticated-user details
- Validating account status before login or token refresh
- Including role and permission details in the authenticated session
- Tracking successful and failed login attempts

### 3.2 Out of Scope

The first version will not include:

- Google login
- Facebook login
- Apple login
- Phone OTP login
- Two-factor authentication
- Biometric authentication
- CAPTCHA integration
- Single Sign-On

---

## 4. Stakeholders and Target Users

### 4.1 Stakeholders

- Business owner
- Product owner
- Backend development team
- Frontend development team
- QA team
- Platform administrators
- Store owners
- Store staff
- Customers

### 4.2 Target Users

| User | Description |
|---|---|
| Admin | Manages the BizCart platform |
| Seller | Owns and manages a store |
| Store Staff | Performs store operations allowed by assigned permissions |
| Customer | Browses products, manages a cart and places orders |

---

## 5. Functional Requirements

### FR-AUTH-001 — User Login

The system must allow a registered user to log in using an email and password.

#### Login Flow

1. User submits email and password.
2. System validates the request.
3. System finds the user by email.
4. System checks the account status.
5. System verifies the password using BCrypt.
6. System generates an access token.
7. System generates a refresh token.
8. System stores the hashed refresh token.
9. System updates `last_login_at`.
10. System records the successful login attempt.
11. System returns tokens and safe user details.

#### Login Response Must Include

- Access token
- Refresh token
- Access-token expiry
- Refresh-token expiry
- User ID
- First name
- Last name
- Email
- User type
- Roles
- Permissions

---

### FR-AUTH-002 — Access Token

The system must generate a JWT access token after successful login.

The access token must contain:

- User ID
- Email
- User type
- Roles
- Issued time
- Expiry time

The access token must be required for protected APIs.

**Recommended expiry:** 15 minutes

---

### FR-AUTH-003 — Refresh Token

The system must generate a refresh token after successful login.

The refresh token must:

- Have a longer expiry than the access token
- Be stored as a hash, not as plain text
- Be linked to a user
- Support multiple devices
- Support revocation
- Expire automatically

**Recommended expiry:** 7 days

---

### FR-AUTH-004 — Refresh Access Token

The system must allow a user to obtain new tokens using a valid refresh token.

#### Refresh Flow

1. User submits the refresh token.
2. System hashes and searches for the token.
3. System checks whether the token exists.
4. System checks whether it is expired.
5. System checks whether it is revoked.
6. System checks whether the user is active.
7. System revokes the old refresh token.
8. System generates a new access token.
9. System generates and stores a new refresh token.
10. System returns the new tokens.

Refresh-token rotation must be implemented.

---

### FR-AUTH-005 — Logout

The system must allow an authenticated user to log out from the current device.

During logout:

- The submitted refresh token must be revoked.
- The revoked token must not be reusable.
- The access token will remain valid until its expiry unless token blacklisting is added later.

---

### FR-AUTH-006 — Logout From All Devices

The system must allow an authenticated user to log out from all devices.

During this action, all active refresh tokens belonging to the user must be revoked.

---

### FR-AUTH-007 — Forgot Password

The system must allow a user to request a password-reset link using an email address.

#### Forgot-Password Flow

1. User submits an email.
2. System validates the email.
3. System checks whether the user exists.
4. System creates a secure reset token.
5. System stores the hashed token.
6. System sends a reset link to the registered email.
7. The token expires after the configured time.

**Recommended expiry:** 15 minutes

For security, the response must be the same whether the email exists or not.

---

### FR-AUTH-008 — Reset Password

The system must allow a user to reset the password using a valid reset token.

#### Reset Flow

1. User submits the reset token.
2. User submits a new password and confirmation password.
3. System validates the token.
4. System checks token expiry.
5. System checks whether the token has already been used.
6. System validates the new password.
7. System updates the BCrypt-encoded password.
8. System marks the token as used.
9. System revokes all existing refresh tokens belonging to the user.

---

### FR-AUTH-009 — Email Verification

The system must support email verification.

#### Verification Flow

1. System generates a verification token.
2. System stores the hashed token.
3. System sends a verification link.
4. User submits or opens the token.
5. System validates the token.
6. System marks `email_verified` as true.
7. System marks the verification token as used.

---

### FR-AUTH-010 — Resend Verification Email

The system must allow a user to request another verification email.

The system must:

- Generate a new token
- Invalidate the previous active token
- Apply a resend rate limit
- Send the new verification email

---

### FR-AUTH-011 — Change Password

An authenticated user must be able to change the password.

The request must contain:

- Current password
- New password
- Confirm new password

The system must:

1. Verify the current password.
2. Validate the new password.
3. Ensure the new password differs from the current password.
4. Update the password.
5. Revoke all active refresh tokens.

---

### FR-AUTH-012 — Get Current User

The system must allow an authenticated user to retrieve current account details.

The response should include:

- User ID
- First name
- Last name
- Email
- Phone
- Profile image
- User type
- Account status
- Email-verification status
- Roles
- Permissions

The password and token hashes must never be returned.

---

### FR-AUTH-013 — Account Status Validation

Authentication must be rejected when the account status is:

- `INACTIVE`
- `BLOCKED`

A `PENDING` user must not receive access to protected business APIs until email verification or admin approval is complete.

---

### FR-AUTH-014 — Login Attempt Tracking

The system must record successful and failed login attempts.

The record should contain:

- User ID when available
- Submitted email
- IP address
- User agent
- Success or failure result
- Failure reason
- Attempt timestamp

---

## 6. Non-Functional Requirements

### 6.1 Security

#### Password Security

- Passwords must never be stored as plain text.
- Passwords must be encoded using BCrypt.
- Passwords must never be returned by an API.
- Passwords must not be written to logs.

#### Token Security

- JWT secrets or private keys must come from environment configuration.
- Refresh tokens must be stored as hashes.
- Reset and verification tokens must be stored as hashes.
- Every token must have an expiry time.
- Refresh-token revocation and rotation must be supported.
- Raw token values must not be logged.

#### API Security

- Protected APIs must require a valid Bearer token.
- Deployed environments must use HTTPS.
- CORS must allow only approved frontend origins.
- Rate limiting should be applied to login, forgot password, reset password, verification resend and token refresh.
- Authentication failures must not expose internal implementation details.
- JPA parameterized queries must be used.
- Forgot-password responses must prevent user enumeration.

#### Session Security

- Multiple active devices may be supported.
- Logout must revoke the current refresh token.
- Password reset and password change must revoke all refresh tokens.
- Blocked or inactive users must not be able to refresh tokens.

### 6.2 Performance

- Login should normally respond within 500 ms, excluding email delivery.
- Refresh-token API should normally respond within 300 ms.
- Current-user API should normally respond within 300 ms.
- Email and token lookup columns must be indexed.
- Role and permission data may be cached.

### 6.3 Scalability

- Access-token validation must remain stateless.
- Refresh tokens must work across multiple backend instances.
- Token validation must not depend on server memory.
- Email sending should be asynchronous where possible.
- The design should support future Redis rate limiting.
- The design should support future social-login integration.

---

## 7. User Roles and Permissions

### 7.1 User Types

| User Type | Description |
|---|---|
| ADMIN | Manages the platform |
| SELLER | Owns and manages a store |
| CUSTOMER | Purchases products |

### 7.2 Authentication Access Matrix

| Action | Admin | Seller | Customer | Public |
|---|---:|---:|---:|---:|
| Login | Yes | Yes | Yes | Yes |
| Refresh token | Yes | Yes | Yes | Token required |
| Logout | Yes | Yes | Yes | No |
| Logout all devices | Yes | Yes | Yes | No |
| Forgot password | Yes | Yes | Yes | Yes |
| Reset password | Yes | Yes | Yes | Valid token required |
| Verify email | Yes | Yes | Yes | Valid token required |
| Resend verification | Yes | Yes | Yes | Yes |
| Change password | Yes | Yes | Yes | No |
| Get current user | Yes | Yes | Yes | No |

### 7.3 Permission Rules

- Public authentication APIs must not require a JWT.
- Protected APIs must require a valid JWT.
- Business authorization must be handled through roles and permissions.
- Blocked or inactive users must not receive new tokens.

---

## 8. Validation Rules

### 8.1 Email

- Required
- Trimmed before processing
- Converted to lowercase before lookup
- Must have a valid email format
- Maximum length: 255 characters

### 8.2 Password

A password must:

- Be required
- Have at least 8 characters
- Have no more than 64 characters
- Include at least one uppercase letter
- Include at least one lowercase letter
- Include at least one number
- Include at least one special character
- Not contain leading or trailing spaces

### 8.3 Confirm Password

- Required
- Must exactly match the new password

### 8.4 Refresh Token

- Required
- Must not be blank
- Must exist in the database
- Must not be expired
- Must not be revoked

### 8.5 Reset Token

- Required
- Must exist
- Must not be expired
- Must not already be used

### 8.6 Verification Token

- Required
- Must exist
- Must not be expired
- Must not already be used

### 8.7 Change Password

- Current password is required.
- New password is required.
- Confirmation password is required.
- Current password must be correct.
- New password must differ from the current password.
- New password and confirmation password must match.

---

## 9. Error Handling and Exception Cases

All endpoints must use a consistent error-response structure.

### 9.1 Standard Error Response

```json
{
  "timestamp": "2026-07-09T10:30:00Z",
  "status": 401,
  "error": "Unauthorized",
  "code": "AUTH_INVALID_CREDENTIALS",
  "message": "Invalid email or password",
  "path": "/api/v1/auth/login"
}
```

### 9.2 Error Cases

| HTTP Status | Error Code | Case |
|---|---|---|
| 400 | AUTH_VALIDATION_FAILED | Invalid request fields |
| 400 | AUTH_PASSWORD_MISMATCH | Password and confirmation do not match |
| 400 | AUTH_NEW_PASSWORD_SAME | New password equals current password |
| 401 | AUTH_INVALID_CREDENTIALS | Invalid email or password |
| 401 | AUTH_INVALID_TOKEN | Invalid access or refresh token |
| 401 | AUTH_REFRESH_TOKEN_EXPIRED | Refresh token expired |
| 401 | AUTH_REFRESH_TOKEN_REVOKED | Refresh token revoked |
| 401 | AUTH_RESET_TOKEN_INVALID | Invalid reset token |
| 401 | AUTH_RESET_TOKEN_EXPIRED | Reset token expired |
| 401 | AUTH_VERIFICATION_TOKEN_INVALID | Invalid verification token |
| 403 | AUTH_ACCOUNT_INACTIVE | Account is inactive |
| 403 | AUTH_ACCOUNT_BLOCKED | Account is blocked |
| 403 | AUTH_EMAIL_NOT_VERIFIED | Email verification is required |
| 404 | AUTH_USER_NOT_FOUND | User not found, only where safe to expose |
| 409 | AUTH_EMAIL_ALREADY_VERIFIED | Email is already verified |
| 429 | AUTH_TOO_MANY_ATTEMPTS | Rate limit exceeded |
| 500 | AUTH_INTERNAL_ERROR | Unexpected internal error |

### 9.3 Security Rules for Errors

- Unknown email and wrong password must both return `Invalid email or password`.
- Forgot-password must not reveal whether an email exists.
- SQL errors, stack traces and internal messages must not be returned.
- Sensitive tokens must not appear in logs.

---

## 10. Data Requirements

### 10.1 Entity: User

Authentication uses the existing `users` table.

#### Table: `users`

| Field | Type | Required | Description |
|---|---|---:|---|
| id | BIGINT | Yes | User ID |
| first_name | VARCHAR(100) | Yes | First name |
| last_name | VARCHAR(100) | Yes | Last name |
| email | VARCHAR(255) | Yes | Unique login email |
| phone | VARCHAR(20) | No | Phone number |
| password | VARCHAR(255) | Yes | BCrypt-encoded password |
| profile_image | VARCHAR(500) | No | Profile image URL |
| user_type | ENUM | Yes | ADMIN, SELLER or CUSTOMER |
| status | ENUM | Yes | PENDING, ACTIVE, INACTIVE or BLOCKED |
| email_verified | BOOLEAN | Yes | Email-verification state |
| last_login_at | DATETIME | No | Last successful login |
| created_at | DATETIME | Yes | Creation time |
| updated_at | DATETIME | Yes | Update time |

**Required indexes**

- Unique index on `email`
- Unique index on `phone` when phone is used
- Index on `status`
- Index on `user_type`

---

### 10.2 Entity: RefreshToken

#### Table: `refresh_tokens`

| Field | Type | Required | Description |
|---|---|---:|---|
| id | BIGINT | Yes | Refresh-token ID |
| user_id | BIGINT | Yes | Related user |
| token_hash | VARCHAR(255) | Yes | Hashed refresh token |
| device_info | VARCHAR(500) | No | Browser or device details |
| ip_address | VARCHAR(45) | No | Login IP address |
| expires_at | DATETIME | Yes | Expiry time |
| revoked_at | DATETIME | No | Revocation time |
| created_at | DATETIME | Yes | Creation time |
| updated_at | DATETIME | Yes | Update time |

**Relationship:** Many refresh tokens belong to one user.

**Required indexes**

- Unique index on `token_hash`
- Index on `user_id`
- Index on `expires_at`

---

### 10.3 Entity: PasswordResetToken

#### Table: `password_reset_tokens`

| Field | Type | Required | Description |
|---|---|---:|---|
| id | BIGINT | Yes | Reset-token ID |
| user_id | BIGINT | Yes | Related user |
| token_hash | VARCHAR(255) | Yes | Hashed reset token |
| expires_at | DATETIME | Yes | Expiry time |
| used_at | DATETIME | No | Token-use time |
| created_at | DATETIME | Yes | Creation time |

**Required indexes**

- Unique index on `token_hash`
- Index on `user_id`
- Index on `expires_at`

---

### 10.4 Entity: VerificationToken

#### Table: `verification_tokens`

| Field | Type | Required | Description |
|---|---|---:|---|
| id | BIGINT | Yes | Verification-token ID |
| user_id | BIGINT | Yes | Related user |
| token_hash | VARCHAR(255) | Yes | Hashed verification token |
| verification_type | ENUM('EMAIL') | Yes | Verification type |
| expires_at | DATETIME | Yes | Expiry time |
| verified_at | DATETIME | No | Verification-completion time |
| created_at | DATETIME | Yes | Creation time |

**Required indexes**

- Unique index on `token_hash`
- Index on `user_id`
- Index on `expires_at`

---

### 10.5 Entity: LoginAttempt

#### Table: `login_attempts`

| Field | Type | Required | Description |
|---|---|---:|---|
| id | BIGINT | Yes | Login-attempt ID |
| user_id | BIGINT | No | User ID when identified |
| email | VARCHAR(255) | No | Submitted email |
| ip_address | VARCHAR(45) | No | Request IP |
| user_agent | VARCHAR(500) | No | Browser/client details |
| was_successful | BOOLEAN | Yes | Attempt result |
| failure_reason | VARCHAR(255) | No | Failure reason |
| attempted_at | DATETIME | Yes | Attempt time |

**Required indexes**

- Index on `email`
- Index on `ip_address`
- Index on `attempted_at`

---

## 11. API and Interface Requirements

### 11.1 Base URL

```text
/api/v1/auth
```

### 11.2 API Summary

| Method | Endpoint | Authentication | Purpose |
|---|---|---|---|
| POST | `/api/v1/auth/login` | Public | Log in a user |
| POST | `/api/v1/auth/refresh-token` | Refresh token | Generate new tokens |
| POST | `/api/v1/auth/logout` | Bearer token | Logout current device |
| POST | `/api/v1/auth/logout-all` | Bearer token | Logout all devices |
| POST | `/api/v1/auth/forgot-password` | Public | Request reset link |
| POST | `/api/v1/auth/reset-password` | Reset token | Reset password |
| POST | `/api/v1/auth/verify-email` | Verification token | Verify email |
| POST | `/api/v1/auth/resend-verification` | Public | Resend verification |
| PUT | `/api/v1/auth/change-password` | Bearer token | Change password |
| GET | `/api/v1/auth/me` | Bearer token | Get current user |

---

### 11.3 Login API

```http
POST /api/v1/auth/login
```

#### Request

```json
{
  "email": "customer@example.com",
  "password": "Password@123"
}
```

#### Success Response

```json
{
  "message": "Login successful",
  "data": {
    "accessToken": "jwt-access-token",
    "refreshToken": "refresh-token",
    "accessTokenExpiresIn": 900,
    "refreshTokenExpiresIn": 604800,
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

---

### 11.4 Refresh Token API

```http
POST /api/v1/auth/refresh-token
```

#### Request

```json
{
  "refreshToken": "refresh-token"
}
```

#### Success Response

```json
{
  "message": "Token refreshed successfully",
  "data": {
    "accessToken": "new-access-token",
    "refreshToken": "new-refresh-token",
    "accessTokenExpiresIn": 900,
    "refreshTokenExpiresIn": 604800
  }
}
```

---

### 11.5 Logout API

```http
POST /api/v1/auth/logout
```

#### Request

```json
{
  "refreshToken": "refresh-token"
}
```

#### Success Response

```json
{
  "message": "Logout successful"
}
```

---

### 11.6 Logout All Devices API

```http
POST /api/v1/auth/logout-all
```

#### Success Response

```json
{
  "message": "Logged out from all devices successfully"
}
```

---

### 11.7 Forgot Password API

```http
POST /api/v1/auth/forgot-password
```

#### Request

```json
{
  "email": "customer@example.com"
}
```

#### Success Response

```json
{
  "message": "If the email is registered, a password reset link has been sent"
}
```

---

### 11.8 Reset Password API

```http
POST /api/v1/auth/reset-password
```

#### Request

```json
{
  "token": "password-reset-token",
  "newPassword": "NewPassword@123",
  "confirmPassword": "NewPassword@123"
}
```

#### Success Response

```json
{
  "message": "Password reset successfully"
}
```

---

### 11.9 Verify Email API

```http
POST /api/v1/auth/verify-email
```

#### Request

```json
{
  "token": "email-verification-token"
}
```

#### Success Response

```json
{
  "message": "Email verified successfully"
}
```

---

### 11.10 Resend Verification API

```http
POST /api/v1/auth/resend-verification
```

#### Request

```json
{
  "email": "customer@example.com"
}
```

#### Success Response

```json
{
  "message": "If verification is required, a new verification email has been sent"
}
```

---

### 11.11 Change Password API

```http
PUT /api/v1/auth/change-password
```

#### Request

```json
{
  "currentPassword": "Password@123",
  "newPassword": "NewPassword@123",
  "confirmPassword": "NewPassword@123"
}
```

#### Success Response

```json
{
  "message": "Password changed successfully. Please log in again"
}
```

---

### 11.12 Current User API

```http
GET /api/v1/auth/me
```

#### Success Response

```json
{
  "message": "Current user retrieved successfully",
  "data": {
    "id": 1,
    "firstName": "Mahendra",
    "lastName": "Singh",
    "email": "customer@example.com",
    "phone": "9876543210",
    "profileImage": null,
    "userType": "CUSTOMER",
    "status": "ACTIVE",
    "emailVerified": true,
    "roles": ["CUSTOMER"],
    "permissions": []
  }
}
```

---

## 12. Suggested DTOs

### Request DTOs

- `LoginRequestDto`
- `RefreshTokenRequestDto`
- `LogoutRequestDto`
- `ForgotPasswordRequestDto`
- `ResetPasswordRequestDto`
- `VerifyEmailRequestDto`
- `ResendVerificationRequestDto`
- `ChangePasswordRequestDto`

### Response DTOs

- `LoginResponseDto`
- `TokenResponseDto`
- `AuthenticatedUserResponseDto`
- `MessageResponseDto`

---

## 13. Suggested Spring Boot Structure

```text
auth
├── controller
│   └── AuthController.java
├── dto
│   ├── request
│   │   ├── LoginRequestDto.java
│   │   ├── RefreshTokenRequestDto.java
│   │   ├── LogoutRequestDto.java
│   │   ├── ForgotPasswordRequestDto.java
│   │   ├── ResetPasswordRequestDto.java
│   │   ├── VerifyEmailRequestDto.java
│   │   ├── ResendVerificationRequestDto.java
│   │   └── ChangePasswordRequestDto.java
│   └── response
│       ├── LoginResponseDto.java
│       ├── TokenResponseDto.java
│       └── AuthenticatedUserResponseDto.java
├── entity
│   ├── RefreshToken.java
│   ├── PasswordResetToken.java
│   ├── VerificationToken.java
│   └── LoginAttempt.java
├── repository
│   ├── RefreshTokenRepository.java
│   ├── PasswordResetTokenRepository.java
│   ├── VerificationTokenRepository.java
│   └── LoginAttemptRepository.java
├── service
│   ├── AuthService.java
│   ├── TokenService.java
│   ├── PasswordService.java
│   └── EmailVerificationService.java
└── exception
    ├── InvalidCredentialsException.java
    ├── InvalidTokenException.java
    ├── TokenExpiredException.java
    ├── AccountBlockedException.java
    └── AccountInactiveException.java
```

Shared security classes may remain under the global `security` package:

```text
security
├── SecurityConfig.java
├── JwtTokenProvider.java
├── JwtAuthenticationFilter.java
└── CustomUserDetailsService.java
```

---

## 14. Assumptions

- The User module and `users` table exist before Authentication implementation is completed.
- Email is the primary login identifier.
- Every user has one primary user type.
- Roles and permissions are managed by the RBAC module.
- An email service will be available for verification and password reset.
- MySQL is the primary database.
- JWT configuration is supplied through environment variables.
- Deployed environments use HTTPS.
- Access-token validation is stateless.
- Refresh tokens are stored in the database.

---

## 15. Constraints

- Social login is not included in version one.
- Phone OTP authentication is not included.
- Two-factor authentication is not included.
- Password reset depends on the email service.
- Access tokens cannot be immediately invalidated unless a blacklist is introduced.
- Refresh-token storage creates database reads during refresh and logout.
- Secrets must never be committed to Git.
- The module must follow the common BizCart response and exception formats.
- Authentication implementation must not break User, Role or Store modules.

---

## 16. Acceptance Criteria

### Login

- [ ] An active registered user can log in with the correct email and password.
- [ ] Successful login returns an access token and refresh token.
- [ ] The access token contains user identity and role information.
- [ ] Incorrect credentials return `401 Unauthorized`.
- [ ] Unknown email and wrong password return the same error message.
- [ ] An inactive user cannot log in.
- [ ] A blocked user cannot log in.
- [ ] Passwords are verified using BCrypt.
- [ ] Successful and failed login attempts are recorded.

### Access and Refresh Tokens

- [ ] Protected APIs reject requests without an access token.
- [ ] Protected APIs reject invalid access tokens.
- [ ] Protected APIs reject expired access tokens.
- [ ] A valid refresh token generates new tokens.
- [ ] Refresh-token rotation generates a new refresh token.
- [ ] The previous refresh token is revoked after rotation.
- [ ] Expired refresh tokens are rejected.
- [ ] Revoked refresh tokens are rejected.
- [ ] Blocked or inactive users cannot refresh tokens.
- [ ] Refresh tokens are stored as hashes.

### Logout

- [ ] Current-device logout revokes the submitted refresh token.
- [ ] A revoked refresh token cannot be reused.
- [ ] Logout-all revokes every active refresh token belonging to the user.

### Forgot and Reset Password

- [ ] Forgot-password accepts a valid email format.
- [ ] Its response does not disclose whether the email exists.
- [ ] A reset token is generated with an expiry.
- [ ] Reset tokens are stored as hashes.
- [ ] A valid token allows password reset.
- [ ] An expired token is rejected.
- [ ] A used token is rejected.
- [ ] New password and confirmation must match.
- [ ] All refresh tokens are revoked after password reset.

### Email Verification

- [ ] A valid verification token verifies the email.
- [ ] Expired verification tokens are rejected.
- [ ] Used verification tokens are rejected.
- [ ] Already verified users receive an appropriate response.
- [ ] Verification-email resend is rate limited.

### Change Password

- [ ] An authenticated user can change the password using the correct current password.
- [ ] An incorrect current password is rejected.
- [ ] The new password meets all password rules.
- [ ] The new password differs from the current password.
- [ ] All refresh tokens are revoked after password change.

### Current User

- [ ] An authenticated user can retrieve current account details.
- [ ] Roles and permissions are included.
- [ ] Passwords and token hashes are never returned.

### Security and Quality

- [ ] Secrets are loaded through environment variables.
- [ ] Sensitive values are not written to logs.
- [ ] Validation errors use a consistent response format.
- [ ] Authentication exceptions use global exception handling.
- [ ] Swagger documents every authentication API.
- [ ] Unit tests cover service success and failure cases.
- [ ] Integration tests cover login, refresh, logout and password reset.
- [ ] Database indexes exist for email and token lookup.

---

## 17. Implementation Readiness Checklist

- [ ] Verify the `users` table and required fields.
- [ ] Create Flyway migrations for authentication tables.
- [ ] Create the Authentication package structure.
- [ ] Configure Spring Security.
- [ ] Configure BCrypt password encoder.
- [ ] Add JWT configuration properties.
- [ ] Create authentication entities.
- [ ] Create repositories.
- [ ] Create request and response DTOs.
- [ ] Implement token generation and validation.
- [ ] Implement refresh-token rotation.
- [ ] Implement `AuthService`.
- [ ] Implement `AuthController`.
- [ ] Configure global exception handling.
- [ ] Define the email-service interface.
- [ ] Add Swagger documentation.
- [ ] Add unit tests.
- [ ] Add integration tests.
