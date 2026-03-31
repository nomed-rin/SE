# Risk Management — Uni-c System
**System:** Uni-c Embroidery Ordering System  
**Date:** 2026-03-31  
**Version:** 1.0

---

## ระดับความเสี่ยง (Risk Scale)

| ระดับ | Likelihood (โอกาสเกิด) | Impact (ผลกระทบ) | Severity Score |
|---|---|---|---|
| 1 — ต่ำมาก | < 10% | ผลกระทบเล็กน้อย | 1–3 |
| 2 — ต่ำ | 10–30% | กระทบบางส่วน | 4–6 |
| 3 — ปานกลาง | 30–50% | กระทบหลายส่วน | 7–9 |
| 4 — สูง | 50–70% | กระทบทั้งระบบ | 10–14 |
| 5 — สูงมาก | > 70% | ระบบล่มทั้งหมด | 15–25 |

> **Severity Score = Likelihood × Impact**

```
         │        IMPACT
         │  1    2    3    4    5
    ─────┼──────────────────────────
    L  5 │  5   10   15   20   25
    I  4 │  4    8   12   16   20
    K  3 │  3    6    9   12   15
    E  2 │  2    4    6    8   10
    L  1 │  1    2    3    4    5
    I    │
    H    │  🟢=1-5  🟡=6-9  🟠=10-14  🔴=15-25
    O    │
    O    │
    D    │
```

---

## สารบัญ Risk

| หมวด | จำนวน Risk |
|---|---|
| [SEC] Security Risks | 7 |
| [TECH] Technical Risks | 7 |
| [OPS] Operational Risks | 3 |
| [BIZ] Business Risks | 3 |
| **รวม** | **20** |

---

## [SEC] Security Risks

### SEC-001 — JWT Token ถูก Steal (Cookie Hijacking)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ผู้ไม่หวังดีขโมย `session` cookie แล้ว impersonate เป็น user หรือ admin |
| **สาเหตุ** | Cookie ตั้งค่า `httpOnly: false` และ `secure: false` ทำให้ JavaScript และ HTTP ธรรมดาเข้าถึง cookie ได้ |
| **Likelihood** | 3 (ปานกลาง) |
| **Impact** | 5 (สูงมาก — admin cookie ถูกขโมย = ควบคุมระบบได้ทั้งหมด) |
| **Severity** | 🔴 **15** |
| **หลักฐานในโค้ด** | `res.cookie('session', session, { httpOnly: false, secure: false, ... })` |

**Mitigation (ป้องกัน):**
- เปลี่ยน `httpOnly: true` — JavaScript ไม่สามารถอ่าน cookie ได้ (ป้องกัน XSS)
- เปลี่ยน `secure: true` บน production — ส่งผ่าน HTTPS เท่านั้น

**Contingency (แผนสำรอง):**
- เพิ่ม JWT blacklist หรือ token version ใน DB เพื่อ revoke token ที่ถูก compromise ได้ทันที

---

### SEC-002 — NoSQL Injection (Partial Protection)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ผู้ไม่หวังดีส่ง MongoDB operator (`$where`, `$regex`) ใน request body เพื่อ bypass authentication |
| **สาเหตุ** | Custom sanitizer ล้างเฉพาะ `req.body` และ `req.params` — **ไม่ครอบคลุม `req.query`** |
| **Likelihood** | 2 (ต่ำ) |
| **Impact** | 5 (สูงมาก) |
| **Severity** | 🔴 **10** |
| **หลักฐานในโค้ด** | `sanitizeObject(req.body); sanitizeObject(req.params);` — ขาด `req.query` |

**Mitigation:**
- เพิ่ม `sanitizeObject(req.query)` ใน custom sanitizer
- ใช้ Mongoose strict schema — ไม่ยอมรับ field ที่ไม่อยู่ใน Schema โดย default

**Contingency:**
- ตรวจ MongoDB Atlas logs สำหรับ query ที่ผิดปกติ
- Rate-limit endpoint ที่ sensitive เช่น `/login`

---

### SEC-003 — Brute Force Attack บน Login

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ผู้ไม่หวังดีลอง password ซ้ำหลาย 1000 ครั้งจนเดาถูก |
| **สาเหตุ** | ไม่มี rate limiting บน `/login` endpoint |
| **Likelihood** | 4 (สูง) |
| **Impact** | 4 (สูง) |
| **Severity** | 🔴 **16** |
| **หลักฐานในโค้ด** | ไม่มี rate-limiter ใน `index.js` หรือ `login.js` |

**Mitigation:**
- ติดตั้ง `express-rate-limit` บน `/login` และ `/register`
- เพิ่ม account lockout หลังจากผิด N ครั้ง (เช่น 5 ครั้ง → lock 15 นาที)
- เพิ่ม CAPTCHA บน frontend

**Contingency:**
- Audit logs บันทึก IP ที่ login ผิดซ้ำ → block อัตโนมัติผ่าน firewall/nginx

---

### SEC-004 — Password Reset Token Timing Attack

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ไม่มีจำกัดจำนวนครั้งในการส่ง reset email → ผู้ไม่หวังดีส่ง request ซ้ำต่อเนื่อง (email flooding) |
| **สาเหตุ** | `/auth/forgot-password` ไม่มี rate limit |
| **Likelihood** | 3 (ปานกลาง) |
| **Impact** | 2 (ต่ำ — spam email, ไม่ได้โจมตีระบบโดยตรง) |
| **Severity** | 🟡 **6** |

**Mitigation:**
- Rate limit `/auth/forgot-password` — เช่น 3 requests ต่อ IP ต่อชั่วโมง
- ตรวจสอบว่า reset token เก่าถูก invalidate เมื่อขอใหม่

**Contingency:**
- Monitor email-service สำหรับ spike ในการส่ง email

---

### SEC-005 — CORS ตั้งค่า `origin: "*"` (Too Permissive)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ทุก domain สามารถเรียก API ได้ — เปิดโอกาสให้ CSRF หรือ unauthorized access จากต่างโดเมน |
| **สาเหตุ** | `cors({ origin: "*", credentials: true })` |
| **Likelihood** | 2 (ต่ำ) |
| **Impact** | 3 (ปานกลาง) |
| **Severity** | 🟡 **6** |
| **หลักฐานในโค้ด** | `app.use(cors({ origin: "*", credentials: true }))` |

**Mitigation:**
- เปลี่ยน `origin` เป็น whitelist: `["https://uni-c.rinthecat.pro", "http://localhost:3000"]`
- note: `credentials: true` + `origin: "*"` จริงๆ จะไม่ work ใน browser — แต่ควรแก้ให้ถูกต้อง

---

### SEC-006 — Stripe Secret Key ใน Environment (Exposure Risk)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | `.ENV` file ถูก commit ขึ้น Git → Stripe secret key หลุด |
| **สาเหตุ** | ชื่อไฟล์ `.ENV` (ตัวใหญ่) อาจไม่ถูก match โดย `.gitignore` ที่ระบุ `.env` (ตัวเล็ก) บน case-sensitive OS |
| **Likelihood** | 3 (ปานกลาง — เคยเกิดขึ้นแล้วใน project นี้ตามประวัติ conversations) |
| **Impact** | 5 (สูงมาก — Stripe key หลุด = ถูกชาร์จเงินได้) |
| **Severity** | 🔴 **15** |
| **หลักฐานในโค้ด** | ไฟล์ชื่อ `.ENV` (ตัวใหญ่) อยู่ใน `SE_Backend/` |

**Mitigation:**
- เปลี่ยนชื่อไฟล์เป็น `.env` (ตัวเล็ก) ให้ตรงกับ `.gitignore`
- ใช้ `git secret scan` หรือ GitHub secret scanning
- Rotate Stripe keys ทันทีถ้าสงสัยว่าหลุด

**Contingency:**
- Revoke Stripe keys ทันทีผ่าน Stripe Dashboard
- ตรวจสอบ Stripe logs สำหรับ unauthorized charges

---

### SEC-007 — Admin Role อยู่ใน JWT (Not DB-Verified)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | Role ใน JWT ไม่ถูก verify กับ DB ทุก request — ถ้า Admin ถูก downgrade ใน DB แต่ token ยังไม่หมดอายุ ยังคง access admin endpoint ได้ |
| **Likelihood** | 1 (ต่ำมาก — ต้องเป็น edge case จงใจ) |
| **Impact** | 4 (สูง) |
| **Severity** | 🟠 **4** |

**Mitigation:**
- สำหรับ endpoint ที่ sensitive มาก ให้ query DB verify role ทุกครั้ง
- ลดอายุ JWT สำหรับ admin users (เช่น 4 ชั่วโมงแทน 24 ชั่วโมง)

---

## [TECH] Technical Risks

### TECH-001 — MongoDB Connection ล้มเหลว (No Retry Logic)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ถ้า MongoDB Atlas ไม่พร้อมตอน startup → `connect()` fail แต่ server ยังคง start → ทุก request จะ error 500 |
| **สาเหตุ** | `db.js` catch error แล้วแค่ `console.log` — ไม่มี retry, ไม่มี process exit |
| **Likelihood** | 2 (ต่ำ) |
| **Impact** | 5 (สูงมาก — ระบบไม่ทำงานทั้งหมด) |
| **Severity** | 🔴 **10** |
| **หลักฐานในโค้ด** | `catch (err) { console.log(err.message); }` ใน `db.js` |

**Mitigation:**
```js
// แนะนำให้แก้ db.js เป็น:
const connect = async () => {
    try {
        await mongoose.connect(process.env.DB);
        console.log("Connected MongoDB");
    } catch (err) {
        console.error("MongoDB connection failed:", err.message);
        process.exit(1); // ← หยุด server ทันที (Docker จะ restart)
    }
};
```

**Contingency:**
- ตั้ง Docker `restart: always` — container จะ restart อัตโนมัติ
- เพิ่ม MongoDB Atlas connection retry ผ่าน Mongoose options

---

### TECH-002 — Dockerfile EXPOSE Port ไม่ตรงกับ App Port

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | `EXPOSE 4040` ใน Dockerfile แต่ app ใช้ `PORT || 8080` → healthcheck และ port mapping อาจผิดพลาด |
| **Likelihood** | 4 (สูง — เกิดขึ้นจริงได้ง่าย) |
| **Impact** | 3 (ปานกลาง — deploy แล้ว external access ไม่ได้) |
| **Severity** | 🟠 **12** |
| **หลักฐานในโค้ด** | `EXPOSE 4040` แต่ `const PORT = process.env.PORT \|\| 8080` |

**Mitigation:**
- ตั้ง `PORT=4040` ใน `.env` หรือ `docker-compose.yml` ให้ตรงกับ `EXPOSE`
- หรือแก้ `EXPOSE 5050` ให้ตรง production port ที่ใช้จริง

---

### TECH-003 — ไม่มี Input Validation บน Order Items (ความยาว String)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ผู้ใช้ส่ง `name` หรือ `surname` ที่ยาวมาก (เช่น 10,000 ตัวอักษร) ทำให้ DB เก็บข้อมูลขยะ หรือ cause performance issue |
| **Likelihood** | 2 (ต่ำ) |
| **Impact** | 2 (ต่ำ) |
| **Severity** | 🟢 **4** |

**Mitigation:**
- เพิ่ม `maxlength` บน Mongoose Schema สำหรับ `name`, `surname`, `note`
- Validate length ใน route handler ก่อน save

---

### TECH-004 — email-service ล่ม → Order Status Update ยังคงสำเร็จ (Silent Failure)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | Admin เปลี่ยน order status สำเร็จ แต่ email ไม่ถูกส่ง → ผู้ใช้ไม่รู้ว่า order เปลี่ยน status |
| **สาเหตุ** | `sendEmail()` คืน false แต่ `order.js` ไม่ตรวจสอบ return value |
| **Likelihood** | 3 (ปานกลาง — email-service อาจ down เป็นครั้งคราว) |
| **Impact** | 2 (ต่ำ — ข้อมูลถูกต้อง แค่ไม่ได้รับ notification) |
| **Severity** | 🟡 **6** |

**Mitigation:**
- ตรวจสอบ return value ของ `sendEmail()` และ log warning ถ้าส่งไม่สำเร็จ
- เพิ่ม email retry queue (เช่น Bull, Agenda) สำหรับ critical notifications

**Contingency:**
- ผู้ใช้ตรวจสอบ status ได้ผ่าน `/order/my-orders` ตลอดเวลา

---

### TECH-005 — Stripe Webhook Replay Attack

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | Stripe ส่ง webhook ซ้ำ (retry) → order ถูก process ซ้ำหลายครั้ง |
| **สาเหตุ** | Stripe ส่ง webhook ซ้ำถ้าไม่ได้รับ 200 response ใน time limit — มีความเป็นไปได้ที่จะ process ซ้ำ |
| **Likelihood** | 2 (ต่ำ) |
| **Impact** | 2 (ต่ำ — มี idempotency check บางส่วน) |
| **Severity** | 🟢 **4** |
| **หลักฐานในโค้ด** | `if (order && order.status === 'pending')` — มี guard แต่อาจพลาดใน race condition |

**Mitigation:**
- เพิ่ม `stripePaymentIntentId` field ใน Order schema และ check ก่อน process
- ใช้ MongoDB atomic update (findOneAndUpdate with condition) แทน find แล้ว save

---

### TECH-006 — ไม่มี Pagination บน List Endpoints

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | เมื่อ orders หรือ users มีจำนวนมาก → `/order/all-orders` และ `/auth/all-users` โหลดข้อมูลทั้งหมดเข้า memory → response ช้า, memory สูง |
| **Likelihood** | 3 (ปานกลาง — ระยะยาว) |
| **Impact** | 3 (ปานกลาง) |
| **Severity** | 🟡 **9** |

**Mitigation:**
- เพิ่ม pagination: `?page=1&limit=20` ใน query string
- ใช้ Mongoose `.skip()` และ `.limit()`

---

### TECH-007 — ไม่มี Input Sanitization บน Email HTML Template

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ถ้า `username` หรือ `order._id` มี HTML characters (`<`, `>`) จะ render เป็น HTML ใน email → XSS ใน email client |
| **สาเหตุ** | `<strong>${user.username}</strong>` ใน email template ไม่มี HTML escaping |
| **Likelihood** | 2 (ต่ำ — username มี validation แต่ไม่ strict ทุกกรณี) |
| **Impact** | 2 (ต่ำ — กระทบแค่ email ไม่ใช่ web app) |
| **Severity** | 🟢 **4** |

**Mitigation:**
- ใช้ utility function escape HTML ก่อนใส่ใน template
- ใช้ email template engine เช่น Handlebars ที่ escape โดย default

---

## [OPS] Operational Risks

### OPS-001 — ไม่มี Health Check บน email-service Container

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | email-service container อาจ running แต่ไม่ตอบสนอง → Docker ถือว่า healthy แต่จริงๆ ไม่ทำงาน |
| **Likelihood** | 2 (ต่ำ) |
| **Impact** | 2 (ต่ำ) |
| **Severity** | 🟢 **4** |
| **หลักฐานในโค้ด** | `email-service/Dockerfile` ไม่มี `HEALTHCHECK` instruction |

**Mitigation:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s \
  CMD wget -qO- http://localhost:5051/health || exit 1
```

---

### OPS-002 — ไม่มี Logging Framework (Console.log Only)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | Log ทุกอย่างไปที่ stdout — ไม่มี log level, ไม่มี log rotation, ไม่มี centralized log storage → ยากต่อการ debug production |
| **Likelihood** | 5 (แน่นอน — ไม่มี logger library) |
| **Impact** | 2 (ต่ำ — ไม่ทำให้ระบบล่ม แต่ debug ยาก) |
| **Severity** | 🟡 **10** |

**Mitigation:**
- ใช้ logging library เช่น `winston` หรือ `pino`
- กำหนด log levels: `error`, `warn`, `info`, `debug`
- เก็บ log ลง file และ rotate

---

### OPS-003 — ไม่มีการ Backup Strategy สำหรับ MongoDB

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ข้อมูล orders และ users สูญหายถ้า MongoDB Atlas มีปัญหา |
| **Likelihood** | 1 (ต่ำมาก — Atlas มี built-in redundancy) |
| **Impact** | 5 (สูงมาก) |
| **Severity** | 🟠 **5** |

**Mitigation:**
- เปิดใช้ MongoDB Atlas Continuous Backup / Point-in-Time Recovery
- ตั้ง automated snapshot ทุกวัน

---

## [BIZ] Business Risks

### BIZ-001 — Stripe Dependency (Single Payment Provider)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | ถ้า Stripe มีปัญหา (downtime, policy change, account suspended) → ระบบรับ payment ไม่ได้ทั้งหมด |
| **Likelihood** | 2 (ต่ำ) |
| **Impact** | 5 (สูงมาก — ไม่มีรายได้) |
| **Severity** | 🔴 **10** |

**Mitigation:**
- วางแผน fallback payment method (เช่น bank transfer พร้อม manual confirmation)
- ติดตาม Stripe Status Page

---

### BIZ-002 — ข้อมูล Order ไม่มีการ Archive

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | Orders สะสมใน collection เดียวกันตลอดเวลา → performance degradation ในระยะยาว |
| **Likelihood** | 3 (ปานกลาง) |
| **Impact** | 2 (ต่ำ) |
| **Severity** | 🟡 **6** |

**Mitigation:**
- วางแผน archiving strategy: orders ที่ `shipped` นานกว่า 1 ปี → ย้ายไป archive collection

---

### BIZ-003 — การเปลี่ยน Product Price กระทบ Order เก่า

| ฟิลด์ | รายละเอียด |
|---|---|
| **ความเสี่ยง** | Admin แก้ `base_price` ใน Item collection → `total_price` ใน order เก่าถูกบันทึกถูกต้องแล้ว แต่ถ้า recalculate จะผิด |
| **Likelihood** | 2 (ต่ำ) |
| **Impact** | 2 (ต่ำ — snapshot price บันทึกไว้ใน Order แล้ว) |
| **Severity** | 🟢 **4** |
| **หลักฐานในโค้ด** | `total_price` เก็บใน Order schema แบบ snapshot — ปลอดภัยแล้ว |

**Status:** ✅ จัดการได้ดีอยู่แล้ว — `total_price` ถูก snapshot ตอนสร้าง Order

---

## Risk Matrix Summary

```
IMPACT →        1          2          3          4          5
              ──────────────────────────────────────────────────
LIKELIHOOD  5 │           OPS-002                            SEC-003
           ↓  │
            4 │                                           TECH-002
              │
            3 │ TECH-003  TECH-004   TECH-006   OPS-003   SEC-001
              │           BIZ-002                         SEC-006
              │           TECH-007               SEC-007  TECH-005
              │
            2 │ TECH-005  TECH-007   SEC-005    SEC-004   TECH-001
              │ BIZ-003   BIZ-001               OPS-001   SEC-002
              │
            1 │                                           BIZ-001
              │                                           OPS-003
```

| โซน | Severity | Risks |
|---|---|---|
| 🔴 Critical (15–25) | ต้องแก้ไขทันที | SEC-001, SEC-003, SEC-006 |
| 🟠 High (10–14) | ต้องวางแผนแก้ไข | SEC-002, TECH-001, TECH-002, BIZ-001, OPS-002 |
| 🟡 Medium (6–9) | ติดตามและวางแผน | SEC-004, SEC-005, TECH-004, TECH-006, BIZ-002 |
| 🟢 Low (1–5) | Monitor | TECH-003, TECH-005, TECH-007, SEC-007, OPS-001, OPS-003, BIZ-003 |

---

## Risk Action Plan (Priority Order)

| Priority | Risk ID | Action | ผู้รับผิดชอบ | Deadline |
|---|---|---|---|---|
| 🔴 P1 | SEC-003 | เพิ่ม rate limiting บน `/login` | Dev | ทันที |
| 🔴 P1 | SEC-001 | เปลี่ยน `httpOnly: true`, `secure: true` | Dev | ทันที |
| 🔴 P1 | SEC-006 | เปลี่ยนชื่อ `.ENV` → `.env`, rotate Stripe keys | Dev | ทันที |
| 🟠 P2 | TECH-001 | เพิ่ม `process.exit(1)` ใน DB connection failure | Dev | สัปดาห์นี้ |
| 🟠 P2 | TECH-002 | ซิงค์ `EXPOSE` port กับ `PORT` env | Dev | สัปดาห์นี้ |
| 🟠 P2 | SEC-002 | เพิ่ม `sanitizeObject(req.query)` | Dev | สัปดาห์นี้ |
| 🟠 P2 | OPS-002 | เพิ่ม logging framework (winston/pino) | Dev | sprint ถัดไป |
| 🟡 P3 | TECH-006 | เพิ่ม pagination บน list endpoints | Dev | sprint ถัดไป |
| 🟡 P3 | SEC-005 | แก้ CORS whitelist | Dev | sprint ถัดไป |
| 🟢 P4 | OPS-001 | เพิ่ม HEALTHCHECK ใน email-service Dockerfile | Dev | เมื่อมีเวลา |

---

## Risk Monitoring Checklist (รายสัปดาห์)

- [ ] ตรวจ Stripe Dashboard — มี unauthorized charge ไหม?
- [ ] ตรวจ MongoDB Atlas — connection errors / unusual queries
- [ ] ตรวจ audit log (`/log` collection) — มี failed login spike ไหม?
- [ ] ตรวจ Docker container health — ทุก service healthy?
- [ ] ตรวจ email-service — มี failed email deliveries ไหม?
- [ ] ตรวจ Stripe Status Page — planned maintenance?
- [ ] ตรวจ GitHub repository — ไม่มีไฟล์ `.env` หลุดขึ้น commit?
