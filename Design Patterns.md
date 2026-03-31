# Design Patterns — Uni-c System
**System:** Uni-c Embroidery Ordering System  
**Date:** 2026-03-31

> เอกสารนี้วิเคราะห์ **Design Patterns** ที่ใช้จริงในโค้ดของระบบ Uni-c  
> แบ่งตาม category: Creational, Structural, Behavioral, และ Architectural

---

## สรุป Patterns ที่พบ

| # | Pattern | Category | หมวดโค้ด |
|---|---|---|---|
| 1 | **Module Pattern** | Structural | ทั้งระบบ |
| 2 | **Singleton** | Creational | DB Connection |
| 3 | **Middleware Chain** | Behavioral | Express middleware |
| 4 | **Chain of Responsibility** | Behavioral | auth → adminOnly → handler |
| 5 | **Repository Pattern (light)** | Structural | Mongoose models |
| 6 | **Factory Pattern** | Creational | Order ID, Stripe PaymentIntent |
| 7 | **Strategy Pattern** | Behavioral | Login ด้วย username หรือ email |
| 8 | **Observer Pattern** | Behavioral | Stripe Webhook |
| 9 | **Proxy Pattern** | Structural | email-service microservice |
| 10 | **Facade Pattern** | Structural | `utils/email.js`, `utils/logger.js` |
| 11 | **Microservice Architecture** | Architectural | email-service แยกออกจาก API |
| 12 | **Token-Based Auth (Stateless)** | Security | JWT Cookie Pattern |

---

## 1. Module Pattern

**Category:** Structural  
**ตำแหน่งในโค้ด:** ทุกไฟล์ (Node.js CommonJS)

### อธิบาย
Node.js ใช้ระบบ `module.exports` / `require()` ซึ่งเป็น implementation ของ **Module Pattern** — ซ่อน implementation detail ภายในไฟล์ และ expose เฉพาะส่วนที่จำเป็น

```js
// database/db.js
const connect = async () => {       // ← private implementation
    await mongoose.connect(process.env.DB);
};
module.exports = connect;            // ← expose เฉพาะ function
```

```js
// middleware/auth.js
const auth = (...) => { ... };       // ← private
const adminOnly = (...) => { ... };  // ← private

module.exports = { auth, adminOnly }; // ← expose เฉพาะที่ต้องการ
```

**ประโยชน์:**  
- แต่ละไฟล์มี scope ของตัวเอง ป้องกัน global namespace pollution  
- ง่ายต่อการ unit test แต่ละ module แยกกัน

---

## 2. Singleton Pattern

**Category:** Creational  
**ตำแหน่งในโค้ด:** `database/db.js` + Node.js module cache

### อธิบาย
การ connect MongoDB เกิดขึ้น **ครั้งเดียว** ตอน startup ผ่าน:

```js
// index.js
require('./database/db')()  // เรียก connect() ครั้งเดียว
```

Node.js **cache** module ทุกตัวหลังจาก `require()` ครั้งแรก  
ดังนั้น Mongoose connection pool ที่สร้างขึ้นจะถูก **share** กับทุก route โดยอัตโนมัติ — ไม่มีการ connect ใหม่ทุก request

```
require('./database/db')() → mongoose.connect() → connection pool
         ↑                                              ↓
     ถูกเรียกครั้งเดียว              ทุก route ใช้ pool เดิม (Singleton)
```

**ประโยชน์:**  
- ประหยัด resource — ไม่สร้าง connection ใหม่ทุก request  
- Mongoose จัดการ connection pool ภายในอัตโนมัติ

---

## 3. Middleware Chain Pattern

**Category:** Behavioral  
**ตำแหน่งในโค้ด:** `index.js` — Express middleware stack

### อธิบาย
Express ใช้ **middleware pipeline** — request ไหลผ่าน middleware ทีละตัวตามลำดับ

```
Request
   │
   ▼
urlencoded parser       (parse form data)
   │
   ▼
/payment/webhook        (raw body — MUST be before json())
   │
   ▼
express.json()          (parse JSON body)
   │
   ▼
mongoSanitize()         (strip $, . from keys)
   │
   ▼
cookieParser()          (parse cookie header)
   │
   ▼
cors()                  (set CORS headers)
   │
   ▼
Route Handler           (business logic)
   │
   ▼
Response
```

**ทำไมลำดับสำคัญมาก:**  
Stripe webhook ต้องการ raw Buffer — ถ้า `express.json()` อยู่ก่อน มันจะ parse และทิ้ง buffer → signature verification จะล้มเหลว

---

## 4. Chain of Responsibility Pattern

**Category:** Behavioral  
**ตำแหน่งในโค้ด:** `middleware/auth.js` + route definitions

### อธิบาย
แต่ละ middleware ตัดสินใจว่าจะ **pass ต่อ** หรือ **หยุดและตอบกลับ** Express middleware ที่ chain กันคือ Chain of Responsibility โดยสมบูรณ์

```js
router.get('/order/all-orders', auth, adminOnly, handler);
//                               (1)   (2)        (3)
```

```
Request → auth middleware
              │
         ┌────┴────┐
         │ มี token? │
         └────┬────┘
           No │                  Yes
              ▼                   ▼
          401 Unauthorized    adminOnly middleware
          (chain หยุด)            │
                              ┌───┴───┐
                              │ Admin? │
                              └───┬───┘
                               No │        Yes
                                  ▼         ▼
                              403 Forbidden  handler()
                              (chain หยุด)   (business logic)
```

**ประโยชน์:**  
- แยก concern: auth ไม่รู้จัก business logic  
- Reusable: `auth` และ `adminOnly` ใช้ได้กับทุก route  
- เพิ่ม middleware ใหม่ (เช่น rate-limiter) ได้โดยไม่แก้ไข handler

---

## 5. Repository Pattern (Light)

**Category:** Structural  
**ตำแหน่งในโค้ด:** `database/Schema/*.js` — Mongoose models

### อธิบาย
Mongoose Model ทำหน้าที่เป็น **Repository** — เป็น abstraction layer ระหว่าง business logic และ database

```js
// routes ไม่รู้จัก MongoDB query syntax โดยตรง
// ใช้ผ่าน Mongoose model แทน

const User = require('../database/Schema/userCollection');

// แทนที่จะเขียน raw MongoDB:
// db.collection('users').findOne({ username: 'john' })

// ใช้ Mongoose repository method:
const user = await User.findOne({ username: req.user.username });
```

**Mongoose methods ที่ใช้เป็น Repository:**

| Method | หน้าที่ |
|---|---|
| `Model.findOne(query)` | ดึงข้อมูล 1 record |
| `Model.find(query)` | ดึงข้อมูลหลาย records |
| `Model.findById(id)` | ดึงโดย MongoDB _id |
| `Model.create(data)` | สร้าง record ใหม่ |
| `Model.findByIdAndUpdate()` | แก้ไขและ return |
| `Model.findOneAndDelete()` | ลบ |

**ประโยชน์:**  
- ถ้าเปลี่ยน database ในอนาคต แก้ไขแค่ Schema layer  
- Business logic ไม่ผูกกับ MongoDB syntax

---

## 6. Factory Pattern

**Category:** Creational  
**ตำแหน่งในโค้ด:** `orderCollection.js`, `routes/api/order/order.js`

### อธิบาย
**Factory** คือ pattern ที่ delegate การสร้าง object ให้ function/method อื่น แทนที่จะ instantiate โดยตรง

#### 6.1 Order ID Factory
```js
// database/Schema/orderCollection.js
const generateOrderId = () => 'ORD-' + crypto.randomBytes(3).toString('hex').toUpperCase();
//    ↑ Factory function สร้าง unique ID รูปแบบ ORD-XXXXXX

const OrderSchema = new mongoose.Schema({
    order_id: {
        default: generateOrderId  // ← เรียก factory อัตโนมัติทุกครั้งที่สร้าง Order
    },
    ...
});
```

#### 6.2 Stripe PaymentIntent Factory
```js
// routes/api/order/order.js
const paymentIntent = await stripe.paymentIntents.create({
    amount: total_price * 100,
    currency: 'thb',
    payment_method_types: ['card', 'promptpay'],
    metadata: { order_id: order._id.toString() },
});
//  ↑ Stripe SDK เป็น Factory — สร้าง PaymentIntent object
//    พร้อม client_secret ที่ frontend ใช้ render payment UI
```

**ประโยชน์:**  
- ซ่อน complexity ของการสร้าง ID และ Payment object  
- แก้รูปแบบ ID ที่จุดเดียว (generateOrderId) โดยไม่กระทบส่วนอื่น

---

## 7. Strategy Pattern

**Category:** Behavioral  
**ตำแหน่งในโค้ด:** `routes/api/auth/login.js`

### อธิบาย
ผู้ใช้ login ได้ทั้งด้วย **username** หรือ **email** — ระบบเลือก strategy การค้นหาตาม input

```js
// login.js
function isEmail(str) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(str);
}

// ── Strategy Selection ──────────────────────────────────
let query;
if (isEmail(username)) {
    // Strategy A: ค้นหาด้วย email (case-insensitive)
    query = { email: new RegExp('^' + username.trim() + '$', 'i') };
} else {
    // Strategy B: ค้นหาด้วย username (exact match)
    query = { username };
}

// ── Execute Strategy ─────────────────────────────────────
const data = await db.findOne(query);
```

```
Login Input
     │
     ▼
 isEmail()?
  /      \
Yes       No
 │         │
 ▼         ▼
Email    Username
Strategy  Strategy
  \       /
   \     /
    ▼   ▼
  findOne(query)
```

**ประโยชน์:**  
- เพิ่ม login strategy ใหม่ (เช่น phone number) ได้โดยไม่แก้ไข core logic  
- แต่ละ strategy ทดสอบแยกได้

---

## 8. Observer Pattern (Event-Driven)

**Category:** Behavioral  
**ตำแหน่งในโค้ด:** `routes/api/payment/payment.js` — Stripe Webhook

### อธิบาย
**Observer Pattern** คือ pattern ที่ object (Subject/Publisher) แจ้งเตือน subscribers เมื่อ state เปลี่ยน

ในระบบนี้ **Stripe** คือ Publisher — ระบบ Uni-c เป็น Subscriber:

```
[ผู้ใช้จ่ายเงิน]
      │
      ▼
  Stripe (Publisher)
      │
      │ HTTP POST /api/payment/webhook
      │ event: payment_intent.succeeded
      ▼
  Uni-c Backend (Subscriber/Observer)
      │
      ├── update Order status (pending → processing)
      └── send email notification to user
```

```js
// payment.js
router.post('/payment/webhook', express.raw(...), async (req, res) => {
    // 1. ตรวจ signature (verify ว่ามาจาก Stripe จริง)
    event = stripe.webhooks.constructEvent(req.body, sig, stripeWebhookSecret);

    // 2. Handle event (Observer reacts to notification)
    if (event.type === 'payment_intent.succeeded') {
        // อัปเดต DB
        order.status = 'processing';
        await order.save();
        // ส่ง email
        await sendEmail({ ... });
    }
    res.json({ received: true });
});
```

**ประโยชน์:**  
- **Decoupled** — Stripe ไม่รู้ว่า Uni-c จะทำอะไรกับ event  
- Idempotent — ถ้า webhook ยิงซ้ำ (order ไม่ใช่ pending) ระบบไม่ทำซ้ำ  
- ง่ายต่อการเพิ่ม handler สำหรับ event ใหม่ (เช่น `payment_intent.payment_failed`)

---

## 9. Proxy Pattern

**Category:** Structural  
**ตำแหน่งในโค้ด:** `utils/email.js`

### อธิบาย
**Proxy** คือ object ที่ทำหน้าที่แทน (represent) object จริง — ในที่นี้ `utils/email.js` เป็น Proxy สำหรับ email-service microservice

```
Route handler (ต้องการส่ง email)
        │
        │ await sendEmail({ to, subject, html })
        ▼
   utils/email.js  ← PROXY
        │
        │ HTTP POST /send-email
        │ + x-api-key header
        ▼
  email-service (Nodemailer)
        │
        └── ส่งจริงผ่าน SMTP
```

```js
// utils/email.js — Proxy hides the complexity
const sendEmail = async ({ to, subject, text, html }) => {
    // Route ไม่รู้ว่า email-service อยู่ที่ไหน หรือใช้ SMTP อะไร
    const response = await fetch(emailServiceUrl, {
        method: 'POST',
        headers: { 'x-api-key': apiKey },
        body: JSON.stringify({ to, subject, html: html || text }),
    });
    ...
};
```

**ประโยชน์:**  
- Route handler ไม่รู้จัก SMTP, Nodemailer, หรือ API key  
- เปลี่ยน email provider ได้โดยแก้แค่ Proxy ไฟล์เดียว  
- ถ้า email-service ล่ม → Proxy คืน `false` แทน throw error (ป้องกัน cascade failure)

---

## 10. Facade Pattern

**Category:** Structural  
**ตำแหน่งในโค้ด:** `utils/logger.js`, `utils/email.js`

### อธิบาย
**Facade** ซ่อน sub-system ที่ซับซ้อนไว้เบื้องหลัง interface ที่เรียบง่าย

#### `utils/logger.js` Facade
```js
// Complex sub-system: Mongoose Log model, error handling, field mapping
// ↓ ซ่อนไว้หมด — route เรียกแค่นี้:

await saveLog({
    action: 'LOGIN',
    detail: `Login successful: ${username}`,
    ip: req.ip,
    user: data._id,
    status: 'success'
});
// façade จัดการ Log.create(), error handling, default values ให้หมด
```

#### `utils/email.js` Facade
```js
// Complex sub-system: fetch(), headers, API key, error handling, HTML fallback
// ↓ route เรียกแค่นี้:

await sendEmail({ to: user.email, subject, html });
// facade จัดการ HTTP call ให้หมด
```

**ประโยชน์:**  
- ลด boilerplate ใน routes  
- ทุก route ส่ง log ในฟอร์แมตเดียวกัน  
- แก้ implementation ได้โดยไม่กระทบ routes

---

## 11. Microservice Architecture

**Category:** Architectural  
**ตำแหน่งในโค้ด:** `email-service/` แยกออกจาก `SE_Backend/`

### อธิบาย
ระบบแบ่งเป็น 2 independent services ที่สื่อสารกันผ่าน HTTP

```
SE_Backend (Main API)          email-service (Mail Microservice)
┌─────────────────────┐        ┌──────────────────────────────┐
│ - Auth              │        │ - SMTP configuration         │
│ - Orders            │──────► │ - Email template rendering   │
│ - Items             │  HTTP  │ - Nodemailer transport       │
│ - Payments          │        │ - API Key auth               │
└─────────────────────┘        └──────────────────────────────┘
```

**ทำไมแยก email-service ออกมา:**

| เหตุผล | คำอธิบาย |
|---|---|
| **Single Responsibility** | SE_Backend ไม่ต้องรู้จัก SMTP |
| **Isolated Credentials** | SMTP credentials อยู่ใน email-service เท่านั้น |
| **Independent Deploy** | อัปเดต email template โดยไม่ restart API |
| **Scalability** | Scale email-service แยกได้ถ้าปริมาณ email เพิ่ม |
| **Reusability** | service อื่นในอนาคตส่ง email ได้ผ่าน endpoint เดียวกัน |

---

## 12. Token-Based Stateless Auth

**Category:** Security / Architectural  
**ตำแหน่งในโค้ด:** `routes/api/auth/login.js`, `middleware/auth.js`

### อธิบาย
ระบบไม่เก็บ session ใน server (Stateless) — ใช้ **JWT** แทน

```
Login Success
     │
     ▼
jwt.sign({ username, role }, JWT_SECRET, { expiresIn: '1d' })
     │
     ▼
res.cookie('session', token, { httpOnly: false, sameSite: 'Lax' })
     │
     ▼
Browser เก็บ cookie อัตโนมัติ

─────────────────────────────────────────

ทุก Request ถัดไป:
     │
     ▼
Browser ส่ง cookie 'session' อัตโนมัติ
     │
     ▼
jwt.verify(token, JWT_SECRET) → { username, role }
     │
     ▼
req.user = decoded  → handler ใช้ได้เลย
```

**JWT Payload:**
```json
{
  "username": "john",
  "role": "Member",
  "iat": 1711900000,
  "exp": 1711986400
}
```

**ข้อดีของ Stateless:**  
- ไม่ต้องมี session store (Redis, DB)  
- Scale horizontally ได้ง่าย — ทุก server instance verify JWT ได้โดยไม่ sync กัน  
- Role (`Member` / `Admin`) อยู่ใน token — ไม่ต้อง query DB เพื่อ check role

---

## สรุปภาพรวม Pattern ในแต่ละ Layer

```
┌─────────────────────────────────────────────────────┐
│                    FRONTEND (Next.js)                │
└─────────────────────────────────────────────────────┘
                           │ REST + Cookie JWT
┌─────────────────────────────────────────────────────┐
│               EXPRESS API (SE_Backend)               │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Middleware Stack (Middleware Chain Pattern)  │  │
│  │  urlencoded → raw → json → sanitize →        │  │
│  │  cookieParser → cors → routes                │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Routes (Chain of Responsibility Pattern)    │  │
│  │  auth → adminOnly → handler                  │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌─────────────┐  ┌──────────────────────────────┐ │
│  │ utils/      │  │ routes/                      │ │
│  │ email.js    │  │ Strategy Pattern (login)     │ │
│  │ (Proxy +    │  │ Factory Pattern (order ID)   │ │
│  │  Facade)    │  │ Observer Pattern (webhook)   │ │
│  │             │  │                              │ │
│  │ logger.js   │  │                              │ │
│  │ (Facade)    │  │                              │ │
│  └─────────────┘  └──────────────────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Database Layer (Repository Pattern)         │  │
│  │  Mongoose Models: User, Order, Item, Log     │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
┌─────────────────────┐    ┌────────────────────────┐
│   MongoDB Atlas     │    │  email-service          │
│   (Singleton conn.) │    │  (Microservice Pattern) │
└─────────────────────┘    └────────────────────────┘
```
