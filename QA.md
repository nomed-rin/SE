# QA Test Cases — Uni-c Backend API
**System:** Uni-c Embroidery Ordering System  
**Base URL (Dev):** `http://localhost:8080/api`  
**Base URL (Prod):** `https://api.rinthecat.pro/api`  
**Date:** 2026-03-31

---

## สารบัญ

| หมวด | จำนวน Test Case |
|---|---|
| [TC-AUTH] Authentication | 28 |
| [TC-ORD] Order | 16 |
| [TC-ITEM] Item | 18 |
| [TC-PAY] Payment Webhook | 4 |
| **รวม** | **66** |

---

## ตำนาน

| สัญลักษณ์ | ความหมาย |
|---|---|
| ✅ | Expected: Pass |
| ❌ | Expected: Fail (ตรวจสอบว่า error ถูกต้อง) |
| 🔒 | ต้องมี JWT session cookie |
| 👑 | ต้องเป็น Admin |

---

## [TC-AUTH] Authentication

### POST `/login`

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| AUTH-001 | Login สำเร็จด้วย username | `{ username: "john", password: "P@ss123!" }` | 200 | `{ success: true, statusCode: 201 }` + cookie `session` | ☐ |
| AUTH-002 | Login สำเร็จด้วย email | `{ username: "john@email.com", password: "P@ss123!" }` | 200 | `{ success: true, statusCode: 201 }` + cookie `session` | ☐ |
| AUTH-003 | ❌ ไม่ส่ง password | `{ username: "john" }` | 400 | `{ message: "require username and password" }` | ☐ |
| AUTH-004 | ❌ ไม่ส่ง username | `{ password: "P@ss123!" }` | 400 | `{ message: "require username and password" }` | ☐ |
| AUTH-005 | ❌ username ไม่มีในระบบ | `{ username: "ghost", password: "P@ss123!" }` | 401 | `{ message: "Username not found" }` | ☐ |
| AUTH-006 | ❌ password ผิด | `{ username: "john", password: "wrongpass" }` | 401 | `{ message: "password is Incorrect" }` | ☐ |
| AUTH-007 | ❌ body ว่างเปล่า | `{}` | 400 | `{ success: false }` | ☐ |
| AUTH-008 | ✅ ตรวจ cookie ที่ได้รับ | login สำเร็จ | - | cookie ชื่อ `session`, maxAge = 86400000ms | ☐ |

---

### POST `/register`

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| AUTH-009 | ✅ สมัครสมาชิกสำเร็จ | `{ username, password: "P@ss123!", confirm_password: "P@ss123!", email, phone_number: "0812345678" }` | 200 | `{ success: true, statusCode: 201 }` + cookie `session` | ☐ |
| AUTH-010 | ❌ password ไม่ตรงกับ confirm | `{ password: "P@ss123!", confirm_password: "WRONG!" }` | 400 | `{ message: "Password and confirm password do not match" }` | ☐ |
| AUTH-011 | ❌ email ผิด format | `{ email: "notanemail" }` | 400 | `{ message: "Invalid email format" }` | ☐ |
| AUTH-012 | ❌ phone ผิด format | `{ phone_number: "abc" }` | 400 | `{ message: "Invalid phone number format" }` | ☐ |
| AUTH-013 | ❌ password อ่อนเกินไป (ไม่มีตัวเลข) | `{ password: "password!" }` | 400 | `{ message: "Password must be at least 8 characters..." }` | ☐ |
| AUTH-014 | ❌ password สั้นเกินไป (< 8 ตัว) | `{ password: "P@1" }` | 400 | `{ message: "Password must be at least 8 characters..." }` | ☐ |
| AUTH-015 | ❌ username ซ้ำ | ส่ง username ที่มีอยู่แล้ว | 401 | `{ message: "username is already use" }` | ☐ |
| AUTH-016 | ❌ email ซ้ำ | ส่ง email ที่มีอยู่แล้ว | 401 | `{ message: "email is already use" }` | ☐ |
| AUTH-017 | ❌ phone ซ้ำ | ส่ง phone ที่มีอยู่แล้ว | 401 | `{ message: "phone number is already use" }` | ☐ |
| AUTH-018 | ❌ ขาด field บังคับ | `{ username: "test" }` (ขาด password, phone) | 400 | `{ message: "require username, password, and phone number" }` | ☐ |

---

### GET `/auth/me` 🔒

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| AUTH-019 | ✅ ดึงข้อมูล user ตัวเอง | cookie `session` ถูกต้อง | 200 | `{ success: true, data: { username, email, role, ... } }` ไม่มี `password`, `reset_token` | ☐ |
| AUTH-020 | ❌ ไม่ส่ง cookie | ไม่มี cookie | 401 | `{ message: "Access denied. No token provided." }` | ☐ |
| AUTH-021 | ❌ cookie หมดอายุ / token ปลอม | session = "invalid_token" | 401 | `{ message: "Invalid or expired token." }` | ☐ |

---

### GET `/auth/all-users` 🔒 👑

| ID | Test Case | Expected Status | Expected Body | ผล |
|---|---|---|---|---|
| AUTH-022 | ✅ Admin ดูรายชื่อ users ทั้งหมด | 200 | `{ success: true, data: [...] }` | ☐ |
| AUTH-023 | ❌ Member พยายามเข้า | 403 | `{ message: "Access denied. Admin only." }` | ☐ |

---

### PUT `/auth/profile` 🔒

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| AUTH-024 | ✅ แก้ไข username สำเร็จ | `{ username: "newname" }` | 200 | `{ success: true, updated: ["username"] }` | ☐ |
| AUTH-025 | ❌ username สั้นเกิน (< 3 ตัว) | `{ username: "ab" }` | 400 | `{ message: "Username must be 3–30 characters" }` | ☐ |
| AUTH-026 | ❌ ไม่มีการเปลี่ยนแปลง | ส่ง field เดิมทุกอย่าง | 400 | `{ message: "No changes provided" }` | ☐ |

---

### PUT `/auth/change-password` 🔒

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| AUTH-027 | ✅ เปลี่ยน password สำเร็จ | `{ current_password: "Old@123!", new_password: "New@456!" }` | 200 | `{ success: true }` | ☐ |
| AUTH-028 | ❌ current_password ผิด | `{ current_password: "wrong", new_password: "New@456!" }` | 400 | `{ message: "Current password is incorrect" }` | ☐ |
| AUTH-029 | ❌ new_password เหมือน current | `{ current_password: "Same@123!", new_password: "Same@123!" }` | 400 | `{ message: "New password must be different..." }` | ☐ |
| AUTH-030 | ❌ new_password สั้นเกิน | `{ new_password: "abc" }` | 400 | `{ message: "New password must be at least 6 characters" }` | ☐ |

---

### POST `/auth/forgot-password`

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| AUTH-031 | ✅ ส่ง reset link สำเร็จ | `{ email: "john@email.com" }` (email ที่มีในระบบ) | 200 | `{ success: true, message: "If that email is registered..." }` | ☐ |
| AUTH-032 | ✅ email ไม่มีในระบบ (anti-enumeration) | `{ email: "ghost@email.com" }` | 200 | `{ success: true, message: "If that email is registered..." }` (เหมือนกัน!) | ☐ |
| AUTH-033 | ❌ ไม่ส่ง email | `{}` | 400 | `{ message: "Email is required" }` | ☐ |
| AUTH-034 | ✅ ตรวจ reset_token ใน DB | หลัง forgot-password สำเร็จ | - | `user.reset_token` ไม่เป็น null, `user.reset_token_expires` อยู่ในอนาคต | ☐ |

---

### POST `/auth/reset-password/:token`

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| AUTH-035 | ✅ reset password สำเร็จ | token จาก email + `{ password: "NewP@ss123!" }` | 200 | `{ success: true, message: "Password reset successfully..." }` | ☐ |
| AUTH-036 | ❌ token ปลอม | token = "fakefakefake" | 400 | `{ message: "Invalid or expired reset token" }` | ☐ |
| AUTH-037 | ❌ token หมดอายุ (> 15 นาที) | ใช้ token หลังจาก 15 นาที | 400 | `{ message: "Invalid or expired reset token" }` | ☐ |
| AUTH-038 | ❌ password สั้นเกิน | `{ password: "abc" }` | 400 | `{ message: "Password must be at least 6 characters" }` | ☐ |
| AUTH-039 | ✅ ตรวจว่า reset_token ถูกล้างหลัง reset | หลัง reset สำเร็จ | - | `user.reset_token === null`, `user.reset_token_expires === null` | ☐ |

---

## [TC-ORD] Order

### POST `/order/create` 🔒

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| ORD-001 | ✅ สร้าง order สำเร็จ | `{ product_type: "student_shirt", items: [{ prefix: "นาย", name: "สมชาย", surname: "ใจดี", quantity: 2 }] }` | 201 | `{ success: true, order_id, total_price, client_secret }` | ☐ |
| ORD-002 | ❌ ไม่ส่ง product_type | `{ items: [...] }` | 400 | `{ message: "product_type and items[] are required" }` | ☐ |
| ORD-003 | ❌ items เป็น array ว่าง | `{ product_type: "student_shirt", items: [] }` | 400 | `{ message: "product_type and items[] are required" }` | ☐ |
| ORD-004 | ❌ product_type ไม่ถูกต้อง | `{ product_type: "fake_type" }` | 400 | `{ message: "Invalid product_type" }` | ☐ |
| ORD-005 | ❌ prefix ไม่ถูกต้อง | `{ items: [{ prefix: "Mr.", ... }] }` | 400 | `{ message: "Invalid prefix: Mr." }` | ☐ |
| ORD-006 | ❌ quantity < 1 | `{ items: [{ quantity: 0, ... }] }` | 400 | `{ message: "Quantity must be at least 1" }` | ☐ |
| ORD-007 | ❌ ไม่มี session | ไม่ส่ง cookie | 401 | `{ message: "Access denied. No token provided." }` | ☐ |
| ORD-008 | ✅ ตรวจ total_price = base_price × quantity | student_shirt base_price = 150, quantity = 3 | 201 | `total_price === 450` | ☐ |

---

### GET `/order/my-orders` 🔒

| ID | Test Case | Expected Status | Expected Body | ผล |
|---|---|---|---|---|
| ORD-009 | ✅ ดู orders ของตัวเอง | 200 | `{ success: true, data: [...] }` มีเฉพาะ orders ของ user นั้น | ☐ |
| ORD-010 | ❌ ไม่มี session | 401 | access denied | ☐ |

---

### GET `/order/all-orders` 🔒 👑

| ID | Test Case | Expected Status | Expected Body | ผล |
|---|---|---|---|---|
| ORD-011 | ✅ Admin ดู orders ทั้งหมด | 200 | `{ data: [...] }` แต่ละ order มี user (username, email, phone) | ☐ |
| ORD-012 | ❌ Member เข้า endpoint นี้ | 403 | `{ message: "Access denied. Admin only." }` | ☐ |

---

### GET `/order/:id` 🔒

| ID | Test Case | Expected Status | Expected Body | ผล |
|---|---|---|---|---|
| ORD-013 | ✅ เจ้าของดู order ตัวเอง | 200 | `{ success: true, data: { order } }` | ☐ |
| ORD-014 | ✅ Admin ดู order ของคนอื่น | 200 | ดูได้ทุก order | ☐ |
| ORD-015 | ❌ Member ดู order ของคนอื่น | 403 | `{ message: "Access denied" }` | ☐ |
| ORD-016 | ❌ id ไม่มีในระบบ | 404 | `{ message: "Order not found" }` | ☐ |

---

### PUT `/order/:id/status` 🔒 👑

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| ORD-017 | ✅ Admin เปลี่ยน status สำเร็จ | `{ status: "processing" }` | 200 | `{ success: true, data: { order } }` + ส่ง email แจ้ง user | ☐ |
| ORD-018 | ❌ status ไม่ถูกต้อง | `{ status: "cancelled" }` | 400 | `{ message: "Invalid or missing status" }` | ☐ |
| ORD-019 | ❌ Member พยายามเปลี่ยน status | - | 403 | `{ message: "Access denied. Admin only." }` | ☐ |
| ORD-020 | ❌ order id ไม่มีในระบบ | valid status แต่ id ไม่มี | 404 | `{ message: "Order not found" }` | ☐ |

---

## [TC-ITEM] Item

### GET `/items` (Public)

| ID | Test Case | Expected Status | Expected Body | ผล |
|---|---|---|---|---|
| ITEM-001 | ✅ ดูสินค้าที่ active ทั้งหมด | 200 | `{ success: true, data: [...] }` เฉพาะ `is_active: true` | ☐ |
| ITEM-002 | ✅ ไม่ต้องมี session | 200 | - | ☐ |

---

### GET `/items/all` 🔒 👑

| ID | Test Case | Expected Status | Expected Body | ผล |
|---|---|---|---|---|
| ITEM-003 | ✅ Admin ดูสินค้าทั้งหมด (รวม inactive) | 200 | `{ data: [...] }` มีทั้ง active และ inactive | ☐ |
| ITEM-004 | ❌ Member เข้า endpoint นี้ | 403 | access denied | ☐ |

---

### GET `/items/:product_type` (Public)

| ID | Test Case | Expected Status | Expected Body | ผล |
|---|---|---|---|---|
| ITEM-005 | ✅ ดูสินค้าเดี่ยว | `/items/student_shirt` | 200 | `{ success: true, data: { ... } }` | ☐ |
| ITEM-006 | ❌ product_type ไม่มีในระบบ | `/items/fake_type` | 404 | `{ message: "Item not found" }` | ☐ |

---

### POST `/items` 🔒 👑

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| ITEM-007 | ✅ Admin สร้างสินค้าใหม่ | `{ product_type: "scout_primary", name_th: "ลูกเสือ", base_price: 200 }` | 201 | `{ success: true, data: { ... } }` | ☐ |
| ITEM-008 | ❌ ขาด product_type | `{ name_th: "test", base_price: 100 }` | 400 | `{ message: "product_type, name_th, and base_price are required" }` | ☐ |
| ITEM-009 | ❌ product_type ซ้ำ | ใช้ product_type ที่มีอยู่แล้ว | 400 | `{ message: "Item with product_type '...' already exists" }` | ☐ |
| ITEM-010 | ❌ Member สร้างสินค้า | - | 403 | access denied | ☐ |
| ITEM-011 | ✅ สร้างพร้อม sizes และ embroidery_options | `{ ..., sizes: [...], embroidery_options: [...] }` | 201 | `{ data: { sizes, embroidery_options } }` | ☐ |

---

### PUT `/items/:product_type` 🔒 👑

| ID | Test Case | Input | Expected Status | Expected Body | ผล |
|---|---|---|---|---|---|
| ITEM-012 | ✅ Admin แก้ราคา | `{ base_price: 180 }` | 200 | `{ success: true, data: { base_price: 180 } }` | ☐ |
| ITEM-013 | ✅ ปิดการขาย (is_active: false) | `{ is_active: false }` | 200 | `{ data: { is_active: false } }` | ☐ |
| ITEM-014 | ❌ product_type ไม่มีในระบบ | PUT `/items/fake_type` | 404 | `{ message: "Item not found" }` | ☐ |
| ITEM-015 | ❌ Member แก้สินค้า | - | 403 | access denied | ☐ |

---

### DELETE `/items/:product_type` 🔒 👑

| ID | Test Case | Expected Status | Expected Body | ผล |
|---|---|---|---|---|
| ITEM-016 | ✅ Admin ลบสินค้า | 200 | `{ success: true, message: "Item deleted" }` | ☐ |
| ITEM-017 | ❌ product_type ไม่มีในระบบ | 404 | `{ message: "Item not found" }` | ☐ |
| ITEM-018 | ❌ Member ลบสินค้า | 403 | access denied | ☐ |

---

## [TC-PAY] Payment Webhook

> [!IMPORTANT]
> ทดสอบ Webhook ต้องใช้ `stripe listen --forward-to localhost:8080/api/payment/webhook`  
> และต้องส่ง raw body ไม่ใช่ JSON parser

| ID | Test Case | Event | Expected Behavior | ผล |
|---|---|---|---|---|
| PAY-001 | ✅ payment_intent.succeeded — pending order | ส่ง Stripe event `payment_intent.succeeded` พร้อม `metadata.order_id` ที่ status = pending | Order status เปลี่ยนเป็น `processing` + ส่ง email แจ้ง user | ☐ |
| PAY-002 | ✅ payment_intent.succeeded — order ไม่ใช่ pending | order status = processing อยู่แล้ว | webhook ตอบ 200 แต่ไม่เปลี่ยน status (idempotent) | ☐ |
| PAY-003 | ❌ Stripe signature ปลอม | ไม่มี / แก้ไข `stripe-signature` header | 400 `Webhook Error: ...` | ☐ |
| PAY-004 | ✅ Stripe ส่ง event อื่น (ไม่ใช่ succeeded) | เช่น `payment_intent.created` | 200 `{ received: true }` — ไม่ทำอะไร | ☐ |

---

## การทดสอบ Security

| ID | Test Case | Expected | ผล |
|---|---|---|---|
| SEC-001 | NoSQL Injection — ส่ง `{ "username": { "$gt": "" } }` | ตัวแปรถูก sanitize → 401 หรือ 400 (ไม่ bypass auth) | ☐ |
| SEC-002 | JWT Tampering — แก้ payload แล้ว encode ใหม่ | 401 Invalid token | ☐ |
| SEC-003 | ตรวจ password ใน DB ต้องเป็น bcrypt hash | hash ต้องขึ้นต้นด้วย `$2b$` | ☐ |
| SEC-004 | reset_token ใน DB ต้องเป็น SHA-256 hash (ไม่ใช่ raw token) | token ใน DB ≠ token ใน email link | ☐ |
| SEC-005 | Forgot-password ส่ง email ที่ไม่มีในระบบ → response ต้องเหมือนกัน | ทั้งสอง case ได้ 200 response body เหมือนกัน | ☐ |
| SEC-006 | email-service: เรียก `/send-email` โดยไม่มี `x-api-key` | 401 Unauthorized | ☐ |
| SEC-007 | ตรวจไม่มี password / reset_token ใน response ของ `/auth/me` | response ไม่มี field `password`, `reset_token`, `reset_token_expires` | ☐ |

---

## Regression Checklist (ตรวจทุกครั้ง deploy)

- [ ] POST `/login` — login ด้วย username ได้ปกติ
- [ ] POST `/login` — login ด้วย email ได้ปกติ
- [ ] POST `/register` — สมัครสมาชิกใหม่ได้ + ได้ cookie
- [ ] GET `/auth/me` — ได้ข้อมูล user หลัง login
- [ ] POST `/order/create` — สร้าง order + ได้ `client_secret`
- [ ] GET `/order/my-orders` — เห็นเฉพาะ orders ของตัวเอง
- [ ] GET `/items` — ดู catalogue ได้โดยไม่ต้อง login
- [ ] Stripe Webhook — payment สำเร็จ → order เป็น processing
- [ ] Email — ได้รับ email เมื่อ order status เปลี่ยน

---

## Environment ที่ใช้ทดสอบ

| ตัวแปร | Dev | Prod |
|---|---|---|
| `STRIPE_MODE` | `test` | `prod` |
| Base URL | `http://localhost:8080/api` | `https://api.rinthecat.pro/api` |
| MongoDB | Atlas (Dev cluster) | Atlas (Prod cluster) |
| Email | cmail.rinthecat.pro | cmail.rinthecat.pro |

> [!WARNING]
> **ห้ามทดสอบ Stripe Live keys บน Dev environment** — ใช้ test keys เสมอในขั้นตอนพัฒนา
