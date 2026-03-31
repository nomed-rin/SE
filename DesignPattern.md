# Design Pattern

---

**Pattern Name: Strategy Pattern**

**Description:** Strategy Pattern เป็นรูปแบบการออกแบบที่ช่วยให้สามารถเลือกวิธีการทำงาน (algorithm) ได้หลายรูปแบบ และสามารถเปลี่ยนได้ในขณะ runtime โดยไม่ต้องแก้ไขโค้ดหลัก

**Problem:** ระบบชำระเงินของ Uni-c มีโหมดการทำงาน 2 แบบ ได้แก่
- การใช้ Stripe Test Keys (สำหรับพัฒนาและทดสอบ)
- การใช้ Stripe Live Keys (สำหรับ production จริง)

หากเขียนรวมกันจะทำให้โค้ดซับซ้อนและเกิดความผิดพลาดได้ง่าย

**Solution:** แยกการเลือก Stripe key ออกเป็น strategy ตาม environment variable `STRIPE_MODE` และให้ระบบเลือก strategy ที่เหมาะสมโดยอัตโนมัติ

**Structure:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- `STRIPE_MODE` (env variable — ตัวกำหนด strategy)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- `stripeSecretKey` — เลือกระหว่าง `STRIPE_SECRET_KEY_TEST` หรือ `STRIPE_SECRET_KEY_LIVE`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- `stripeWebhookSecret` — เลือกระหว่าง `STRIPE_WEBHOOK_SECRET_TEST` หรือ `STRIPE_WEBHOOK_SECRET_LIVE`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- `stripe` (context) — ใช้ key ที่ถูกเลือกมาสร้าง Stripe instance

**Example in System:** เมื่อ `.ENV` ตั้ง `STRIPE_MODE=test` ระบบจะเลือก Test Keys โดยอัตโนมัติ เมื่อเปลี่ยนเป็น `STRIPE_MODE=prod` ระบบจะสลับไปใช้ Live Keys ทันทีโดยไม่ต้องแก้โค้ด

**Benefits:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- เปลี่ยน environment ได้ง่ายโดยแก้เพียง 1 ค่าใน `.ENV`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- ป้องกันการใช้ Live Keys ผิดพลาดในขณะพัฒนา

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- โค้ด payment logic ไม่ต้องเปลี่ยนเมื่อ deploy

---

**Pattern Name: Model-View-Controller (MVC)**

**Description:** MVC เป็นรูปแบบที่แยกส่วนของระบบออกเป็น 3 ส่วน ได้แก่ Model, View และ Controller

**Problem:** หากเขียน business logic, database query และ HTTP response รวมกัน จะทำให้ระบบแก้ไขยากและทดสอบได้ยาก

**Solution:** แยกหน้าที่ของแต่ละส่วนออกจากกันตามโครงสร้างของโปรเจกต์

**Structure:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- Model: Database Schema (`userCollection`, `orderCollection`, `itemCollection`)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- View: Frontend Next.js (`uni-c.rinthecat.pro`)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- Controller: Route handlers ใน `/routes/api/` รับ request และเรียกใช้ Model

**Example in System:** ผู้ใช้กรอกข้อมูล → Controller (`POST /order/create`) รับค่า → ดึงข้อมูลจาก Model (`Item`, `User`, `Order`) → คำนวณราคา → ส่งผลลัพธ์กลับไปที่ View (Frontend)

**Benefits:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- โค้ดเป็นระเบียบ แยกหน้าที่ชัดเจน

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- พัฒนาและแก้ไขได้ง่ายในแต่ละ layer

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- สามารถทดสอบ Model และ Controller แยกจากกันได้

---

**Pattern Name: Facade Pattern**

**Description:** Facade Pattern เป็นรูปแบบการออกแบบที่ช่วยรวมการทำงานของหลายระบบย่อย (subsystems) ให้มี interface เดียวที่ใช้งานง่าย

**Problem:** การส่งอีเมลในระบบ Uni-c มีหลายขั้นตอน ได้แก่
- สร้าง HTTP request ไปยัง Email Service
- แนบ `x-api-key` ใน header
- กำหนด `to`, `subject`, `text`, `html`
- จัดการ error จาก external service

หากเรียกขั้นตอนเหล่านี้ตรงๆ ในทุก route จะทำให้ระบบซับซ้อน

**Solution:** สร้าง `sendEmail()` utility function ที่ห่อหุ้มความซับซ้อนทั้งหมดไว้ภายใน ให้ route เรียกใช้ได้ด้วยคำสั่งเดียว

**Structure:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- `sendEmail({ to, subject, text, html })` (Facade)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- Email Service HTTP Client (subsystem ที่ถูกซ่อนไว้)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- `EMAIL_SERVICE_URL` + `EMAIL_SERVICE_API_KEY` (config ภายใน Facade)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- Route handlers (`order.js`, `payment.js`) — ผู้ใช้งาน Facade

**Example in System:** เมื่อ Admin เปลี่ยนสถานะ order, route เรียกแค่ `await sendEmail({ to: email, subject, text, html })` โดยไม่ต้องรู้รายละเอียด HTTP request ของ Email Service เลย

**Benefits:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- ลดความซับซ้อนใน route handler

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- เปลี่ยน Email Provider ได้โดยแก้ที่ `utils/email.js` ที่เดียว

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- โค้ดอ่านง่ายและ maintain ง่าย

---

**Pattern Name: Chain of Responsibility Pattern**

**Description:** Chain of Responsibility Pattern เป็นรูปแบบที่ส่ง request ผ่านลำดับของ handler ต่อกันเป็นทอดๆ แต่ละ handler ตัดสินใจว่าจะประมวลผลหรือส่งต่อ

**Problem:** API ของ Uni-c มี endpoint ที่ต้องการระดับสิทธิ์ต่างกัน ได้แก่
- บาง endpoint ทุกคนเข้าได้ (Public)
- บาง endpoint ต้องมี JWT session (Member)
- บาง endpoint ต้องเป็น Admin เท่านั้น

หากตรวจสอบสิทธิ์รวมไว้ในแต่ละ route จะทำให้โค้ดซ้ำซ้อน

**Solution:** สร้าง middleware chain ที่แยกหน้าที่ชัดเจน ให้แต่ละ middleware ตรวจสอบเงื่อนไขของตัวเอง แล้วจึงส่งต่อด้วย `next()`

**Structure:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- `auth` middleware — ตรวจสอบ JWT token จาก cookie `session`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- `adminOnly` middleware — ตรวจสอบว่า `req.user.role === 'Admin'`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- Route handler — ประมวลผล business logic เมื่อผ่านทุก handler แล้ว

**Example in System:** เมื่อ Admin เรียก `PUT /order/:id/status` → `auth` ตรวจ JWT ก่อน → ถ้าผ่านจึงส่งต่อให้ `adminOnly` ตรวจ role → ถ้าผ่านจึงเข้าสู่ route handler ที่อัปเดต status และส่ง email แจ้งผู้ใช้

**Benefits:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- แยก authentication และ authorization ออกจากกันชัดเจน

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- เพิ่ม/ลด middleware ได้โดยไม่กระทบ route อื่น

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- โค้ดไม่ซ้ำซ้อน ใช้ middleware ร่วมกันได้ทุก route
