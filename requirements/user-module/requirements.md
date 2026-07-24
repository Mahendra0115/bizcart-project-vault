# BizCart — User Module Requirements

## 1. Module Overview

The User Module manages user profile updates, address management, and admin-controlled account-status operations for BizCart.  
Authentication features such as login, logout, password reset, JWT, OAuth, and OTP are handled separately by the Authentication Module.

---

## 2. Objective / Goals

The goal of this module is to allow users to manage their profile and delivery addresses, while allowing admins to manage user account status securely.  
The module must reuse the existing `User` entity and must not duplicate authentication logic.

- Provide safe customer profile update.
- Provide address CRUD and default-address handling.
- Provide admin-only account-status management.
- Keep user data consistent with existing backend structure.

---

## 3. Scope

### In Scope

- Customer profile update.
- Customer profile view using User Module API.
- Address create, list, view, update, delete.
- Set default address.
- Admin account-status management.
- User status transition validation.
- User/address validation and error handling.

### Out of Scope

- Login and logout.
- Registration.
- Password change.
- Forgot password and reset password.
- Email verification.
- Phone OTP verification.
- Google OAuth 2.0 login.
- Username update.
- Email update.
- Profile image upload service.
- Role and permission assignment.
- Seller approval workflow.

---

## 4. Functional Requirements

### FR-USER-001 — View Customer Profile

The system must allow an authenticated user to view their profile details.

- Return safe user fields only.
- Do not return password, token version, or token-related fields.
- Response must include profile, account status, and verification flags.

### FR-USER-002 — Update Customer Profile

The system must allow an authenticated user to update allowed profile fields.

- User can update first name, last name, phone, and profile image.
- User cannot update email, username, password, user type, status, or verification fields.
- Phone number must be unique if provided.
- Updated profile must return safe user details.

### FR-USER-003 — Add Address

The system must allow an authenticated user to add a delivery address.

- Address must be linked to the authenticated user.
- First address should become default automatically.
- If new address is marked default, existing default address must be unset.
- Required address fields must be validated.

### FR-USER-004 — View Addresses

The system must allow an authenticated user to view their saved addresses.

- User can view only their own addresses.
- Deleted addresses must not appear in the active address list.
- Default address should be clearly marked.

### FR-USER-005 — Update Address

The system must allow an authenticated user to update their own address.

- User cannot update another user’s address.
- Address ownership must be validated.
- Default-address logic must be handled safely.

### FR-USER-006 — Delete Address

The system must allow an authenticated user to delete their own address.

- User cannot delete another user’s address.
- Soft delete is preferred for future order/shipping reference.
- If default address is deleted, another address may be selected as default.

### FR-USER-007 — Set Default Address

The system must allow an authenticated user to set one address as default.

- Only one default address is allowed per user.
- Setting one address as default must unset other default addresses.
- Operation must be transactional.

### FR-USER-008 — Manage Account Status

The system must allow admins to activate, deactivate, block, and unblock users.

- Only admin users can update account status.
- Admin cannot block or deactivate their own account through this API.
- Status transition must be validated.
- Blocking or deactivation should revoke active sessions if Auth service supports it.

---

## 5. Non-Functional Requirements

### Security

- All profile and address APIs must require a valid bearer token.
- Account-status APIs must require admin access.
- Users must access only their own profile and addresses.
- Sensitive fields must never be returned in API responses.

### Performance

- Profile update should complete within 300 ms under normal load.
- Address list should complete within 300 ms for normal user data.
- Account-status update should complete within 300 ms excluding session revocation.

### Scalability

- Address table must support future use by Order, Shipping, and Checkout modules.
- Default-address update must work correctly in a multi-user system.
- Service design should remain modular and reusable.

---

## 6. User Stories / Use Cases

- As a customer, I want to update my profile so that my personal details stay current.
- As a customer, I want to add multiple addresses so that I can choose delivery locations during checkout.
- As a customer, I want to set a default address so that checkout can preselect my preferred delivery address.
- As a customer, I want to update or delete old addresses so that my address book stays clean.
- As an admin, I want to block a user so that suspicious accounts cannot access the platform.
- As an admin, I want to reactivate a user so that valid users can regain access.

---

## 7. API / Data Requirements

### API Summary

| Method | Endpoint | Access | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/me/profile` | Bearer token | View current user profile |
| PUT | `/api/v1/users/me/profile` | Bearer token | Update current user profile |
| GET | `/api/v1/users/me/addresses` | Bearer token | List addresses |
| POST | `/api/v1/users/me/addresses` | Bearer token | Add address |
| GET | `/api/v1/users/me/addresses/{addressId}` | Bearer token | View address |
| PUT | `/api/v1/users/me/addresses/{addressId}` | Bearer token | Update address |
| DELETE | `/api/v1/users/me/addresses/{addressId}` | Bearer token | Delete address |
| PUT | `/api/v1/users/me/addresses/{addressId}/default` | Bearer token | Set default address |
| PATCH | `/api/v1/users/admin/users/{userId}/status` | Admin | Update account status |

### User Data

Reuse existing `users` table.

Editable profile fields:

- `first_name`
- `last_name`
- `phone`
- `profile_image`

Status-management field:

- `status`

### Address Data

Create new `addresses` table.

Required fields:

- `id`
- `user_id`
- `full_name`
- `phone`
- `address_line_1`
- `address_line_2`
- `landmark`
- `city`
- `state`
- `postal_code`
- `country`
- `address_type`
- `default_address`
- `deleted`
- `created_at`
- `updated_at`

---

## 8. Dependencies

- Authentication Module
- Existing User entity and User repository
- Role and Permission Module
- Common API response structure
- Global exception handler
- Spring Security authenticated principal
- MySQL database
- Future Order, Shipping, and Checkout modules

---

## 9. Assumptions and Constraints

### Assumptions

- Authentication Module is already completed.
- Existing `User` entity will be reused.
- Address management belongs to User Module for this phase.
- Admin users already have proper admin role.
- Email and username updates are not required in this phase.

### Constraints

- Authentication logic must not be duplicated.
- User cannot update restricted fields through profile update.
- Only admin can update account status.
- Only one default address is allowed per user.
- Address deletion should not break future order history.

---

## 10. Acceptance Criteria

### Profile

- [ ] Authenticated user can view profile.
- [ ] Authenticated user can update allowed profile fields.
- [ ] Restricted fields cannot be updated.
- [ ] Duplicate phone number returns conflict error.
- [ ] Password and token fields are never returned.

### Address

- [ ] User can create an address.
- [ ] First address becomes default automatically.
- [ ] User can list only their own addresses.
- [ ] User can update only their own address.
- [ ] User can delete only their own address.
- [ ] User can set one default address.
- [ ] Only one default address exists per user.

### Account Status

- [ ] Admin can activate user.
- [ ] Admin can deactivate user.
- [ ] Admin can block user.
- [ ] Admin can unblock user.
- [ ] Non-admin cannot update account status.
- [ ] Admin cannot block or deactivate own account.
- [ ] Invalid status transition returns validation error.
