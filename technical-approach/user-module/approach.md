# BizCart — User Module Technical Approach

## 1. Introduction / Background

The User Module will manage user profile updates, address management, and admin-controlled account-status operations in BizCart.

The Authentication Module already handles login, logout, JWT, refresh token, password reset, email verification, phone OTP, and Google OAuth.  
Therefore, this approach focuses only on the remaining user-management responsibilities.

---

## 2. Objective

The objective of this approach is to define how the User Module should be designed and implemented in the BizCart Spring Boot backend.

This approach will solve:

- Safe customer profile update.
- Customer address management.
- Admin-controlled account-status management.
- Clear separation between Authentication Module and User Module.
- Reuse of the existing `User` entity and `users` table.

---

## 3. Current System / As-Is

The current backend already has an Authentication Module and an existing `User` entity.

Current available user-related behavior:

- User registration is handled by Authentication Module.
- Login and current user retrieval are handled by Authentication Module.
- `GET /api/v1/auth/me` returns current authenticated-user details.
- User account status values already exist:
  - `PENDING`
  - `ACTIVE`
  - `INACTIVE`
  - `BLOCKED`
- Existing user fields include:
  - `firstName`
  - `lastName`
  - `username`
  - `email`
  - `phone`
  - `profileImage`
  - `userType`
  - `status`
  - `emailVerified`
  - `adminApproved`
  - `tokenVersion`
  - `lastLoginAt`

Current gaps:

- Profile update API is not documented.
- Address management is not implemented/documented.
- Admin account-status management APIs are not documented.

---

## 4. Proposed Approach / To-Be

The User Module will be created as a dedicated module inside the existing Spring Boot modular monolith.

The module will contain three main parts:

```text
User Module
├── Profile Management
├── Address Management
└── Account Status Management
```

### 4.1 Profile Management

Profile management will allow authenticated users to update only safe profile fields.

Allowed update fields:

```text
firstName
lastName
phone
profileImage
```

Restricted fields:

```text
id
email
username
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

The profile update API will use an allowlist approach so that only approved fields can be modified.

### 4.2 Address Management

Address management will allow authenticated users to create, view, update, delete, and set default addresses.

The address will be linked with the authenticated user.

Rules:

- A user can have multiple addresses.
- A user can have only one default address.
- User cannot access another user's address.
- Soft delete is preferred for future order/shipping references.
- First address should become default automatically.

### 4.3 Account Status Management

Account-status management will be admin-only.

Supported actions:

```text
ACTIVATE
DEACTIVATE
BLOCK
UNBLOCK
```

Status changes will update the existing `status` field in the `users` table.

Rules:

- Only admin can update account status.
- Admin cannot block or deactivate their own account using this API.
- Invalid status transitions must be rejected.
- Blocking or deactivation should revoke active sessions if Auth Module provides a session-revocation method.

---

## 5. High-Level Architecture

### 5.1 Component Flow

```text
Frontend
   |
   v
User Controller
   |
   v
User Service
   |
   v
User Repository / Address Repository
   |
   v
MySQL
```

### 5.2 Profile Update Flow

```text
Authenticated User
   |
   v
PUT /api/v1/users/me/profile
   |
   v
Validate DTO
   |
   v
Load User by authenticated userId
   |
   v
Update allowed fields only
   |
   v
Save User
   |
   v
Return safe profile response
```

### 5.3 Address Management Flow

```text
Authenticated User
   |
   v
Address API
   |
   v
Validate ownership
   |
   v
Create / Update / Delete / Set Default
   |
   v
Save Address
   |
   v
Return address response
```

### 5.4 Account Status Flow

```text
Admin User
   |
   v
PATCH /api/v1/users/admin/users/{userId}/status
   |
   v
Validate admin access
   |
   v
Validate status transition
   |
   v
Update user status
   |
   v
Optionally revoke sessions
   |
   v
Return status response
```

---

## 6. Technology Stack

| Technology | Usage | Reason |
|---|---|---|
| Java 21 | Backend language | Existing BizCart backend language |
| Spring Boot | Application framework | Existing backend framework |
| Spring Web | REST APIs | Controller layer |
| Spring Security | Access control | Bearer token and admin authorization |
| Spring Data JPA | Persistence | Repository abstraction |
| Hibernate | ORM | Entity-table mapping |
| MySQL | Database | Existing primary database |
| Flyway | Migration | Version-controlled DB schema |
| Bean Validation | Request validation | DTO field validation |
| Springdoc OpenAPI | API documentation | Swagger support |
| JUnit 5 | Testing | Unit and integration testing |
| Mockito | Unit testing | Mock service/repository dependencies |

---

## 7. Design Considerations

### 7.1 Security

- Profile and address APIs must require bearer token.
- Status-management API must require admin role.
- Users must access only their own profile and addresses.
- Password, token, and internal security fields must never be returned.
- Profile update must use field allowlisting.
- Address ownership must be checked using `userId`.

### 7.2 Performance

- Profile update should be a simple single-user update.
- Address list should query by indexed `user_id`.
- Default-address update should run inside a transaction.
- Admin status update should avoid unnecessary joins.

### 7.3 Scalability

- Address table should support future Order, Checkout, and Shipping modules.
- Soft delete should be preferred so future order history does not break.
- User Module should remain independent from Authentication internals.
- Services should be separated by responsibility.

### 7.4 Data Consistency

- Only one default address should exist per user.
- Phone number should remain unique if used as a profile field.
- Account status must use existing enum values only.
- Restricted user fields must not be modified from profile APIs.

---

## 8. Alternative Approaches Considered

| Option | Description | Decision |
|---|---|---|
| Put profile update inside Auth Module | Extend Auth APIs to update profile | Rejected because Auth Module should focus on authentication only |
| Keep addresses in separate Address Module now | Build standalone Address Module | Rejected for now; address management is user-owned in this phase |
| Hard delete addresses | Permanently remove address row | Rejected because future orders may need address history |
| Allow user to update email here | Add email update in profile API | Rejected because email update needs verification flow |
| Allow username update | User can change username | Rejected for this phase due to uniqueness and future profile URL impact |

---

## 9. Risks and Mitigation

| Risk | Mitigation |
|---|---|
| User updates sensitive fields | Use DTO allowlist and mapper allowlist |
| Duplicate phone number conflict | Add uniqueness check and handle DB conflict |
| User accesses another user's address | Fetch address by `addressId` and `userId` together |
| Multiple default addresses | Use transaction when setting default address |
| Admin blocks own account | Add self-action validation |
| Blocked user keeps active token | Revoke sessions if Auth service supports it |
| Address deletion breaks future orders | Use soft delete instead of hard delete |

---

## 10. Assumptions and Constraints

### Assumptions

- Authentication Module is already completed.
- Existing `User` entity and `users` table will be reused.
- Admin users already have valid admin role.
- Address management belongs to User Module in this phase.
- Email and username update are not required in this phase.
- Order and Shipping modules may use addresses later.

### Constraints

- Authentication logic must not be duplicated.
- Password update must remain in Authentication Module.
- Phone OTP verification must remain in Authentication Module.
- Google OAuth login must remain in Authentication Module.
- Only one default address is allowed per user.
- Account-status management must be admin-only.

---

## 11. Timeline / Phases

### Phase 1 — Profile Management

- Create profile DTOs.
- Create profile response mapper.
- Implement profile view API.
- Implement profile update API.
- Add profile validation and tests.

### Phase 2 — Address Management

- Create `Address` entity.
- Create Flyway migration for `addresses` table.
- Create `AddressRepository`.
- Implement create/list/view/update/delete APIs.
- Implement set-default-address API.
- Add ownership and default-address tests.

### Phase 3 — Account Status Management

- Create status update DTO.
- Implement admin status-management service.
- Implement status transition validation.
- Add admin-only controller API.
- Add security and service tests.

### Phase 4 — Final Review

- Add Swagger/OpenAPI documentation.
- Run unit and integration tests.
- Run Auth regression tests.
- Prepare PR summary and QA checklist.

---

## 12. Conclusion / Recommendation

The recommended approach is to build the User Module as a separate module from Authentication while reusing the existing `User` entity and database structure.

Profile update, address management, and account-status management should be implemented in small focused services.  
This keeps the project clean, avoids duplicate authentication logic, and prepares the backend for future Checkout, Order, and Shipping modules.
