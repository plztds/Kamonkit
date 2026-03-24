# การวิเคราะห์รูปแบบ System Design สำหรับระบบกมลกิจยานยนต์

เอกสารฉบับนี้วิเคราะห์รูปแบบการออกแบบระบบ (System Design) ที่นิยมใช้ในปัจจุบัน พร้อมข้อดี-ข้อเสีย สถานการณ์ที่เหมาะสม และคำแนะนำสำหรับระบบกมลกิจยานยนต์โดยเฉพาะ

---

## ส่วนที่ 1: รูปแบบ System Design ในปัจจุบัน

---

### 1.1 สถาปัตยกรรมระบบ (System Architecture)

#### 1.1.1 Monolith (แบบก้อนเดียว)

**แนวคิด:** ทุกส่วนของระบบ (ทั้ง business logic, การเข้าถึง database, API endpoint) อยู่ใน codebase เดียว deploy เป็น process เดียว

```
┌──────────────────────────────┐
│         Monolith App         │
│                              │
│  ┌────────┐ ┌────────┐      │
│  │ ขายรถ   │ │ ซ่อมรถ  │      │
│  └────────┘ └────────┘      │
│  ┌────────┐ ┌────────┐      │
│  │ อะไหล่  │ │ บัญชี   │      │
│  └────────┘ └────────┘      │
│                              │
│  ┌──────────────────────┐   │
│  │    Single Database    │   │
│  └──────────────────────┘   │
└──────────────────────────────┘
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | พัฒนาง่ายและเร็ว, deploy ไม่ยุ่งยาก, debug ง่ายเพราะทุกอย่างอยู่ที่เดียว, ไม่ต้องจัดการ network ระหว่าง service, transaction ข้าม module ทำได้ง่าย (ACID) |
| **ข้อเสีย** | เมื่อโค้ดใหญ่ขึ้น module เริ่มพันกัน (spaghetti code), scale ได้แค่ทั้งก้อน (ไม่สามารถ scale เฉพาะส่วนที่โหลดหนักได้), ถ้า deploy ผิดพลาดมีผลกระทบทั้งระบบ, เปลี่ยน tech stack ยาก |
| **เหมาะกับ** | ระบบขนาดเล็ก-กลาง, ทีมเล็ก (1–5 คน), ช่วงเริ่มต้นพัฒนา (MVP), ระบบที่ต้อง ACID transaction ข้าม module |
| **ไม่เหมาะกับ** | ระบบที่ต้อง scale แต่ละส่วนแยกกัน, ทีมใหญ่หลายทีมพัฒนาพร้อมกัน |
| **ตัวอย่าง** | ร้านค้าออนไลน์ขนาดเล็ก, ระบบ ERP สำหรับ SME, ระบบ CRM ภายในองค์กร |

---

#### 1.1.2 Modular Monolith (ก้อนเดียวแบ่ง Module)

**แนวคิด:** ยังคง deploy เป็น process เดียว แต่ภายในแบ่งโค้ดเป็น module ชัดเจนตาม business domain แต่ละ module มี boundary ชัด สื่อสารกันผ่าน interface ที่กำหนดไว้ ไม่เรียกข้าม module โดยตรง

```
┌──────────────────────────────────────┐
│           Modular Monolith            │
│                                       │
│  ┌─────────┐   ┌─────────┐           │
│  │ Vehicle  │──▶│  Sales   │           │
│  │ Module   │   │ Module   │           │
│  └─────────┘   └────┬────┘           │
│                      │                │
│  ┌─────────┐   ┌────▼────┐           │
│  │  Parts   │──▶│ Repair   │          │
│  │ Module   │   │ Module   │          │
│  └─────────┘   └─────────┘           │
│                                       │
│  ┌─────────┐   ┌─────────┐           │
│  │  Cash    │   │Collection│          │
│  │ Module   │   │ Module   │          │
│  └─────────┘   └─────────┘           │
│                                       │
│  ┌──────────────────────────────┐    │
│  │       Shared Database        │    │
│  │   (แต่ละ module มี schema    │    │
│  │    ของตัวเอง)                 │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | ได้ข้อดีของ Monolith (deploy ง่าย, ACID transaction) + โค้ดเป็นระเบียบตาม domain, ป้องกัน spaghetti code ด้วย module boundary, ถ้าต้องแยกเป็น Microservice ในอนาคตทำได้ง่ายกว่า, เข้าใจง่ายเพราะแบ่งตาม business domain |
| **ข้อเสีย** | ต้องมีวินัยในการรักษา boundary (ถ้าไม่ระวังจะกลายเป็น Monolith ธรรมดา), ยังคง scale ได้แค่ทั้งก้อน, deploy ยังเป็นก้อนเดียว |
| **เหมาะกับ** | ระบบที่มี business logic ซับซ้อนหลาย domain, ทีมเล็กที่ต้องการโค้ดเป็นระเบียบ, ระบบที่อาจ scale เป็น Microservice ในอนาคต |
| **ไม่เหมาะกับ** | ระบบที่ต้อง scale แต่ละส่วนแยกกันตั้งแต่วันแรก |
| **ตัวอย่าง** | Shopify (เริ่มจาก Modular Monolith ก่อนแยก Microservice), ระบบ ERP ขนาดกลาง |

---

#### 1.1.3 Microservices (แยกย่อยเป็น Service)

**แนวคิด:** แยกระบบออกเป็น service ย่อยๆ แต่ละ service รับผิดชอบ 1 domain ทำงานเป็น process แยก มี database แยก สื่อสารกันผ่าน API หรือ message queue

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Vehicle   │  │  Sales   │  │  Repair  │
│ Service   │  │ Service  │  │ Service  │
│ ┌──────┐  │  │ ┌──────┐ │  │ ┌──────┐ │
│ │ DB   │  │  │ │ DB   │ │  │ │ DB   │ │
│ └──────┘  │  │ └──────┘ │  │ └──────┘ │
└─────┬─────┘  └────┬─────┘  └────┬─────┘
      │              │             │
      ▼              ▼             ▼
┌──────────────────────────────────────┐
│          API Gateway / Mesh          │
└──────────────────────────────────────┘
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | scale แต่ละ service แยกได้, ใช้ tech stack ต่างกันได้แต่ละ service, deploy แยกกัน (ไม่กระทบ service อื่น), ทีมหลายทีมพัฒนาพร้อมกันได้ |
| **ข้อเสีย** | ซับซ้อนมาก (network, distributed transaction, service discovery), debug ยาก, ต้องจัดการ data consistency ข้าม service, infrastructure cost สูง, ต้องการ DevOps expertise |
| **เหมาะกับ** | ระบบขนาดใหญ่ที่ต้อง scale สูง, ทีมใหญ่ (10+ คน) หลายทีมพัฒนาพร้อมกัน, ระบบที่แต่ละส่วนมี load ต่างกันมาก |
| **ไม่เหมาะกับ** | ทีมเล็ก, ระบบที่ต้อง ACID transaction ข้ามหลาย domain, ช่วงเริ่มต้นพัฒนา |
| **ตัวอย่าง** | Netflix, Grab, Lazada, ธนาคารขนาดใหญ่ |

---

#### 1.1.4 Serverless / FaaS (Function as a Service)

**แนวคิด:** ไม่ต้องจัดการ server เอง เขียนโค้ดเป็น function เล็กๆ ที่ทำงานตาม event (เช่น HTTP request, schedule) ผู้ให้บริการ cloud จัดการ scaling และ infrastructure ให้

```
HTTP Request ──▶ ┌────────────┐ ──▶ Database
                 │  Function   │
Schedule ──────▶ │ (Lambda /   │ ──▶ Storage
                 │  Workers)   │
Event ─────────▶ └────────────┘ ──▶ Queue
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | ไม่ต้อง manage server, จ่ายเฉพาะตอนใช้งาน (pay-per-use), scale อัตโนมัติ, เหมาะกับ event-driven workload |
| **ข้อเสีย** | Cold start (ครั้งแรกช้า), จำกัดเวลาทำงาน (เช่น Lambda 15 นาที), vendor lock-in สูง, debug ยาก, ไม่เหมาะกับ long-running process, connection pool กับ database จัดการยาก |
| **เหมาะกับ** | API ที่ traffic ไม่สม่ำเสมอ (บางเวลาไม่มีคนใช้เลย), webhook/notification handler, scheduled jobs, MVP ที่ต้องการลดค่าใช้จ่าย |
| **ไม่เหมาะกับ** | ระบบที่ต้อง response เร็วตลอด (cold start เป็นปัญหา), ระบบที่มี business logic ซับซ้อน, ระบบที่ต้องใช้ WebSocket หรือ long-running connection |
| **ตัวอย่าง** | LINE Bot, Webhook handler, Notification service, Image processing |

---

#### 1.1.5 Event-Driven Architecture (สถาปัตยกรรมขับเคลื่อนด้วย Event)

**แนวคิด:** ส่วนต่างๆ ของระบบสื่อสารกันผ่าน event (เหตุการณ์) โดยมี message broker (เช่น RabbitMQ, Kafka) เป็นตัวกลาง เมื่อเกิดเหตุการณ์ ระบบส่ง event ออกไป ส่วนอื่นที่สนใจ event นั้นจะรับไปทำงานต่อ

```
Producer ──▶ ┌───────────────┐ ──▶ Consumer A
             │ Message Broker │
Producer ──▶ │  (Kafka /      │ ──▶ Consumer B
             │   RabbitMQ)    │
             └───────────────┘ ──▶ Consumer C
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | loose coupling ระหว่าง module, รองรับ async processing (งานที่ไม่ต้องตอบทันที), สามารถ replay event ได้, เหมาะกับระบบ notification/alert |
| **ข้อเสีย** | ซับซ้อนในการ debug (ไม่มี call stack ตรงๆ), eventual consistency (ข้อมูลอาจไม่ sync ทันที), ต้อง manage message broker, ยากในการรับประกันลำดับ event |
| **เหมาะกับ** | ระบบที่มี notification/alert เยอะ, ระบบที่ต้อง decouple ส่วนต่างๆ, ระบบ real-time analytics |
| **ไม่เหมาะกับ** | ระบบเล็กที่ไม่ต้องการ async, ระบบที่ต้องการ consistency ทันทีทุก transaction |
| **ตัวอย่าง** | ระบบแจ้งเตือน, ระบบ e-commerce ขนาดใหญ่ (order → payment → shipping), IoT |

---

### ตารางสรุปเปรียบเทียบสถาปัตยกรรม

| เกณฑ์ | Monolith | Modular Monolith | Microservices | Serverless | Event-Driven |
|-------|----------|-------------------|---------------|------------|--------------|
| ความซับซ้อนเริ่มต้น | ต่ำ | ปานกลาง | สูงมาก | ปานกลาง | สูง |
| ทีมที่เหมาะ | 1–5 คน | 1–10 คน | 10+ คน | 1–3 คน | 5+ คน |
| ต้นทุน infrastructure | ต่ำ | ต่ำ | สูง | ต่ำ-กลาง | กลาง-สูง |
| ACID transaction | ง่าย | ง่าย | ยากมาก | ยาก | ยากมาก |
| การ scale | ทั้งก้อน | ทั้งก้อน | แยกส่วนได้ | อัตโนมัติ | แยกส่วนได้ |
| การ deploy | ง่าย | ง่าย | ยุ่งยาก | ง่าย | ยุ่งยาก |
| การ maintain ระยะยาว | อาจยุ่งเหยิง | ดี | ดีถ้ามีทีมพอ | กลางๆ | กลางๆ |
| เวลาพัฒนาเริ่มต้น | เร็วที่สุด | เร็ว | ช้า | เร็ว | ช้า |

---

### 1.2 รูปแบบ API

#### 1.2.1 REST (Representational State Transfer)

**แนวคิด:** ใช้ HTTP methods (GET, POST, PUT, DELETE) กับ resource URL เพื่อจัดการข้อมูล ส่งข้อมูลเป็น JSON

```
GET    /api/customers          → ดึงรายชื่อลูกค้า
GET    /api/customers/123      → ดึงลูกค้ารหัส 123
POST   /api/customers          → สร้างลูกค้าใหม่
PUT    /api/customers/123      → แก้ไขลูกค้ารหัส 123
DELETE /api/customers/123      → ลบลูกค้ารหัส 123
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | เข้าใจง่าย เป็นมาตรฐาน, เครื่องมือรองรับเยอะ (Postman, Swagger, etc.), cache ได้ง่าย (HTTP cache), ทุก platform รองรับ (Web, Mobile, Desktop) |
| **ข้อเสีย** | อาจเกิด over-fetching (ดึงข้อมูลมาเกินที่ต้องการ) หรือ under-fetching (ต้องเรียกหลาย endpoint), ไม่มี built-in type safety |
| **เหมาะกับ** | ระบบ CRUD ทั่วไป, ระบบที่ต้อง public API, ระบบที่ต้องรองรับหลาย platform |

#### 1.2.2 GraphQL

**แนวคิด:** Client เขียน query เพื่อระบุว่าต้องการข้อมูลอะไรบ้าง Server ส่งเฉพาะที่ขอ ทุก request ส่งไป endpoint เดียว (POST /graphql)

```graphql
query {
  customer(id: 123) {
    name
    phone
    contracts {
      contractNumber
      status
      remainingAmount
    }
  }
}
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | ไม่มีปัญหา over/under-fetching, type system ชัดเจน (schema), เหมาะกับ frontend ที่หน้าจอหลากหลาย, developer experience ดี |
| **ข้อเสีย** | เรียนรู้ยากกว่า REST, cache ยากกว่า, อาจเกิด N+1 query, security ซับซ้อนกว่า (ต้อง limit query depth), ไม่เหมาะกับ file upload |
| **เหมาะกับ** | ระบบที่มี frontend หลายแบบ (web, mobile) ต้องการข้อมูลต่างกัน, ระบบที่ data relationship ซับซ้อน |

#### 1.2.3 tRPC

**แนวคิด:** Type-safe RPC สำหรับ TypeScript monorepo — client เรียก function ฝั่ง server ได้โดยตรง มี type safety ตั้งแต่ server ถึง client ไม่ต้องเขียน API spec แยก

```typescript
// Server
const appRouter = router({
  getCustomer: publicProcedure
    .input(z.object({ id: z.number() }))
    .query(({ input }) => db.customer.findById(input.id)),
});

// Client — type-safe, autocomplete ได้
const customer = await trpc.getCustomer.query({ id: 123 });
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | type-safe ตั้งแต่ server ถึง client (ไม่ต้อง generate type), developer experience ดีมาก (autocomplete ทุกจุด), ไม่ต้องเขียน API documentation แยก, เร็วในการพัฒนา |
| **ข้อเสีย** | ใช้ได้เฉพาะ TypeScript, ต้องเป็น monorepo, ไม่เหมาะกับ public API (ไม่เป็นมาตรฐาน), third-party ใช้ยาก, ใช้กับ React Native ได้แต่ต้อง setup เพิ่ม |
| **เหมาะกับ** | TypeScript monorepo, ระบบภายในที่ทุก client เป็น TypeScript, Solo developer ที่ต้องการ velocity สูง |

#### 1.2.4 gRPC

**แนวคิด:** ใช้ Protocol Buffers (Protobuf) เป็น format แทน JSON มี code generation สำหรับหลายภาษา สื่อสารผ่าน HTTP/2

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | เร็วมาก (binary format), type-safe, รองรับ streaming, เหมาะกับ service-to-service |
| **ข้อเสีย** | browser รองรับจำกัด (ต้องใช้ gRPC-Web), เรียนรู้ยาก, debug ยาก (binary), ไม่เหมาะกับ frontend โดยตรง |
| **เหมาะกับ** | Microservices สื่อสารกันภายใน, ระบบ real-time ที่ต้องการ low latency |

#### ตารางสรุปเปรียบเทียบ API

| เกณฑ์ | REST | GraphQL | tRPC | gRPC |
|-------|------|---------|------|------|
| เรียนรู้ง่าย | ง่ายที่สุด | ปานกลาง | ง่าย (ถ้ารู้ TS) | ยาก |
| Type Safety | ไม่มี (ต้องเสริม) | มี (schema) | ดีที่สุด | มี (Protobuf) |
| รองรับ Multi-platform | ดีที่สุด | ดี | เฉพาะ TS | จำกัด |
| Performance | ดี | ดี | ดี | ดีที่สุด |
| เหมาะกับ Public API | ใช่ | ได้ | ไม่ | ไม่ |
| เวลาพัฒนา | เร็ว | ปานกลาง | เร็วที่สุด | ช้า |

---

### 1.3 รูปแบบ Frontend

#### 1.3.1 SPA (Single Page Application)

**แนวคิด:** โหลด HTML/CSS/JS ทั้งหมดครั้งแรกครั้งเดียว หลังจากนั้นการเปลี่ยนหน้าทำผ่าน JavaScript โดยไม่ reload หน้าใหม่ ดึงข้อมูลจาก API

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | UX ลื่นไหล (ไม่ reload หน้า), แยก frontend/backend ชัดเจน, เหมาะกับ app ที่ใช้งานต่อเนื่อง |
| **ข้อเสีย** | SEO ไม่ดี (ต้องเสริม), initial load ช้า (โหลด JS เยอะ), ต้องจัดการ state ฝั่ง client |
| **เหมาะกับ** | ระบบภายในองค์กร (ไม่ต้อง SEO), Dashboard, Admin panel |
| **ตัวอย่าง framework** | React (Vite), Vue.js, Angular |

#### 1.3.2 SSR (Server-Side Rendering)

**แนวคิด:** Server render HTML สำหรับทุก request ส่ง HTML สำเร็จรูปมาที่ browser

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | SEO ดี, initial load เร็ว (ได้ HTML ทันที), ทำงานได้แม้ไม่มี JavaScript |
| **ข้อเสีย** | server load สูงกว่า, เปลี่ยนหน้าช้ากว่า SPA (ต้อง reload), UX ไม่ลื่นเท่า SPA |
| **เหมาะกับ** | เว็บที่ต้องการ SEO, Content-heavy website |
| **ตัวอย่าง framework** | Next.js (Pages Router), Nuxt.js, Laravel Blade |

#### 1.3.3 Hybrid (SSR + SPA)

**แนวคิด:** ผสม SSR กับ SPA — หน้าแรกใช้ SSR (โหลดเร็ว, SEO ดี) หลังจากนั้นใช้ client-side navigation (ลื่นเหมือน SPA)

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | ได้ทั้ง SEO และ UX ที่ลื่น, ยืดหยุ่น เลือกได้ว่าหน้าไหน SSR หน้าไหน client-side |
| **ข้อเสีย** | ซับซ้อนกว่า SPA/SSR อย่างเดียว, ต้องเข้าใจว่าอะไร render ที่ server อะไร render ที่ client |
| **เหมาะกับ** | ระบบที่ต้องการทั้ง SEO และ interactive UI |
| **ตัวอย่าง framework** | Next.js (App Router), Nuxt 3, SvelteKit |

#### 1.3.4 PWA (Progressive Web App)

**แนวคิด:** Web App ที่เสริมความสามารถคล้าย Native App — ทำงาน offline ได้, ติดตั้งบนหน้าจอมือถือได้, รับ Push Notification ได้

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | ไม่ต้องผ่าน App Store, อัปเดตทันที, ทำงาน offline ได้ (Service Worker), cross-platform (Web + Mobile) ด้วย codebase เดียว |
| **ข้อเสีย** | ความสามารถจำกัดกว่า Native App (เช่น GPS ทำงานไม่ดีเท่า, ไม่เข้าถึง hardware บางอย่าง), iOS รองรับจำกัดกว่า Android, UX ไม่เนียนเท่า Native |
| **เหมาะกับ** | ระบบที่ต้อง offline + ไม่อยากทำ Native App แยก |

#### ตารางสรุปเปรียบเทียบ Frontend

| เกณฑ์ | SPA | SSR | Hybrid | PWA |
|-------|-----|-----|--------|-----|
| SEO | ไม่ดี | ดี | ดี | ปานกลาง |
| Initial Load | ช้า | เร็ว | เร็ว | ปานกลาง |
| UX ลื่นไหล | ดีที่สุด | ปานกลาง | ดี | ดี |
| Offline | ไม่ได้ | ไม่ได้ | ไม่ได้ | ได้ |
| ความซับซ้อน | ปานกลาง | ต่ำ | สูง | สูง |

---

### 1.4 การจัดการ Database

#### 1.4.1 Single Database (ฐานข้อมูลเดียว)

**แนวคิด:** ทุก module ใช้ database เดียวกัน (อาจแยก schema ภายใน)

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | ACID transaction ง่าย, JOIN ข้าม table ได้, consistency รับประกัน, manage ง่าย (backup, migration ที่เดียว) |
| **ข้อเสีย** | bottleneck เมื่อโหลดสูงมาก, ถ้า DB ล่ม ล่มทั้งระบบ |
| **เหมาะกับ** | ระบบขนาดเล็ก-ใหญ่ (PostgreSQL รับได้ถึงหลักล้าน row), ระบบที่ต้อง ACID |

#### 1.4.2 Database per Service (แยก DB ต่อ Service)

**แนวคิด:** แต่ละ service มี database ของตัวเอง ไม่แชร์กัน

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | แต่ละ service เป็นอิสระ, scale แยกได้, เลือก DB type ที่เหมาะกับ workload ได้ |
| **ข้อเสีย** | ไม่สามารถ JOIN ข้าม service ได้, ต้องใช้ eventual consistency, distributed transaction ยากมาก |
| **เหมาะกับ** | Microservices architecture |

#### 1.4.3 CQRS (Command Query Responsibility Segregation)

**แนวคิด:** แยก model สำหรับเขียน (Command) กับอ่าน (Query) ออกจากกัน เพื่อ optimize แต่ละด้านแยก

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | optimize read/write แยกกันได้, read model ออกแบบให้ตรงกับ UI ได้เลย, scale read แยกจาก write |
| **ข้อเสีย** | ซับซ้อนมาก, eventual consistency ระหว่าง read/write model, ต้อง sync ข้อมูล |
| **เหมาะกับ** | ระบบที่ read หนักมากกว่า write (เช่น reporting), ระบบที่ต้อง query ซับซ้อนหลายรูปแบบ |

#### 1.4.4 Event Sourcing (เก็บ Event แทนเก็บ State)

**แนวคิด:** แทนที่จะเก็บ "สถานะปัจจุบัน" ของข้อมูล ให้เก็บ "เหตุการณ์ทั้งหมด" ที่เกิดขึ้น แล้วสร้างสถานะจากการ replay event

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | audit trail สมบูรณ์ (ย้อนดูทุก event ได้), สามารถ rebuild state ได้, เหมาะกับระบบการเงิน |
| **ข้อเสีย** | ซับซ้อนมาก, query ข้อมูลปัจจุบันต้อง replay event (หรือสร้าง projection), storage โตเร็ว |
| **เหมาะกับ** | ระบบธนาคาร, ระบบ trading, ระบบที่ต้อง audit trail ละเอียดมาก |

#### ตารางสรุปเปรียบเทียบ Database

| เกณฑ์ | Single DB | DB per Service | CQRS | Event Sourcing |
|-------|-----------|----------------|------|----------------|
| ความซับซ้อน | ต่ำ | สูง | สูงมาก | สูงมาก |
| ACID Transaction | ง่าย | ยากมาก | ปานกลาง | ยาก |
| Audit Trail | ต้องเสริม | ต้องเสริม | ต้องเสริม | built-in |
| เหมาะกับทีมเล็ก | ใช่ | ไม่ | ไม่ | ไม่ |

---

### 1.5 การจัดโครงสร้างโปรเจกต์

#### 1.5.1 Monorepo (โค้ดทั้งหมดอยู่ repo เดียว)

```
kamonkit/
├── apps/
│   ├── api/              ← Backend
│   ├── web/              ← Web App (เสมียน/ช่าง)
│   └── mobile/           ← Mobile App (เจ้าของ/Collector)
├── packages/
│   ├── shared/           ← Type, Utility ที่ใช้ร่วมกัน
│   ├── db/               ← Database schema, migration
│   └── ui/               ← Shared UI components
└── package.json
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | share code ระหว่าง app ง่าย (types, validation, utils), refactor ทีเดียวแก้ทั้งระบบ, ใช้ type เดียวกันตั้งแต่ DB ถึง frontend, CI/CD ที่เดียว |
| **ข้อเสีย** | repo ใหญ่ขึ้นเรื่อยๆ, CI อาจช้าถ้าไม่มี incremental build, ต้องใช้ tool จัดการ (Turborepo, Nx) |
| **เหมาะกับ** | ทีมเล็ก, ระบบที่มีหลาย app ใช้ backend เดียวกัน, TypeScript full-stack |

#### 1.5.2 Polyrepo (แยก repo ต่อ app)

```
kamonkit-api/         ← repo สำหรับ Backend
kamonkit-web/         ← repo สำหรับ Web App
kamonkit-mobile/      ← repo สำหรับ Mobile App
```

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | แต่ละ repo เบา CI เร็ว, แยก deploy อิสระ, เหมาะกับทีมหลายทีม |
| **ข้อเสีย** | share code ยาก (ต้อง publish package), type อาจไม่ sync กัน, refactor ต้องแก้หลาย repo |
| **เหมาะกับ** | ทีมใหญ่ที่แต่ละทีมดูแล repo แยก, Microservices |

---

### 1.6 รูปแบบ Deployment

#### 1.6.1 VPS / Self-hosted

**แนวคิด:** เช่า Virtual Private Server (เช่น DigitalOcean, Vultr, Linode) หรือเครื่อง server ของตัวเอง จัดการเองทั้งหมด

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | ควบคุมได้ 100%, ราคาถูกกว่า managed service, ไม่ vendor lock-in, เหมาะกับระบบที่ต้องเก็บข้อมูลในประเทศ |
| **ข้อเสีย** | ต้อง manage เอง (OS update, security, backup, monitoring), ถ้า server ล่มต้องแก้เอง, scale ช้า (ต้องเพิ่มเครื่อง) |
| **ค่าใช้จ่ายโดยประมาณ** | 500–2,000 บาท/เดือน สำหรับ SME |

#### 1.6.2 PaaS (Platform as a Service)

**แนวคิด:** Push code แล้ว platform จัดการ deploy, scaling, SSL, monitoring ให้ เช่น Railway, Fly.io, Render, Vercel

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | deploy ง่ายมาก (git push แล้วจบ), ไม่ต้อง manage server, มี SSL/monitoring/auto-restart ให้, เหมาะกับทีมเล็ก |
| **ข้อเสีย** | ราคาแพงกว่า VPS เมื่อ scale ขึ้น, ควบคุมได้น้อยกว่า, อาจ vendor lock-in, server อยู่ต่างประเทศ |
| **ค่าใช้จ่ายโดยประมาณ** | 1,000–5,000 บาท/เดือน |

#### 1.6.3 Cloud Managed (AWS / GCP / Azure)

**แนวคิด:** ใช้บริการ managed ของ cloud provider เช่น RDS (database), ECS (container), S3 (storage), CloudFront (CDN)

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | scale ได้ไม่จำกัด, reliability สูง (SLA 99.9%+), มีบริการครบ (DB, storage, queue, CDN, notification), มี region ในเอเชีย (Singapore) |
| **ข้อเสีย** | เรียนรู้ยาก, ราคาคาดเดายาก (pay-per-use), vendor lock-in, overkill สำหรับระบบเล็ก |
| **ค่าใช้จ่ายโดยประมาณ** | 2,000–20,000+ บาท/เดือน |

#### 1.6.4 Container Orchestration (Docker + Kubernetes)

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **ข้อดี** | deploy consistent ทุก environment, scale horizontal ได้, self-healing, rolling update |
| **ข้อเสีย** | ซับซ้อนมาก, ต้องมี DevOps expertise, overkill สำหรับระบบเล็ก, ค่าใช้จ่าย cluster สูง |
| **เหมาะกับ** | ระบบ Microservices ขนาดใหญ่, ทีมที่มี DevOps |

#### ตารางสรุปเปรียบเทียบ Deployment

| เกณฑ์ | VPS | PaaS | Cloud Managed | K8s |
|-------|-----|------|---------------|-----|
| ความง่าย | ปานกลาง | ง่ายที่สุด | ยาก | ยากที่สุด |
| ราคา (SME) | ถูกที่สุด | ปานกลาง | แพง | แพงที่สุด |
| ควบคุมได้ | สูงสุด | ต่ำ | สูง | สูง |
| เหมาะกับทีมเล็ก | ใช่ | ใช่ | ไม่แนะนำ | ไม่ |

---

## ส่วนที่ 2: วิเคราะห์ลักษณะระบบกมลกิจยานยนต์

จากเอกสาร requirements (prompts/motorcycle-shop-system.md) สรุปลักษณะสำคัญที่ส่งผลต่อการเลือก System Design:

### 2.1 ขนาดและปริมาณการใช้งาน

| ปัจจัย | ค่าประมาณ |
|--------|----------|
| จำนวนผู้ใช้พร้อมกัน | 5–15 คน (เสมียน 1–2, ช่าง 2–3, เจ้าของ 1, Collector 2–3) |
| ปริมาณ transaction/วัน | 50–200 รายการ (ค่างวด, งานซ่อม, ขายอะไหล่) |
| ปริมาณข้อมูล | หลักพัน–หมื่น record (ลูกค้า, สัญญา, อะไหล่) |
| ปริมาณไฟล์ (รูปลูกค้า/งานซ่อม) | หลักร้อย–พัน ไฟล์ |
| จำนวนสาขา (ปัจจุบัน) | 1 สาขา (แต่ต้องรองรับหลายสาขาในอนาคต) |

**สรุป:** ขนาดเล็ก ไม่ต้อง scale สูง — database เดียวรับไหวสบาย

### 2.2 ความซับซ้อนของ Business Logic

| Domain | ความซับซ้อน | รายละเอียด |
|--------|------------|-----------|
| ระบบผ่อนชำระ | **สูง** | คำนวณ Flat Rate/Effective Rate, ค่าปรับ 3 แบบ, ตัดสดก่อนกำหนด (60/70/100%), ประกันรถหาย, กฎหมายเช่าซื้อ |
| ระบบ Collection | **สูง** | Offline-first mobile, GPS logging, Field payment + ใบเสร็จชั่วคราว, Cash handover workflow |
| ระบบบัญชีประจำวัน | **ปานกลาง** | เปิด-ปิดรอบ, แยกประเภทเงินเข้า-ออก, ตรวจนับเงินสด, auto-link จาก module อื่น |
| ระบบซ่อม | **ปานกลาง** | เปิด-ปิด Job, ตัดสต็อกอัตโนมัติ, ค่าแรง, warranty |
| ระบบอะไหล่ | **ปานกลาง** | Barcode, stock movement, ตรวจนับ, low stock alert |
| ภาษี/กฎหมาย | **สูง** | VAT 7%, ใบกำกับภาษี, ภ.ง.ด.1, ประกันสังคม, PDPA |

**สรุป:** Business logic ซับซ้อนหลาย domain → ต้องมี module boundary ชัดเจน

### 2.3 ข้อกำหนดเฉพาะ

| ข้อกำหนด | ผลต่อ Design |
|----------|-------------|
| **Multi-platform** (Web + Mobile) | ต้อง shared backend API |
| **Offline-first** สำหรับ Collector | Mobile app ต้อง cache ข้อมูล + sync เมื่อมี internet |
| **ACID Transaction** (การเงิน, สัญญา, สต็อก) | Database เดียวง่ายกว่า → ไม่ควรแยก DB per service |
| **Audit Trail** | ต้องบันทึกทุก change → database trigger หรือ application-level logging |
| **Solo Developer** + Claude Code | ต้องเลือก tech ที่ maintain ง่าย ไม่ over-engineer |
| **PDPA compliance** | ต้อง mask PII, consent logging, access control |
| **Thermal Printer + Barcode Scanner** | Web App ต้องรองรับ USB device |
| **รองรับหลายสาขาในอนาคต** | ออกแบบ schema ให้มี branch_id แต่ยังไม่ต้อง implement |

---

## ส่วนที่ 3: คำแนะนำสำหรับระบบกมลกิจยานยนต์

### 3.1 สถาปัตยกรรมระบบ → **Modular Monolith**

**เหตุผล:**

1. **Solo Developer** — Microservices ซับซ้อนเกินไป ต้อง manage หลาย service, distributed transaction, service discovery ไม่คุ้มกับทีม 1 คน
2. **ACID Transaction จำเป็น** — การสร้างสัญญาเช่าซื้อต้องทำหลายอย่างพร้อมกัน (เปลี่ยนสถานะรถ + สร้างงวด + บันทึกเงินดาวน์ + ตัดสต็อก) ถ้าอย่างใดอย่างหนึ่งล้มเหลวต้อง rollback ทั้งหมด → Single DB ทำได้ง่ายกว่ามาก
3. **Business logic ซับซ้อนหลาย domain** — ต้องแบ่ง module ชัดเจน ไม่ให้โค้ดพันกัน แต่ไม่จำเป็นต้องแยก deploy
4. **รองรับอนาคต** — ถ้าวันหนึ่งต้องแยก module ไปเป็น Microservice (เช่น Collection module) ก็ทำได้ง่ายเพราะ boundary ชัดอยู่แล้ว
5. **Deploy ง่าย** — process เดียว server เดียว ไม่ต้องจัดการ network ระหว่าง service

**โครงสร้าง Module ที่แนะนำ:**

| Module | ความรับผิดชอบ |
|--------|-------------|
| `vehicle` | ข้อมูลรถ, สถานะรถ, trade-in, warranty |
| `customer` | ลูกค้า, ผู้ค้ำประกัน, รูปลูกค้า |
| `sales` | สัญญาซื้อขาย, เช่าซื้อ, เลขสัญญา, เล่มทะเบียน |
| `installment` | ค่างวด, ค่าปรับ, ตัดสด, ประกันรถหาย |
| `repair` | งานซ่อม, Job Order, ค่าแรง |
| `inventory` | อะไหล่, สต็อก, Barcode, Purchase Order |
| `insurance-tax` | พ.ร.บ., ต่อภาษี, ตรวจสภาพ |
| `cash` | เงินสดประจำวัน, เปิด-ปิดรอบ, Cash Transaction |
| `collection` | ทวงหนี้, Collector task, Field payment |
| `report` | รายงานทุกประเภท |
| `auth` | Authentication, RBAC, Audit trail |
| `notification` | LINE Notify, Push Notification |

---

### 3.2 รูปแบบ API → **tRPC (หลัก) + REST (เสริม)**

**เหตุผล:**

1. **Solo Developer ใช้ TypeScript ทั้ง stack** — tRPC ให้ type safety ตั้งแต่ server ถึง client โดยไม่ต้องเขียน API spec แยก ลดโอกาสเกิด bug
2. **พัฒนาเร็ว** — ไม่ต้องเขียน API endpoint + DTO + type definition + client SDK แยก ทุกอย่างอยู่ใน TypeScript
3. **Monorepo** — tRPC ทำงานดีที่สุดใน monorepo เพราะ share type ได้โดยตรง
4. **REST เสริม** — สำหรับ endpoint ที่ต้อง public (เช่น webhook จาก LINE, file upload) หรือ third-party เรียกใช้

---

### 3.3 รูปแบบ Frontend → **Hybrid (Next.js) สำหรับ Web + React Native (Expo) สำหรับ Mobile**

**เหตุผล:**

**Web App (เสมียน + ช่าง):**
1. **Next.js** — รวม SSR + SPA ในตัว เลือกได้ว่าหน้าไหนต้อง SSR (login, print preview) หน้าไหนเป็น SPA (dashboard, form ต่างๆ)
2. **ทำงานบน PC** — หน้าจอใหญ่ ใช้ keyboard + mouse + barcode scanner ได้ดี
3. **ไม่ต้อง SEO** — ระบบภายในไม่ต้อง index โดย search engine แต่ SSR ช่วยให้ initial load เร็ว

**Mobile App (เจ้าของ + Collector):**
1. **React Native (Expo)** — share ความรู้ TypeScript + React กับ Web App, Expo ทำให้ build/deploy ง่าย
2. **Offline-first** — Collector ต้องทำงานในพื้นที่ไม่มี internet → React Native + SQLite/WatermelonDB สำหรับ local storage
3. **GPS + Camera** — React Native เข้าถึง native API ได้ดี (ถ่ายรูป, GPS logging)
4. **Push Notification** — Expo Push Notification service ง่ายกว่า FCM/APNs โดยตรง

**ทำไมไม่ใช้ PWA แทน Mobile App?**
- PWA บน iOS มีข้อจำกัดเรื่อง background GPS, Push Notification, Offline storage
- Collector ต้องใช้ GPS อย่างต่อเนื่อง + offline → Native App ทำได้ดีกว่ามาก

---

### 3.4 Database → **Single PostgreSQL Database**

**เหตุผล:**

1. **ACID Transaction สำคัญ** — สร้างสัญญา + เปลี่ยนสถานะรถ + บันทึกค่างวด + ตัดเงินดาวน์ ต้องทำใน transaction เดียว
2. **ปริมาณข้อมูลน้อย** — หลักพัน-หมื่น record PostgreSQL รับไหวสบายๆ (PostgreSQL รับได้หลักล้าน row ต่อ table)
3. **JOIN ข้าม module** — รายงานต้อง JOIN ลูกค้า + สัญญา + ค่างวด + เงินสด → single DB ทำได้ง่ายและเร็ว
4. **Audit Trail** — ใช้ trigger หรือ application-level ก็ได้ ไม่จำเป็นต้อง Event Sourcing
5. **JSON support** — PostgreSQL มี JSONB สำหรับข้อมูลที่โครงสร้างไม่แน่นอน
6. **Full-text search** — ค้นหาลูกค้า/อะไหล่ด้วยข้อความภาษาไทยได้

**ไม่เลือก DB per Service:** เพราะไม่ได้ใช้ Microservices — การแยก DB จะทำให้ JOIN ไม่ได้ และต้องจัดการ distributed transaction ซึ่งซับซ้อนเกินไป

**ไม่เลือก CQRS/Event Sourcing:** เพราะระบบขนาดเล็ก การแยก read/write model เพิ่มความซับซ้อนโดยไม่จำเป็น Audit Trail ใช้ log table ธรรมดาก็พอ

---

### 3.5 โครงสร้างโปรเจกต์ → **Monorepo (Turborepo)**

**เหตุผล:**

1. **Share Type** — type ของ Customer, Contract, Installment ใช้ร่วมกันทั้ง Backend, Web, Mobile — Monorepo share ได้โดยตรง ไม่ต้อง publish package
2. **Share Validation** — Zod schema เดียวกันใช้ได้ทั้ง frontend form validation และ backend API validation
3. **Solo Developer** — คนเดียวดูแล จัดการ repo เดียวง่ายกว่าหลาย repo
4. **Claude Code** — AI-assisted coding ทำงานได้ดีกับ monorepo เพราะเห็นภาพรวมทั้งระบบ
5. **Turborepo** — incremental build ทำให้ CI ไม่ช้าแม้ repo จะใหญ่

```
kamonkit/
├── apps/
│   ├── api/                    ← Backend (Hono.js + Bun)
│   │   └── src/
│   │       ├── modules/        ← แบ่งตาม business domain
│   │       │   ├── vehicle/
│   │       │   ├── customer/
│   │       │   ├── sales/
│   │       │   ├── installment/
│   │       │   ├── repair/
│   │       │   ├── inventory/
│   │       │   ├── insurance-tax/
│   │       │   ├── cash/
│   │       │   ├── collection/
│   │       │   ├── report/
│   │       │   ├── auth/
│   │       │   └── notification/
│   │       ├── middleware/
│   │       └── jobs/           ← Scheduled jobs
│   ├── web/                    ← Web App (Next.js)
│   └── mobile/                 ← Mobile App (Expo)
├── packages/
│   ├── shared/                 ← Type + Validation (Zod)
│   ├── db/                     ← Drizzle schema + migration
│   └── ui/                     ← Shared UI components
├── turbo.json
└── package.json
```

---

### 3.6 Deployment → **VPS (เริ่มต้น) + PaaS (ทางเลือก)**

**เหตุผล:**

1. **ราคาประหยัด** — VPS (เช่น DigitalOcean/Vultr) ราคา 500–1,500 บาท/เดือน สำหรับ SME คุ้มค่าที่สุด
2. **ข้อมูลอยู่ในควบคุม** — ระบบมีข้อมูลส่วนบุคคล (PDPA) และข้อมูลการเงิน → ควบคุม server เองได้ดีกว่า
3. **Solo Developer manage ได้** — Monolith deploy เป็น Docker container 1 ตัว + PostgreSQL 1 ตัว ไม่ซับซ้อน
4. **ถ้าไม่อยาก manage server** → ใช้ PaaS (Railway/Fly.io) แทน deploy ง่ายกว่าแต่ราคาสูงกว่า
5. **Mobile App** → Expo EAS Build + OTA Update (ไม่ต้อง deploy เอง)

**โครงสร้าง Deploy:**

```
┌─────────────────────────────────────┐
│           VPS (DigitalOcean)        │
│                                     │
│  ┌───────────┐  ┌───────────────┐  │
│  │  Caddy     │  │  Backend API  │  │
│  │ (Reverse   │─▶│  (Docker)     │  │
│  │  Proxy +   │  └───────────────┘  │
│  │  SSL)      │  ┌───────────────┐  │
│  │            │─▶│  Web App      │  │
│  └───────────┘  │  (Docker)     │  │
│                  └───────────────┘  │
│  ┌───────────────┐                  │
│  │  PostgreSQL    │  ← Docker       │
│  └───────────────┘                  │
│  ┌───────────────┐                  │
│  │  Redis         │  ← Docker       │
│  └───────────────┘                  │
│  ┌───────────────┐                  │
│  │  File Storage  │  ← Local volume │
│  └───────────────┘                  │
│                                     │
│  Backup: Daily → Cloud Storage     │
└─────────────────────────────────────┘

Mobile App: Expo EAS Build
           → App Store / Google Play
           → OTA Update
```

---

### สรุปคำแนะนำทั้งหมด

| มิติ | เลือก | เหตุผลหลัก |
|------|-------|-----------|
| **สถาปัตยกรรม** | Modular Monolith | Solo dev + ACID transaction + Business logic ซับซ้อนหลาย domain |
| **API** | tRPC (หลัก) + REST (เสริม) | Type safety end-to-end + พัฒนาเร็ว + Monorepo |
| **Frontend (Web)** | Next.js (Hybrid SSR/SPA) | เสมียน/ช่างใช้ PC + Barcode + Printer |
| **Frontend (Mobile)** | React Native (Expo) | Collector ต้อง Offline + GPS + Camera |
| **Database** | Single PostgreSQL | ACID + JOIN ข้าม module + ข้อมูลไม่เยอะ |
| **โครงสร้างโปรเจกต์** | Monorepo (Turborepo) | Share type/validation + Solo dev + AI-assisted coding |
| **Deployment** | VPS + Docker | ราคาประหยัด + ควบคุมข้อมูลได้ + ไม่ซับซ้อน |
