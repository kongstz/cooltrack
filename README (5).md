# 🌡️ AC Service Management System

> ระบบจัดการงานซ่อมและบำรุงรักษาเครื่องปรับอากาศ สำหรับธุรกิจร้านแอร์
> Tech Stack: React + TypeScript · Node.js/Express · MySQL · Prisma ORM · PM2
> Project Codename: **cooltrack**

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Core Features](#core-features)
3. [User Roles & Permissions](#user-roles--permissions)
4. [Module Breakdown](#module-breakdown)
5. [Data Models](#data-models)
6. [QR Code System](#qr-code-system)
7. [Job Order System](#job-order-system)
8. [Quotation System](#quotation-system)
9. [MA Notification System](#ma-notification-system)
10. [Tech Stack & Architecture](#tech-stack--architecture)
11. [API Routes Overview](#api-routes-overview)
12. [Deployment: Production (VPS + PM2)](#deployment-production-vps--pm2)
13. [Deployment: Demo (Vercel + PlanetScale)](#deployment-demo-vercel--planetscale)
14. [Environment Variables](#environment-variables)
15. [Known Gaps / TODO](#known-gaps--todo)
16. [Server & Pricing Guide](#server--pricing-guide)

---

## Project Naming

```
# Folder Structure
cooltrack/
├── frontend/
└── backend/

# Database
DB name     : cooltrack_db
DB user     : cooltrack_user
DB password : (ตั้งเองตอน setup server)
DB host     : localhost
DB port     : 3306

# PM2
PM2 app name : cooltrack-api

# Ports
Backend API  : 3001
Frontend dev : 5173  (vite default)
```

> ชื่อโปรแกรม (brand) ค่อยกำหนดทีหลัง ชื่อ folder/DB ด้านบนใช้ไปก่อน
> ถ้าเปลี่ยนชื่อโปรแกรม แค่แก้ `.env` กับ Nginx config เท่านั้น ไม่กระทบ code

---

## Overview

ระบบนี้ออกแบบมาสำหรับ **ร้านบริการแอร์** ที่ต้องการ:
- บันทึกประวัติการซ่อม/MA ของแอร์แต่ละตัวแบบ digital
- จัดการหลาย Site/Location ในระบบเดียว
- ช่างสแกน QR Code แล้วบันทึกงานจากมือถือได้ทันที
- แอดมินเห็น dashboard ภาพรวมทุก site
- ออกใบเสนอราคาในระบบ พร้อม VAT 7%

---

## Core Features

| Feature | Description |
|---|---|
| 🏢 Multi-Site / Station | แบ่ง location ได้ เช่น โรงพยาบาล A, B, อาคาร C |
| 📱 QR Code per AC Unit | สแกนแล้วเห็นประวัติและ log ทั้งหมดของแอร์ตัวนั้น |
| 🔧 Job Order (ใบงาน) | สร้าง track pending → on process → done |
| 📄 Quotation (ใบเสนอราคา) | เลือก item → คำนวณ VAT 7% อัตโนมัติ |
| 🔔 MA Alert | แจ้งเตือนล่วงหน้า 30 วัน ก่อนครบรอบ MA |
| 📸 Photo Attachment | แนบรูปก่อน/หลังงานได้ ไม่จำกัดจำนวน |
| 👷 Tech Verification | ช่างยืนยัน username/password ก่อนบันทึกงาน |
| 📊 Admin Dashboard | เห็นสถิติทุก site: งานค้าง, เสร็จแล้ว, ใกล้ MA |

---

## User Roles & Permissions

### 1. Admin
- เห็นข้อมูลทั้งหมดทุก Station
- สร้าง/แก้ไข/ลบ User, Station, AC Master Item
- กำหนดสิทธิ์ User แต่ละคนว่าดู Station ไหนได้บ้าง
- เห็น Dashboard ภาพรวม (งานค้าง, จบ, ใกล้ MA ทุก site)
- จัดการ Job Order ทุก site

### 2. Staff / Technician
- เห็นเฉพาะ Station ที่แอดมินกำหนดให้
- สร้างใบงาน (Job Order) ได้
- บันทึกงานซ่อม/MA ของแอร์ใน station ตัวเอง
- ยืนยันตัวตนด้วย username/password ก่อนบันทึก
- สิทธิ์เพิ่มเติมขึ้นอยู่กับที่แอดมินกำหนด

> **Note for Codex:** Permission ควรทำเป็น role-based + station-based access control
> เก็บใน `user_station_permissions` table และ `user_feature_permissions` table แยกกัน

---

## Module Breakdown

### Module 1: Station Management
- CRUD Station (แอดมินเท่านั้น)
- กำหนด User ให้แต่ละ Station
- ตัวอย่าง Station: `โรงพยาบาลปากน้ำโพ1`, `อาคารสำนักงาน B`

### Module 2: AC Master Item
- สร้าง AC Unit แต่ละตัว (MASTER record)
- ข้อมูล: รหัสแอร์ (Air ID), ยี่ห้อ, รุ่น, SN, ตำแหน่งติดตั้ง, วันติดตั้ง, อาคาร, ชั้น
- ผูกกับ Station
- แอดมิน → dropdown เลือก Station ได้
- Staff → สร้างได้เฉพาะใน Station ของตัวเอง
- รองรับการกรอก SN เพื่อดึงข้อมูลเหมือนสแกน QR Code

### Module 3: QR Code System
- Generate QR Code จาก Air ID หรือ SN
- สแกนแล้วเปิดหน้า AC Detail ทันที
- แสดง: ข้อมูลแอร์, ประวัติซ่อม/MA, Log ทั้งหมด, รอบ MA ถัดไป
- มีปุ่ม "บันทึกกิจกรรมใหม่" และ "สร้างใบเสนอราคา" ในหน้าเดียวกัน

### Module 4: Job Order (ใบงาน)
- รหัส Format: `JOB-YYYYMM000001` (reset ทุกต้นเดือน)
- สถานะ: `pending` → `on_process` → `done`
- แสดงอายุงาน (กี่วันที่รับไว้แล้ว)
- แนบรูปได้ไม่จำกัด (ก่อน/หลัง)
- ระบุ ชื่อลูกค้า, เบอร์โทร, รายละเอียดงาน
- เชื่อมกับ AC Master Item (optional)

### Module 5: Service Record (บันทึกการซ่อม/MA)
- แยกประเภท: `repair` | `ma` | `install` | `other`
- ช่างยืนยัน username/password ก่อนบันทึก
- กรอกรายละเอียดงาน, วัสดุที่ใช้, ค่าแรง
- กำหนดวัน MA ถัดไป
- แนบรูปได้

### Module 6: Quotation (ใบเสนอราคา)
- สร้างจากหน้า AC Detail หรือ Job Order
- เลือก Item พร้อมราคา (ใส่เองได้)
- ระบุรายละเอียดงานอิสระ
- คำนวณ subtotal, VAT 7%, total อัตโนมัติ
- บันทึก log ผูกกับ AC Master Item นั้น
- Print / Export PDF

### Module 7: MA Alert & Dashboard
- แสดงแอร์ที่ใกล้ครบ MA ภายใน 30 วัน
- Staff เห็นเฉพาะ station ตัวเอง
- Admin เห็นทุก station
- หลัง MA แล้ว → alert หายไปอัตโนมัติ

---

## Data Models

```typescript
// Station
Station {
  id: string
  name: string
  address: string
  contact_person?: string
  contact_phone?: string
  created_at: Date
}

// AC Master Item
AcUnit {
  id: string             // Air ID เช่น AC-004718 (ตั้งเองได้)
  serial_number: string  // SN จากป้ายเครื่อง
  station_id: string
  location_detail: string  // เช่น "เคาน์เตอร์พยาบาล W32"
  building: string
  floor?: string
  brand: string
  model: string
  installed_date: Date
  next_ma_date?: Date
  qr_code_url: string
  created_by: string
  created_at: Date
}

// Service Log
ServiceLog {
  id: string
  ac_unit_id: string
  job_order_id?: string
  type: 'repair' | 'ma' | 'install' | 'other'
  description: string
  technician_id: string   // verified by username/password
  performed_date: Date
  next_ma_date?: Date
  photos: string[]        // เก็บเป็น JSON string ใน MySQL
  materials_used?: string
  notes?: string
  created_at: Date
}

// Job Order
JobOrder {
  id: string              // JOB-202608000001
  station_id?: string
  ac_unit_id?: string
  customer_name: string
  customer_phone: string
  description: string
  status: 'pending' | 'on_process' | 'done'
  assigned_technician_id?: string
  photos: string[]        // เก็บเป็น JSON string ใน MySQL
  received_at: Date
  started_at?: Date
  completed_at?: Date
  created_by: string
}

// Quotation
Quotation {
  id: string
  ac_unit_id?: string
  job_order_id?: string
  items: QuotationItem[]  // เก็บเป็น JSON string ใน MySQL
  detail_note: string
  subtotal: number
  vat_amount: number      // 7%
  total: number
  created_by: string
  created_at: Date
}

QuotationItem {
  description: string
  quantity: number
  unit_price: number
  total: number
}

// User
User {
  id: string
  username: string
  password_hash: string
  full_name: string
  role: 'admin' | 'staff'
  phone?: string
  is_active: boolean
  created_at: Date
}

// User-Station Permission
UserStationPermission {
  user_id: string
  station_id: string
  can_view: boolean
  can_create_job: boolean
  can_record_service: boolean
  can_create_quotation: boolean
}
```

> **Note for Codex (MySQL):** Array fields เช่น `photos`, `items` ให้ใช้ `TEXT` column
> แล้วทำ `JSON.stringify` / `JSON.parse` ใน service layer หรือใช้ Prisma `Json` type

---

## QR Code System

```
Flow:
1. Admin/Staff สร้าง AC Unit → ระบบ Generate QR Code อัตโนมัติ
2. QR Code encode URL: https://yourdomain.com/ac/{ac_unit_id}
3. Print QR Code ติดที่ตัวเครื่อง
4. ช่างสแกน → เปิดหน้า AC Detail ทันที
5. หน้า AC Detail แสดง:
   - ข้อมูลเครื่อง (brand, model, SN, location)
   - ประวัติการซ่อม/MA ล่าสุด
   - รอบ MA ถัดไป
   - ปุ่ม [บันทึกกิจกรรมใหม่]
   - ปุ่ม [สร้างใบเสนอราคา]

Alternative: กรอก SN ในระบบ → ดึงข้อมูลเหมือนสแกน QR
```

---

## Job Order System

### ID Format
```
JOB-{YYYY}{MM}{NNNNNN}
ตัวอย่าง: JOB-202608000001

- YYYY   = ปี (2026)
- MM     = เดือน (08)
- NNNNNN = ลำดับ 6 หลัก reset ทุกต้นเดือน
```

### Status Flow
```
[pending] → [on_process] → [done]
   ↑ รับงานแล้ว    ↑ เริ่มทำ      ↑ เสร็จแล้ว
   บันทึก received_at  บันทึก started_at  บันทึก completed_at
```

### Age Calculation
แสดง "รับไว้ X วันแล้ว" โดยคำนวณจาก `received_at` ถึง now

---

## Quotation System

```
รายการ Items (เพิ่มได้ไม่จำกัด):
- รายละเอียด | จำนวน | ราคาต่อหน่วย | รวม

Subtotal = sum(items)
VAT 7%   = Subtotal × 0.07
Total    = Subtotal + VAT

- ใส่ Detail Note อิสระ (textarea)
- บันทึก log ผูกกับ AC Unit
- Export เป็น PDF ได้
```

---

## MA Notification System

```
Logic:
- ทุก AC Unit มี next_ma_date
- ถ้า next_ma_date - today <= 30 วัน → แสดงใน alert list (สีเหลือง)
- ถ้า next_ma_date - today <= 7 วัน  → แสดง highlight สีแดง
- ถ้า next_ma_date < today           → เกินกำหนดแล้ว (สีแดงเข้ม)
- หลังบันทึก MA แล้ว → อัพเดต next_ma_date → ออกจาก alert list

Admin Dashboard:
- เห็น MA Alert ทุก station (grouped by station)
- Count: ใกล้ครบ MA / เกินกำหนด / งานค้าง / งานเสร็จ

Staff Dashboard:
- เห็นเฉพาะ station ของตัวเอง
```

---

## Tech Stack & Architecture

### Production (VPS)
```
Frontend : React + TypeScript + Vite
Backend  : Node.js + Express + TypeScript
Database : MySQL 8.x  ← ถนัดอยู่แล้ว เหมือน XAMPP
ORM      : Prisma (รองรับ MySQL เต็มที่)
Auth     : JWT
Upload   : Multer + Sharp (resize รูปก่อนเก็บ)
Process  : PM2
Proxy    : Nginx
OS       : Ubuntu 22.04 LTS
```

### Demo (Vercel)
```
Frontend : React + TypeScript + Vite → Vercel (static)
Backend  : Express แปลงเป็น Vercel Serverless Functions
Database : PlanetScale (MySQL-compatible, free tier)
           หรือ Neon.tech (PostgreSQL, ถ้าไม่ติด MySQL)
Storage  : Cloudinary (รูปภาพ, free tier)
```

### Libraries หลัก
```
qrcode / react-qr-code     → Generate QR Code
@react-pdf/renderer        → Export PDF ใบเสนอราคา
react-query                → Data fetching + cache
react-router-dom v6        → Routing
tailwindcss                → Styling
dayjs                      → จัดการวันที่ (เบากว่า moment)
bcryptjs                   → Hash password
jsonwebtoken               → JWT
multer                     → File upload
sharp                      → Resize รูป
```

### Project Structure
```
cooltrack/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── AcDetail.tsx        ← QR Code landing page
│   │   │   ├── JobOrders.tsx
│   │   │   ├── Quotation.tsx
│   │   │   ├── Stations.tsx
│   │   │   └── Settings.tsx
│   │   ├── hooks/
│   │   ├── types/
│   │   └── utils/
│   ├── package.json
│   └── vite.config.ts
│
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   ├── controllers/
│   │   ├── middleware/
│   │   │   ├── auth.ts
│   │   │   └── permissions.ts
│   │   ├── services/
│   │   └── utils/
│   ├── prisma/
│   │   └── schema.prisma    ← provider = "mysql"
│   ├── uploads/             ← รูปภาพ (production)
│   └── package.json
│
├── ecosystem.config.js      ← PM2 config
└── README.md
```

---

## API Routes Overview

```
AUTH
POST   /api/auth/login
POST   /api/auth/logout
GET    /api/auth/me

STATIONS
GET    /api/stations
POST   /api/stations
PUT    /api/stations/:id
DELETE /api/stations/:id

AC UNITS
GET    /api/ac-units                    (filter by station)
POST   /api/ac-units
GET    /api/ac-units/:id               (QR Code landing)
GET    /api/ac-units/by-sn/:sn         (กรอก SN)
PUT    /api/ac-units/:id
POST   /api/ac-units/:id/qr-code       (regenerate QR)

SERVICE LOGS
GET    /api/ac-units/:id/logs
POST   /api/ac-units/:id/logs          (ต้องยืนยัน tech password)

JOB ORDERS
GET    /api/jobs                        (filter by status, station)
POST   /api/jobs
GET    /api/jobs/:id
PUT    /api/jobs/:id/status
POST   /api/jobs/:id/photos

QUOTATIONS
GET    /api/quotations
POST   /api/quotations
GET    /api/quotations/:id
GET    /api/quotations/:id/pdf

USERS
GET    /api/users
POST   /api/users
PUT    /api/users/:id
PUT    /api/users/:id/permissions

DASHBOARD
GET    /api/dashboard/summary           (admin: all stations)
GET    /api/dashboard/ma-alerts         (upcoming MA within 30 days)
```

---

## Deployment: Production (VPS + PM2)

### ขั้นตอน

```bash
# 1. SSH เข้า Server
ssh user@your-server-ip

# 2. Install Node.js v20 LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 3. Install PM2
npm install -g pm2

# 4. Install MySQL
sudo apt install mysql-server -y
sudo mysql_secure_installation

# 5. สร้าง Database + User
sudo mysql -u root -p
> CREATE DATABASE cooltrack_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
> CREATE USER 'cooltrack_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
> GRANT ALL PRIVILEGES ON cooltrack_db.* TO 'cooltrack_user'@'localhost';
> FLUSH PRIVILEGES;

# 6. Clone & Install
git clone https://github.com/yourrepo/cooltrack.git
cd cooltrack
cd backend && npm install
cd ../frontend && npm install && npm run build

# 7. Setup .env (ดู Environment Variables ด้านล่าง)

# 8. Migrate DB ด้วย Prisma
cd backend
npx prisma migrate deploy
npx prisma db seed   # ถ้ามี seed data (admin user)

# 9. Start PM2
pm2 start ecosystem.config.js
pm2 save && pm2 startup
```

### ecosystem.config.js
```javascript
module.exports = {
  apps: [
    {
      name: 'cooltrack-api',
      cwd: './backend',
      script: 'npm',
      args: 'start',
      env: {
        NODE_ENV: 'production',
        PORT: 3001
      }
    }
  ]
}
```

### Nginx Config
```nginx
server {
    listen 80;
    server_name yourdomain.com;

    # Frontend (static build)
    location / {
        root /var/www/cooltrack/frontend/dist;
        try_files $uri $uri/ /index.html;
    }

    # Backend API
    location /api {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Uploaded files (รูปภาพ)
    location /uploads {
        alias /var/www/cooltrack/backend/uploads;
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

### Auto Backup MySQL (cron)
```bash
# เพิ่มใน crontab: crontab -e
0 2 * * * mysqldump -u cooltrack_user -pStrongPassword123! cooltrack_db > /backup/cooltrack_$(date +\%Y\%m\%d).sql
# ล้าง backup เก่ากว่า 30 วัน
0 3 * * * find /backup -name "*.sql" -mtime +30 -delete
```

---

## Deployment: Demo (Vercel + PlanetScale)

> ใช้สำหรับสาธิตให้ลูกค้าดู ไม่เหมาะ production จริง (Serverless มี cold start, ไม่มี local file storage)

### Database สำหรับ Demo

**ตัวเลือก 1: PlanetScale** (MySQL-compatible, ฟรี 5GB)
```
- สมัคร planetscale.com
- สร้าง database → copy connection string
- ใส่ใน DATABASE_URL แทน
- schema.prisma: provider = "mysql"
  + relationMode = "prisma"  ← PlanetScale ต้องการ
```

**ตัวเลือก 2: Neon.tech** (PostgreSQL, ฟรี 512MB)
```
- ถ้าไม่ติด MySQL ก็ใช้ได้เลย
- schema.prisma: provider = "postgresql"
- connection string ได้จาก neon.tech dashboard
```

### รูปภาพสำหรับ Demo: Cloudinary
```
- สมัคร cloudinary.com (free tier: 25GB)
- ใช้ multer-storage-cloudinary แทน multer ปกติ
- ไม่ต้องเก็บรูปบน server เลย
```

### vercel.json (root ของ project)
```json
{
  "version": 2,
  "builds": [
    {
      "src": "backend/src/index.ts",
      "use": "@vercel/node"
    },
    {
      "src": "frontend/package.json",
      "use": "@vercel/static-build",
      "config": { "distDir": "dist" }
    }
  ],
  "routes": [
    { "src": "/api/(.*)", "dest": "backend/src/index.ts" },
    { "src": "/(.*)", "dest": "frontend/dist/$1" }
  ]
}
```

### Deploy ขั้นตอน
```bash
# 1. สมัคร vercel.com + เชื่อม GitHub repo
# 2. สมัคร planetscale.com → copy DATABASE_URL
# 3. สมัคร cloudinary.com → copy API credentials
# 4. ใส่ Environment Variables ใน Vercel Dashboard:
#    DATABASE_URL, JWT_SECRET, CLOUDINARY_URL ฯลฯ
# 5. git push → Vercel deploy อัตโนมัติ
```

---

## Environment Variables

### Production (.env บน VPS)
```env
# Database
DATABASE_URL="mysql://cooltrack_user:StrongPassword123!@localhost:3306/cooltrack_db"

# Auth
JWT_SECRET="your-super-secret-key-change-this-min-32-chars"
JWT_EXPIRES_IN="7d"

# Server
PORT=3001
NODE_ENV=production

# File Upload (local)
UPLOAD_DIR="./uploads"
MAX_FILE_SIZE_MB=10

# Frontend URL (for CORS)
FRONTEND_URL="https://yourdomain.com"
```

### Demo (.env บน Vercel)
```env
# Database (PlanetScale หรือ Neon)
DATABASE_URL="mysql://user:pass@host/db?sslaccept=strict"

# Auth
JWT_SECRET="your-super-secret-key-change-this-min-32-chars"
JWT_EXPIRES_IN="7d"

# Server
NODE_ENV=production

# Cloudinary (รูปภาพ)
CLOUDINARY_CLOUD_NAME="your-cloud-name"
CLOUDINARY_API_KEY="your-api-key"
CLOUDINARY_API_SECRET="your-api-secret"

# Frontend
VITE_API_URL="https://your-project.vercel.app/api"
```

---

## Known Gaps / TODO

### ต้องตัดสินใจก่อน Build
- [ ] **Air ID vs SN**: ตอนนี้ออกแบบให้มีทั้งสอง (Air ID ตั้งเอง + SN จากป้ายเครื่อง) — ยืนยันกับลูกค้าก่อน
- [ ] **Offline support**: ถ้าช่างทำงานในที่ไม่มีเน็ต อาจต้อง PWA + local cache
- [ ] **รูปภาพ Production**: Local disk (ง่ายกว่า ไม่มีค่าใช้จ่าย) vs Cloudinary (ไม่ต้องดูแล storage เอง)

### Features ที่ยังไม่ได้ออกแบบละเอียด
- [ ] **Checklist Template**: PM mention checklist — ต้องการ template กรอกในระบบไหม หรือแค่ช่อง text?
- [ ] **Work Order PDF**: นอกจาก Quotation ยังต้องการ PDF ใบงานด้วยไหม?
- [ ] **Report/Export**: Export ประวัติ MA รายเดือน/รายปี เป็น Excel ได้ไหม?
- [ ] **Line Notify**: MA Alert แจ้งผ่าน Line OA หรือแค่ใน dashboard?
- [ ] **Customer Portal**: ลูกค้าจะเข้าดูประวัติเองได้ไหม หรือดูได้แค่ผ่านช่าง?
- [ ] **Inventory/Stock**: ติดตาม stock อะไหล่ที่ใช้ไปไหม?
- [ ] **Invoice**: ออกใบแจ้งหนี้ได้เลยหรือแค่ใบเสนอราคา?

---

## 💰 Server & Pricing Guide

### VPS ที่แนะนำ (ราคา 2026)

| Provider | Spec | ราคา/เดือน | หมายเหตุ |
|---|---|---|---|
| **Vultr** ⭐ | 1 vCPU, 2GB RAM, 55GB SSD | ~$6 (~210 บาท) | DC Singapore, เร็ว |
| **Hostinger VPS** | 1 vCPU, 4GB RAM, 50GB SSD | ~$5-7 (~175-245 บาท) | ถูกที่สุด |
| **DigitalOcean** | 1 vCPU, 2GB RAM, 50GB SSD | $12 (~420 บาท) | Support ดี |
| **AWS Lightsail** | 1 vCPU, 2GB RAM, 60GB SSD | $10 (~350 บาท) | เสถียร แต่แพงกว่า |

> แนะนำ Vultr หรือ Hostinger $5-7/เดือน รองรับ 20 concurrent users สบาย

### ต้นทุนรายปี
```
Server (Vultr $6/เดือน) : ~2,500 บาท/ปี
Domain (.com)           :   ~500 บาท/ปี
SSL                     :  ฟรี (Let's Encrypt)
Backup                  :  ฟรี (cron script)
─────────────────────────────────────────
รวมต้นทุน               : ~3,000 บาท/ปี
```

### ราคาโปรแกรมที่แนะนำเก็บลูกค้า

#### Option A: One-time + MA รายปี
```
ค่าพัฒนา (ครั้งแรก)  : 35,000 - 60,000 บาท
ค่า MA รายปี         : 15-20% ของค่าพัฒนา
                       เช่น Dev 50,000 → MA 7,500-10,000 บาท/ปี

MA ครอบคลุม:
  - Bug fix & security update
  - Minor feature update
  - Server monitoring & backup
  - Support ผ่านโทรศัพท์/Line
```

#### Option B: SaaS รายปี (แนะนำ — recurring income)
```
Tier        | Station  | Users  | ราคา/ปี
────────────|──────────|────────|──────────────
Basic       | 1        | 5      | 6,000 บาท
Standard    | 5        | 15     | 15,000 บาท
Pro         | ไม่จำกัด | ไม่จำกัด| 25,000 บาท

+ ค่า Setup ครั้งแรก: 5,000 - 10,000 บาท
```

#### สรุปแนะนำ
```
ลูกค้าร้านแอร์ขนาดกลาง (1-3 สาขา):
  → SaaS Standard 15,000 บาท/ปี + Setup 8,000 บาท

ถ้าขายหลายร้าน (SaaS multi-tenant):
  → Server เดียวรองรับได้ 10+ ร้าน
  → Margin ดีขึ้นเรื่อยๆ ทุกร้านที่เพิ่ม
  → ต้นทุน server อาจเพิ่มเป็น $12-20/เดือน แต่ยังคุ้มมาก
```

---

*Last updated: 2026-08-28*
*Version: 1.2.0*
