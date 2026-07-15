from pathlib import Path

content = """# BizCart — Backend Modules and Table Schemas

## 1. Project Overview

**Project Name:** BizCart  
**Project Type:** Small Business Commerce Platform  
**Backend:** Spring Boot  
**Database:** MySQL  

This document contains only the important backend modules required for a small business e-commerce platform.

---

# 2. Backend Modules

1. Auth Module
2. User Module
3. Admin Module
4. Role and Permission Module
5. Store Module
6. Address Module
7. Category Module
8. Product Module
9. Product Image Module
10. Product Variant Module
11. Product Attribute Module
12. Inventory Module
13. Stock Transaction Module
14. Cart Module
15. Wishlist Module
16. Coupon Module
17. Order Module
18. Payment Module
19. Refund Module
20. Shipping Module
21. Review Module
22. Notification Module
23. Media Module
24. Report Module
25. Audit Log Module

---

# 3. Module-Wise Table Schemas

## 3.1 Auth Module

### Responsibilities

- Login
- Logout
- Refresh token
- Forgot password
- Reset password
- Email verification

### Table: `refresh_tokens`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Refresh token ID |
| user_id | BIGINT | FK | User ID |
| token | VARCHAR(500) | UNIQUE | Refresh token |
| expires_at | DATETIME |  | Token expiry |
| revoked | BOOLEAN |  | Token revoked status |
| created_at | DATETIME |  | Created date |

### Table: `password_reset_tokens`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Reset token ID |
| user_id | BIGINT | FK | User ID |
| token | VARCHAR(500) | UNIQUE | Password reset token |
| expires_at | DATETIME |  | Token expiry |
| used | BOOLEAN |  | Token used status |
| created_at | DATETIME |  | Created date |

---

## 3.2 User Module

### Table: `users`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | User ID |
| first_name | VARCHAR(100) |  | First name |
| last_name | VARCHAR(100) |  | Last name |
| email | VARCHAR(255) | UNIQUE | User email |
| phone | VARCHAR(20) | UNIQUE | Phone number |
| password | VARCHAR(255) |  | Encrypted password |
| profile_image | VARCHAR(500) |  | Profile image |
| user_type | ENUM('ADMIN','SELLER','CUSTOMER') |  | User type |
| status | ENUM('ACTIVE','INACTIVE','BLOCKED') |  | User status |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.3 Admin Module

Admin users will be stored in the `users` table with `user_type = ADMIN`.

### Table: `admin_profiles`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Admin profile ID |
| user_id | BIGINT | FK, UNIQUE | User ID |
| designation | VARCHAR(100) |  | Admin designation |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.4 Role and Permission Module

### Table: `roles`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Role ID |
| name | VARCHAR(100) | UNIQUE | Role name |
| description | VARCHAR(500) |  | Role description |
| created_at | DATETIME |  | Created date |

### Table: `permissions`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Permission ID |
| name | VARCHAR(100) | UNIQUE | Permission name |
| module_name | VARCHAR(100) |  | Module name |
| action_name | VARCHAR(100) |  | Action name |
| created_at | DATETIME |  | Created date |

### Table: `user_roles`

| Column | Type | Key | Description |
|---|---|---|---|
| user_id | BIGINT | PK, FK | User ID |
| role_id | BIGINT | PK, FK | Role ID |

### Table: `role_permissions`

| Column | Type | Key | Description |
|---|---|---|---|
| role_id | BIGINT | PK, FK | Role ID |
| permission_id | BIGINT | PK, FK | Permission ID |

---

## 3.5 Store Module

### Table: `stores`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Store ID |
| owner_id | BIGINT | FK | Store owner user ID |
| name | VARCHAR(150) |  | Store name |
| slug | VARCHAR(180) | UNIQUE | Store URL slug |
| description | TEXT |  | Store description |
| email | VARCHAR(255) |  | Store email |
| phone | VARCHAR(20) |  | Store phone |
| logo_url | VARCHAR(500) |  | Store logo |
| banner_url | VARCHAR(500) |  | Store banner |
| status | ENUM('ACTIVE','INACTIVE') |  | Store status |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.6 Address Module

### Table: `addresses`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Address ID |
| user_id | BIGINT | FK | User ID |
| name | VARCHAR(150) |  | Receiver name |
| phone | VARCHAR(20) |  | Receiver phone |
| address_line_1 | VARCHAR(255) |  | Address line 1 |
| address_line_2 | VARCHAR(255) |  | Address line 2 |
| city | VARCHAR(100) |  | City |
| state | VARCHAR(100) |  | State |
| postal_code | VARCHAR(20) |  | Postal code |
| country | VARCHAR(100) |  | Country |
| is_default | BOOLEAN |  | Default address |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.7 Category Module

### Table: `categories`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Category ID |
| store_id | BIGINT | FK | Store ID |
| parent_id | BIGINT | FK | Parent category ID |
| name | VARCHAR(150) |  | Category name |
| slug | VARCHAR(180) |  | Category slug |
| description | TEXT |  | Category description |
| image_url | VARCHAR(500) |  | Category image |
| is_active | BOOLEAN |  | Category status |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.8 Product Module

### Table: `products`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Product ID |
| store_id | BIGINT | FK | Store ID |
| category_id | BIGINT | FK | Category ID |
| name | VARCHAR(200) |  | Product name |
| slug | VARCHAR(220) |  | Product slug |
| description | TEXT |  | Product description |
| sku | VARCHAR(100) | UNIQUE | Product SKU |
| price | DECIMAL(10,2) |  | Regular price |
| sale_price | DECIMAL(10,2) |  | Sale price |
| status | ENUM('ACTIVE','INACTIVE','OUT_OF_STOCK') |  | Product status |
| is_featured | BOOLEAN |  | Featured product |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.9 Product Image Module

### Table: `product_images`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Product image ID |
| product_id | BIGINT | FK | Product ID |
| image_url | VARCHAR(500) |  | Product image URL |
| is_primary | BOOLEAN |  | Primary image |
| sort_order | INT |  | Image display order |
| created_at | DATETIME |  | Created date |

---

## 3.10 Product Variant Module

### Table: `product_variants`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Variant ID |
| product_id | BIGINT | FK | Product ID |
| name | VARCHAR(150) |  | Variant name |
| sku | VARCHAR(100) | UNIQUE | Variant SKU |
| price | DECIMAL(10,2) |  | Variant price |
| sale_price | DECIMAL(10,2) |  | Variant sale price |
| is_active | BOOLEAN |  | Variant status |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.11 Product Attribute Module

### Table: `attributes`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Attribute ID |
| name | VARCHAR(100) |  | Attribute name |
| created_at | DATETIME |  | Created date |

### Table: `attribute_values`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Attribute value ID |
| attribute_id | BIGINT | FK | Attribute ID |
| value | VARCHAR(100) |  | Attribute value |
| created_at | DATETIME |  | Created date |

### Table: `variant_attribute_values`

| Column | Type | Key | Description |
|---|---|---|---|
| variant_id | BIGINT | PK, FK | Variant ID |
| attribute_value_id | BIGINT | PK, FK | Attribute value ID |

---

## 3.12 Inventory Module

### Table: `inventories`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Inventory ID |
| product_id | BIGINT | FK | Product ID |
| variant_id | BIGINT | FK | Variant ID |
| quantity | INT |  | Available stock |
| reserved_quantity | INT |  | Reserved stock |
| low_stock_limit | INT |  | Low stock limit |
| updated_at | DATETIME |  | Updated date |

---

## 3.13 Stock Transaction Module

### Table: `stock_transactions`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Transaction ID |
| inventory_id | BIGINT | FK | Inventory ID |
| transaction_type | ENUM('STOCK_IN','STOCK_OUT','SALE','RETURN','ADJUSTMENT') |  | Stock transaction type |
| quantity | INT |  | Changed quantity |
| reference_type | VARCHAR(50) |  | Order or manual reference |
| reference_id | BIGINT |  | Reference ID |
| created_at | DATETIME |  | Created date |

---

## 3.14 Cart Module

### Table: `carts`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Cart ID |
| user_id | BIGINT | FK | Customer ID |
| store_id | BIGINT | FK | Store ID |
| status | ENUM('ACTIVE','ORDERED','ABANDONED') |  | Cart status |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

### Table: `cart_items`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Cart item ID |
| cart_id | BIGINT | FK | Cart ID |
| product_id | BIGINT | FK | Product ID |
| variant_id | BIGINT | FK | Variant ID |
| quantity | INT |  | Product quantity |
| unit_price | DECIMAL(10,2) |  | Product price |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.15 Wishlist Module

### Table: `wishlists`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Wishlist ID |
| user_id | BIGINT | FK, UNIQUE | Customer ID |
| created_at | DATETIME |  | Created date |

### Table: `wishlist_items`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Wishlist item ID |
| wishlist_id | BIGINT | FK | Wishlist ID |
| product_id | BIGINT | FK | Product ID |
| created_at | DATETIME |  | Created date |

---

## 3.16 Coupon Module

### Table: `coupons`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Coupon ID |
| store_id | BIGINT | FK | Store ID |
| code | VARCHAR(50) | UNIQUE | Coupon code |
| discount_type | ENUM('PERCENTAGE','FIXED') |  | Discount type |
| discount_value | DECIMAL(10,2) |  | Discount value |
| minimum_order_amount | DECIMAL(10,2) |  | Minimum order amount |
| maximum_discount | DECIMAL(10,2) |  | Maximum discount |
| start_date | DATETIME |  | Coupon start date |
| end_date | DATETIME |  | Coupon end date |
| usage_limit | INT |  | Maximum usage |
| is_active | BOOLEAN |  | Coupon status |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

### Table: `coupon_usages`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Coupon usage ID |
| coupon_id | BIGINT | FK | Coupon ID |
| user_id | BIGINT | FK | Customer ID |
| order_id | BIGINT | FK | Order ID |
| used_at | DATETIME |  | Used date |

---

## 3.17 Order Module

### Table: `orders`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Order ID |
| order_number | VARCHAR(50) | UNIQUE | Order number |
| user_id | BIGINT | FK | Customer ID |
| store_id | BIGINT | FK | Store ID |
| shipping_address_id | BIGINT | FK | Shipping address |
| billing_address_id | BIGINT | FK | Billing address |
| coupon_id | BIGINT | FK | Coupon ID |
| subtotal | DECIMAL(10,2) |  | Order subtotal |
| discount_amount | DECIMAL(10,2) |  | Discount amount |
| tax_amount | DECIMAL(10,2) |  | Tax amount |
| shipping_amount | DECIMAL(10,2) |  | Shipping amount |
| total_amount | DECIMAL(10,2) |  | Total amount |
| order_status | ENUM('PENDING','CONFIRMED','PROCESSING','SHIPPED','DELIVERED','CANCELLED') |  | Order status |
| payment_status | ENUM('PENDING','PAID','FAILED','REFUNDED') |  | Payment status |
| ordered_at | DATETIME |  | Order date |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

### Table: `order_items`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Order item ID |
| order_id | BIGINT | FK | Order ID |
| product_id | BIGINT | FK | Product ID |
| variant_id | BIGINT | FK | Variant ID |
| product_name | VARCHAR(200) |  | Product name snapshot |
| sku | VARCHAR(100) |  | SKU snapshot |
| quantity | INT |  | Quantity |
| unit_price | DECIMAL(10,2) |  | Unit price |
| total_price | DECIMAL(10,2) |  | Total price |
| created_at | DATETIME |  | Created date |

### Table: `order_status_history`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Status history ID |
| order_id | BIGINT | FK | Order ID |
| status | VARCHAR(50) |  | Order status |
| note | VARCHAR(500) |  | Status note |
| changed_by | BIGINT | FK | Changed by user ID |
| created_at | DATETIME |  | Created date |

---

## 3.18 Payment Module

### Table: `payments`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Payment ID |
| order_id | BIGINT | FK | Order ID |
| transaction_id | VARCHAR(255) | UNIQUE | Payment transaction ID |
| payment_method | ENUM('COD','CARD','UPI','NET_BANKING','WALLET') |  | Payment method |
| payment_status | ENUM('PENDING','SUCCESS','FAILED','REFUNDED') |  | Payment status |
| amount | DECIMAL(10,2) |  | Payment amount |
| gateway_response | JSON |  | Payment gateway response |
| paid_at | DATETIME |  | Payment date |
| created_at | DATETIME |  | Created date |

---

## 3.19 Refund Module

### Table: `refunds`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Refund ID |
| payment_id | BIGINT | FK | Payment ID |
| order_id | BIGINT | FK | Order ID |
| amount | DECIMAL(10,2) |  | Refund amount |
| reason | VARCHAR(500) |  | Refund reason |
| refund_status | ENUM('PENDING','PROCESSING','COMPLETED','FAILED') |  | Refund status |
| refunded_at | DATETIME |  | Refund date |
| created_at | DATETIME |  | Created date |

---

## 3.20 Shipping Module

### Table: `shipping_methods`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Shipping method ID |
| store_id | BIGINT | FK | Store ID |
| name | VARCHAR(100) |  | Shipping method name |
| charge | DECIMAL(10,2) |  | Shipping charge |
| estimated_days | INT |  | Estimated delivery days |
| is_active | BOOLEAN |  | Shipping method status |
| created_at | DATETIME |  | Created date |

### Table: `shipments`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Shipment ID |
| order_id | BIGINT | FK | Order ID |
| shipping_method_id | BIGINT | FK | Shipping method ID |
| tracking_number | VARCHAR(150) |  | Tracking number |
| courier_name | VARCHAR(100) |  | Courier company |
| shipment_status | ENUM('PENDING','SHIPPED','IN_TRANSIT','DELIVERED','FAILED') |  | Shipment status |
| shipped_at | DATETIME |  | Shipping date |
| delivered_at | DATETIME |  | Delivery date |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.21 Review Module

### Table: `reviews`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Review ID |
| user_id | BIGINT | FK | Customer ID |
| product_id | BIGINT | FK | Product ID |
| order_item_id | BIGINT | FK, UNIQUE | Purchased order item ID |
| rating | INT |  | Rating from 1 to 5 |
| comment | TEXT |  | Review comment |
| status | ENUM('PENDING','APPROVED','REJECTED') |  | Review status |
| created_at | DATETIME |  | Created date |
| updated_at | DATETIME |  | Updated date |

---

## 3.22 Notification Module

### Table: `notifications`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Notification ID |
| user_id | BIGINT | FK | User ID |
| title | VARCHAR(255) |  | Notification title |
| message | TEXT |  | Notification message |
| notification_type | ENUM('ORDER','PAYMENT','SHIPPING','GENERAL') |  | Notification type |
| is_read | BOOLEAN |  | Read status |
| created_at | DATETIME |  | Created date |

---

## 3.23 Media Module

### Table: `media_files`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Media file ID |
| uploaded_by | BIGINT | FK | Uploaded user ID |
| file_name | VARCHAR(255) |  | File name |
| file_url | VARCHAR(500) |  | File URL |
| file_type | VARCHAR(100) |  | File MIME type |
| file_size | BIGINT |  | File size |
| created_at | DATETIME |  | Created date |

---

## 3.24 Report Module

Report data can be generated from existing tables.

### Reports

- Sales report
- Order report
- Product report
- Inventory report
- Customer report
- Payment report

No separate table is required in the first version.

---

## 3.25 Audit Log Module

### Table: `audit_logs`

| Column | Type | Key | Description |
|---|---|---|---|
| id | BIGINT | PK | Audit log ID |
| user_id | BIGINT | FK | User who performed action |
| module_name | VARCHAR(100) |  | Module name |
| action_name | VARCHAR(100) |  | Action name |
| entity_type | VARCHAR(100) |  | Entity name |
| entity_id | BIGINT |  | Entity ID |
| old_data | JSON |  | Old data |
| new_data | JSON |  | New data |
| created_at | DATETIME |  | Created date |

---

# 4. Main Table Relationships

- One user can have multiple addresses.
- One seller can own one or multiple stores.
- One store can have multiple categories.
- One category can have multiple products.
- One product can have multiple images.
- One product can have multiple variants.
- One product variant can have multiple attribute values.
- One product or variant can have one inventory record.
- One customer can have one active cart per store.
- One cart can have multiple cart items.
- One customer can have one wishlist.
- One wishlist can have multiple products.
- One customer can place multiple orders.
- One order can have multiple order items.
- One order can have one or multiple payment attempts.
- One order can have one shipment.
- One payment can have one or multiple refunds.
- One product can have multiple reviews.
- One user can receive multiple notifications.
- One user can create multiple audit log records.

---

# 5. Recommended Spring Boot Package Structure

```text
com.bizcart
├── auth
├── user
├── admin
├── role
├── store
├── address
├── category
├── product
├── inventory
├── cart
├── wishlist
├── coupon
├── order
├── payment
├── refund
├── shipping
├── review
├── notification
├── media
├── report
├── audit
├── security
├── config
├── common
└── exception