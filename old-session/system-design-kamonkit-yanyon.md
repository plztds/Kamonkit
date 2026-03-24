# System Design: ระบบจัดการร้านกมลกิจยานยนต์

> เอกสารฉบับนี้ครอบคลุมการออกแบบระบบทั้งหมด ตั้งแต่ Architecture, Database, Tech Stack, Business Logic, UX/UI ไปจนถึง Phase Planning

---

## สารบัญ

1. [System Architecture Design](#1-system-architecture-design)
2. [Database Design (ERD & Schema)](#2-database-design-erd--schema)
3. [Tech Stack Recommendation](#3-tech-stack-recommendation)
4. [Business Logic Design](#4-business-logic-design)
5. [UX/UI Direction](#5-uxui-direction)
6. [Pain Points & Pitfalls](#6-pain-points--pitfalls)
7. [Phase Planning](#7-phase-planning)

---

## 1. System Architecture Design

### 1.1 ภาพรวม Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS (4 กลุ่มผู้ใช้)                    │
│                                                                 │
│  ┌──────────────────────┐      ┌──────────────────────────┐    │
│  │  Web App (PC)         │      │  Mobile App (มือถือ)      │    │
│  │  - เสมียน (Clerk)      │      │  - เจ้าของร้าน (Owner)    │    │
│  │  - ช่างซ่อม (Tech)     │      │  - พนง.ติดตามหนี้         │    │
│  │  Next.js + shadcn/ui  │      │    (Collector)            │    │
│  │                       │      │  React Native (Expo)      │    │
│  │  อุปกรณ์เสริม:         │      │                          │    │
│  │  - Thermal Printer    │      │                          │    │
│  │  - Barcode Scanner    │      │                          │    │
│  └──────────┬───────────┘      └────────────┬─────────────┘    │
│             │                                │                  │
└─────────────┼────────────────────────────────┼─────────────────┘
              │                                │
              ▼                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API GATEWAY / REVERSE PROXY                │
│                         (Caddy / Nginx)                         │
│                   ┌─────────────────────┐                       │
│                   │   Rate Limiting      │                      │
│                   │   SSL Termination    │                      │
│                   │   Request Routing    │                      │
│                   └─────────────────────┘                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND API SERVER                            │
│                     (Shared Backend)                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    API Layer (REST)                      │    │
│  │              Hono.js on Bun Runtime                      │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │                 Authentication Layer                     │    │
│  │              JWT + Role-Based Access Control             │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │                  Service Layer                           │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │    │
│  │  │ Vehicle  │ │ Customer │ │  Sales   │ │  Repair  │   │    │
│  │  │ Service  │ │ Service  │ │ Service  │ │ Service  │   │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │    │
│  │  │Inventory │ │Insurance │ │  Cash    │ │ Report   │   │    │
│  │  │ Service  │ │ Service  │ │ Service  │ │ Service  │   │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │    │
│  │  ┌──────────┐                                           │    │
│  │  │Collection│  ← ระบบติดตามหนี้                          │    │
│  │  │ Service  │                                           │    │
│  │  └──────────┘                                           │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │                  Data Access Layer                       │    │
│  │                  Drizzle ORM + Zod                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATA & INFRA LAYER                         │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │   PostgreSQL      │  │   Redis           │                    │
│  │   (Primary DB)    │  │   (Cache/Session/ │                    │
│  │                   │  │    Message Queue)  │                    │
│  │  - Master Data    │  │                   │                    │
│  │  - Transactions   │  │  - Session Store  │                    │
│  │  - Reports        │  │  - Rate Limiting  │                    │
│  │  - Collection     │  │  - Cache Layer    │                    │
│  │    Activities     │  │  - BullMQ Queue   │                    │
│  └──────────────────┘  └──────────────────┘                     │
│                                                                 │
│  ┌──────────────────┐                                           │
│  │   File Storage    │                                           │
│  │   (Local/S3)      │                                           │
│  │                   │                                           │
│  │  - สำเนาเอกสาร     │                                           │
│  │  - รูปถ่ายงานซ่อม    │                                           │
│  │  - ใบเสร็จ/สัญญา    │                                           │
│  └──────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

User Groups:
┌──────────────────────┬────────────────┬─────────────────────┐
│ กลุ่ม                 │ Platform หลัก   │ อุปกรณ์เสริม          │
├──────────────────────┼────────────────┼─────────────────────┤
│ เจ้าของร้าน (Owner)   │ Mobile         │ -                   │
│ เสมียน (Clerk)        │ Web/PC         │ Thermal Printer     │
│ ช่างซ่อม (Technician) │ Web/PC         │ Barcode Scanner     │
│ พนง.ติดตามหนี้        │ Mobile         │ -                   │
│  (Collector)         │                │                     │
└──────────────────────┴────────────────┴─────────────────────┘
```

### 1.2 Architecture Decisions

| Decision | เลือก | เหตุผล |
|----------|-------|--------|
| API Style | REST (JSON) | เรียบง่าย เข้าใจง่าย เครื่องมือรองรับเยอะ เหมาะกับ CRUD-heavy app |
| Backend Runtime | Bun | เร็วกว่า Node.js, มี built-in test runner, รองรับ TypeScript natively |
| Backend Framework | Hono.js | Lightweight, เร็ว, Type-safe, ทำงานได้ดีบน Bun |
| ORM | Drizzle ORM | Type-safe, Performance ดี, SQL-like syntax เข้าใจง่าย, Migration ดี |
| Database | PostgreSQL | Mature, ACID compliant, JSON support, Full-text search |
| Cache / Queue | Redis + BullMQ | Session, Rate limiting, Caching, Message Queue สำหรับ background jobs |
| Auth Strategy | JWT + RBAC | Stateless, รองรับ Multi-platform, Role-based permissions (4 roles) |
| Validation | Zod | Type-safe validation, ใช้ร่วมกับ Drizzle ได้ดี, Share schema ระหว่าง FE/BE |
| Monorepo | Turborepo | Share code ระหว่าง packages, Incremental builds |
| Web Frontend | Next.js 15 | SSR, สำหรับ Clerk + Technician บน PC |
| Mobile | React Native (Expo) | สำหรับ Owner + Collector บนมือถือ |
| UI Library | shadcn/ui + Tailwind | Customizable, Accessible, Copy-paste components |

### 1.3 Monorepo Structure

```
kamonkit-yanyon/
├── apps/
│   ├── api/                    # Backend API (Hono.js + Bun)
│   │   ├── src/
│   │   │   ├── routes/         # Route handlers แยกตาม module
│   │   │   ├── services/       # Business logic layer
│   │   │   ├── middleware/     # Auth, logging, error handling
│   │   │   ├── jobs/           # Scheduled jobs (penalty calc, alerts)
│   │   │   └── utils/
│   │   └── drizzle/            # DB migrations & seeds
│   │
│   ├── web/                    # Web App สำหรับ Clerk + Technician (Next.js)
│   │   ├── src/
│   │   │   ├── app/            # App Router pages
│   │   │   │   ├── (clerk)/    # Clerk pages (สัญญา, รับชำระ, เงินสด)
│   │   │   │   └── (tech)/     # Technician pages (งานซ่อม, อะไหล่)
│   │   │   ├── components/     # UI components
│   │   │   ├── hooks/          # Custom hooks
│   │   │   └── lib/            # Utilities
│   │   └── public/
│   │
│   └── mobile/                 # Mobile App สำหรับ Owner + Collector (Expo)
│       ├── src/
│       │   ├── screens/
│       │   │   ├── owner/      # Owner dashboard, รายงาน
│       │   │   └── collector/  # ติดตามหนี้, บันทึกผล, รับเงิน
│       │   ├── components/
│       │   ├── navigation/
│       │   └── hooks/
│       └── app.json
│
├── packages/
│   ├── shared/                 # Shared types, constants, utils
│   │   ├── src/
│   │   │   ├── types/          # TypeScript interfaces (Vehicle, Customer, etc.)
│   │   │   ├── constants/      # Enums, status codes
│   │   │   ├── validators/     # Zod schemas (shared validation)
│   │   │   └── utils/          # Shared utility functions
│   │   └── package.json
│   │
│   ├── db/                     # Database schema & client (Drizzle)
│   │   ├── src/
│   │   │   ├── schema/         # Table definitions
│   │   │   ├── migrations/     # SQL migrations
│   │   │   └── client.ts       # DB connection
│   │   └── package.json
│   │
│   └── ui/                     # Shared UI components (for web)
│       ├── src/
│       │   ├── components/     # shadcn/ui based components
│       │   └── styles/
│       └── package.json
│
├── turbo.json
├── package.json
└── docker-compose.yml          # PostgreSQL + Redis for local dev
```

### 1.4 API Design (RESTful Endpoints)

```
# === Master Data ===
GET/POST             /api/v1/vehicles
GET/PUT/DELETE       /api/v1/vehicles/:id
GET/POST             /api/v1/customers
GET/PUT              /api/v1/customers/:id
GET                  /api/v1/customers/:id/history    # ประวัติซื้อ/ซ่อม
GET/POST             /api/v1/parts
GET/PUT              /api/v1/parts/:id
GET                  /api/v1/parts/low-stock           # อะไหล่ใกล้หมด
GET/POST             /api/v1/parts/categories

# === Sales & Contracts ===
GET/POST             /api/v1/sales/contracts
GET/PUT              /api/v1/sales/contracts/:id
GET                  /api/v1/sales/contracts/:id/installments
POST                 /api/v1/sales/contracts/:id/payments      # รับชำระค่างวด
GET                  /api/v1/sales/contracts/:id/penalties      # ค่าปรับ
PUT                  /api/v1/sales/contracts/:id/registration   # จัดการเล่มทะเบียน

# === Repair / Service ===
GET/POST             /api/v1/repairs/jobs
GET/PUT              /api/v1/repairs/jobs/:id
PUT                  /api/v1/repairs/jobs/:id/status            # เปลี่ยนสถานะ
POST                 /api/v1/repairs/jobs/:id/parts             # เพิ่มอะไหล่ในงานซ่อม
POST                 /api/v1/repairs/jobs/:id/complete          # ปิดงานซ่อม

# === Inventory ===
POST                 /api/v1/inventory/stock-in
POST                 /api/v1/inventory/stock-out
GET                  /api/v1/inventory/balance
GET                  /api/v1/inventory/movements/:partId        # ประวัติเคลื่อนไหว

# === Insurance & Tax ===
GET/POST             /api/v1/services/insurance-tax
GET/PUT              /api/v1/services/insurance-tax/:id
PUT                  /api/v1/services/insurance-tax/:id/status

# === Daily Cash ===
POST                 /api/v1/cash/open-day                     # ตั้งยอดเริ่มต้น
GET/POST             /api/v1/cash/transactions                  # รายรับ-รายจ่าย
POST                 /api/v1/cash/close-day                     # สรุปยอดประจำวัน
GET                  /api/v1/cash/summary/:date

# === Reports ===
GET                  /api/v1/reports/daily-summary
GET                  /api/v1/reports/sales
GET                  /api/v1/reports/receivables
GET                  /api/v1/reports/collection-aging
GET                  /api/v1/reports/repairs
GET                  /api/v1/reports/inventory

# === Collection (ติดตามหนี้) ===
GET                  /api/v1/collections/assignments            # งานติดตามที่มอบหมาย
POST                 /api/v1/collections/assignments            # มอบหมายงานติดตาม
GET                  /api/v1/collections/assignments/:id
PUT                  /api/v1/collections/assignments/:id/status # อัปเดตสถานะงาน
GET                  /api/v1/collections/my-tasks               # งานของ Collector (mobile)
GET                  /api/v1/collections/debtors                # รายชื่อลูกหนี้ค้างชำระ
GET                  /api/v1/collections/debtors/:customerId    # ประวัติลูกหนี้ (สัญญา, รถ, ค้ำประกัน)
POST                 /api/v1/collections/activities              # บันทึกกิจกรรมติดตาม
GET                  /api/v1/collections/activities/:assignmentId # ประวัติกิจกรรมติดตาม
POST                 /api/v1/collections/field-payment           # รับชำระเงินสดหน้างาน
POST                 /api/v1/collections/submit-cash             # ส่งเงินกลับร้าน

# === Auth ===
POST                 /api/v1/auth/login
POST                 /api/v1/auth/refresh
POST                 /api/v1/auth/logout
GET                  /api/v1/auth/me
```

---

## 2. Database Design (ERD & Schema)

### 2.1 Entity Relationship Diagram (Text-based)

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│    users      │     │    customers      │     │    vehicles       │
├──────────────┤     ├──────────────────┤     ├──────────────────┤
│ id (PK)      │     │ id (PK)          │     │ id (PK)          │
│ username     │     │ citizen_id       │     │ brand            │
│ password_hash│     │ first_name       │     │ model            │
│ full_name    │     │ last_name        │     │ year             │
│ role (enum)  │     │ phone            │     │ color            │
│ is_active    │     │ phone2           │     │ chassis_no (UQ)  │
│ created_at   │     │ address          │     │ engine_no (UQ)   │
│ updated_at   │     │ subdistrict      │     │ license_plate    │
└──────────────┘     │ district         │     │ condition (enum) │
                     │ province         │     │   new/used       │
                     │ postal_code      │     │ status (enum)    │
                     │ notes            │     │   in_stock/sold/ │
                     │ created_at       │     │   reserved       │
                     │ updated_at       │     │ cost_price       │
                     └────────┬─────────┘     │ selling_price    │
                              │               │ customer_id (FK) │
                              │               │ notes            │
                              │               │ created_at       │
                              │               │ updated_at       │
                              │               └────────┬─────────┘
                              │                        │
                              ▼                        ▼
                     ┌──────────────────────────────────────────┐
                     │            sales_contracts                │
                     ├──────────────────────────────────────────┤
                     │ id (PK)                                  │
                     │ contract_number (UQ)                     │
                     │ customer_id (FK → customers)             │
                     │ vehicle_id (FK → vehicles)               │
                     │ contract_date                            │
                     │ payment_type (enum: cash/installment)    │
                     │ total_price                              │
                     │ down_payment                             │
                     │ finance_amount                           │
                     │ interest_rate                            │
                     │ total_installments (จำนวนงวด)             │
                     │ installment_amount (ยอดต่องวด)            │
                     │ due_day_of_month (วันครบกำหนดของเดือน)     │
                     │ grace_period_days (วันผ่อนผัน)            │
                     │ penalty_type (enum: fixed/percent_daily/ │
                     │   percent_monthly)                       │
                     │ penalty_rate                             │
                     │ registration_status (enum)               │
                     │   held_by_shop / returned_to_customer    │
                     │ status (enum)                            │
                     │   active/completed/defaulted             │
                     │ clerk_id (FK → users)                    │
                     │ notes                                    │
                     │ created_at                               │
                     │ updated_at                               │
                     └────────────────────┬─────────────────────┘
                                          │
                     ┌────────────────────┼────────────────────┐
                     ▼                    ▼                    ▼
          ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
          │  installments     │ │installment_      │ │ penalty_         │
          │                  │ │  payments         │ │   records        │
          ├──────────────────┤ ├──────────────────┤ ├──────────────────┤
          │ id (PK)          │ │ id (PK)          │ │ id (PK)          │
          │ contract_id (FK) │ │ installment_id   │ │ installment_id   │
          │ installment_no   │ │   (FK)           │ │   (FK)           │
          │ due_date         │ │ payment_date     │ │ penalty_amount   │
          │ amount           │ │ amount_paid      │ │ days_overdue     │
          │ status (enum)    │ │ penalty_paid     │ │ calculated_at    │
          │   pending/paid/  │ │ payment_method   │ │ is_paid          │
          │   overdue/       │ │ received_by (FK) │ │ paid_at          │
          │   partial        │ │ receipt_number   │ └──────────────────┘
          │ paid_date        │ │ notes            │
          │ created_at       │ │ created_at       │
          └──────────────────┘ └──────────────────┘


┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  part_categories  │     │    parts          │     │ stock_movements  │
├──────────────────┤     ├──────────────────┤     ├──────────────────┤
│ id (PK)          │◄────│ id (PK)          │────►│ id (PK)          │
│ name             │     │ category_id (FK) │     │ part_id (FK)     │
│ description      │     │ part_code (UQ)   │     │ type (enum)      │
│ created_at       │     │ barcode          │     │   in/out         │
└──────────────────┘     │ name             │     │ quantity         │
                         │ description      │     │ reference_type   │
                         │ unit             │     │   (repair/sale/  │
                         │ cost_price       │     │    adjustment/   │
                         │ selling_price    │     │    stock_in)     │
                         │ quantity_in_stock│     │ reference_id     │
                         │ min_stock_level  │     │ unit_cost        │
                         │ location         │     │ performed_by(FK) │
                         │ is_active        │     │ notes            │
                         │ created_at       │     │ created_at       │
                         │ updated_at       │     └──────────────────┘
                         └────────┬─────────┘
                                  │
                                  │ (ใช้ใน repair)
                                  ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   repair_jobs     │────►│ repair_job_parts  │     │ repair_job_      │
├──────────────────┤     ├──────────────────┤     │   services       │
│ id (PK)          │     │ id (PK)          │     ├──────────────────┤
│ job_number (UQ)  │     │ job_id (FK)      │     │ id (PK)          │
│ customer_id (FK) │     │ part_id (FK)     │     │ job_id (FK)      │
│ vehicle_plate    │     │ quantity         │     │ description      │
│ vehicle_brand    │     │ unit_price       │     │ labor_cost       │
│ vehicle_model    │     │ total_price      │     │ created_at       │
│ technician_id(FK)│     │ created_at       │     └──────────────────┘
│ description      │     └──────────────────┘
│ status (enum)    │
│   open/in_prog/  │     ┌──────────────────┐
│   completed/     │     │insurance_tax_    │
│   cancelled      │     │  services        │
│ total_parts_cost │     ├──────────────────┤
│ total_labor_cost │     │ id (PK)          │
│ total_amount     │     │ customer_id (FK) │
│ payment_status   │     │ vehicle_plate    │
│   paid/unpaid/   │     │ service_type     │
│   partial        │     │   (enum: prb/tax/│
│ paid_amount      │     │    both)         │
│ notes            │     │ service_date     │
│ created_at       │     │ expiry_date      │
│ updated_at       │     │ fee_amount       │
│ completed_at     │     │ service_charge   │
└──────────────────┘     │ total_amount     │
                         │ payment_status   │
                         │ document_type    │
                         │   (enum: original│
                         │    /copy)        │
                         │ document_status  │
                         │   (enum: received│
                         │    /processing/  │
                         │    completed/    │
                         │    returned)     │
                         │ registration_    │
                         │   source (enum)  │
                         │   shop_held/     │
                         │   customer_      │
                         │   provided       │
                         │ clerk_id (FK)    │
                         │ notes            │
                         │ created_at       │
                         │ updated_at       │
                         └──────────────────┘


┌──────────────────┐     ┌──────────────────┐
│  daily_cash_     │     │ cash_            │
│   sessions       │     │   transactions   │
├──────────────────┤     ├──────────────────┤
│ id (PK)          │     │ id (PK)          │
│ session_date(UQ) │     │ session_id (FK)  │
│ opening_balance  │     │ type (enum)      │
│ total_income     │     │   income/expense │
│ total_expense    │     │ category (enum)  │
│ closing_balance  │     │   installment/   │
│ status (enum)    │     │   repair/parts/  │
│   open/closed    │     │   insurance_tax/ │
│ opened_by (FK)   │     │   vehicle_sale/  │
│ closed_by (FK)   │     │   petty_cash/    │
│ notes            │     │   supplier/      │
│ created_at       │     │   other          │
│ updated_at       │     │ amount           │
└──────────────────┘     │ description      │
                         │ reference_type   │
                         │ reference_id     │
                         │ performed_by(FK) │
                         │ created_at       │
                         └──────────────────┘
```

### 2.2 Drizzle Schema (TypeScript)

```typescript
// packages/db/src/schema/enums.ts

import { pgEnum } from 'drizzle-orm/pg-core';

export const userRoleEnum = pgEnum('user_role', ['admin', 'clerk', 'technician', 'collector']);
export const vehicleConditionEnum = pgEnum('vehicle_condition', ['new', 'used']);
export const vehicleStatusEnum = pgEnum('vehicle_status', ['in_stock', 'sold', 'reserved']);
export const paymentTypeEnum = pgEnum('payment_type', ['cash', 'installment']);
export const installmentStatusEnum = pgEnum('installment_status', ['pending', 'paid', 'overdue', 'partial']);
export const contractStatusEnum = pgEnum('contract_status', ['active', 'completed', 'defaulted']);
export const penaltyTypeEnum = pgEnum('penalty_type', ['fixed', 'percent_daily', 'percent_monthly']);
export const registrationStatusEnum = pgEnum('registration_status', ['held_by_shop', 'returned_to_customer']);
export const repairJobStatusEnum = pgEnum('repair_job_status', ['open', 'in_progress', 'completed', 'cancelled']);
export const paymentStatusEnum = pgEnum('payment_status', ['paid', 'unpaid', 'partial']);
export const stockMovementTypeEnum = pgEnum('stock_movement_type', ['in', 'out']);
export const stockReferenceTypeEnum = pgEnum('stock_reference_type', ['repair', 'sale', 'adjustment', 'stock_in']);
export const serviceTypeEnum = pgEnum('service_type', ['prb', 'tax', 'both']);
export const documentTypeEnum = pgEnum('document_type', ['original', 'copy']);
export const documentStatusEnum = pgEnum('document_status', ['received', 'processing', 'completed', 'returned']);
export const registrationSourceEnum = pgEnum('registration_source', ['shop_held', 'customer_provided']);
export const cashTransactionTypeEnum = pgEnum('cash_transaction_type', ['income', 'expense']);
export const cashCategoryEnum = pgEnum('cash_category', [
  'installment', 'repair', 'parts', 'insurance_tax',
  'vehicle_sale', 'petty_cash', 'supplier', 'other'
]);
export const cashSessionStatusEnum = pgEnum('cash_session_status', ['open', 'closed']);

// Collection (ติดตามหนี้)
export const collectionAssignmentStatusEnum = pgEnum('collection_assignment_status', [
  'assigned', 'in_progress', 'completed', 'cancelled'
]);
export const collectionActivityResultEnum = pgEnum('collection_activity_result', [
  'contacted_promise_to_pay',   // ติดต่อได้ นัดจ่าย
  'contacted_cannot_pay',       // ติดต่อได้ แต่ยังจ่ายไม่ได้
  'not_home',                   // ไม่อยู่บ้าน
  'unreachable',                // ติดต่อไม่ได้
  'paid_on_site',               // จ่ายเงินหน้างาน
  'partial_paid_on_site',       // จ่ายบางส่วนหน้างาน
  'other'                       // อื่นๆ
]);
export const fieldPaymentStatusEnum = pgEnum('field_payment_status', [
  'collected',     // เก็บเงินแล้ว ยังไม่ส่งร้าน
  'submitted',     // ส่งเงินกลับร้านแล้ว
  'verified'       // ร้านตรวจรับแล้ว
]);
```

```typescript
// packages/db/src/schema/users.ts

import { pgTable, uuid, varchar, boolean, timestamp } from 'drizzle-orm/pg-core';
import { userRoleEnum } from './enums';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  username: varchar('username', { length: 50 }).unique().notNull(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  fullName: varchar('full_name', { length: 100 }).notNull(),
  role: userRoleEnum('role').notNull(),
  isActive: boolean('is_active').default(true).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

```typescript
// packages/db/src/schema/customers.ts

import { pgTable, uuid, varchar, text, timestamp } from 'drizzle-orm/pg-core';

export const customers = pgTable('customers', {
  id: uuid('id').primaryKey().defaultRandom(),
  citizenId: varchar('citizen_id', { length: 13 }).unique(),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  phone: varchar('phone', { length: 20 }).notNull(),
  phone2: varchar('phone2', { length: 20 }),
  address: text('address'),
  subdistrict: varchar('subdistrict', { length: 100 }),
  district: varchar('district', { length: 100 }),
  province: varchar('province', { length: 100 }),
  postalCode: varchar('postal_code', { length: 5 }),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

```typescript
// packages/db/src/schema/vehicles.ts

import { pgTable, uuid, varchar, text, numeric, timestamp } from 'drizzle-orm/pg-core';
import { vehicleConditionEnum, vehicleStatusEnum } from './enums';
import { customers } from './customers';

export const vehicles = pgTable('vehicles', {
  id: uuid('id').primaryKey().defaultRandom(),
  brand: varchar('brand', { length: 50 }).notNull(),
  model: varchar('model', { length: 100 }).notNull(),
  year: varchar('year', { length: 4 }),
  color: varchar('color', { length: 30 }),
  chassisNo: varchar('chassis_no', { length: 50 }).unique().notNull(),
  engineNo: varchar('engine_no', { length: 50 }).unique().notNull(),
  licensePlate: varchar('license_plate', { length: 20 }),
  condition: vehicleConditionEnum('condition').notNull(),
  status: vehicleStatusEnum('status').default('in_stock').notNull(),
  costPrice: numeric('cost_price', { precision: 12, scale: 2 }).notNull(),
  sellingPrice: numeric('selling_price', { precision: 12, scale: 2 }).notNull(),
  customerId: uuid('customer_id').references(() => customers.id),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

```typescript
// packages/db/src/schema/sales.ts

import { pgTable, uuid, varchar, date, numeric, integer, timestamp, text } from 'drizzle-orm/pg-core';
import { paymentTypeEnum, contractStatusEnum, penaltyTypeEnum,
         registrationStatusEnum, installmentStatusEnum } from './enums';
import { customers } from './customers';
import { vehicles } from './vehicles';
import { users } from './users';

export const salesContracts = pgTable('sales_contracts', {
  id: uuid('id').primaryKey().defaultRandom(),
  contractNumber: varchar('contract_number', { length: 20 }).unique().notNull(),
  customerId: uuid('customer_id').references(() => customers.id).notNull(),
  vehicleId: uuid('vehicle_id').references(() => vehicles.id).notNull(),
  contractDate: date('contract_date').notNull(),
  paymentType: paymentTypeEnum('payment_type').notNull(),
  totalPrice: numeric('total_price', { precision: 12, scale: 2 }).notNull(),
  downPayment: numeric('down_payment', { precision: 12, scale: 2 }).default('0'),
  financeAmount: numeric('finance_amount', { precision: 12, scale: 2 }).default('0'),
  interestRate: numeric('interest_rate', { precision: 5, scale: 2 }).default('0'),
  totalInstallments: integer('total_installments').default(0),
  installmentAmount: numeric('installment_amount', { precision: 12, scale: 2 }).default('0'),
  dueDayOfMonth: integer('due_day_of_month'),
  gracePeriodDays: integer('grace_period_days').default(7),
  penaltyType: penaltyTypeEnum('penalty_type').default('fixed'),
  penaltyRate: numeric('penalty_rate', { precision: 10, scale: 2 }).default('0'),
  registrationStatus: registrationStatusEnum('registration_status'),
  status: contractStatusEnum('status').default('active').notNull(),
  clerkId: uuid('clerk_id').references(() => users.id).notNull(),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

export const installments = pgTable('installments', {
  id: uuid('id').primaryKey().defaultRandom(),
  contractId: uuid('contract_id').references(() => salesContracts.id).notNull(),
  installmentNo: integer('installment_no').notNull(),
  dueDate: date('due_date').notNull(),
  amount: numeric('amount', { precision: 12, scale: 2 }).notNull(),
  status: installmentStatusEnum('status').default('pending').notNull(),
  paidDate: date('paid_date'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const installmentPayments = pgTable('installment_payments', {
  id: uuid('id').primaryKey().defaultRandom(),
  installmentId: uuid('installment_id').references(() => installments.id).notNull(),
  paymentDate: date('payment_date').notNull(),
  amountPaid: numeric('amount_paid', { precision: 12, scale: 2 }).notNull(),
  penaltyPaid: numeric('penalty_paid', { precision: 12, scale: 2 }).default('0'),
  paymentMethod: varchar('payment_method', { length: 30 }).default('cash'),
  receivedBy: uuid('received_by').references(() => users.id).notNull(),
  receiptNumber: varchar('receipt_number', { length: 30 }),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const penaltyRecords = pgTable('penalty_records', {
  id: uuid('id').primaryKey().defaultRandom(),
  installmentId: uuid('installment_id').references(() => installments.id).notNull(),
  penaltyAmount: numeric('penalty_amount', { precision: 12, scale: 2 }).notNull(),
  daysOverdue: integer('days_overdue').notNull(),
  calculatedAt: timestamp('calculated_at').defaultNow().notNull(),
  isPaid: boolean('is_paid').default(false).notNull(),
  paidAt: timestamp('paid_at'),
});

import { boolean } from 'drizzle-orm/pg-core';
```

```typescript
// packages/db/src/schema/parts.ts

import { pgTable, uuid, varchar, text, numeric, integer, boolean, timestamp } from 'drizzle-orm/pg-core';
import { stockMovementTypeEnum, stockReferenceTypeEnum } from './enums';
import { users } from './users';

export const partCategories = pgTable('part_categories', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 100 }).unique().notNull(),
  description: text('description'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const parts = pgTable('parts', {
  id: uuid('id').primaryKey().defaultRandom(),
  categoryId: uuid('category_id').references(() => partCategories.id),
  partCode: varchar('part_code', { length: 50 }).unique().notNull(),
  barcode: varchar('barcode', { length: 100 }),
  name: varchar('name', { length: 200 }).notNull(),
  description: text('description'),
  unit: varchar('unit', { length: 20 }).default('ชิ้น'),
  costPrice: numeric('cost_price', { precision: 12, scale: 2 }).notNull(),
  sellingPrice: numeric('selling_price', { precision: 12, scale: 2 }).notNull(),
  quantityInStock: integer('quantity_in_stock').default(0).notNull(),
  minStockLevel: integer('min_stock_level').default(5).notNull(),
  location: varchar('location', { length: 50 }),
  isActive: boolean('is_active').default(true).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

export const stockMovements = pgTable('stock_movements', {
  id: uuid('id').primaryKey().defaultRandom(),
  partId: uuid('part_id').references(() => parts.id).notNull(),
  type: stockMovementTypeEnum('type').notNull(),
  quantity: integer('quantity').notNull(),
  referenceType: stockReferenceTypeEnum('reference_type').notNull(),
  referenceId: uuid('reference_id'),
  unitCost: numeric('unit_cost', { precision: 12, scale: 2 }),
  performedBy: uuid('performed_by').references(() => users.id).notNull(),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

```typescript
// packages/db/src/schema/repairs.ts

import { pgTable, uuid, varchar, text, numeric, integer, timestamp } from 'drizzle-orm/pg-core';
import { repairJobStatusEnum, paymentStatusEnum } from './enums';
import { customers } from './customers';
import { users } from './users';
import { parts } from './parts';

export const repairJobs = pgTable('repair_jobs', {
  id: uuid('id').primaryKey().defaultRandom(),
  jobNumber: varchar('job_number', { length: 20 }).unique().notNull(),
  customerId: uuid('customer_id').references(() => customers.id),
  vehiclePlate: varchar('vehicle_plate', { length: 20 }).notNull(),
  vehicleBrand: varchar('vehicle_brand', { length: 50 }),
  vehicleModel: varchar('vehicle_model', { length: 100 }),
  technicianId: uuid('technician_id').references(() => users.id).notNull(),
  description: text('description'),
  status: repairJobStatusEnum('status').default('open').notNull(),
  totalPartsCost: numeric('total_parts_cost', { precision: 12, scale: 2 }).default('0'),
  totalLaborCost: numeric('total_labor_cost', { precision: 12, scale: 2 }).default('0'),
  totalAmount: numeric('total_amount', { precision: 12, scale: 2 }).default('0'),
  paymentStatus: paymentStatusEnum('payment_status').default('unpaid').notNull(),
  paidAmount: numeric('paid_amount', { precision: 12, scale: 2 }).default('0'),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
  completedAt: timestamp('completed_at'),
});

export const repairJobParts = pgTable('repair_job_parts', {
  id: uuid('id').primaryKey().defaultRandom(),
  jobId: uuid('job_id').references(() => repairJobs.id).notNull(),
  partId: uuid('part_id').references(() => parts.id).notNull(),
  quantity: integer('quantity').notNull(),
  unitPrice: numeric('unit_price', { precision: 12, scale: 2 }).notNull(),
  totalPrice: numeric('total_price', { precision: 12, scale: 2 }).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const repairJobServices = pgTable('repair_job_services', {
  id: uuid('id').primaryKey().defaultRandom(),
  jobId: uuid('job_id').references(() => repairJobs.id).notNull(),
  description: varchar('description', { length: 255 }).notNull(),
  laborCost: numeric('labor_cost', { precision: 12, scale: 2 }).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

```typescript
// packages/db/src/schema/insurance-tax.ts

import { pgTable, uuid, varchar, date, numeric, text, timestamp } from 'drizzle-orm/pg-core';
import { serviceTypeEnum, paymentStatusEnum, documentTypeEnum,
         documentStatusEnum, registrationSourceEnum } from './enums';
import { customers } from './customers';
import { users } from './users';

export const insuranceTaxServices = pgTable('insurance_tax_services', {
  id: uuid('id').primaryKey().defaultRandom(),
  customerId: uuid('customer_id').references(() => customers.id).notNull(),
  vehiclePlate: varchar('vehicle_plate', { length: 20 }).notNull(),
  serviceType: serviceTypeEnum('service_type').notNull(),
  serviceDate: date('service_date').notNull(),
  expiryDate: date('expiry_date'),
  feeAmount: numeric('fee_amount', { precision: 12, scale: 2 }).notNull(),
  serviceCharge: numeric('service_charge', { precision: 12, scale: 2 }).default('0'),
  totalAmount: numeric('total_amount', { precision: 12, scale: 2 }).notNull(),
  paymentStatus: paymentStatusEnum('payment_status').default('unpaid').notNull(),
  documentType: documentTypeEnum('document_type'),
  documentStatus: documentStatusEnum('document_status').default('received'),
  registrationSource: registrationSourceEnum('registration_source').notNull(),
  clerkId: uuid('clerk_id').references(() => users.id).notNull(),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

```typescript
// packages/db/src/schema/cash.ts

import { pgTable, uuid, date, numeric, text, timestamp } from 'drizzle-orm/pg-core';
import { cashSessionStatusEnum, cashTransactionTypeEnum, cashCategoryEnum } from './enums';
import { users } from './users';

export const dailyCashSessions = pgTable('daily_cash_sessions', {
  id: uuid('id').primaryKey().defaultRandom(),
  sessionDate: date('session_date').unique().notNull(),
  openingBalance: numeric('opening_balance', { precision: 12, scale: 2 }).notNull(),
  totalIncome: numeric('total_income', { precision: 12, scale: 2 }).default('0'),
  totalExpense: numeric('total_expense', { precision: 12, scale: 2 }).default('0'),
  closingBalance: numeric('closing_balance', { precision: 12, scale: 2 }),
  status: cashSessionStatusEnum('status').default('open').notNull(),
  openedBy: uuid('opened_by').references(() => users.id).notNull(),
  closedBy: uuid('closed_by').references(() => users.id),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

export const cashTransactions = pgTable('cash_transactions', {
  id: uuid('id').primaryKey().defaultRandom(),
  sessionId: uuid('session_id').references(() => dailyCashSessions.id).notNull(),
  type: cashTransactionTypeEnum('type').notNull(),
  category: cashCategoryEnum('category').notNull(),
  amount: numeric('amount', { precision: 12, scale: 2 }).notNull(),
  description: text('description'),
  referenceType: varchar('reference_type', { length: 50 }),
  referenceId: uuid('reference_id'),
  performedBy: uuid('performed_by').references(() => users.id).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

```typescript
// packages/db/src/schema/collections.ts

import { pgTable, uuid, varchar, date, text, numeric, integer, timestamp } from 'drizzle-orm/pg-core';
import { collectionAssignmentStatusEnum, collectionActivityResultEnum,
         fieldPaymentStatusEnum } from './enums';
import { users } from './users';
import { customers } from './customers';
import { salesContracts } from './sales';

// มอบหมายงานติดตามหนี้ให้ Collector
export const collectionAssignments = pgTable('collection_assignments', {
  id: uuid('id').primaryKey().defaultRandom(),
  contractId: uuid('contract_id').references(() => salesContracts.id).notNull(),
  customerId: uuid('customer_id').references(() => customers.id).notNull(),
  collectorId: uuid('collector_id').references(() => users.id).notNull(),
  assignedBy: uuid('assigned_by').references(() => users.id).notNull(),
  assignedDate: date('assigned_date').notNull(),
  dueDate: date('due_date'),                    // กำหนดต้องติดตามภายในวันที่
  priority: integer('priority').default(0),      // 0=ปกติ, 1=ด่วน, 2=ด่วนมาก
  totalOverdueAmount: numeric('total_overdue_amount', { precision: 12, scale: 2 }).notNull(),
  overdueInstallments: integer('overdue_installments').notNull(), // จำนวนงวดที่ค้าง
  status: collectionAssignmentStatusEnum('status').default('assigned').notNull(),
  notes: text('notes'),
  completedAt: timestamp('completed_at'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

// บันทึกกิจกรรมติดตามหนี้ (แต่ละครั้งที่ Collector ออกไปหา/โทรหา)
export const collectionActivities = pgTable('collection_activities', {
  id: uuid('id').primaryKey().defaultRandom(),
  assignmentId: uuid('assignment_id').references(() => collectionAssignments.id).notNull(),
  collectorId: uuid('collector_id').references(() => users.id).notNull(),
  activityDate: timestamp('activity_date').defaultNow().notNull(),
  result: collectionActivityResultEnum('result').notNull(),
  promiseDate: date('promise_date'),             // วันที่ลูกค้านัดจ่าย (ถ้ามี)
  amountCollected: numeric('amount_collected', { precision: 12, scale: 2 }).default('0'),
  fieldReceiptNumber: varchar('field_receipt_number', { length: 30 }), // เลขที่ใบเสร็จชั่วคราว
  fieldPaymentStatus: fieldPaymentStatusEnum('field_payment_status'),
  gpsLatitude: numeric('gps_latitude', { precision: 10, scale: 7 }),
  gpsLongitude: numeric('gps_longitude', { precision: 10, scale: 7 }),
  notes: text('notes'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

### 2.3 Indexes สำคัญ

```sql
-- ค้นหาลูกค้า
CREATE INDEX idx_customers_phone ON customers(phone);
CREATE INDEX idx_customers_name ON customers(first_name, last_name);
CREATE INDEX idx_customers_citizen_id ON customers(citizen_id);

-- ค้นหารถ
CREATE INDEX idx_vehicles_license_plate ON vehicles(license_plate);
CREATE INDEX idx_vehicles_status ON vehicles(status);

-- สัญญาที่ active
CREATE INDEX idx_sales_contracts_status ON sales_contracts(status);
CREATE INDEX idx_sales_contracts_customer ON sales_contracts(customer_id);

-- งวดที่ค้างชำระ
CREATE INDEX idx_installments_status ON installments(status);
CREATE INDEX idx_installments_due_date ON installments(due_date);
CREATE INDEX idx_installments_contract ON installments(contract_id);

-- งานซ่อม
CREATE INDEX idx_repair_jobs_status ON repair_jobs(status);
CREATE INDEX idx_repair_jobs_technician ON repair_jobs(technician_id);
CREATE INDEX idx_repair_jobs_created ON repair_jobs(created_at);

-- อะไหล่
CREATE INDEX idx_parts_barcode ON parts(barcode);
CREATE INDEX idx_parts_low_stock ON parts(quantity_in_stock, min_stock_level)
  WHERE is_active = true;

-- เงินสด
CREATE INDEX idx_cash_transactions_session ON cash_transactions(session_id);
CREATE INDEX idx_cash_transactions_category ON cash_transactions(category);
CREATE INDEX idx_daily_cash_sessions_date ON daily_cash_sessions(session_date);

-- ติดตามหนี้ (Collection)
CREATE INDEX idx_collection_assignments_collector ON collection_assignments(collector_id);
CREATE INDEX idx_collection_assignments_status ON collection_assignments(status);
CREATE INDEX idx_collection_assignments_customer ON collection_assignments(customer_id);
CREATE INDEX idx_collection_assignments_contract ON collection_assignments(contract_id);
CREATE INDEX idx_collection_activities_assignment ON collection_activities(assignment_id);
CREATE INDEX idx_collection_activities_collector ON collection_activities(collector_id);
CREATE INDEX idx_collection_activities_payment_status ON collection_activities(field_payment_status)
  WHERE field_payment_status = 'collected';
```

---

## 3. Tech Stack Recommendation

> **หลักการเลือก:** เสถียร, เล็ก, เร็ว, ประหยัด — เหมาะกับ SME ที่ต้องการระบบที่ดูแลง่าย ค่าใช้จ่ายต่ำ

### 3.1 สรุป Tech Stack ที่เลือก

| Layer | Technology | เหตุผลหลัก |
|-------|-----------|-----------|
| **Runtime** | Bun | เร็ว, Built-in TS, Test runner, Package manager ในตัว |
| **Backend Framework** | Hono.js | Lightweight, Type-safe, เร็วสุดในกลุ่ม |
| **ORM** | Drizzle ORM | Type-safe SQL, ไม่มี binary dependency, Migration ดี |
| **Validation** | Zod | Type inference, Share schema FE/BE |
| **Database** | PostgreSQL 16 | ACID, JSON, Full-text search, Mature |
| **Cache / Queue** | Redis + BullMQ | Session, Cache, Message Queue สำหรับ background jobs |
| **Web Frontend** | Next.js 15 (App Router) | SSR, สำหรับ Clerk + Technician บน PC |
| **UI Library** | shadcn/ui + Tailwind CSS | Customizable, Accessible, ไม่ lock-in |
| **Mobile** | React Native (Expo) | สำหรับ Owner + Collector, share TS types กับ backend/web |
| **State Management** | TanStack Query | Server state caching, Auto-refetch |
| **Monorepo** | Turborepo | Incremental builds, Task caching |
| **Auth** | JWT (jose library) | Lightweight, Stateless, 4 roles |
| **Scheduled Jobs** | BullMQ (Redis-based) | คำนวณค่าปรับ, มอบหมายงานติดตาม, แจ้งเตือน |
| **Barcode** | USB HID (keyboard emulation) + html5-qrcode | USB scanner ทำงานเป็น keyboard อัตโนมัติ, html5-qrcode สำหรับกล้อง/QR |
| **PDF/Receipts** | @react-pdf/renderer | ใบเสร็จ PDF, สัญญา, ใบเสร็จชั่วคราว Collector |
| **Thermal Printer** | QZ Tray + ESC/POS | พิมพ์ใบเสร็จ 80mm ผ่าน USB, ไม่มี dialog, ตัดกระดาษอัตโนมัติ |
| **Deployment** | VPS (Hetzner) + Docker | ประหยัด, ควบคุมได้เต็มที่ |
| **CI/CD** | GitHub Actions | Auto deploy on push, Run tests |

---

### 3.2 เปรียบเทียบทางเลือก Tech Stack (10 หมวด)

#### หมวด 1: Backend Runtime

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **Bun** ✅ | เร็วมาก (3-5x Node), Built-in TS/Test/Bundler, ใช้ RAM น้อย | Ecosystem ยังไม่ mature เท่า Node, บาง package อาจไม่ compatible | โปรเจกต์ใหม่ที่ต้องการ speed + simplicity | ฟรี, ~50MB install |
| **Node.js** | Ecosystem ใหญ่สุด, เสถียรมาก, Community ใหญ่, LTS support | ช้ากว่า Bun, ต้องติดตั้ง TS compiler แยก | โปรเจกต์ที่ต้องการ stability สูงสุด | ฟรี, ~100MB install |
| **Deno** | Security by default, Built-in TS, Web standard APIs | Ecosystem เล็ก, npm compatibility ยังไม่ 100% | โปรเจกต์ที่เน้น security | ฟรี, ~80MB install |

> **เลือก Bun** — เร็ว, เบา, Built-in TS native, ลดเครื่องมือที่ต้องตั้งค่า

#### หมวด 2: Backend Framework

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **Hono.js** ✅ | เร็วสุด, Type-safe, Multi-runtime, API surface เล็ก | Ecosystem เล็กกว่า Express, ยังใหม่ | API ที่ต้องการ type-safe + performance | ฟรี, ~14KB |
| **Elysia** | เร็วมากบน Bun, Type-safe, Plugin system ดี | Bun-only (ไม่ portable), Community เล็ก | Bun-first projects | ฟรี, ~25KB |
| **Fastify** | เร็ว, Plugin architecture ดี, JSON Schema validation | TS support ปานกลาง, Boilerplate มากกว่า Hono | Large-scale REST APIs | ฟรี, ~2MB |
| **Express** | Ecosystem ใหญ่สุด, ง่ายมาก, ทุกคนรู้จัก | ช้า, ไม่มี built-in type-safe, ไม่มี async error handling | Legacy projects, prototype | ฟรี, ~600KB |
| **NestJS** | Enterprise-grade, DI, Modular architecture | ใหญ่มาก, Learning curve สูง, Overhead เยอะ | Enterprise apps ที่มีทีมใหญ่ | ฟรี, ~15MB+ |

> **เลือก Hono.js** — type-safe มาก, เร็วสุด, ทำงานบน Bun ได้ดี, เล็กสุด

#### หมวด 3: ORM / Query Builder

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **Drizzle** ✅ | SQL-like syntax, Type-safe, Performance ดีสุด, Migration tool, ไม่มี binary | ยังใหม่, Community เล็กกว่า Prisma | โปรเจกต์ที่คนเขียน SQL เป็น | ฟรี, ~500KB |
| **Prisma** | DX ดีมาก, Schema DSL ชัดเจน, Prisma Studio, Community ใหญ่ | มี binary engine (~15MB), Startup ช้ากว่า, ไม่ SQL-like | โปรเจกต์ที่ต้องการ DX สูง + ไม่เก่ง SQL | ฟรี (OSS), ~15MB binary |
| **Kysely** | SQL-like, Type-safe, Performance ดี, Lightweight | ไม่มี Migration tool ในตัว, ต้องเขียน query เอง | Query builder only (ไม่ต้องการ ORM features) | ฟรี, ~200KB |
| **TypeORM** | Entity pattern คุ้นเคย, Decorator-based | Performance แย่, Bug เยอะ, Maintenance ช้า | Legacy projects, คนมาจาก Java/C# | ฟรี, ~5MB |

> **เลือก Drizzle** — SQL-like เข้าใจง่าย, ไม่มี binary, เร็ว, Migration ดี

#### หมวด 4: Database

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **PostgreSQL** ✅ | ACID, JSON, Full-text search, Extensions (pg_cron), Mature, ฟรี | ใช้ RAM มากกว่า SQLite/MySQL, ต้อง config tuning | Production app ที่ต้องการ data integrity + advanced features | ฟรี, RAM ~256MB+ |
| **MySQL** | เร็วสำหรับ read-heavy, Replication ง่าย, คุ้นเคยกันดี | JSON support แย่กว่า PG, ไม่มี CTE ที่ดี (version เก่า) | Web apps ทั่วไป, WordPress ecosystem | ฟรี, RAM ~200MB+ |
| **SQLite (embedded)** | ไม่ต้อง server, ไฟล์เดียว, เร็วมาก read, Zero config | ไม่รองรับ concurrent write ดี, ไม่มี built-in replication | Embedded apps, Edge, ต้นแบบ | ฟรี, ~1MB |
| **Turso (LibSQL)** | SQLite-compatible + replicated, Edge-friendly, ฟรี tier | ยังใหม่, Ecosystem เล็ก, ไม่เหมาะ write-heavy | Edge/distributed apps ที่ต้องการ SQLite + sync | Free tier 9GB, จ่ายจาก $29/mo |

> **เลือก PostgreSQL** — ACID สำหรับธุรกรรมเงิน, JSON สำหรับ flexible data, Mature + ฟรี

#### หมวด 5: Web Frontend

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **Next.js (SSR)** ✅ | SSR/SSG, File-based routing, React ecosystem, Server Actions | Bundle ใหญ่, Complexity สูงขึ้นเรื่อยๆ, Vercel-centric, Satisfaction ลดลง (68% → 55% ใน State of JS 2025) | Full-stack web apps ที่ต้องการ SEO + SSR | ฟรี (OSS), Build ~50MB+ |
| **Vite + React (SPA)** | เร็วมาก dev/build, เบา, Simple, HMR ดี | ไม่มี SSR built-in, SEO ต้องทำเพิ่ม | Internal tools / Dashboard ที่ไม่ต้องการ SEO | ฟรี, Build ~5MB |
| **SvelteKit** | Svelte เร็ว+เบา, Built-in SSR, ง่ายกว่า React, Satisfaction อันดับ 1 ใน State of JS 2025 | Ecosystem เล็กกว่า React มาก, หาคนยากกว่า | โปรเจกต์ที่ต้องการ performance สูง + ทีมเล็ก | ฟรี, Build ~3MB |
| **Remix / React Router 7** | Nested routing ดี, Web standards, Data loading pattern ดี | Community เล็กกว่า Next.js | Web apps ที่ต้องการ progressive enhancement | ฟรี, Build ~20MB |
| **TanStack Start** | Type-safe routing + data loading, File-based routing, React ecosystem | ยังใหม่มาก (stable 2024), Community เล็ก | โปรเจกต์ที่ต้องการ type-safe routing สูงสุด | ฟรี |

> **เลือก Next.js** — Pragmatic choice: SSR สำหรับ Clerk + Tech ที่ใช้บน PC, React ecosystem share กับ Mobile แม้ satisfaction จะลดลงใน 2025 survey แต่ยังเป็นตัวเลือกที่ mature ที่สุด

#### หมวด 6: Mobile Framework

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **React Native (Expo)** ✅ | Share TS types กับ backend/web, Expo = วิธีที่ React Native docs แนะนำอย่างเป็นทางการ (2026), New Architecture (Fabric + TurboModules) เป็น default แล้ว, OTA Update ผ่าน EAS Update ไม่ต้อง submit App Store | Performance ไม่เท่า Native | ทีม TS/React ที่ต้องการ cross-platform | ฟรี (OSS), App ~20-40MB |
| **Flutter** | UI สวย, Performance ดี, Single codebase iOS+Android | Dart language (ไม่ share types กับ TS), ใหญ่กว่า | Apps ที่เน้น custom UI + animation | ฟรี, App ~15-30MB |
| **Capacitor** | Web tech (HTML/CSS/JS), ใช้ Web App ที่มีอยู่ได้เลย | Performance แย่กว่า RN/Flutter, UX ไม่ native | Web app ที่ต้องการ wrap เป็น mobile app | ฟรี, App ~10-20MB |
| **PWA** | ไม่ต้อง install, ใช้ Web tech, ไม่ต้องผ่าน App Store | ไม่มี Push Notification บน iOS (จำกัด), Offline จำกัด | Apps ง่ายๆ ที่ไม่ต้องการ native features | ฟรี, ไม่ต้อง build |

> **เลือก React Native (Expo)** — share TypeScript types ทั้งระบบ, Owner + Collector ใช้บนมือถือ, OTA updates ผ่าน EAS Update, New Architecture เป็น default แล้วใน 2026

#### หมวด 7: UI Component Library

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **shadcn/ui** ✅ | Copy-paste (ไม่ lock-in), Accessible, Customizable, Tailwind-based | ต้อง copy เอง, ไม่มี theme builder | โปรเจกต์ที่ต้องการ full control + design system | ฟรี, per-component ~2-5KB |
| **Ant Design** | Component ครบมาก, Table/Form ดีเยี่ยม, Enterprise-ready | Bundle ใหญ่มาก (~1MB+), Style override ยาก, ดูเหมือนกันหมด | Enterprise apps, Admin panels | ฟรี, ~1.2MB |
| **Mantine** | DX ดีมาก, Hook library ดี, Theme system ยืดหยุ่น | Bundle ปานกลาง, Community เล็กกว่า Ant Design | โปรเจกต์ที่ต้องการ DX + flexibility | ฟรี, ~500KB |
| **DaisyUI + Tailwind** | ง่ายมาก, Class-based, Theme ง่าย, เบา | Component ไม่ครบ, Accessibility ต้องทำเอง | โปรเจกต์เล็กที่ต้องการ speed-to-build | ฟรี, ~30KB CSS |

> **เลือก shadcn/ui** — ไม่ lock-in, ปรับแต่งได้ทุกอย่าง, Accessible, ใช้กับ Tailwind ที่ Mobile ก็ใช้ได้ (NativeWind)

#### หมวด 8: Message Queue

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **BullMQ (Redis)** ✅ | Feature ครบ (retry, delay, priority, repeat), Dashboard (Bull Board), ใช้ Redis ที่มีอยู่ | ต้องมี Redis, Memory-based (ข้อมูลหายถ้า Redis ล่ม) | Background jobs, scheduled tasks ในระบบที่มี Redis อยู่แล้ว | ฟรี, ใช้ Redis ร่วม |
| **Bee-Queue** | Simple, เร็ว, Lightweight | Feature น้อยกว่า BullMQ, ไม่มี repeat jobs | Simple queue ที่ไม่ต้องการ features เยอะ | ฟรี, ใช้ Redis ร่วม |
| **RabbitMQ** | Enterprise-grade, Protocol standards (AMQP), Persistent | ต้อง deploy แยก, ใช้ RAM เยอะ (~200MB+), ซับซ้อนกว่า | Microservices, Enterprise ที่ต้องการ message guarantee | ฟรี (OSS), RAM ~200MB+ |
| **pg-boss (PostgreSQL)** | ใช้ PostgreSQL ที่มีอยู่ (ไม่ต้อง Redis), Persistent by default | ช้ากว่า Redis-based, Polling-based | โปรเจกต์ที่ไม่ต้องการ Redis | ฟรี, ใช้ PG ร่วม |

> **เลือก BullMQ** — มี Redis อยู่แล้ว, Features ครบ (คำนวณค่าปรับ, มอบหมายงาน Collector, แจ้งเตือน)

#### หมวด 9: In-memory DB / Cache

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย/ขนาด |
|----------|------|---------|---------|----------------|
| **Redis** ✅ | มาตรฐาน, Ecosystem ใหญ่สุด, Pub/Sub, Streams, Lua scripting | License เปลี่ยน (SSPL), Single-threaded | Cache, Session, Queue, Rate limiting | ฟรี (OSS), RAM ~50MB+ |
| **Valkey** | Redis fork (BSD license), Compatible 100%, Community-driven | ยังใหม่มาก, Community เล็กกว่า Redis | ต้องการ Redis-compatible + open license | ฟรี (BSD), RAM ~50MB+ |
| **KeyDB** | Multi-threaded Redis, เร็วกว่า Redis บาง workload | Community เล็ก, Snap Inc. owned | High-throughput ที่ต้องการ multi-thread | ฟรี (BSD), RAM ~50MB+ |
| **Dragonfly** | เร็วกว่า Redis 25x (benchmark), Multi-threaded, ใช้ RAM น้อยกว่า | ยังใหม่, Production adoption น้อย | High-performance caching | ฟรี (BSL), RAM ~100MB+ |
| **Memcached** | Simple, เร็ว, Multi-threaded | ไม่มี persistence, ไม่มี data structures, Feature น้อย | Simple cache-only use case | ฟรี, RAM ~50MB+ |

> **เลือก Redis** — มาตรฐาน, ใช้เป็นทั้ง Cache + Session + BullMQ Queue, Ecosystem ใหญ่สุด

#### หมวด 10: Deployment & Hosting

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ | ค่าใช้จ่าย |
|----------|------|---------|---------|-----------|
| **VPS + Docker** ✅ | ควบคุมเต็มที่, ราคาถูก, Persistent storage | ต้อง maintain เอง, DevOps knowledge ต้องมี | SME ที่ต้องการประหยัด + full control | Hetzner CX22: ~$5/mo, CX32: ~$15/mo |
| **Railway** | Deploy ง่ายมาก (git push), Free tier, Database included | แพงเมื่อ scale, ไม่มี persistent volume (บาง plan) | Prototype, Side projects | Free tier → $5/mo+ |
| **Fly.io** | Edge deployment, Docker-based, ราคาดี | ซับซ้อนกว่า Railway, Volume management ยาก | Apps ที่ต้องการ global edge | Free tier → $5/mo+ |
| **Hetzner Cloud** | ราคาถูกสุดใน EU, Specs ดี, Network ดี | Datacenter ใน EU เท่านั้น (latency จาก Asia) | VPS ราคาถูก specs สูง | CX22: €4.5/mo, CX32: €14/mo |
| **Self-hosted (ร้าน)** | ไม่ต้องพึ่ง Internet, Latency ต่ำสุด, ค่าใช้จ่ายครั้งเดียว | ต้อง maintain hardware, ไม่มี redundancy, สำรองข้อมูลยาก | ร้านที่ Internet ไม่เสถียร | Mini PC ~5,000-10,000 บาท (ครั้งเดียว) |

> **เลือก VPS + Docker (Hetzner)** — ราคาถูก ~$15/mo ได้ 4 vCPU + 8GB RAM, เพียงพอสำหรับร้านเดียว

---

### 3.3 Code Sharing Strategy

```
packages/shared/
├── types/          ← ใช้ร่วมกันทุก platform (Web + Mobile + API)
│   ├── customer.ts
│   ├── vehicle.ts
│   ├── contract.ts
│   ├── collection.ts   ← ใหม่: types สำหรับ Collector
│   └── ...
├── validators/     ← Zod schemas ใช้ validate ทั้ง FE และ BE
│   ├── customer.ts
│   ├── collection.ts   ← ใหม่: validation สำหรับ Collection
│   └── ...
├── constants/      ← Enums, status labels, config
│   ├── status.ts
│   ├── roles.ts        ← admin, clerk, technician, collector
│   └── ...
└── utils/          ← Pure functions ที่ใช้ร่วมกัน
    ├── formatCurrency.ts
    ├── calculatePenalty.ts
    └── ...
```

---

### 3.4 Hardware Integration

#### Barcode Scanner (USB HID)

USB Barcode Scanner ทำงานเป็น **keyboard emulation** — เมื่อสแกน barcode เครื่องจะส่ง keystrokes เหมือนพิมพ์จากแป้นพิมพ์ ตามด้วย `Enter`

**ข้อดี:** Web App รับได้เลยโดยไม่ต้องทำ Desktop App และไม่ต้องติดตั้ง driver พิเศษ

**Pattern: Global Keydown Listener**

แยกแยะ scanner (keystroke < 30ms) กับคนพิมพ์ (keystroke > 100ms) — UX แบบ 7-11 ไม่ต้อง focus input ไม่ต้องกด Enter เอง:

```typescript
// hooks/useBarcodeScanner.ts
export function useBarcodeScanner(onScan: (barcode: string) => void) {
  useEffect(() => {
    let buffer = '';
    let lastKeyTime = 0;

    const handleKeyDown = (e: KeyboardEvent) => {
      const now = Date.now();
      const timeDiff = now - lastKeyTime;
      lastKeyTime = now;

      // Scanner ส่ง keystroke เร็วมาก (< 30ms) — คนพิมพ์ช้ากว่า
      if (timeDiff > 100) buffer = ''; // reset ถ้า gap ใหญ่ (คนพิมพ์)

      if (e.key === 'Enter' && buffer.length > 0) {
        onScan(buffer);
        buffer = '';
      } else if (e.key.length === 1) {
        buffer += e.key;
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [onScan]);
}
```

#### Thermal Printer 80mm (QZ Tray)

Browser ไม่สามารถเข้าถึง USB printer ได้โดยตรง — ต้องมีตัวกลาง

**QZ Tray** คือ Java app ที่ install บนเครื่อง Clerk **ครั้งเดียว** รับคำสั่งจาก Web App ผ่าน WebSocket แล้วส่ง ESC/POS commands ไปยัง thermal printer

```
Web App (Next.js) → WebSocket → QZ Tray (localhost) → USB → Thermal Printer
```

**ข้อดี:**
- พิมพ์ทันที **ไม่มี print dialog**
- ตัดกระดาษอัตโนมัติ (ESC/POS cut command)
- ไม่ต้องทำ Desktop App (Electron/Tauri)
- รองรับ printer ทุกยี่ห้อที่ใช้ ESC/POS (Epson, Star, SNBC ฯลฯ)

```typescript
// lib/thermalPrinter.ts
import qz from 'qz-tray';

export async function printReceipt(lines: string[]) {
  await qz.websocket.connect();
  const config = qz.configs.create('PRINTER_NAME');
  const data = [
    '\x1B\x40',          // ESC @ — initialize printer
    ...lines.map(l => l + '\n'),
    '\x1D\x56\x00',      // GS V 0 — cut paper
  ];
  await qz.print(config, data);
  await qz.websocket.disconnect();
}
```

---

## 4. Business Logic Design

### 4.1 ระบบสินเชื่อผ่อนชำระ (Installment System)

#### Flow การสร้างสัญญาผ่อนชำระ

```
1. เสมียนเลือก "สร้างสัญญาซื้อขาย"
2. เลือกลูกค้า (หรือสร้างใหม่)
3. เลือกรถจาก Stock (status = in_stock)
4. กำหนดเงื่อนไข:
   - ราคาขาย (total_price)
   - เงินดาวน์ (down_payment)
   - จำนวนงวด (total_installments)
   - อัตราดอกเบี้ย (interest_rate) → optional, บวกเข้าราคาขายแล้ว
   - วันที่ครบกำหนดในแต่ละเดือน (due_day_of_month)
   - วันผ่อนผัน (grace_period_days)
   - ประเภทค่าปรับ (penalty_type) + อัตราค่าปรับ (penalty_rate)
5. ระบบคำนวณอัตโนมัติ:
   - finance_amount = total_price - down_payment
   - installment_amount = finance_amount / total_installments
   - สร้าง installment records ตามจำนวนงวด พร้อมกำหนด due_date
6. Vehicle status → "sold"
7. Registration status → "held_by_shop" (กรณีผ่อน)
8. บันทึก Cash Transaction สำหรับเงินดาวน์
```

#### Pseudo-code: สร้างสัญญาผ่อนชำระ

```typescript
async function createInstallmentContract(input: CreateContractInput) {
  return db.transaction(async (tx) => {
    // 1. ตรวจสอบรถว่ายัง in_stock
    const vehicle = await tx.query.vehicles.findFirst({
      where: eq(vehicles.id, input.vehicleId)
    });
    if (vehicle.status !== 'in_stock') throw new Error('Vehicle not available');

    // 2. คำนวณยอดผ่อน
    const financeAmount = input.totalPrice - input.downPayment;
    const installmentAmount = Math.ceil(financeAmount / input.totalInstallments);

    // 3. สร้างสัญญา
    const [contract] = await tx.insert(salesContracts).values({
      contractNumber: generateContractNumber(),
      customerId: input.customerId,
      vehicleId: input.vehicleId,
      contractDate: new Date(),
      paymentType: 'installment',
      totalPrice: input.totalPrice,
      downPayment: input.downPayment,
      financeAmount,
      totalInstallments: input.totalInstallments,
      installmentAmount,
      dueDayOfMonth: input.dueDayOfMonth,
      gracePeriodDays: input.gracePeriodDays ?? 7,
      penaltyType: input.penaltyType ?? 'fixed',
      penaltyRate: input.penaltyRate ?? 100, // 100 บาท default
      registrationStatus: 'held_by_shop',
      status: 'active',
      clerkId: input.clerkId,
    }).returning();

    // 4. สร้าง installment records
    const installmentRecords = [];
    for (let i = 1; i <= input.totalInstallments; i++) {
      const dueDate = calculateDueDate(
        input.contractDate,
        i,
        input.dueDayOfMonth
      );
      installmentRecords.push({
        contractId: contract.id,
        installmentNo: i,
        dueDate,
        amount: i === input.totalInstallments
          ? financeAmount - (installmentAmount * (input.totalInstallments - 1)) // งวดสุดท้ายปัดเศษ
          : installmentAmount,
        status: 'pending',
      });
    }
    await tx.insert(installments).values(installmentRecords);

    // 5. อัปเดตสถานะรถ
    await tx.update(vehicles)
      .set({ status: 'sold', customerId: input.customerId })
      .where(eq(vehicles.id, input.vehicleId));

    // 6. บันทึกเงินดาวน์เข้า Cash
    if (input.downPayment > 0) {
      await recordCashTransaction(tx, {
        type: 'income',
        category: 'vehicle_sale',
        amount: input.downPayment,
        description: `เงินดาวน์ สัญญา ${contract.contractNumber}`,
        referenceType: 'sales_contract',
        referenceId: contract.id,
        performedBy: input.clerkId,
      });
    }

    return contract;
  });
}
```

### 4.2 การคิดค่าปรับล่าช้า (Late Payment Penalty)

#### Logic การคำนวณ

```typescript
function calculatePenalty(
  installment: Installment,
  contract: SalesContract,
  asOfDate: Date = new Date()
): PenaltyCalculation {
  // ไม่คิดค่าปรับถ้ายังไม่เกิน due_date + grace_period
  const graceDueDate = addDays(installment.dueDate, contract.gracePeriodDays);

  if (asOfDate <= graceDueDate) {
    return { penaltyAmount: 0, daysOverdue: 0 };
  }

  const daysOverdue = differenceInDays(asOfDate, installment.dueDate);
  const daysChargeable = daysOverdue - contract.gracePeriodDays;
  let penaltyAmount = 0;

  switch (contract.penaltyType) {
    case 'fixed':
      // ค่าปรับคงที่ เช่น 100 บาท/งวดที่ค้าง (ไม่สะสม)
      penaltyAmount = contract.penaltyRate;
      break;

    case 'percent_daily':
      // เช่น 0.1% ต่อวัน ของยอดค่างวด
      penaltyAmount = installment.amount * (contract.penaltyRate / 100) * daysChargeable;
      break;

    case 'percent_monthly':
      // เช่น 3% ต่อเดือน ของยอดค่างวด (คิดเป็นรายเดือน ปัดขึ้น)
      const monthsOverdue = Math.ceil(daysChargeable / 30);
      penaltyAmount = installment.amount * (contract.penaltyRate / 100) * monthsOverdue;
      break;
  }

  return {
    penaltyAmount: Math.round(penaltyAmount), // ปัดเป็นจำนวนเต็ม
    daysOverdue,
    daysChargeable,
  };
}
```

#### Scheduled Job: อัปเดตสถานะ Overdue

```typescript
// ทำงานทุกวัน เวลา 00:05
cron.schedule('5 0 * * *', async () => {
  const today = new Date();

  // 1. หางวดที่เลยกำหนดแล้วยังไม่จ่าย
  const overdueInstallments = await db.query.installments.findMany({
    where: and(
      eq(installments.status, 'pending'),
      lt(installments.dueDate, today)
    ),
    with: { contract: true }
  });

  for (const inst of overdueInstallments) {
    // 2. อัปเดตสถานะเป็น overdue
    await db.update(installments)
      .set({ status: 'overdue' })
      .where(eq(installments.id, inst.id));

    // 3. คำนวณและบันทึกค่าปรับ
    const penalty = calculatePenalty(inst, inst.contract, today);
    if (penalty.penaltyAmount > 0) {
      await db.insert(penaltyRecords).values({
        installmentId: inst.id,
        penaltyAmount: penalty.penaltyAmount,
        daysOverdue: penalty.daysOverdue,
        calculatedAt: today,
        isPaid: false,
      });
    }
  }
});
```

### 4.3 การรับชำระค่างวด (Payment Processing)

```typescript
async function processInstallmentPayment(input: PaymentInput) {
  return db.transaction(async (tx) => {
    const installment = await tx.query.installments.findFirst({
      where: eq(installments.id, input.installmentId),
      with: { contract: true }
    });

    // 1. คำนวณค่าปรับ ณ วันที่จ่าย
    const penalty = calculatePenalty(installment, installment.contract, new Date());
    const totalDue = Number(installment.amount) + penalty.penaltyAmount;

    // 2. บันทึกการชำระ
    await tx.insert(installmentPayments).values({
      installmentId: input.installmentId,
      paymentDate: new Date(),
      amountPaid: input.amountPaid,
      penaltyPaid: penalty.penaltyAmount,
      paymentMethod: input.paymentMethod ?? 'cash',
      receivedBy: input.clerkId,
      receiptNumber: generateReceiptNumber(),
    });

    // 3. อัปเดตสถานะงวด
    await tx.update(installments).set({
      status: 'paid',
      paidDate: new Date(),
    }).where(eq(installments.id, input.installmentId));

    // 4. อัปเดต penalty record
    if (penalty.penaltyAmount > 0) {
      await tx.update(penaltyRecords).set({
        isPaid: true,
        paidAt: new Date(),
      }).where(
        and(
          eq(penaltyRecords.installmentId, input.installmentId),
          eq(penaltyRecords.isPaid, false)
        )
      );
    }

    // 5. ตรวจสอบว่าผ่อนครบหรือยัง
    const remainingInstallments = await tx.query.installments.findMany({
      where: and(
        eq(installments.contractId, installment.contractId),
        ne(installments.status, 'paid')
      )
    });

    if (remainingInstallments.length === 0) {
      // ผ่อนครบ → ปิดสัญญา + คืนเล่มทะเบียน
      await tx.update(salesContracts).set({
        status: 'completed',
        registrationStatus: 'returned_to_customer',
      }).where(eq(salesContracts.id, installment.contractId));
    }

    // 6. บันทึก Cash Transaction
    await recordCashTransaction(tx, {
      type: 'income',
      category: 'installment',
      amount: input.amountPaid,
      description: `ค่างวด #${installment.installmentNo} สัญญา ${installment.contract.contractNumber}`,
      referenceType: 'installment_payment',
      referenceId: input.installmentId,
      performedBy: input.clerkId,
    });

    return {
      installmentAmount: installment.amount,
      penaltyAmount: penalty.penaltyAmount,
      totalPaid: input.amountPaid,
      receiptNumber: generateReceiptNumber(),
      isContractCompleted: remainingInstallments.length === 0,
    };
  });
}
```

### 4.4 งานซ่อม + ตัดสต็อกอัตโนมัติ

```typescript
async function addPartToRepairJob(input: {
  jobId: string;
  partId: string;
  quantity: number;
  clerkId: string;
}) {
  return db.transaction(async (tx) => {
    // 1. ตรวจสอบสต็อก
    const part = await tx.query.parts.findFirst({
      where: eq(parts.id, input.partId)
    });

    if (part.quantityInStock < input.quantity) {
      throw new Error(`สต็อกไม่พอ: ${part.name} เหลือ ${part.quantityInStock} ${part.unit}`);
    }

    const totalPrice = Number(part.sellingPrice) * input.quantity;

    // 2. เพิ่มอะไหล่ในงานซ่อม
    await tx.insert(repairJobParts).values({
      jobId: input.jobId,
      partId: input.partId,
      quantity: input.quantity,
      unitPrice: part.sellingPrice,
      totalPrice: String(totalPrice),
    });

    // 3. ตัดสต็อก
    await tx.update(parts).set({
      quantityInStock: part.quantityInStock - input.quantity,
    }).where(eq(parts.id, input.partId));

    // 4. บันทึก Stock Movement
    await tx.insert(stockMovements).values({
      partId: input.partId,
      type: 'out',
      quantity: input.quantity,
      referenceType: 'repair',
      referenceId: input.jobId,
      unitCost: part.costPrice,
      performedBy: input.clerkId,
    });

    // 5. อัปเดตยอดรวมงานซ่อม
    await tx.execute(sql`
      UPDATE repair_jobs SET
        total_parts_cost = (
          SELECT COALESCE(SUM(total_price), 0)
          FROM repair_job_parts WHERE job_id = ${input.jobId}
        ),
        total_amount = total_labor_cost + (
          SELECT COALESCE(SUM(total_price), 0)
          FROM repair_job_parts WHERE job_id = ${input.jobId}
        ),
        updated_at = NOW()
      WHERE id = ${input.jobId}
    `);

    // 6. เช็ค Low Stock Alert
    if (part.quantityInStock - input.quantity <= part.minStockLevel) {
      // TODO: ส่ง notification แจ้งเตือน
      console.warn(`⚠️ Low stock alert: ${part.name} เหลือ ${part.quantityInStock - input.quantity}`);
    }
  });
}
```

### 4.5 Daily Cash Management

```typescript
// เปิดวัน
async function openCashDay(input: { openingBalance: number; clerkId: string }) {
  const today = new Date().toISOString().split('T')[0];

  // ตรวจสอบว่ายังไม่เปิดวันนี้
  const existing = await db.query.dailyCashSessions.findFirst({
    where: eq(dailyCashSessions.sessionDate, today)
  });
  if (existing) throw new Error('วันนี้เปิดรอบแล้ว');

  return db.insert(dailyCashSessions).values({
    sessionDate: today,
    openingBalance: String(input.openingBalance),
    totalIncome: '0',
    totalExpense: '0',
    status: 'open',
    openedBy: input.clerkId,
  }).returning();
}

// บันทึก transaction (รายรับ/รายจ่าย)
async function recordCashTransaction(
  txOrDb: typeof db,
  input: CashTransactionInput
) {
  const session = await txOrDb.query.dailyCashSessions.findFirst({
    where: and(
      eq(dailyCashSessions.sessionDate, new Date().toISOString().split('T')[0]),
      eq(dailyCashSessions.status, 'open')
    )
  });
  if (!session) throw new Error('ยังไม่ได้เปิดรอบเงินสดวันนี้');

  // บันทึก transaction
  await txOrDb.insert(cashTransactions).values({
    sessionId: session.id,
    ...input,
  });

  // อัปเดตยอดรวมใน session
  if (input.type === 'income') {
    await txOrDb.update(dailyCashSessions).set({
      totalIncome: sql`${dailyCashSessions.totalIncome} + ${input.amount}`,
    }).where(eq(dailyCashSessions.id, session.id));
  } else {
    await txOrDb.update(dailyCashSessions).set({
      totalExpense: sql`${dailyCashSessions.totalExpense} + ${input.amount}`,
    }).where(eq(dailyCashSessions.id, session.id));
  }
}

// ปิดวัน (สรุปยอด)
async function closeCashDay(input: { clerkId: string }) {
  const today = new Date().toISOString().split('T')[0];
  const session = await db.query.dailyCashSessions.findFirst({
    where: and(
      eq(dailyCashSessions.sessionDate, today),
      eq(dailyCashSessions.status, 'open')
    )
  });
  if (!session) throw new Error('ไม่พบรอบเงินสดที่เปิดอยู่');

  const closingBalance =
    Number(session.openingBalance) +
    Number(session.totalIncome) -
    Number(session.totalExpense);

  return db.update(dailyCashSessions).set({
    closingBalance: String(closingBalance),
    status: 'closed',
    closedBy: input.clerkId,
  }).where(eq(dailyCashSessions.id, session.id)).returning();
}
```

### 4.6 ระบบติดตามหนี้ (Collection System)

#### Flow การติดตามหนี้

```
1. เสมียน/Admin ดูรายชื่อลูกหนี้ค้างชำระจากระบบ
2. มอบหมายงานติดตาม (assignCollectionTask)
   → เลือก Collector + เลือกลูกหนี้ + กำหนด priority
3. Collector เห็นงานบน Mobile App
   → ดูข้อมูลลูกหนี้: ชื่อ, ที่อยู่, เบอร์โทร, ยอดค้าง
   → ดูประวัติลูกค้า: สัญญา, รถที่ซื้อ, ข้อมูลค้ำประกัน
4. Collector ออกพื้นที่ติดตาม
   → บันทึกผล (recordCollectionVisit):
     - ติดต่อได้ นัดจ่าย → บันทึกวันนัด
     - ติดต่อได้ แต่จ่ายไม่ได้ → บันทึกหมายเหตุ
     - ไม่อยู่บ้าน / ติดต่อไม่ได้
     - จ่ายเงินหน้างาน → ออกใบเสร็จชั่วคราว
5. กรณีรับเงินหน้างาน:
   → บันทึกยอดเงินที่เก็บ + ออกใบเสร็จชั่วคราว
   → สถานะ: "collected" (เก็บเงินแล้ว ยังไม่ส่งร้าน)
6. Collector กลับร้าน → ส่งเงินให้เสมียน
   → เสมียนตรวจรับเงิน → สถานะ: "submitted" → "verified"
   → ระบบบันทึก Cash Transaction + อัปเดตงวดที่จ่าย
```

#### Pseudo-code: มอบหมายงานติดตาม

```typescript
async function assignCollectionTask(input: {
  contractId: string;
  collectorId: string;
  assignedBy: string;
  priority?: number;
  notes?: string;
}) {
  // 1. ดึงข้อมูลสัญญาและงวดค้างชำระ
  const contract = await db.query.salesContracts.findFirst({
    where: eq(salesContracts.id, input.contractId),
    with: {
      customer: true,
      installments: {
        where: inArray(installments.status, ['overdue', 'pending']),
      },
    },
  });

  if (!contract) throw new Error('ไม่พบสัญญา');

  const overdueInstallments = contract.installments.filter(
    (i) => i.status === 'overdue'
  );
  if (overdueInstallments.length === 0) {
    throw new Error('ไม่มีงวดค้างชำระ');
  }

  const totalOverdue = overdueInstallments.reduce(
    (sum, i) => sum + Number(i.amount), 0
  );

  // 2. ตรวจสอบว่ายังไม่มีงานติดตามที่ยังเปิดอยู่สำหรับสัญญานี้
  const existingAssignment = await db.query.collectionAssignments.findFirst({
    where: and(
      eq(collectionAssignments.contractId, input.contractId),
      inArray(collectionAssignments.status, ['assigned', 'in_progress']),
    ),
  });
  if (existingAssignment) {
    throw new Error('มีงานติดตามที่เปิดอยู่แล้วสำหรับสัญญานี้');
  }

  // 3. สร้างงานมอบหมาย
  const [assignment] = await db.insert(collectionAssignments).values({
    contractId: input.contractId,
    customerId: contract.customerId,
    collectorId: input.collectorId,
    assignedBy: input.assignedBy,
    assignedDate: new Date().toISOString().split('T')[0],
    priority: input.priority ?? 0,
    totalOverdueAmount: String(totalOverdue),
    overdueInstallments: overdueInstallments.length,
    status: 'assigned',
    notes: input.notes,
  }).returning();

  // 4. TODO: ส่ง Push Notification ไปหา Collector

  return assignment;
}
```

#### Pseudo-code: บันทึกผลการติดตาม

```typescript
async function recordCollectionVisit(input: {
  assignmentId: string;
  collectorId: string;
  result: CollectionActivityResult;
  promiseDate?: string;
  amountCollected?: number;
  gpsLatitude?: number;
  gpsLongitude?: number;
  notes?: string;
}) {
  return db.transaction(async (tx) => {
    // 1. ตรวจสอบงาน
    const assignment = await tx.query.collectionAssignments.findFirst({
      where: and(
        eq(collectionAssignments.id, input.assignmentId),
        eq(collectionAssignments.collectorId, input.collectorId),
      ),
    });
    if (!assignment) throw new Error('ไม่พบงานติดตาม');

    // 2. อัปเดตสถานะงานเป็น in_progress (ถ้ายังเป็น assigned)
    if (assignment.status === 'assigned') {
      await tx.update(collectionAssignments).set({
        status: 'in_progress',
      }).where(eq(collectionAssignments.id, input.assignmentId));
    }

    // 3. บันทึกกิจกรรม
    const activityData: any = {
      assignmentId: input.assignmentId,
      collectorId: input.collectorId,
      result: input.result,
      promiseDate: input.promiseDate,
      gpsLatitude: input.gpsLatitude,
      gpsLongitude: input.gpsLongitude,
      notes: input.notes,
    };

    // 4. กรณีเก็บเงินหน้างาน
    if (
      input.result === 'paid_on_site' ||
      input.result === 'partial_paid_on_site'
    ) {
      if (!input.amountCollected || input.amountCollected <= 0) {
        throw new Error('ต้องระบุยอดเงินที่เก็บ');
      }
      activityData.amountCollected = String(input.amountCollected);
      activityData.fieldReceiptNumber = generateFieldReceiptNumber();
      activityData.fieldPaymentStatus = 'collected';
    }

    const [activity] = await tx.insert(collectionActivities)
      .values(activityData)
      .returning();

    return {
      activity,
      fieldReceiptNumber: activityData.fieldReceiptNumber,
    };
  });
}
```

---

## 5. UX/UI Direction

### 5.1 หลักการ UX ภาพรวม

| หลักการ | รายละเอียด |
|---------|-----------|
| **Less is More** | แต่ละหน้าจอมีจุดประสงค์เดียว ไม่ยัดข้อมูลจนรกตา |
| **Big Touch Targets** | ปุ่มใหญ่ ข้อความใหญ่ โดยเฉพาะบน Mobile (min 44px) |
| **Minimal Typing** | ใช้ Dropdown, Quick Select, Barcode Scan แทนการพิมพ์เท่าที่ทำได้ |
| **Instant Feedback** | ทุก Action ต้องมี feedback ชัดเจน (success toast, error alert) |
| **Forgiving Design** | ยืนยันก่อนลบ, Undo ได้เมื่อเป็นไปได้ |

### 5.2 UX ตาม Role

#### เสมียน (Clerk) — Web App บนจอ PC

```
┌─────────────────────────────────────────────────────────────────┐
│  🏪 กมลกิจยานยนต์              วันที่ 10 มี.ค. 2569    [ออกจากระบบ]│
├─────────────┬───────────────────────────────────────────────────┤
│             │                                                   │
│  📊 แดชบอร์ด │  ┌─────────────────────────────────────────────┐  │
│             │  │         หน้าหลัก - Quick Actions              │  │
│  📋 สัญญา    │  │                                               │  │
│    ซื้อขาย    │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐        │  │
│             │  │  │ 💰 รับ   │ │ 🔧 เปิด  │ │ 📦 ขาย   │        │  │
│  💰 รับชำระ   │  │  │ค่างวด   │ │งานซ่อม   │ │อะไหล่    │        │  │
│    ค่างวด    │  │  └─────────┘ └─────────┘ └─────────┘        │  │
│             │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐        │  │
│  🔧 งานซ่อม  │  │  │ 🏍 สร้าง │ │ 📋 พ.ร.บ. │ │ 💵 เงินสด │        │  │
│             │  │  │สัญญาขาย │ │ต่อภาษี   │ │ประจำวัน  │        │  │
│  📦 อะไหล่   │  │  └─────────┘ └─────────┘ └─────────┘        │  │
│    & สต็อก   │  │                                               │  │
│             │  │  ── แจ้งเตือน ──────────────────────           │  │
│  📋 พ.ร.บ.  │  │  ⚠️ ลูกค้า 3 ราย ค้างชำระเกิน 30 วัน          │  │
│    & ภาษี   │  │  ⚠️ อะไหล่ 5 รายการ ใกล้หมดสต็อก              │  │
│             │  │  📌 งานซ่อม 2 งาน รอปิด Job                   │  │
│  💵 เงินสด   │  │                                               │  │
│    ประจำวัน   │  └─────────────────────────────────────────────┘  │
│             │                                                   │
│  📊 รายงาน   │                                                   │
│             │                                                   │
└─────────────┴───────────────────────────────────────────────────┘
```

**หลักการสำคัญสำหรับเสมียน:**
- **Quick Actions** บนหน้าแรก — งานที่ทำบ่อยที่สุดต้องเข้าถึงได้ภายใน 1 คลิก
- **Sidebar navigation** — เห็น Menu ได้ตลอด ไม่ต้องกดหลายครั้ง
- **รองรับ Keyboard Shortcut** — F1-F6 สำหรับ Quick Actions
- **Barcode Scanner** — สแกนอะไหล่แล้วเพิ่มเข้าตะกร้าได้เลย
- **ใบเสร็จ** — พิมพ์ได้ทันทีหลังบันทึก (Thermal Printer)

#### หน้ารับชำระค่างวด (ตัวอย่างละเอียด)

```
┌─────────────────────────────────────────────────────────────────┐
│  💰 รับชำระค่างวด                                    [กลับหน้าหลัก]│
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🔍 ค้นหา: [________________] [ค้นหา]                           │
│     ค้นหาด้วย: ชื่อ / เบอร์โทร / เลขสัญญา / ทะเบียนรถ            │
│                                                                 │
│  ── ข้อมูลลูกค้า ──────────────────────────────────────          │
│  ชื่อ: นายสมชาย ใจดี           เบอร์โทร: 089-xxx-xxxx           │
│  สัญญา: KK-2569-0042         รถ: Honda Wave 110i (กท-1234)     │
│                                                                 │
│  ── สถานะการผ่อน ──────────────────────────────────────          │
│  ┌──────┬──────────┬──────────┬──────────┬──────────┐           │
│  │ งวดที่ │ ครบกำหนด  │ จำนวนเงิน │ สถานะ    │ จ่ายวันที่ │           │
│  ├──────┼──────────┼──────────┼──────────┼──────────┤           │
│  │  1   │ 15/01/69 │ 2,500.-  │ ✅ จ่ายแล้ว│ 15/01/69 │           │
│  │  2   │ 15/02/69 │ 2,500.-  │ ✅ จ่ายแล้ว│ 18/02/69 │           │
│  │  3   │ 15/03/69 │ 2,500.-  │ ⚠️ ค้าง   │    -     │ ← เลือก  │
│  │  4   │ 15/04/69 │ 2,500.-  │ ⏳ รอ     │    -     │           │
│  │  ... │          │          │          │          │           │
│  └──────┴──────────┴──────────┴──────────┴──────────┘           │
│                                                                 │
│  ── รายละเอียดการชำระ ──────────────────────────────────          │
│  ┌─────────────────────────────────────────────┐                │
│  │  งวดที่ 3 — ครบกำหนด 15/03/2569              │                │
│  │                                              │                │
│  │  ค่างวด:                          2,500 บาท  │                │
│  │  ค้างชำระ:                       25 วัน       │                │
│  │  ค่าปรับล่าช้า (100 บาท/งวด):      100 บาท   │                │
│  │  ─────────────────────────────────           │                │
│  │  ยอดรวมที่ต้องชำระ:              2,600 บาท    │                │
│  │                                              │                │
│  │  วิธีชำระ: [เงินสด ▼]                         │                │
│  │                                              │                │
│  │        [ 🖨 พิมพ์ใบเสร็จ + บันทึก ]             │                │
│  └─────────────────────────────────────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### ช่างซ่อม (Technician) — Web App บนจอ PC + Barcode Scanner

```
┌─────────────────────────────────────────────────────────────────┐
│  🔧 งานซ่อม — สมศักดิ์                    10 มี.ค. 69   [ออกจากระบบ]│
├─────────────┬───────────────────────────────────────────────────┤
│             │                                                   │
│  🔧 งานของฉัน│  ┌─────────────────────────────────────────────┐  │
│             │  │         งานซ่อมที่ได้รับมอบหมาย                  │  │
│  📦 เบิกอะไหล่│  │                                               │  │
│             │  │  ┌───────────────────────────────────────────┐│  │
│  📊 สรุปงาน  │  │  │ 🟡 JOB-0089  Honda Click — กท-5678       ││  │
│             │  │  │ เปลี่ยนผ้าเบรค + เปลี่ยนน้ำมัน                 ││  │
│             │  │  │ สถานะ: กำลังซ่อม                            ││  │
│             │  │  │ [📦 เบิกอะไหล่] [📸 แนบรูป] [✅ เสร็จแล้ว]   ││  │
│             │  │  └───────────────────────────────────────────┘│  │
│             │  │                                               │  │
│             │  │  ┌───────────────────────────────────────────┐│  │
│             │  │  │ 🔴 JOB-0090  Yamaha Fino — ขก-9012        ││  │
│             │  │  │ เครื่องดับขณะขับ                               ││  │
│             │  │  │ สถานะ: รอซ่อม                               ││  │
│             │  │  │ [▶️ เริ่มซ่อม]                                ││  │
│             │  │  └───────────────────────────────────────────┘│  │
│             │  │                                               │  │
│             │  │        [➕ เปิดงานซ่อมใหม่]                      │  │
│             │  └─────────────────────────────────────────────┘  │
│             │                                                   │
│             │  ── Barcode Scanner Workflow ──                    │
│             │  ┌─────────────────────────────────────────────┐  │
│             │  │  🔫 สแกน Barcode อะไหล่ → [______________]   │  │
│             │  │  ผลลัพธ์: ผ้าเบรค Honda (BRK-H001)           │  │
│             │  │  จำนวน: [1] ชิ้น   ราคา: 250 บาท             │  │
│             │  │  [เพิ่มเข้างาน JOB-0089]                      │  │
│             │  └─────────────────────────────────────────────┘  │
│             │                                                   │
└─────────────┴───────────────────────────────────────────────────┘
```

**หลักการสำคัญสำหรับช่าง:**
- **Web/PC interface** — จอใหญ่ดูข้อมูลได้เยอะ ใช้ Keyboard + Mouse
- **Barcode Scanner (USB/Wireless)** — สแกน Barcode อะไหล่แล้วเพิ่มเข้างานได้เลย
- **Sidebar navigation แบบเสมียน** — เข้าถึง Menu ได้ตลอด
- **แนบรูปงานซ่อม** — อัปโหลดรูปก่อน/หลังซ่อม
- **Status update แค่ 1 คลิก** — เริ่มซ่อม / เสร็จแล้ว
- **Keyboard Shortcut** — Enter เพื่อเพิ่มอะไหล่หลังสแกน

#### เจ้าของร้าน (Owner) — Mobile App

```
┌──────────────────────────────────┐
│  📊 กมลกิจยานยนต์     10 มี.ค. 69│
├──────────────────────────────────┤
│                                  │
│  ── รายได้วันนี้ ─────────────     │
│  ┌────────────────────────────┐  │
│  │    💰 12,500 บาท           │  │
│  │    รายรับ    15,000        │  │
│  │    รายจ่าย   -2,500        │  │
│  └────────────────────────────┘  │
│                                  │
│  ── แยกตามประเภท ─────────────    │
│  ค่างวดรถ      ████████ 7,500   │
│  ค่าซ่อม       █████   3,500    │
│  ค่าอะไหล่     ███     2,000    │
│  พ.ร.บ./ภาษี  ██      1,500    │
│  เงินออก       ██     -2,500    │
│                                  │
│  ── แจ้งเตือน ─────────────────   │
│  🔴 ลูกค้า 3 ราย ค้างเกิน 30 วัน  │
│  🟡 อะไหล่ 5 รายการ ใกล้หมด      │
│  🟢 งานซ่อมวันนี้ 4 งาน (เสร็จ 2) │
│                                  │
│  [📋 ดูลูกหนี้ค้าง]  [📊 ดูรายงาน] │
│                                  │
├──────────────────────────────────┤
│   🏠 หลัก  📊 รายงาน  🔔 แจ้งเตือน│
└──────────────────────────────────┘
```

**หลักการสำคัญสำหรับเจ้าของร้าน:**
- **สรุปยอดประจำวัน** — เปิดแอปมาเห็นทันทีว่าวันนี้เงินเข้าเท่าไหร่
- **Push Notification** — แจ้งเมื่อมีค่างวดค้างเกิน X วัน, เปิด/ปิดร้าน, ยอดผิดปกติ
- **Drill-down** — แตะตัวเลขแล้วดูรายละเอียดได้
- **ไม่ต้องบันทึกข้อมูลใดๆ** — Read-only dashboard

#### พนักงานติดตามหนี้ (Collector) — Mobile App

```
┌──────────────────────────────────┐
│  💼 ติดตามหนี้        สมปอง (วันนี้)│
├──────────────────────────────────┤
│                                  │
│  ── งานวันนี้ (3 ราย) ──────────  │
│                                  │
│  ┌────────────────────────────┐  │
│  │ 🔴 ด่วน — นายสมชาย ใจดี    │  │
│  │ ค้าง 3 งวด (7,500 บาท)    │  │
│  │ 📍 123 ถ.มิตรภาพ อ.เมือง    │  │
│  │ 📞 089-xxx-xxxx            │  │
│  │                            │  │
│  │ [📞 โทร] [🗺 นำทาง] [📝 บันทึก]│  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │ 🟡 ปกติ — นางสมหญิง สุขใจ  │  │
│  │ ค้าง 1 งวด (2,500 บาท)    │  │
│  │ 📍 45/2 ซ.สุขสวัสดิ์         │  │
│  │ 📞 091-xxx-xxxx            │  │
│  │                            │  │
│  │ [📞 โทร] [🗺 นำทาง] [📝 บันทึก]│  │
│  └────────────────────────────┘  │
│                                  │
│  ── สรุปวันนี้ ────────────────   │
│  ติดตามแล้ว: 1/3 ราย             │
│  เก็บเงินได้: 2,500 บาท          │
│  ยังไม่ส่งร้าน: 2,500 บาท        │
│                                  │
├──────────────────────────────────┤
│  💼 งาน   📋 ประวัติ   💰 เงิน   │
└──────────────────────────────────┘
```

#### Collector — หน้าบันทึกผลติดตาม

```
┌──────────────────────────────────┐
│  📝 บันทึกผลติดตาม    [← กลับ]   │
├──────────────────────────────────┤
│                                  │
│  ลูกค้า: นายสมชาย ใจดี            │
│  สัญญา: KK-2569-0042            │
│  รถ: Honda Wave 110i (กท-1234)  │
│  ค้าง 3 งวด — 7,500 บาท         │
│                                  │
│  ── ผลการติดตาม ─────────────    │
│  ┌────────────────────────────┐  │
│  │ ○ ติดต่อได้ นัดจ่าย         │  │
│  │ ○ ติดต่อได้ แต่จ่ายไม่ได้    │  │
│  │ ○ ไม่อยู่บ้าน               │  │
│  │ ○ ติดต่อไม่ได้              │  │
│  │ ● จ่ายเงินหน้างาน ←        │  │
│  │ ○ จ่ายบางส่วนหน้างาน       │  │
│  └────────────────────────────┘  │
│                                  │
│  ── รับเงิน ─────────────────    │
│  จำนวนเงิน: [    2,500    ] บาท  │
│  (งวดที่ 3 — ค่างวด 2,500 บาท)  │
│                                  │
│  หมายเหตุ: [________________]    │
│                                  │
│  ┌────────────────────────────┐  │
│  │  📍 GPS: 16.4321, 102.8236 │  │
│  │  (บันทึกตำแหน่งอัตโนมัติ)    │  │
│  └────────────────────────────┘  │
│                                  │
│  [    💾 บันทึก + ออกใบเสร็จ    ]  │
│                                  │
└──────────────────────────────────┘
```

**หลักการสำคัญสำหรับ Collector:**
- **Task list แบบ Card** — เห็นงานวันนี้ทั้งหมด จัดลำดับตาม priority
- **โทร + นำทาง 1 แตะ** — กดโทรหาลูกค้า หรือเปิด Google Maps นำทาง
- **บันทึกผลง่าย** — เลือก Radio button + ใส่หมายเหตุ
- **รับเงินหน้างาน** — ระบุยอด + ออกใบเสร็จชั่วคราว (PDF)
- **GPS อัตโนมัติ** — บันทึกตำแหน่งเป็นหลักฐานว่าออกพื้นที่จริง
- **สรุปเงินที่เก็บ** — ดูยอดรวมที่ยังไม่ส่งร้าน

### 5.3 Role Permission Matrix

| Feature | Owner | Clerk | Technician | Collector |
|---------|-------|-------|------------|-----------|
| Dashboard / สรุปรายได้ | ✅ (Mobile) | ✅ (Web) | ❌ | ❌ |
| จัดการลูกค้า | ดูอย่างเดียว | CRUD | ❌ | ดูอย่างเดียว |
| จัดการรถ / สต็อกรถ | ดูอย่างเดียว | CRUD | ❌ | ❌ |
| สร้างสัญญาซื้อขาย | ❌ | CRUD | ❌ | ❌ |
| รับชำระค่างวด (ที่ร้าน) | ❌ | CRUD | ❌ | ❌ |
| งานซ่อม | ดูอย่างเดียว | เปิด/ปิด Job | CRUD (งานตัวเอง) | ❌ |
| อะไหล่ / สต็อก | ดูอย่างเดียว | CRUD | เบิกอะไหล่ (Barcode) | ❌ |
| เงินสดประจำวัน | ดูอย่างเดียว | CRUD | ❌ | ❌ |
| พ.ร.บ. / ภาษี | ดูอย่างเดียว | CRUD | ❌ | ❌ |
| รายงาน | ✅ ทั้งหมด | ✅ บางส่วน | ❌ | ❌ |
| มอบหมายงานติดตาม | ❌ | CRUD | ❌ | ❌ |
| ดูงานติดตาม | ดูทั้งหมด | ดูทั้งหมด | ❌ | เฉพาะงานตัวเอง |
| บันทึกผลติดตาม | ❌ | ❌ | ❌ | CRUD |
| รับเงินหน้างาน | ❌ | ❌ | ❌ | ✅ |
| ตรวจรับเงิน Collector | ❌ | ✅ | ❌ | ❌ |
| ดูประวัติลูกหนี้ | ✅ | ✅ | ❌ | ✅ (เฉพาะงาน) |

### 5.4 Color & Design System

```
Primary Colors:
  - Primary Blue:    #2563EB  (สำหรับ actions หลัก)
  - Success Green:   #16A34A  (จ่ายแล้ว, เสร็จ)
  - Warning Amber:   #D97706  (ค้างชำระ, ใกล้หมด)
  - Danger Red:      #DC2626  (เกินกำหนด, ข้อผิดพลาด)
  - Neutral Gray:    #6B7280  (ข้อมูลทั่วไป)

Typography:
  - Web: Inter / Sarabun (รองรับภาษาไทย)
  - Mobile: System font (SF Pro / Roboto)
  - ขนาดตัวอักษรขั้นต่ำ: 14px (Web), 16px (Mobile)
  - ตัวเลขเงิน: Tabular nums, font-weight: 600

Spacing:
  - Base unit: 4px
  - Component padding: 12-16px
  - Section gap: 24-32px
```

---

## 6. Pain Points & Pitfalls

### 6.1 ปัญหาที่มักเจอ + วิธีป้องกัน

| # | ปัญหา | สาเหตุ | วิธีป้องกัน |
|---|--------|--------|------------|
| 1 | **ยอดเงินสดไม่ตรง** | เสมียนลืมบันทึกรายจ่าย หรือบันทึกผิดประเภท | บังคับเปิด/ปิดรอบเงินสดทุกวัน + ตรวจสอบยอดจริง vs ระบบตอนปิดร้าน + ต้องใส่เหตุผลเมื่อยอดไม่ตรง |
| 2 | **สต็อกอะไหล่ไม่ตรง** | เบิกอะไหล่แล้วลืมบันทึก | บังคับให้เพิ่มอะไหล่ใน Job Order ก่อนปิดงาน + Stock audit ระบบนับสต็อกเป็นระยะ |
| 3 | **ลูกค้าจ่ายค่างวดแต่ไม่ได้บันทึก** | เสมียนรับเงินแล้วลืม key | ทุกการรับเงินต้องพิมพ์ใบเสร็จ (บังคับ) + ใบเสร็จมีเลขที่ Running |
| 4 | **ข้อมูลลูกค้าซ้ำ** | พิมพ์ชื่อไม่ตรง สร้างลูกค้าใหม่ | ค้นหาด้วยเบอร์โทรก่อนสร้าง + แจ้งเตือนเมื่อพบข้อมูลคล้าย + Merge duplicate |
| 5 | **ค่าปรับคำนวณผิด** | ลอจิกซับซ้อน คำนวณมือ | ระบบคำนวณอัตโนมัติเท่านั้น ไม่ให้แก้ไขค่าปรับด้วยมือ (ยกเว้น Admin override) |
| 6 | **ลืมต่อ พ.ร.บ./ภาษี** | ไม่มีระบบเตือน | แจ้งเตือนล่วงหน้า 30 วัน ก่อนหมดอายุ |
| 7 | **Internet ดับ ใช้งานไม่ได้** | ร้านอาจอยู่ในพื้นที่ Internet ไม่เสถียร | Phase แรก Deploy เป็น Local server ในร้าน + ทำ Offline-capable ใน Phase หลัง (Service Worker / Local-first) |
| 8 | **Barcode ซ้ำกันระหว่างร้าน/Supplier** | ใช้ Barcode ของผู้ผลิต | ระบบรองรับทั้ง Barcode ผู้ผลิต + Internal Part Code |
| 9 | **เสมียนเปลี่ยนบ่อย** | อาจจ้างคนใหม่ | UI ง่าย + มี Onboarding tutorial ในระบบ + แยก User account ชัดเจน |
| 10 | **ข้อมูลถูกลบหรือแก้ไข** | Human error หรือ จงใจ | Soft delete เท่านั้น (ไม่ลบจริง) + Audit log ทุก action สำคัญ + Role-based permissions |
| 11 | **เงินที่ Collector เก็บมา vs ยอดในระบบไม่ตรง** | Collector ลืมบันทึก หรือบันทึกยอดผิด หรือเงินหาย | บังคับบันทึกยอดเงินทุกครั้งที่เก็บ + ออกใบเสร็จชั่วคราวทันที + เสมียนต้องตรวจรับเงินและยืนยันยอด + GPS log เป็นหลักฐาน |
| 12 | **Collector Offline ขณะออกพื้นที่** | พื้นที่ห่างไกล ไม่มี Internet | บันทึกข้อมูลลง Local Storage ก่อน → Sync เมื่อมี Internet (Offline-first) + Queue pending activities |
| 13 | **Collector แก้ไขข้อมูลหนี้** | Collector อาจพยายามแก้ยอดหนี้/ลดยอด | Collector ไม่มีสิทธิ์แก้ไขข้อมูลสัญญา/งวด → ทำได้แค่ "ดู" + "บันทึกผลติดตาม" + "รับเงิน" เท่านั้น |

### 6.2 Security Considerations

```
1. Authentication & Authorization
   - JWT token expire 24 ชม., Refresh token 7 วัน
   - JWT Storage: เก็บใน httpOnly cookie (ป้องกัน XSS) — ไม่ใช้ localStorage
   - Refresh Token Rotation: ทุก refresh ออก token ใหม่ + revoke token เก่า (ป้องกัน token theft)
   - Password Hashing: Argon2id (OWASP 2023+ recommendation — แข็งแกร่งกว่า bcrypt)
   - Login Lockout: block 15 นาที หลัง fail 5 ครั้ง (Redis TTL)
   - Role-based access control (RBAC):
     - Admin: CRUD ทุกอย่าง + รายงาน + ตั้งค่า
     - Clerk: CRUD ธุรกรรม + ดูรายงานบางส่วน + มอบหมาย/ตรวจรับงานติดตาม
     - Technician: CRUD เฉพาะงานซ่อมที่ตัวเองรับผิดชอบ
     - Collector: ดูงานติดตามตัวเอง + บันทึกผล + รับเงินหน้างาน (Read-only สัญญา/ลูกค้า)
   - Audit Log: บันทึก user, action, timestamp, before/after

2. Data Protection
   - ข้อมูลลูกค้า (PII) เก็บอย่างปลอดภัย
   - PII Masking ใน Log/Response: เลขบัตร → `1234-XXXX-XXXX-5678`, โทรศัพท์ → `08X-XXX-1234`
   - Database backup อัตโนมัติทุกวัน (pg_dump)
   - Backup Encryption: pg_dump → encrypt ด้วย gpg ก่อน upload S3/offsite
   - HTTPS เท่านั้น (TLS 1.3)
   - Secrets Management: ไม่ commit .env, ใช้ .env.example แทน, production ใช้ environment variables จาก VPS config

3. Input Validation & Attack Prevention
   - Zod validation ทุก endpoint
   - Parameterized queries (Drizzle ORM ป้องกัน SQL Injection)
   - Rate limiting สำหรับ login attempts
   - CORS: whitelist เฉพาะ domain ของร้าน (kamonkit.com) — ไม่ใช้ *
   - CSP Header: Content-Security-Policy ป้องกัน XSS ใน browser
   - CSRF: ใช้ SameSite=Strict cookie + Origin header check (เนื่องจากใช้ httpOnly cookie)
   - XSS Output Encoding: React escape HTML by default — ห้ามใช้ `dangerouslySetInnerHTML` กับ user input โดยไม่ sanitize, ใช้ DOMPurify ถ้ามี rich text

4. Financial Transaction Security
   - Atomic Transactions: ทุก payment ใช้ PostgreSQL transaction (BEGIN/COMMIT) — ป้องกันเงินหาย
   - Idempotency Key: ทุก POST payment ส่ง X-Idempotency-Key ป้องกัน double charge จาก retry
   - ห้าม Soft Delete บน Transaction: payment records ลบไม่ได้ — แก้ได้ด้วย reversal entry เท่านั้น
   - Collector Cash Validation: ยอดที่ Collector รับต้องตรงกับยอดใน contract ± tolerance เล็กน้อย

5. Privacy & Compliance (PDPA)
   - PDPA (พรบ.คุ้มครองข้อมูลส่วนบุคคล): ระบบเก็บ PII → ต้องมี consent + purpose limitation
   - GPS Collector: แจ้ง Collector ว่า GPS ถูก log ขณะปฏิบัติงาน — ไม่ log นอกเวลางาน
   - Data Retention: ลูกค้าปิดสัญญา → archive ข้อมูล 5 ปี ตาม พรบ.บัญชี → ลบ

6. Dependency & Environment Security
   - Dependency Scan: bun audit / npm audit ใน CI pipeline
   - Docker Image: ใช้ node:alpine / oven/bun:alpine (minimal attack surface), scan ด้วย Trivy
   - VPS Hardening: SSH key only (ไม่ใช้ password), UFW firewall เปิดแค่ port 80/443/22

7. OWASP Top 10 (2021) Coverage Mapping

| # | OWASP Category | Mitigation ในระบบนี้ |
|---|---|---|
| A01 | Broken Access Control | RBAC 4 roles + permission matrix, Audit Log |
| A02 | Cryptographic Failures | Argon2id, TLS 1.3, Backup encryption (gpg) |
| A03 | Injection (SQL, XSS, etc.) | Drizzle parameterized queries, Zod, CSP Header, Output Encoding |
| A04 | Insecure Design | Financial atomic transactions, No delete on payment, Soft delete only |
| A05 | Security Misconfiguration | CORS whitelist, VPS hardening, SSH key only, UFW |
| A06 | Vulnerable & Outdated Components | bun audit ใน CI, Trivy Docker scan |
| A07 | Identification & Authentication Failures | Argon2id, httpOnly cookie, Refresh Token Rotation, Login Lockout |
| A08 | Software & Data Integrity Failures | Idempotency Key, Atomic transactions, Dependency scan |
| A09 | Security Logging & Monitoring Failures | Audit Log (user, action, timestamp, before/after) |
| A10 | Server-Side Request Forgery (SSRF) | Validate/whitelist ทุก URL ที่รับจาก user input, ไม่ fetch URL จาก request body โดยตรง |
```

### 6.3 Soft Delete & Audit Log

```typescript
// Soft delete pattern — เพิ่ม column เหล่านี้ในทุก table ที่สำคัญ
// deletedAt: timestamp('deleted_at'),
// deletedBy: uuid('deleted_by').references(() => users.id),

// Audit Log table
export const auditLogs = pgTable('audit_logs', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  action: varchar('action', { length: 50 }).notNull(), // CREATE, UPDATE, DELETE
  tableName: varchar('table_name', { length: 50 }).notNull(),
  recordId: uuid('record_id').notNull(),
  oldValues: jsonb('old_values'), // ค่าก่อนเปลี่ยน
  newValues: jsonb('new_values'), // ค่าหลังเปลี่ยน
  ipAddress: varchar('ip_address', { length: 45 }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

---

## 7. Phase Planning

### Phase 0: Foundation & Infrastructure (สัปดาห์ที่ 1-2)

```
🎯 เป้าหมาย: วาง foundation ให้แข็งแรง

Tasks:
  ✅ ตั้ง Monorepo (Turborepo)
  ✅ ตั้ง Backend API skeleton (Hono.js + Bun)
  ✅ ตั้ง Database + Drizzle ORM + Migrations
  ✅ ตั้ง Authentication (JWT + RBAC)
  ✅ ตั้ง Web App skeleton (Next.js)
  ✅ Docker Compose (PostgreSQL + Redis)
  ✅ CI/CD pipeline (GitHub Actions)
  ✅ Seed data สำหรับ dev/test

Deliverable: โครงสร้างพร้อม, Login ได้, Deploy dev environment ได้
```

### Phase 1: Core Business — ขายรถ + ผ่อนชำระ (สัปดาห์ที่ 3-5)

```
🎯 เป้าหมาย: ใช้จัดการการขายรถและรับค่างวดได้จริง ← จุดที่ทำเงินให้ร้าน

Tasks:
  ✅ CRUD ลูกค้า (Customer)
  ✅ CRUD รถจักรยานยนต์ (Vehicle)
  ✅ สร้างสัญญาซื้อขาย (Cash + Installment)
  ✅ ระบบผ่อนชำระ — สร้างงวด, กำหนดวันครบกำหนด
  ✅ รับชำระค่างวด — ค้นหาลูกค้า, เลือกงวด, บันทึกการจ่าย
  ✅ คำนวณค่าปรับล่าช้าอัตโนมัติ
  ✅ จัดการเล่มทะเบียน (held_by_shop / returned)
  ✅ Overdue detection (Scheduled Job)
  ✅ พิมพ์ใบเสร็จค่างวด (PDF/Thermal)

Deliverable: เสมียนใช้รับค่างวดได้จริงผ่าน Web App
Platform: Web App (Clerk)
```

### Phase 2: งานซ่อม + อะไหล่ (สัปดาห์ที่ 6-8)

```
🎯 เป้าหมาย: จัดการงานซ่อมและสต็อกอะไหล่ได้

Tasks:
  ✅ CRUD อะไหล่ (Parts) + หมวดหมู่
  ✅ Stock In/Out + Barcode Scanner (Web)
  ✅ เปิด/ปิด Job Order (งานซ่อม)
  ✅ เพิ่มอะไหล่ + ค่าแรงในงานซ่อม
  ✅ ตัดสต็อกอัตโนมัติ
  ✅ Low Stock Alert
  ✅ ขายอะไหล่ปลีก (ไม่ผ่านงานซ่อม)
  ✅ พิมพ์ใบเสร็จงานซ่อม

Deliverable: ระบบจัดการงานซ่อมและอะไหล่ใช้ได้จริง
Platform: Web App (Clerk)
```

### Phase 3: เงินสดประจำวัน + พ.ร.บ./ภาษี (สัปดาห์ที่ 9-10)

```
🎯 เป้าหมาย: จัดการเงินสดประจำวันและบริการ พ.ร.บ.

Tasks:
  ✅ เปิด/ปิดรอบเงินสดประจำวัน
  ✅ บันทึกรายรับ-รายจ่ายแยกประเภท
  ✅ Auto-link Cash Transaction จาก Module อื่น (ค่างวด, ค่าซ่อม)
  ✅ สรุปยอดประจำวัน
  ✅ บริการ พ.ร.บ. / ต่อภาษี
  ✅ จัดการเอกสารทะเบียน (รับ/คืน)

Deliverable: ระบบเงินสดและ พ.ร.บ. ใช้ได้จริง
Platform: Web App (Clerk)
```

### Phase 4: รายงาน + Dashboard เจ้าของร้าน (สัปดาห์ที่ 11-12)

```
🎯 เป้าหมาย: เจ้าของร้านดูรายงานได้

Tasks:
  ✅ Daily Summary Report
  ✅ Sales Report (รายสัปดาห์/เดือน)
  ✅ Receivable Report (ลูกหนี้ค้างชำระ)
  ✅ Collection Aging Report
  ✅ Repair Report + ผลงานช่าง
  ✅ Inventory Report
  ✅ Export PDF / Excel
  ✅ Owner Dashboard (Web)

Deliverable: รายงานครบถ้วน, เจ้าของร้านดูผ่าน Web ได้
Platform: Web App (Owner section)
```

### Phase 5: Mobile App — เจ้าของร้าน + Collector (สัปดาห์ที่ 13-16)

```
🎯 เป้าหมาย: เจ้าของร้านและ Collector ใช้งานผ่านมือถือได้

Tasks:
  ✅ ตั้ง Expo project
  ✅ Owner App:
    - Dashboard สรุปรายวัน
    - ดูรายงาน
    - Push Notification (ค้างชำระ, เปิด/ปิดร้าน, ยอดผิดปกติ)
    - ดูลูกหนี้ค้างชำระ

Deliverable: Mobile App (Owner) พร้อมใช้
Platform: iOS + Android (Expo)
```

### Phase 5.5: Collection Module — ติดตามหนี้ (สัปดาห์ที่ 17-19)

```
🎯 เป้าหมาย: ระบบติดตามหนี้ครบวงจร (Web มอบหมาย + Mobile ออกพื้นที่)

Tasks:
  ✅ DB Schema: collection_assignments + collection_activities
  ✅ API: Collection endpoints (มอบหมาย, บันทึกผล, รับเงิน, ส่งเงิน)
  ✅ Web (Clerk): หน้ามอบหมายงานติดตาม + ดูสถานะ + ตรวจรับเงิน
  ✅ Mobile (Collector):
    - ดูรายชื่อลูกหนี้ที่ได้รับมอบหมาย (ที่อยู่, เบอร์โทร)
    - โทร + นำทาง (Google Maps) 1 แตะ
    - บันทึกผลติดตาม (ติดต่อได้/ไม่ได้, นัดจ่าย)
    - รับชำระเงินสดหน้างาน + ออกใบเสร็จชั่วคราว (PDF)
    - ดูประวัติลูกค้า: สัญญา, รถ, ค้ำประกัน
    - GPS tracking ตำแหน่ง
    - สรุปเงินที่เก็บ + ส่งเงินกลับร้าน
  ✅ Offline support สำหรับ Collector (Local Storage → Sync)
  ✅ Push Notification แจ้ง Collector เมื่อได้รับงานใหม่

Deliverable: Collector ออกพื้นที่ติดตามหนี้ได้จริง เก็บเงินส่งร้านได้
Platform: Web (Clerk มอบหมาย) + Mobile (Collector ออกพื้นที่)
```

### Phase 6: Enhancement & Polish (สัปดาห์ที่ 20-23)

```
🎯 เป้าหมาย: ปรับปรุงจากข้อมูลการใช้งานจริง

Tasks:
  ✅ Offline support (Service Worker / Local-first)
  ✅ Search optimization (Full-text search)
  ✅ Performance tuning
  ✅ แจ้งเตือน พ.ร.บ./ภาษี ใกล้หมดอายุ
  ✅ Customer portal (ลูกค้าเช็คยอดค้างผ่าน LINE OA)
  ✅ Data export / backup automation
  ✅ Onboarding tutorial สำหรับเสมียนใหม่
  ✅ Bug fixes จาก production usage

Deliverable: ระบบเสถียร ใช้งานจริงได้ดี
```

### สรุป Timeline

```
Phase 0   ─── Foundation ──────── [████]                      2 สัปดาห์
Phase 1   ─── ขายรถ + ผ่อนชำระ ── [██████]                    3 สัปดาห์  ← เริ่มใช้งานจริง
Phase 2   ─── งานซ่อม + อะไหล่ ── [██████]                    3 สัปดาห์
Phase 3   ─── เงินสด + พ.ร.บ. ─── [████]                      2 สัปดาห์
Phase 4   ─── รายงาน + Dashboard  [████]                      2 สัปดาห์
Phase 5   ─── Mobile (Owner) ──── [████████]                  4 สัปดาห์
Phase 5.5 ─── Collection Module ─ [██████]                    3 สัปดาห์  ← Collector ใช้งาน
Phase 6   ─── Enhancement ──────  [████████]                  4 สัปดาห์
                                  ─────────────────────────
                                  รวม ~23 สัปดาห์ (~6 เดือน)

⭐ Key Milestone:
  - สัปดาห์ที่ 5:  เสมียนเริ่มใช้รับค่างวดได้จริง
  - สัปดาห์ที่ 8:  ช่างใช้ Web + Barcode Scanner ได้
  - สัปดาห์ที่ 10: ระบบหลัก (ขาย+ซ่อม+เงินสด) ครบ
  - สัปดาห์ที่ 12: รายงานครบ, เจ้าของร้านดูได้
  - สัปดาห์ที่ 16: Mobile App (Owner) พร้อมใช้
  - สัปดาห์ที่ 19: Collector ออกพื้นที่ติดตามหนี้ได้จริง
  - สัปดาห์ที่ 23: ระบบครบสมบูรณ์
```

---

## Appendix A: Deployment Architecture

```
Production Environment (VPS):

┌─────────────────────────────────────────┐
│           VPS (4 vCPU, 8GB RAM)         │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │ Docker Compose                  │    │
│  │                                 │    │
│  │  ┌───────────┐ ┌────────────┐  │    │
│  │  │ Caddy     │ │ API Server │  │    │
│  │  │ (Reverse  │ │ (Bun +     │  │    │
│  │  │  Proxy +  │→│  Hono.js)  │  │    │
│  │  │  SSL)     │ │ Port: 3000 │  │    │
│  │  └───────────┘ └────────────┘  │    │
│  │                                 │    │
│  │  ┌───────────┐ ┌────────────┐  │    │
│  │  │ Next.js   │ │ PostgreSQL │  │    │
│  │  │ (Web App) │ │ Port: 5432 │  │    │
│  │  │ Port: 3001│ │            │  │    │
│  │  └───────────┘ └────────────┘  │    │
│  │                                 │    │
│  │  ┌───────────┐                  │    │
│  │  │ Redis     │                  │    │
│  │  │ Port: 6379│                  │    │
│  │  └───────────┘                  │    │
│  └─────────────────────────────────┘    │
│                                         │
│  Volumes:                               │
│    /data/postgres  (DB data)            │
│    /data/uploads   (Files)              │
│    /data/backups   (Daily backups)      │
│                                         │
└─────────────────────────────────────────┘

Estimated Cost: ~$20-30/month (Hetzner CX31 or DigitalOcean)
```

## Appendix B: Receipt/Document Templates

```
สำหรับใบเสร็จค่างวด:
┌─────────────────────────────────────┐
│         กมลกิจยานยนต์               │
│    ที่อยู่ร้าน xxxxxxxxxxxxx          │
│    โทร: xxx-xxx-xxxx               │
│─────────────────────────────────────│
│ ใบเสร็จรับเงิน เลขที่: RC-2569-0001│
│ วันที่: 10/03/2569                  │
│─────────────────────────────────────│
│ ลูกค้า: นายสมชาย ใจดี               │
│ สัญญา: KK-2569-0042               │
│ รถ: Honda Wave 110i (กท-1234)      │
│─────────────────────────────────────│
│ ค่างวดที่ 3          2,500.00 บาท  │
│ ค่าปรับล่าช้า          100.00 บาท  │
│─────────────────────────────────────│
│ รวมทั้งสิ้น          2,600.00 บาท  │
│─────────────────────────────────────│
│ ชำระโดย: เงินสด                     │
│ ผู้รับเงิน: นางสาวสมหญิง             │
│                                     │
│ งวดคงเหลือ: 9 งวด                  │
│ ยอดค้างชำระ: 22,500.00 บาท         │
│─────────────────────────────────────│
│    ขอบคุณที่ใช้บริการ                 │
└─────────────────────────────────────┘
```

---

> **เอกสารนี้เป็น Living Document** — จะอัปเดตตาม feedback จากการพัฒนาและใช้งานจริง
>
> สร้างโดย: System Design Session
> วันที่: 10 มีนาคม 2569
> อัปเดตล่าสุด: 10 มีนาคม 2569 — เพิ่ม Collector role, ทบทวน Tech Stack ครบ 10 หมวด, ปรับช่างเป็น Web/PC
