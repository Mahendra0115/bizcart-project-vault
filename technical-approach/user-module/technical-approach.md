# BizCart — User Module Technical Approach

## Document Information

| Field | Value |
|---|---|
| Project | BizCart — Small Business Commerce Platform |
| Module | User Module |
| Backend | Spring Boot Modular Monolith |
| Database | MySQL |
| Document Type | Technical Approach |
| Status | Ready for Senior Review |
| Version | 1.0 |
| Related Requirement | `bizcart-user-module-requirements.md` |
| API Base Path | `/api/v1/users` |

---

## 1. Overview / Introduction

The User Module will manage user profile data, customer addresses, and account-status operations that are outside the Authentication Module.

Authentication concerns such as login, logout, JWT, refresh token, forgot password, reset password, email verification, phone OTP verification, Google OAuth 2.0 login, and password change are already handled by the Authentication Module and must not be duplicated here.

This User Module approach focuses only on:

- Customer profile update
- Account-status management
- Address management

The module will reuse the existing `users` table and existing user-related enums already created in the backend.

---

## 2. Objective

The objective of the User Module is to provide a clean and secure way to manage user profile information, user addresses, and admin-controlled account lifecycle operations.

The module must:

- Allow authenticated users to update allowed profile fields.
- Prevent users from updating authentication-sensitive fields directly.
- Allow users to manage their own delivery addresses.
- Allow admins to activate, deactivate, block, and unblock user accounts.
- Maintain compatibility with the existing `User` entity and database naming.
- Follow existing BizCart response, validation, exception, and security patterns.
- Keep the implementation simple and suitable for a small e-commerce project.

---

## 3. Existing Backend Alignment

The current backend already contains a `User` entity under the user package.

The User Module must align with the existing structure:

```text
com.mahendra.bizcart_backend.user
├── entity
├── enums
└── repository
```

The existing user fields include:

```text
id
firstName
lastName
username
email
phone
password
profileImage
userType
status
emailVerified
adminApproved
tokenVersion
lastLoginAt
createdAt
updatedAt
```

The existing account status enum values are:

```text
PENDING
ACTIVE
INACTIVE
BLOCKED
```

The User Module must not create a duplicate `User` entity.

---

## 4. Final Architecture Decisions

### 4.1 Module Boundary Decision

The User Module will own profile management, address management, and account-status management.

The Authentication Module will continue to own:

- Login
- Logout
- Token generation
- Token refresh
- Password management
- Email verification
- Phone OTP verification
- Social login

This separation keeps the authentication layer secure and prevents user-profile APIs from modifying authentication-sensitive data.

### 4.2 User Entity Reuse Decision

The module will reuse the existing `User` entity and `users` table.

Allowed profile update fields:

```text
firstName
lastName
phone
profileImage
```

Restricted fields:

```text
id
username
email
password
userType
status
emailVerified
adminApproved
tokenVersion
lastLoginAt
createdAt
updatedAt
```

Restricted fields must not be updated through the profile update API.

### 4.3 Address Ownership Decision

Addresses will be managed as a separate entity linked to `User`.

A user may have multiple addresses, but only one default address should exist per user at a time.

Address data will be used later by Cart, Order, Shipping, and Checkout modules.

### 4.4 Account Status Management Decision

Account-status changes will be admin-controlled.

Regular customers and sellers must not be able to activate, deactivate, block, or unblock their own accounts.

Account-status actions will update the existing `status` field in the `users` table.

When a user is blocked, inactive, or reactivated, the module should consider whether active refresh sessions need to be revoked. If the Auth Module exposes a session-revocation service, the User Module should call it for blocking and deactivation.

---

## 5. Proposed Approach / Solution

## 5.1 Customer Profile Update Approach

The authenticated user will submit profile fields that are safe to update.

### Flow

1. The user sends a profile update request.
2. The backend identifies the user from the authenticated principal.
3. The backend loads the user from the database.
4. The backend validates editable fields.
5. The backend checks whether the phone number is already used by another user.
6. The backend updates only allowed fields.
7. The backend saves the user.
8. The backend returns a safe profile response.

### Editable Fields

```text
firstName
lastName
phone
profileImage
```

### Non-editable Fields

```text
email
username
password
userType
status
emailVerified
adminApproved
tokenVersion
lastLoginAt
```

Email update should not be included in this phase because it requires re-verification and additional security handling.

Username update should not be included in this phase because it affects unique identity and profile URLs in future modules.

---

## 5.2 Account Status Management Approach

Admins will manage user account status from admin-facing APIs.

### Supported Actions

```text
ACTIVATE
DEACTIVATE
BLOCK
UNBLOCK
```

### Flow

1. Admin sends status-management request for a target user.
2. Backend validates that the requester has admin authority.
3. Backend loads the target user.
4. Backend validates the requested status transition.
5. Backend updates the `status` field.
6. Backend records reason/note where applicable.
7. Backend optionally revokes active sessions for blocked or inactive users.
8. Backend returns updated user status details.

### Status Transition Rules

| Current Status | Allowed Action | New Status |
|---|---|---|
| PENDING | ACTIVATE | ACTIVE |
| ACTIVE | DEACTIVATE | INACTIVE |
| ACTIVE | BLOCK | BLOCKED |
| INACTIVE | ACTIVATE | ACTIVE |
| INACTIVE | BLOCK | BLOCKED |
| BLOCKED | UNBLOCK | ACTIVE or INACTIVE |

### Notes

- `BLOCKED` means the user should not be allowed to access protected APIs.
- `INACTIVE` means the account is disabled but not necessarily punished.
- `PENDING` means onboarding or approval is incomplete.
- Seller approval remains outside this module unless explicitly added later.

---

## 5.3 Address Management Approach

Users will manage their own addresses.

### Supported Address Actions

- Add address
- View address list
- View address by ID
- Update address
- Delete address
- Set default address

### Address Flow

#### Add Address

1. User submits address details.
2. Backend validates address fields.
3. Backend creates address linked to authenticated user.
4. If it is the first address, it becomes default automatically.
5. If `defaultAddress=true`, other user addresses are changed to non-default.
6. Backend returns the created address.

#### Update Address

1. User submits updated address details.
2. Backend loads address by ID and authenticated user ID.
3. Backend rejects access if address belongs to another user.
4. Backend updates allowed address fields.
5. Backend handles default-address logic.
6. Backend returns updated address.

#### Delete Address

1. User requests address deletion.
2. Backend checks ownership.
3. Backend deletes or soft-deletes the address.
4. If deleted address was default, backend may assign another active address as default.
5. Backend returns success response.

#### Set Default Address

1. User requests default change for an address.
2. Backend validates ownership.
3. Backend sets all user addresses to non-default.
4. Backend marks selected address as default.
5. Backend returns updated address.

---

## 6. Technology Stack and Justification

| Technology | Usage | Reason |
|---|---|---|
| Java 21 | Backend language | LTS and already used in BizCart |
| Spring Boot | Application framework | Existing backend framework |
| Spring Web | REST APIs | User and address endpoints |
| Spring Security | Access control | Authenticated and admin-only APIs |
| Spring Data JPA | Persistence | Existing repository pattern |
| Hibernate | ORM | Entity mapping |
| MySQL 8 | Database | Existing primary database |
| Bean Validation | DTO validation | Request validation |
| Flyway | DB migrations | Version-controlled schema changes |
| JUnit 5 | Testing | Unit and integration tests |
| Mockito | Unit tests | Mock services and repositories |
| Testcontainers | Integration tests | Real MySQL validation where needed |
| Springdoc OpenAPI | API docs | Swagger documentation |

---

## 7. Module Breakdown and File Structure

Recommended practical structure:

```text
user/
├── controller/
│   ├── UserProfileController.java
│   ├── UserAddressController.java
│   └── UserStatusController.java
│
├── dto/
│   ├── request/
│   │   ├── UpdateProfileRequestDto.java
│   │   ├── CreateAddressRequestDto.java
│   │   ├── UpdateAddressRequestDto.java
│   │   └── UpdateUserStatusRequestDto.java
│   │
│   └── response/
│       ├── UserProfileResponseDto.java
│       ├── AddressResponseDto.java
│       └── UserStatusResponseDto.java
│
├── entity/
│   ├── User.java
│   └── Address.java
│
├── enums/
│   ├── AccountStatus.java
│   ├── UserType.java
│   └── AddressType.java
│
├── repository/
│   ├── UserRepository.java
│   └── AddressRepository.java
│
├── service/
│   ├── UserProfileService.java
│   ├── UserAddressService.java
│   └── UserStatusService.java
│
├── mapper/
│   ├── UserMapper.java
│   └── AddressMapper.java
│
└── exception/
    ├── UserNotFoundException.java
    ├── AddressNotFoundException.java
    ├── AddressAccessDeniedException.java
    ├── DuplicatePhoneException.java
    └── InvalidAccountStatusTransitionException.java
```

### Structure Rules

- Reuse existing `User` entity.
- Do not duplicate authentication services.
- Do not expose password or token fields.
- Keep address logic inside User Module for now.
- Order and Shipping modules may reuse addresses later.

---

## 8. Database Design Approach

## 8.1 Users Table

The module will reuse the existing `users` table.

Fields used by profile update:

```text
first_name
last_name
phone
profile_image
updated_at
```

Fields used by account-status management:

```text
status
updated_at
```

Fields returned in profile response:

```text
id
first_name
last_name
username
email
phone
profile_image
user_type
status
email_verified
admin_approved
last_login_at
created_at
updated_at
```

Sensitive fields must never be returned:

```text
password
token_version
```

## 8.2 Addresses Table

A new `addresses` table will be created.

```text
addresses
├── id
├── created_at
├── updated_at
├── user_id
├── full_name
├── phone
├── address_line_1
├── address_line_2
├── landmark
├── city
├── state
├── postal_code
├── country
├── address_type
├── default_address
└── deleted
```

### Required Indexes

```text
ix_addresses_user_id
ix_addresses_user_default
ix_addresses_user_deleted
```

### Relationship

```text
users 1 ---- many addresses
```

Each address belongs to exactly one user.

### Default Address Rule

Only one non-deleted default address should be active for a user.

The service layer must enforce this rule in a transaction.

---

## 9. API Design Approach

## 9.1 Endpoint Summary

| Method | Endpoint | Access | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/me/profile` | Bearer token | Get current user profile |
| PUT | `/api/v1/users/me/profile` | Bearer token | Update current user profile |
| GET | `/api/v1/users/me/addresses` | Bearer token | List current user addresses |
| POST | `/api/v1/users/me/addresses` | Bearer token | Add new address |
| GET | `/api/v1/users/me/addresses/{addressId}` | Bearer token | Get address by ID |
| PUT | `/api/v1/users/me/addresses/{addressId}` | Bearer token | Update address |
| DELETE | `/api/v1/users/me/addresses/{addressId}` | Bearer token | Delete address |
| PUT | `/api/v1/users/me/addresses/{addressId}/default` | Bearer token | Set default address |
| GET | `/api/v1/users/admin/users` | Admin | List users for management |
| GET | `/api/v1/users/admin/users/{userId}` | Admin | Get user details |
| PATCH | `/api/v1/users/admin/users/{userId}/status` | Admin | Update user status |

---

## 9.2 Update Profile API

```http
PUT /api/v1/users/me/profile
```

### Request

```json
{
  "firstName": "Mahendra",
  "lastName": "Singh",
  "phone": "9876543210",
  "profileImage": "https://cdn.bizcart.local/profiles/user-1.png"
}
```

### Success Response

```json
{
  "message": "Profile updated successfully",
  "data": {
    "id": 1,
    "firstName": "Mahendra",
    "lastName": "Singh",
    "username": "mahendra.singh",
    "email": "customer@example.com",
    "phone": "9876543210",
    "profileImage": "https://cdn.bizcart.local/profiles/user-1.png",
    "userType": "CUSTOMER",
    "status": "ACTIVE",
    "emailVerified": true,
    "adminApproved": false
  }
}
```

---

## 9.3 Create Address API

```http
POST /api/v1/users/me/addresses
```

### Request

```json
{
  "fullName": "Mahendra Singh",
  "phone": "9876543210",
  "addressLine1": "B-101, Sector 62",
  "addressLine2": "Near Metro Station",
  "landmark": "Noida Electronic City",
  "city": "Noida",
  "state": "Uttar Pradesh",
  "postalCode": "201309",
  "country": "India",
  "addressType": "HOME",
  "defaultAddress": true
}
```

### Success Response

```json
{
  "message": "Address created successfully",
  "data": {
    "id": 10,
    "fullName": "Mahendra Singh",
    "phone": "9876543210",
    "addressLine1": "B-101, Sector 62",
    "addressLine2": "Near Metro Station",
    "landmark": "Noida Electronic City",
    "city": "Noida",
    "state": "Uttar Pradesh",
    "postalCode": "201309",
    "country": "India",
    "addressType": "HOME",
    "defaultAddress": true
  }
}
```

---

## 9.4 Update Account Status API

```http
PATCH /api/v1/users/admin/users/{userId}/status
```

### Request

```json
{
  "status": "BLOCKED",
  "reason": "Suspicious activity detected"
}
```

### Success Response

```json
{
  "message": "User status updated successfully",
  "data": {
    "id": 1,
    "email": "customer@example.com",
    "userType": "CUSTOMER",
    "oldStatus": "ACTIVE",
    "newStatus": "BLOCKED",
    "reason": "Suspicious activity detected"
  }
}
```

---

## 10. Validation Approach

## 10.1 Profile Validation

| Field | Validation |
|---|---|
| firstName | Required, 2-100 characters |
| lastName | Required, 2-100 characters |
| phone | Optional, valid mobile format, unique when present |
| profileImage | Optional, valid URL, max 500 characters |

## 10.2 Address Validation

| Field | Validation |
|---|---|
| fullName | Required, 2-150 characters |
| phone | Required, valid mobile format |
| addressLine1 | Required, 5-255 characters |
| addressLine2 | Optional, max 255 characters |
| landmark | Optional, max 150 characters |
| city | Required, 2-100 characters |
| state | Required, 2-100 characters |
| postalCode | Required, 4-20 characters |
| country | Required, default India |
| addressType | Required, HOME, WORK, OTHER |
| defaultAddress | Optional boolean |

## 10.3 Account Status Validation

- Status must be one of `PENDING`, `ACTIVE`, `INACTIVE`, `BLOCKED`.
- Admin cannot block their own account through this API.
- Invalid status transition must return validation error.
- Reason is required when blocking a user.
- Reason is optional for activate, deactivate, and unblock actions.

---

## 11. Error Handling Approach

The User Module will use the common BizCart error-response format.

### Important Error Codes

| HTTP Status | Code | Case |
|---:|---|---|
| 400 | `USER_VALIDATION_FAILED` | Invalid request fields |
| 400 | `USER_INVALID_STATUS_TRANSITION` | Invalid status transition |
| 401 | `AUTH_INVALID_TOKEN` | Missing or invalid bearer token |
| 403 | `USER_ACCESS_DENIED` | User tried to access another user's address |
| 403 | `USER_ADMIN_ACCESS_REQUIRED` | Admin role required |
| 404 | `USER_NOT_FOUND` | User not found |
| 404 | `USER_ADDRESS_NOT_FOUND` | Address not found |
| 409 | `USER_PHONE_ALREADY_EXISTS` | Phone already used by another account |
| 409 | `USER_DEFAULT_ADDRESS_REQUIRED` | Cannot remove default without replacement |
| 500 | `USER_INTERNAL_ERROR` | Unexpected server error |

---

## 12. Security Approach

### 12.1 Profile Security

- Profile update requires a valid bearer token.
- User can update only their own profile.
- User cannot update email through this API.
- User cannot update username through this API.
- User cannot update password through this API.
- Password and token-related fields must never be returned.

### 12.2 Address Security

- Address APIs require a valid bearer token.
- Users can access only their own addresses.
- Address ownership must be checked by `addressId` and authenticated `userId`.
- Address deletion must not affect addresses belonging to other users.

### 12.3 Account Status Security

- Account-status APIs require admin role.
- Admin cannot block or deactivate themselves from the same API.
- Status change should be logged for audit.
- Blocking or deactivating a user should revoke active sessions if Auth service supports it.

---

## 13. Transaction Management

The following operations must be transactional:

- Profile update
- Create address with default address handling
- Update address with default address handling
- Delete default address
- Set default address
- Account-status update
- Session revocation after account status change, if applicable

---

## 14. Testing Approach

## 14.1 Unit Tests

Primary targets:

- `UserProfileService`
- `UserAddressService`
- `UserStatusService`
- `UserMapper`
- `AddressMapper`

Important scenarios:

- Successful profile update
- Duplicate phone rejection
- Restricted fields ignored
- User not found
- Address create
- Address update
- Address delete
- Set default address
- Address ownership check
- Account status activate
- Account status block
- Invalid status transition
- Admin self-block rejection

## 14.2 Controller Tests

Tests must verify:

- Missing bearer token returns 401
- Non-admin user cannot update status
- User cannot access another user's address
- Validation errors return common error format
- Successful profile update returns safe response
- Successful address create/update/delete
- Successful status change

## 14.3 Repository Tests

Tests should verify:

- User lookup by ID
- Phone uniqueness check excluding current user
- Address lookup by ID and user ID
- Active address listing by user ID
- Default address query by user ID

## 14.4 Integration Tests

### Profile Update

```text
Create active user
→ Authenticate
→ Update profile
→ Verify user row updated
→ Verify restricted fields unchanged
```

### Address Management

```text
Create active user
→ Add first address
→ Verify default address true
→ Add second default address
→ Verify first address default false
→ Delete default address
→ Verify fallback/default behavior
```

### Account Status Management

```text
Create admin user
→ Create customer
→ Update status ACTIVE to BLOCKED
→ Verify user status changed
→ Verify customer cannot access protected APIs after block
```

---

## 15. Risks and Mitigation

| Risk | Mitigation |
|---|---|
| User updates sensitive fields | Use DTO allowlist and mapper allowlist |
| Duplicate phone race condition | Use DB unique index and convert conflict to 409 |
| User accesses another address | Query address by `addressId` and `userId` together |
| Multiple default addresses | Enforce default update in transaction |
| Admin blocks own account | Explicit self-action validation |
| Status changed but active token remains valid | Revoke sessions or rely on short token expiry plus status recheck |
| Address used by existing order gets deleted | Prefer soft delete when Order module starts using addresses |

---

## 16. Assumptions and Dependencies

### Assumptions

- Authentication module is already implemented.
- Existing `User` entity and `users` table are reused.
- Email and username update are not part of this phase.
- Password update is handled by Authentication Module.
- Address management belongs to User Module for now.
- Address records may later be referenced by Order and Shipping modules.
- Admin users already have proper roles.
- Common API response and exception handling already exist.

### Internal Dependencies

- Authentication Module
- User Repository
- Common BaseEntity
- Common exception handler
- Spring Security authenticated principal
- Role and Permission Module
- Future Audit Log Module

### External Dependencies

- MySQL
- Frontend profile page
- Frontend address management page
- Frontend admin user-management page

---

## 17. Required Configuration

No new external provider configuration is required for the first version.

Recommended configurable values:

```text
USER_PROFILE_IMAGE_MAX_LENGTH
USER_ADDRESS_MAX_COUNT
USER_ADDRESS_DEFAULT_COUNTRY
USER_STATUS_CHANGE_REVOKE_SESSIONS
```

Default values:

```text
USER_ADDRESS_MAX_COUNT=10
USER_ADDRESS_DEFAULT_COUNTRY=India
USER_STATUS_CHANGE_REVOKE_SESSIONS=true
```

---

## 18. Out of Scope

The following are not included in this User Module phase:

- Login
- Logout
- Registration
- Password change
- Forgot password
- Reset password
- Email verification
- Phone OTP verification
- Google OAuth 2.0 login
- Email update flow
- Username update flow
- Profile image upload service
- Admin creation
- Role and permission assignment
- Seller approval workflow
- Order address snapshot
- Address verification by courier APIs
- KYC verification

---

## 19. Implementation Sequence

1. Review existing `User` entity and repository.
2. Create User Module request and response DTOs.
3. Add `Address` entity.
4. Add Flyway migration for `addresses` table.
5. Add `AddressRepository`.
6. Implement `UserProfileService`.
7. Implement profile update API.
8. Implement `UserAddressService`.
9. Implement address create/list/detail/update/delete/default APIs.
10. Implement `UserStatusService`.
11. Implement admin account-status API.
12. Add validation and exception classes.
13. Add Swagger/OpenAPI annotations.
14. Add unit tests.
15. Add controller/security tests.
16. Add repository/integration tests.
17. Run full regression with Auth Module.
18. Prepare PR summary and QA checklist.

---

## 20. Final Development Readiness

The User Module will be considered ready for implementation when:

- Requirements document is reviewed.
- Address table design is approved.
- Profile editable fields are approved.
- Account-status transition rules are approved.
- Admin-only status management security is approved.
- API endpoints are approved by frontend requirements.
- Auth Module integration points are clear.
- Test scenarios are reviewed.
