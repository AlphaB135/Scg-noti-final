# SCG NOTI – Apple‑Level Secure Web App (README)

> **Goal:** ระบบแจ้งเตือน & จัดการงานภายใน SCG ที่มีความปลอดภัยระดับ Apple‑level  
> **Stack:** React TSX + Tailwind • Express + Prisma (SQL Server) • Redis + BullMQ • Docker • pnpm mono‑repo

---

## 🗂️ สารบัญ
1. [แผนงาน & สถานะ](#แผนงาน--สถานะ)
2. [Security Backlog รายละเอียด](#security-backlog-รายละเอียด)
3. [โครงสร้างโฟลเดอร์](#โครงสร้างโฟลเดอร์)
4. [เครื่องมือที่ใช้](#เครื่องมือที่ใช้)
5. [ไฟล์ .env ตัวอย่าง](#ไฟล์-env-ตัวอย่าง)
6. [Docker‑compose (Dev Infra)](#docker-compose-dev-infra)
7. [คู่มือเซ็ตอัป (Dev)](#คู่มือเซ็ตอัป-dev)
8. [สคริปต์ pnpm ที่ใช้บ่อย](#สคริปต์-pnpm-ที่ใช้บ่อย)

---

## แผนงาน & สถานะ

| Phase | หมวดงาน | งานย่อย (✅ = เสร็จ) |
|-------|---------|-----------------------|
| **🚀 Phase A – Core Foundation** | • Scaffold repo + pnpm workspace ✅<br>• Express + TS backend ✅<br>• Prisma schema (User+SessionGuard) ✅<br>• Auth Flow (login/refresh/logout) ⏳<br>• Security middleware (Helmet/CORS/rate‑limit) ⏳<br>• Redis session store ⏳<br>• React TSX + Tailwind scaffold ✅<br>• AuthContext + ProtectedRoute ⏳ |
| **🟧 Phase B – Feature & Security** | • Dashboard vertical slice ⏳<br>• Notifications CRUD ⏳<br>• Approvals module ⏳<br>• Work Calendar ⏳<br>• Settings 7 แท็บ ⏳<br>• CSRF double‑submit ⏳<br>• Account Lockout (×5) ⏳<br>• Refresh‑token rotation job ⏳ |
| **🟩 Phase C – Scale & Nice‑to‑Have** | • 2FA (TOTP / LINE OTP) ⏳<br>• Device fingerprint trust list ⏳<br>• IP allow‑list per role ⏳<br>• Webhook retry log ⏳<br>• Prometheus / Grafana / Loki ⏳<br>• k8s Helm chart ⏳ |

---

## Security Backlog รายละเอียด

| # | Feature | สถานะ | หมายเหตุ |
|---|---------|-------|----------|
| 1 | Refresh‑token rotation 15 min | ⏳ | BullMQ job + `currentRefreshId` |
| 2 | IP binding กับ session | ⏳ | compare `req.ip` ทุก `/me` |
| 3 | Account lockout ≥5 fails | ⏳ | `status = LOCKED` |
| 4 | CSRF double‑submit token | ⏳ | `csurf` middleware |
| 5 | Security headers (Helmet) | ⏳ | `app.use(helmet())` |
| 6 | Audit Log Prisma model | ⏳ | action, ip, ua, status |
| 7 | 2FA (TOTP / LINE OTP) | ⏳ | phase C |
| 8 | Device fingerprint binding | ⏳ | UA+platform hash |
| 9 | Secure session recovery | ⏳ | backup codes / OTP |
|10 | Anomaly detection alert | ⏳ | IP/device change →

---

## 🗂️ โครงสร้างโฟลเดอร์
```text
scg-noti/
├── backend/
│   ├── src/
│   │   ├── config/            # env, logger, helmet, rate‑limit
│   │   ├── prisma/            # schema.prisma, migrations/
│   │   ├── middleware/
│   │   │   ├── auth/          # guards, refresh‑check
│   │   │   ├── csrf.ts
│   │   │   └── auditLog.ts
│   │   ├── modules/
│   │   │   ├── auth/          # login, refresh, 2FA, lockout
│   │   │   ├── notifications/
│   │   │   ├── approvals/
│   │   │   ├── rpa/
│   │   │   ├── auditLogs/
│   │   │   └── settings/      # system & user settings
│   │   ├── routes/            # express.Router split by module
│   │   ├── utils/             # mailer, sms, webhook, hasher
│   │   ├── jobs/              # cron / bullmq (rotating token, digest)
│   │   └── server.ts
│   ├── tests/                 # jest / supertest
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   │   ├── lib/
│   │   │   ├── real-api.ts    # axios withCredentials = true
│   │   │   └── types.ts       # shared TS types
│   │   ├── hooks/             # useAuth, useFetch, useGuard
│   │   ├── contexts/          # AuthContext, ThemeContext
│   │   ├── layouts/           # DashboardLayout, SettingsLayout
│   │   ├── pages/
│   │   │   ├── dashboard/
│   │   │   ├── notifications/
│   │   │   ├── approvals/
│   │   │   ├── rpa/
│   │   │   ├── audit-logs/
│   │   │   └── settings/
│   │   │        ├── profile.tsx
│   │   │        ├── notifications.tsx
│   │   │        ├── system.tsx
│   │   │        ├── appearance.tsx
│   │   │        ├── security.tsx
│   │   │        ├── integrations.tsx
│   │   │        └── data.tsx
│   │   ├── components/
│   │   │   ├── ui/            # buttons, cards, modal
│   │   │   ├── charts/        # dashboard KPI bars
│   │   │   ├── tables/
│   │   │   └── calendar/      # WorkCalendar component
│   │   ├── routes.tsx         # React‑Router / TanStack
│   │   └── main.tsx
│   ├── public/
│   └── vite.config.ts
│
└── docs/                      # ADR, API spec (OpenAPI), ER‑diagram

```
#### เครื่องมือที่ใช้

| Layer | Tool | Purpose |
|-------|------|---------|
| FE | **React 18 TSX**, **Vite**, **Tailwind** | SPA |
| Data Fetch | **TanStack Query** | cache / retry |
| BE | **Express + TypeScript** | REST API |
| ORM | **Prisma** | Type‑safe SQL Server |
| Cache/Queue | **Redis 7**, **BullMQ** | session, jobs |
| Security | **Helmet**, **CSURF**, **express‑rate‑limit** | hardening |
| Dev/CI | **pnpm workspaces**, **Docker**, **GitHub Actions** | mono‑repo |
| Testing | Jest/Supertest • Vitest/RTL • Cypress | unit‑to‑E2E |

---

## ไฟล์ .env ตัวอย่าง

`backend/.env.example`
```env
DATABASE_URL="sqlserver://sa:StrongP@ssw0rd@localhost:1433;database=scgnoti;encrypt=true"
REDIS_URL="redis://localhost:6379"
JWT_SECRET="supersecretjwt"
NODE_ENV=development

```
##Docker‑compose (Dev Infra)

version: "3.9"
services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: Y
      SA_PASSWORD: StrongP@ssw0rd
    ports: ["1433:1433"]
    volumes: ["mssql_data:/var/opt/mssql"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  mssql_data:




##คู่มือเซ็ตอัป (Dev)
git clone https://github.com/your-org/scg-noti-final.git
cd scg-noti-final
pnpm install                       # ติดตั้งทุก workspace

# เรียกใช้ Docker infra
cd infra && docker compose up -d && cd ..

# Migrate DB & seed (ถ้ามี)
pnpm --filter backend prisma migrate dev --name init

# Start dev servers (2 เทอร์มินัล)
pnpm --filter backend dev      # http://localhost:3001
pnpm --filter frontend dev     # http://localhost:5173



##สคริปต์ pnpm ที่ใช้บ่อย









## auth.controller.ts 
อธิบายโค้ด
login()

รับ email กับ password

เช็คว่าผู้ใช้มีอยู่หรือไม่ และรหัสผ่านตรงหรือเปล่า

ถ้าผ่าน จะสร้าง JWT แล้วเก็บไว้ใน HttpOnly Cookie (ปลอดภัยกว่า localStorage)

ตอบกลับว่า login สำเร็จ

logout()

ล้าง cookie ชื่อ token เพื่อลบ session

ตอบกลับว่า logout แล้ว

me()

ดึงข้อมูลผู้ใช้จาก req.user.id (ซึ่งต้องมี middleware decode JWT แล้วแนบไว้ก่อนหน้า)

ดึงข้อมูลจากฐานข้อมูลพร้อม adminProfile (กรณีเป็น admin)

ส่งข้อมูลกลับ


##pnpm --filter backend exec prisma db seed
อัพข้อมูลขึ้นdata base 


