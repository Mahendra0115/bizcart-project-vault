# BizCart — User Module Requirements

## 1. Document Overview

| Field | Value |
|---|---|
| Project | BizCart — Small Business Commerce Platform |
| Module | User Module |
| Backend | Spring Boot Modular Monolith |
| Database | MySQL |
| Document Type | Requirements Document |
| Status | Ready for Senior Review |
| Version | 1.0 |
| Related Existing Module | Authentication Module |
| API Base Path | `/api/v1/users` |

---

## 2. Purpose

The User Module manages user profile data, customer/seller account status administration and customer address management for the BizCart platform.

This document covers only the pending User Module requirements that are not already completed inside the Authentication Module.

The module will provide:

- Customer profile update
- Basic profile data management
- Account-status management for admin-controlled user lifecycle
- Address CRUD for authenticated users
- Default address selection
- Address ownership validation
- User data safety rules for e-commerce use cases

---

## 3. Reference Alignment

The current backend already has a shared `User` entity and `users` table created during Authentication work. User Module implementation must reuse the existing user model and must not create a duplicate user table.

Current aligned package:

```text
com.mahendra.bizcart_backend.user
```

Existing user-related structure:

```text
user/
├── entity/
│   ├── User.java
│   ├── Role.java
│   ├── Permission.java
│   ├── UserRole.java
│   └── RolePermission.java
├── enums/
│   ├── AccountStatus.java
│   └── UserType.java
└── repository/
    ├── UserRepository.java
    ├── RoleRepository.java
    └── PermissionRepository.java
```

Existing `users` fields that must be reused:

```text
id
created_at
updated_at
first_name
last_name
username
email
phone
password
profile_image
user_type
status
email_verified
admin_approved
token_version
last_login_at
```

Existing status enum values:

```text
PENDING
ACTIVE
INACTIVE
BLOCKED
```

Existing user-type enum values:

```text
ADMIN
SELLER
CUSTOMER
```

---

## 4. Feature Documentation Status

| Feature | Documentation Status |
|---|---|
| Customer profile | Partial. `GET /api/v1/auth/me` returns current user details, but profile update API and flow are not documented. |
| Address management | Pending. Address module documentation currently has only placeholder content. Actual requirements, APIs and design are required. |
| Account status | Partial. Status values and authentication enforcement are documented, but activate, deactivate, block and unblock management APIs are not documented. |

---

## 5. Scope

### 5.1 In Scope

- Authenticated user profile update
- Updating first name, last name, username, phone and profile image
- Validating duplicate username and phone
- Preventing direct password, role and status update through profile API
- Admin user listing for account-status management
- Admin activate, deactivate, block and unblock user accounts
- Capturing status-change reason and actor
- Customer address creation
- Customer address update
- Customer address listing
- Customer address detail retrieval
- Customer address delete/removal
- Setting one default address per user
- Address ownership validation
- Address validation for Indian e-commerce use cases
- Basic audit-ready status-change history

### 5.2 Out of Scope

The following are already handled or will be handled by other modules:

- Login
- Logout
- Registration
- JWT access-token generation
- Refresh-token management
- Password reset
- Password change
- Email verification
- Phone OTP verification
- Google OAuth 2.0 login
- Role and permission assignment
- Seller approval workflow beyond account status
- Order address snapshot
- Shipping-rate calculation
- Payment or checkout workflow
- Admin profile management
- Store staff membership management

---

## 6. Functional Requirements

## 6.1 Customer Profile Update

### FR-USER-001 — Update Own Profile

The system must allow an authenticated user to update their own basic profile information.

Allowed update fields:

- First name
- Last name
- Username
- Phone
- Profile image URL

The user must not be allowed to update the following fields through this API:

- Email
- Password
- User type
- Account status
- Email verification status
- Admin approval status
- Token version
- Roles
- Permissions

### FR-USER-002 — Profile Update Validation

The system must validate profile update input before saving.

Validation rules:

- First name is required and must not exceed 100 characters.
- Last name is required and must not exceed 100 characters.
- Username is required, unique and must not exceed 50 characters.
- Phone is optional but must be unique when provided.
- Profile image URL is optional and must not exceed 500 characters.
- Empty strings must be trimmed and rejected where the field is required.

### FR-USER-003 — Username Update

The system may allow username update if the new username is unique.

If the submitted username already belongs to another user, the system must return a conflict response.

Username must be normalized before persistence.

Recommended username rules:

```text
Minimum length: 3
Maximum length: 50
Allowed characters: letters, numbers, dots, underscores and hyphens
```

### FR-USER-004 — Phone Update

The system may allow phone update from the profile API.

When the phone number is changed:

- It must be unique.
- It must be normalized.
- Phone verification must not be assumed by this module.
- Phone OTP verification belongs to the Authentication Module.

If a future `phone_verified` field is added, changing the phone number should reset `phone_verified=false`.

### FR-USER-005 — Safe Profile Response

Profile update response must return only safe user fields.

The response must not include:

- Password
- Token hashes
- JWT information
- Internal security secrets
- Refresh-token data

---

## 6.2 Account Status Management

### FR-USER-006 — Admin User Listing

The system must allow admins to list users for management purposes.

The listing should support:

- Pagination
- Sorting
- Search by name, email, username or phone
- Filter by user type
- Filter by account status

The listing must not expose password or token-related data.

### FR-USER-007 — View User Details for Admin

The system must allow admins to view safe details of a specific user.

The response should include:

- User ID
- First name
- Last name
- Username
- Email
- Phone
- Profile image
- User type
- Status
- Email verified
- Admin approved
- Last login time
- Created time
- Updated time

The response must not include password or token hashes.

### FR-USER-008 — Activate User Account

The system must allow an admin to activate a user account.

Allowed transition examples:

```text
PENDING  → ACTIVE
INACTIVE → ACTIVE
BLOCKED  → ACTIVE
```

Activation must be controlled by admin permission.

The system must record:

- Target user ID
- Previous status
- New status
- Changed by admin user ID
- Reason
- Timestamp

### FR-USER-009 — Deactivate User Account

The system must allow an admin to deactivate a user account.

Allowed transition examples:

```text
ACTIVE  → INACTIVE
PENDING → INACTIVE
```

Deactivated users must not be allowed to login or refresh tokens based on existing Authentication rules.

The system should revoke active refresh tokens for the user when deactivated.

### FR-USER-010 — Block User Account

The system must allow an admin to block a user account.

Allowed transition examples:

```text
ACTIVE   → BLOCKED
PENDING  → BLOCKED
INACTIVE → BLOCKED
```

Blocked users must not be allowed to login or refresh tokens.

The system should revoke active refresh tokens for the user when blocked.

### FR-USER-011 — Unblock User Account

The system must allow an admin to unblock a user account.

Recommended transition:

```text
BLOCKED → ACTIVE
```

If business approval is still pending for a seller, the system may move the user to `PENDING` instead of `ACTIVE`.

Final seller approval rules may be handled by Seller/User Management in a later phase.

### FR-USER-012 — Prevent Self-Blocking

An admin must not be allowed to block or deactivate their own account through the account-status management API.

### FR-USER-013 — Status Change Reason

Status update request must require a reason.

Reason requirements:

- Required
- Minimum 5 characters
- Maximum 500 characters
- Must be stored for audit and support review

---

## 6.3 Address Management

### FR-USER-014 — Create Address

The system must allow an authenticated user to create an address.

Address fields:

- Full name
- Phone
- Address line 1
- Address line 2
- Landmark
- City
- State
- Postal code
- Country
- Address type
- Default address flag

Supported address types:

```text
HOME
WORK
OTHER
```

### FR-USER-015 — List Own Addresses

The system must allow an authenticated user to list their own active addresses.

The user must not be able to view addresses belonging to another user.

### FR-USER-016 — Get Own Address Detail

The system must allow an authenticated user to view details of their own address by address ID.

If the address does not belong to the authenticated user, the system must return a safe not-found or forbidden response.

### FR-USER-017 — Update Own Address

The system must allow an authenticated user to update their own address.

The system must validate all address fields before updating.

### FR-USER-018 — Delete Own Address

The system must allow an authenticated user to delete or deactivate their own address.

Recommended approach:

```text
Soft delete using deleted_at or is_active=false
```

Address should not be hard-deleted if it may be referenced by an order in the future.

### FR-USER-019 — Set Default Address

The system must allow a user to set one address as default.

Rules:

- Only one default address is allowed per user.
- When one address becomes default, other addresses for the same user must become non-default.
- Default update must happen in a transaction.

### FR-USER-020 — Address Limit

The system should limit the number of active addresses per user.

Recommended limit:

```text
20 active addresses per user
```

This limit is suitable for a small e-commerce platform and prevents unnecessary data growth.

---

## 7. API Requirements

## 7.1 Base URL

```text
/api/v1/users
```

---

## 7.2 API Summary

| Method | Endpoint | Authentication | Purpose |
|---|---|---|---|
| PATCH | `/api/v1/users/me/profile` | Bearer token | Update own profile |
| GET | `/api/v1/users` | Admin Bearer token | List users for management |
| GET | `/api/v1/users/{userId}` | Admin Bearer token | View user details |
| PATCH | `/api/v1/users/{userId}/status` | Admin Bearer token | Update account status |
| POST | `/api/v1/users/me/addresses` | Bearer token | Create own address |
| GET | `/api/v1/users/me/addresses` | Bearer token | List own addresses |
| GET | `/api/v1/users/me/addresses/{addressId}` | Bearer token | Get own address detail |
| PUT | `/api/v1/users/me/addresses/{addressId}` | Bearer token | Update own address |
| DELETE | `/api/v1/users/me/addresses/{addressId}` | Bearer token | Delete own address |
| PATCH | `/api/v1/users/me/addresses/{addressId}/default` | Bearer token | Set default address |

---

## 7.3 Update Own Profile API

```http
PATCH /api/v1/users/me/profile
```

### Request

```json
{
  "firstName": "Mahendra",
  "lastName": "Singh",
  "username": "mahendra.singh",
  "phone": "9876543210",
  "profileImage": "https://cdn.bizcart.local/profiles/1.png"
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
    "profileImage": "https://cdn.bizcart.local/profiles/1.png",
    "userType": "CUSTOMER",
    "status": "ACTIVE",
    "emailVerified": true
  }
}
```

---

## 7.4 Admin User List API

```http
GET /api/v1/users?search=mahendra&userType=CUSTOMER&status=ACTIVE&page=0&size=20&sort=createdAt,desc
```

### Success Response

```json
{
  "message": "Users retrieved successfully",
  "data": {
    "content": [
      {
        "id": 1,
        "firstName": "Mahendra",
        "lastName": "Singh",
        "username": "mahendra.singh",
        "email": "customer@example.com",
        "phone": "9876543210",
        "userType": "CUSTOMER",
        "status": "ACTIVE",
        "emailVerified": true,
        "adminApproved": true,
        "lastLoginAt": "2026-07-21T10:30:00",
        "createdAt": "2026-07-20T10:30:00"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 1,
    "totalPages": 1
  }
}
```

---

## 7.5 Update Account Status API

```http
PATCH /api/v1/users/{userId}/status
```

### Request

```json
{
  "status": "BLOCKED",
  "reason": "Suspicious account activity reported by support team"
}
```

### Success Response

```json
{
  "message": "User status updated successfully",
  "data": {
    "id": 1,
    "previousStatus": "ACTIVE",
    "newStatus": "BLOCKED",
    "changedBy": 10,
    "reason": "Suspicious account activity reported by support team",
    "changedAt": "2026-07-21T10:30:00"
  }
}
```

---

## 7.6 Create Address API

```http
POST /api/v1/users/me/addresses
```

### Request

```json
{
  "fullName": "Mahendra Singh",
  "phone": "9876543210",
  "addressLine1": "D-77, Sector 63",
  "addressLine2": "Near Metro Station",
  "landmark": "Noida Sector 62",
  "city": "Noida",
  "state": "Uttar Pradesh",
  "postalCode": "201301",
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
    "id": 101,
    "fullName": "Mahendra Singh",
    "phone": "9876543210",
    "addressLine1": "D-77, Sector 63",
    "addressLine2": "Near Metro Station",
    "landmark": "Noida Sector 62",
    "city": "Noida",
    "state": "Uttar Pradesh",
    "postalCode": "201301",
    "country": "India",
    "addressType": "HOME",
    "defaultAddress": true
  }
}
```

---

## 7.7 List Addresses API

```http
GET /api/v1/users/me/addresses
```

### Success Response

```json
{
  "message": "Addresses retrieved successfully",
  "data": [
    {
      "id": 101,
      "fullName": "Mahendra Singh",
      "phone": "9876543210",
      "addressLine1": "D-77, Sector 63",
      "city": "Noida",
      "state": "Uttar Pradesh",
      "postalCode": "201301",
      "country": "India",
      "addressType": "HOME",
      "defaultAddress": true
    }
  ]
}
```

---

## 8. Validation Rules

## 8.1 Profile Validation

| Field | Required | Rule |
|---|---:|---|
| firstName | Yes | 1–100 characters |
| lastName | Yes | 1–100 characters |
| username | Yes | 3–50 characters, unique |
| phone | No | Valid phone format, unique |
| profileImage | No | Maximum 500 characters |

---

## 8.2 Account Status Validation

| Field | Required | Rule |
|---|---:|---|
| status | Yes | Must be one of `PENDING`, `ACTIVE`, `INACTIVE`, `BLOCKED` |
| reason | Yes | 5–500 characters |
| userId | Yes | Must identify an existing user |

Rules:

- Admin cannot block or deactivate their own account.
- Status changes must be performed only by users with admin permissions.
- Unknown user ID must return `404 Not Found`.
- Invalid status transition must return `400 Bad Request`.

---

## 8.3 Address Validation

| Field | Required | Rule |
|---|---:|---|
| fullName | Yes | 2–150 characters |
| phone | Yes | Valid phone format |
| addressLine1 | Yes | 5–255 characters |
| addressLine2 | No | Maximum 255 characters |
| landmark | No | Maximum 150 characters |
| city | Yes | 2–100 characters |
| state | Yes | 2–100 characters |
| postalCode | Yes | 4–20 characters |
| country | Yes | Default `India`, maximum 100 characters |
| addressType | Yes | `HOME`, `WORK`, `OTHER` |
| defaultAddress | No | Boolean |

---

## 9. Data Requirements

## 9.1 Existing Entity: User

User Module must reuse the existing `User` entity.

New profile update implementation should update only these fields:

```text
firstName
lastName
username
phone
profileImage
updatedAt
```

The following fields must not be updated by profile update API:

```text
email
password
userType
status
emailVerified
adminApproved
tokenVersion
lastLoginAt
createdAt
```

---

## 9.2 New Entity: Address

### Table: `addresses`

| Field | Type | Required | Description |
|---|---|---:|---|
| id | BIGINT | Yes | Address ID |
| user_id | BIGINT | Yes | Owner user ID |
| full_name | VARCHAR(150) | Yes | Receiver full name |
| phone | VARCHAR(20) | Yes | Receiver phone number |
| address_line_1 | VARCHAR(255) | Yes | Primary address line |
| address_line_2 | VARCHAR(255) | No | Secondary address line |
| landmark | VARCHAR(150) | No | Nearby landmark |
| city | VARCHAR(100) | Yes | City |
| state | VARCHAR(100) | Yes | State |
| postal_code | VARCHAR(20) | Yes | Postal code |
| country | VARCHAR(100) | Yes | Country |
| address_type | VARCHAR(50) | Yes | HOME, WORK or OTHER |
| is_default | BOOLEAN | Yes | Default address flag |
| is_active | BOOLEAN | Yes | Soft-delete flag |
| created_at | DATETIME | Yes | Created time |
| updated_at | DATETIME | Yes | Updated time |

### Required Indexes

```text
ix_addresses_user_id
ix_addresses_user_default
ix_addresses_user_active
```

### Relationships

```text
User 1 ---- many Address
```

Each address must belong to exactly one user.

---

## 9.3 New Entity: UserStatusHistory

### Table: `user_status_history`

| Field | Type | Required | Description |
|---|---|---:|---|
| id | BIGINT | Yes | History record ID |
| user_id | BIGINT | Yes | Target user |
| previous_status | VARCHAR(50) | Yes | Previous status |
| new_status | VARCHAR(50) | Yes | New status |
| changed_by | BIGINT | Yes | Admin user who changed status |
| reason | VARCHAR(500) | Yes | Status-change reason |
| created_at | DATETIME | Yes | Change time |

### Required Indexes

```text
ix_user_status_history_user_id
ix_user_status_history_changed_by
ix_user_status_history_created_at
```

### Relationships

```text
User 1 ---- many UserStatusHistory
Admin User 1 ---- many UserStatusHistory
```

---

## 10. Security and Authorization Requirements

### 10.1 Profile Security

- Only authenticated users can update their own profile.
- A user must not update another user's profile through `/me` endpoints.
- Profile update must not accept role, status, password or email fields.
- API responses must not expose sensitive authentication data.

### 10.2 Admin Status Security

- Only admin users can manage account status.
- Admin must not block or deactivate their own account.
- Blocking or deactivation should revoke active refresh tokens.
- Status change must be recorded in `user_status_history`.

### 10.3 Address Security

- Only authenticated users can manage their own addresses.
- Address APIs must validate ownership by `user_id`.
- Users must not access, update or delete another user's address.
- Deleted addresses should be soft deleted for future order-history safety.

---

## 11. Error Handling

All endpoints must follow the common BizCart error-response format.

### Error Codes

| HTTP Status | Error Code | Case |
|---|---|---|
| 400 | USER_VALIDATION_FAILED | Invalid user request |
| 400 | USER_INVALID_STATUS_TRANSITION | Invalid account-status transition |
| 400 | USER_ADDRESS_LIMIT_EXCEEDED | Maximum active address limit reached |
| 401 | AUTH_UNAUTHORIZED | Missing or invalid authentication |
| 403 | USER_FORBIDDEN | User does not have permission |
| 403 | USER_SELF_STATUS_CHANGE_NOT_ALLOWED | Admin tried to block/deactivate own account |
| 404 | USER_NOT_FOUND | User not found |
| 404 | USER_ADDRESS_NOT_FOUND | Address not found |
| 409 | USER_USERNAME_ALREADY_EXISTS | Username already exists |
| 409 | USER_PHONE_ALREADY_EXISTS | Phone already exists |
| 500 | USER_INTERNAL_ERROR | Unexpected server error |

---

## 12. Suggested DTOs

### Request DTOs

```text
UpdateProfileRequestDto
UpdateUserStatusRequestDto
CreateAddressRequestDto
UpdateAddressRequestDto
```

### Response DTOs

```text
UserProfileResponseDto
AdminUserSummaryResponseDto
AdminUserDetailResponseDto
UserStatusUpdateResponseDto
AddressResponseDto
```

---

## 13. Suggested Spring Boot Structure

```text
user
├── controller
│   ├── UserProfileController.java
│   ├── UserAdminController.java
│   └── UserAddressController.java
├── dto
│   ├── request
│   │   ├── UpdateProfileRequestDto.java
│   │   ├── UpdateUserStatusRequestDto.java
│   │   ├── CreateAddressRequestDto.java
│   │   └── UpdateAddressRequestDto.java
│   └── response
│       ├── UserProfileResponseDto.java
│       ├── AdminUserSummaryResponseDto.java
│       ├── AdminUserDetailResponseDto.java
│       ├── UserStatusUpdateResponseDto.java
│       └── AddressResponseDto.java
├── entity
│   ├── User.java
│   ├── Address.java
│   └── UserStatusHistory.java
├── enums
│   ├── AccountStatus.java
│   ├── UserType.java
│   └── AddressType.java
├── repository
│   ├── UserRepository.java
│   ├── AddressRepository.java
│   └── UserStatusHistoryRepository.java
├── service
│   ├── UserProfileService.java
│   ├── UserAdminService.java
│   └── UserAddressService.java
├── mapper
│   ├── UserMapper.java
│   └── AddressMapper.java
└── exception
    ├── UserNotFoundException.java
    ├── AddressNotFoundException.java
    ├── DuplicateUsernameException.java
    ├── DuplicatePhoneException.java
    └── InvalidUserStatusTransitionException.java
```

---

## 14. Acceptance Criteria

### Customer Profile Update

- [ ] Authenticated user can update own first name and last name.
- [ ] Authenticated user can update username when unique.
- [ ] Authenticated user can update phone when unique.
- [ ] Profile update does not allow email, password, role or status changes.
- [ ] Duplicate username returns conflict response.
- [ ] Duplicate phone returns conflict response.
- [ ] Response does not expose password or token fields.

### Account Status Management

- [ ] Admin can list users with pagination.
- [ ] Admin can filter users by status and user type.
- [ ] Admin can activate a user.
- [ ] Admin can deactivate a user.
- [ ] Admin can block a user.
- [ ] Admin can unblock a user.
- [ ] Status change requires a reason.
- [ ] Status change creates history record.
- [ ] Admin cannot block or deactivate their own account.
- [ ] Blocking or deactivation revokes active refresh sessions where supported.

### Address Management

- [ ] Authenticated user can create an address.
- [ ] Authenticated user can list only own addresses.
- [ ] Authenticated user can update only own address.
- [ ] Authenticated user can delete only own address.
- [ ] Authenticated user can set one default address.
- [ ] Setting default address clears previous default address.
- [ ] Address validation works for required fields.
- [ ] Deleted addresses are not returned in active address listing.

---

## 15. Implementation Sequence

1. Review existing `User` entity and `users` table.
2. Create User Module requirements review branch in vault if required.
3. Create `AddressType` enum.
4. Create `Address` entity.
5. Create `UserStatusHistory` entity.
6. Add Flyway migration for `addresses` and `user_status_history`.
7. Add repositories for address and status history.
8. Add profile update DTOs and mapper.
9. Implement profile update API.
10. Add admin user listing and detail APIs.
11. Implement status update API.
12. Add status-change history creation.
13. Revoke refresh sessions when account is blocked or deactivated.
14. Add address create/list/detail/update/delete APIs.
15. Add set-default-address transaction handling.
16. Add unit tests.
17. Add controller/security tests.
18. Add repository/integration tests.
19. Add Swagger documentation.
20. Complete QA checklist.

---

## 16. Senior Review Checklist

- [ ] Scope excludes authentication workflows.
- [ ] Existing `User` entity is reused.
- [ ] No duplicate user table is introduced.
- [ ] Profile update fields are safe and limited.
- [ ] Email and password changes remain outside this document.
- [ ] Account-status rules align with Authentication Module.
- [ ] Address ownership validation is clearly documented.
- [ ] Address soft-delete approach is acceptable.
- [ ] Status-change history is sufficient for audit needs.
- [ ] API naming follows existing BizCart style.
- [ ] Requirements are suitable for a small/personal e-commerce project.

---

## 17. Notes

This User Module requirements document is intentionally medium-scope. It is designed for a personal/small e-commerce platform, so it avoids enterprise-level complexity such as advanced KYC, profile privacy settings, identity verification workflows, account merge workflows and customer support case management.
