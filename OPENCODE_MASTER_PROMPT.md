# OPENCODE MASTER PROMPT — Clínica Automotriz Arellan Hnos
# Revisión, Refactorización, Base de Datos y Conexión Full-Stack

---

## ROL Y CONTEXTO

Actúa como ingeniero de software senior full-stack, arquitecto de sistemas distribuidos y especialista en DDD (Domain-Driven Design) con más de 15 años de experiencia en sistemas empresariales de alta disponibilidad. Tu objetivo es revisar, refactorizar, completar y conectar el ecosistema digital completo de "Clínica Automotriz Arellan Hnos", asegurando que cada producto digital sea funcional, robusto, interconectado y escalable para millones de usuarios.

---

## CONTEXTO DEL NEGOCIO

"Clínica Automotriz Arellan Hnos" es un taller automotriz familiar en Lima, Perú (Surquillo). Está siendo digitalizado completamente. Los dueños son Edgar y Juan (gerentes), Ana (operaciones) y una hija (finanzas). Tienen trabajadores, mecánicos y un practicante.

**Problema crítico resuelto por este sistema:** control de personal, finanzas, inventario, órdenes de trabajo, clientes y trazabilidad completa de todas las operaciones.

---

## STACK TECNOLÓGICO CONFIRMADO

```
BACKEND      → NestJS (TypeScript) + Prisma ORM + PostgreSQL + Redis
FRONTEND WEB → Next.js 14+ (App Router) + TypeScript + Tailwind CSS
MOBILE/PWA   → React Native / Expo o Next.js PWA
MECHANIC UI  → Next.js (tablet-first) + TypeScript + Tailwind CSS
CLIENT PORTAL→ Next.js + TypeScript + Tailwind CSS
STATUS DASH  → Upptime (ya configurado)
DESIGN SYSTEM→ Turborepo + Storybook + shadcn/ui base
AUTH         → JWT + Refresh Tokens + RBAC (roles: OWNER, MANAGER, FINANCE, MECHANIC, APPRENTICE, CLIENT)
CACHE        → Redis (sesiones, rate limiting, real-time)
NOTIFICACIONES → WebSockets (Socket.io) + Push Notifications
INFRA        → Docker + Docker Compose (local) → preparado para Railway/Render/AWS ECS
```

---

## ESTRUCTURA DE REPOSITORIOS EXISTENTE

```
CLINICA AUTOMOTRIZ ARELLA.../
├── .agents/                    ← Skills de OpenCode
├── .claude/                    ← Config Claude
├── .opencode/                  ← Config OpenCode
├── arellan-business-ops/       ← Operaciones y procesos de negocio
├── arellan-client-portal/      ← Portal de clientes (Next.js)
├── arellan-data-intelligence/  ← Analytics y BI
├── arellan-design-system/      ← Design system compartido
├── arellan-docs-governance/    ← Documentación
├── arellan-frontend-web/       ← Panel admin web (Next.js) ← ACTIVO
├── arellan-hardware-iot/       ← Módulo IoT futuro
├── arellan-infrastructure/     ← Docker, CI/CD, configs cloud
├── arellan-mechanic-ui/        ← UI para mecánicos en tablet (Next.js)
├── arellan-mobile-app/         ← App gerencial PWA/móvil
├── arellan-platform/           ← BACKEND PRINCIPAL (NestJS)
├── arellan-security-compliance/← Documentación legal y seguridad
├── arellan-status-dashboard/   ← Upptime (ya configurado)
├── ECOSYSTEM.md
├── opencode.json
└── skills-lock.json
```

### BACKEND (arellan-platform) — Estructura confirmada:
```
arellan-platform/
├── prisma/
│   ├── schema.prisma           ← REVISAR Y EXPANDIR
│   ├── seed.ts                 ← COMPLETAR con datos realistas
│   └── migrations/
│       └── 20260602003437_init/
│           └── migration.sql
└── src/
    ├── app.module.ts
    ├── main.ts
    ├── common/
    │   ├── decorators/
    │   │   ├── current-user.decorator.ts
    │   │   └── roles.decorator.ts
    │   ├── filters/
    │   │   └── http-exception.filter.ts
    │   ├── guards/
    │   │   ├── jwt-auth.guard.ts
    │   │   ├── mfa-required.guard.ts
    │   │   └── roles.guard.ts
    │   ├── health/
    │   │   ├── health.controller.ts
    │   │   └── health.module.ts
    │   ├── interceptors/
    │   │   └── audit.interceptor.ts
    │   ├── prisma/
    │   │   ├── prisma.module.ts
    │   │   └── prisma.service.ts
    │   ├── public/
    │   │   ├── public.controller.ts
    │   │   └── public.module.ts
    │   └── redis/
    │       ├── redis.module.ts
    │       └── redis.service.ts
    └── modules/
        ├── audit/
        ├── auth/
        ├── clients/
        ├── finance/
        ├── inventory/
        ├── orders/
        ├── personnel/
        └── vehicles/
```

---

## MISIÓN PRINCIPAL — QUÉ DEBES HACER AHORA

### 1. REVISAR TODO EL CÓDIGO EXISTENTE

Lee todos los archivos del proyecto antes de hacer cambios. Para cada producto:
- Identifica código incompleto, placeholders, TODOs, stubs vacíos
- Detecta inconsistencias entre módulos
- Encuentra endpoints que no están conectados al frontend
- Identifica queries Prisma que faltan o están incompletas
- Verifica que los DTOs tienen validación completa con `class-validator`
- Verifica que los guards están aplicados correctamente en todos los controllers

### 2. EXPANDIR Y COMPLETAR EL SCHEMA PRISMA

**OBLIGATORIO:** El schema de Prisma debe contener TODOS estos modelos con sus relaciones completas:

```prisma
// ═══════════════════════════════════════
// USUARIOS Y AUTENTICACIÓN
// ═══════════════════════════════════════

model User {
  id            String       @id @default(cuid())
  email         String       @unique
  password      String
  role          UserRole     @default(MECHANIC)
  status        UserStatus   @default(ACTIVE)
  mfaEnabled    Boolean      @default(false)
  mfaSecret     String?
  lastLoginAt   DateTime?
  lastLoginIp   String?
  failedAttempts Int         @default(0)
  lockedUntil   DateTime?
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt
  deletedAt     DateTime?    // Soft delete

  profile       Personnel?
  sessions      Session[]
  auditLogs     AuditLog[]
  approvals     Approval[]   @relation("ApprovedBy")
  requests      Approval[]   @relation("RequestedBy")

  @@index([email])
  @@index([role])
}

enum UserRole {
  SUPER_ADMIN
  OWNER        // Edgar, Juan
  MANAGER      // Ana
  FINANCE      // Hija de Edgar/Juan
  MECHANIC
  APPRENTICE
  CLIENT
}

enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
  PENDING_VERIFICATION
}

model Session {
  id           String    @id @default(cuid())
  userId       String
  user         User      @relation(fields: [userId], references: [id])
  token        String    @unique
  deviceInfo   String?
  ipAddress    String?
  userAgent    String?
  expiresAt    DateTime
  createdAt    DateTime  @default(now())
  revokedAt    DateTime?

  @@index([userId])
  @@index([token])
}

// ═══════════════════════════════════════
// PERSONAL
// ═══════════════════════════════════════

model Personnel {
  id              String           @id @default(cuid())
  userId          String           @unique
  user            User             @relation(fields: [userId], references: [id])
  firstName       String
  lastName        String
  dni             String           @unique
  phone           String?
  emergencyPhone  String?
  address         String?
  birthDate       DateTime?
  nationality     String           @default("PE")
  contractType    ContractType     @default(FULL_TIME)
  position        String
  department      String?
  salary          Decimal          @db.Decimal(10, 2)
  salaryType      SalaryType       @default(MONTHLY)
  startDate       DateTime
  endDate         DateTime?
  photo           String?          // URL
  documents       Json?            // { dni_scan, contract, ... }
  notes           String?
  createdAt       DateTime         @default(now())
  updatedAt       DateTime         @updatedAt
  deletedAt       DateTime?

  attendance      Attendance[]
  vehicleUsages   VehicleUsage[]
  workOrders      WorkOrder[]      @relation("AssignedTo")
  commissions     Commission[]

  @@index([dni])
}

enum ContractType {
  FULL_TIME
  PART_TIME
  APPRENTICE
  CONTRACTOR
  TEMP
}

enum SalaryType {
  HOURLY
  DAILY
  WEEKLY
  MONTHLY
}

model Attendance {
  id           String    @id @default(cuid())
  personnelId  String
  personnel    Personnel @relation(fields: [personnelId], references: [id])
  date         DateTime  @db.Date
  checkIn      DateTime?
  checkOut     DateTime?
  type         AttendanceType @default(PRESENT)
  notes        String?
  verifiedBy   String?
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt

  @@unique([personnelId, date])
  @@index([personnelId])
  @@index([date])
}

enum AttendanceType {
  PRESENT
  ABSENT
  LATE
  HALF_DAY
  PERMISSION
  VACATION
  SICK_LEAVE
  HOLIDAY
}

// ═══════════════════════════════════════
// CLIENTES
// ═══════════════════════════════════════

model Client {
  id           String        @id @default(cuid())
  userId       String?       @unique // Para clientes con portal
  type         ClientType    @default(INDIVIDUAL)
  firstName    String
  lastName     String?
  companyName  String?
  dni          String?
  ruc          String?
  email        String?
  phone        String        
  phone2       String?
  address      String?
  district     String?
  city         String        @default("Lima")
  notes        String?
  source       String?       // Referido, Google, walk-in
  creditLimit  Decimal?      @db.Decimal(10, 2)
  creditBalance Decimal      @default(0) @db.Decimal(10, 2)
  isVip        Boolean       @default(false)
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt
  deletedAt    DateTime?

  vehicles     Vehicle[]
  workOrders   WorkOrder[]
  invoices     Invoice[]
  quotes       Quote[]

  @@index([phone])
  @@index([dni])
  @@index([ruc])
}

enum ClientType {
  INDIVIDUAL
  COMPANY
}

// ═══════════════════════════════════════
// VEHÍCULOS
// ═══════════════════════════════════════

model Vehicle {
  id            String         @id @default(cuid())
  clientId      String
  client        Client         @relation(fields: [clientId], references: [id])
  plate         String         @unique
  brand         String
  model         String
  year          Int
  color         String?
  vin           String?        @unique
  engineType    EngineType     @default(GASOLINE)
  engineCC      Int?
  mileage       Int?
  fuelType      FuelType       @default(GASOLINE)
  transmission  TransmissionType @default(MANUAL)
  photos        String[]       // URLs
  notes         String?
  status        VehicleStatus  @default(ACTIVE)
  createdAt     DateTime       @default(now())
  updatedAt     DateTime       @updatedAt

  workOrders    WorkOrder[]
  usageLogs     VehicleUsage[] // Para vehículos DEL TALLER

  @@index([clientId])
  @@index([plate])
}

enum EngineType {
  GASOLINE
  DIESEL
  HYBRID
  ELECTRIC
  GAS
}

enum FuelType {
  GASOLINE
  DIESEL
  GAS
  ELECTRIC
  HYBRID
}

enum TransmissionType {
  MANUAL
  AUTOMATIC
  CVT
  SEMI_AUTO
}

enum VehicleStatus {
  ACTIVE
  IN_SERVICE
  WAITING_PARTS
  COMPLETED
  DELIVERED
  INACTIVE
}

// Control de uso de vehículos DEL TALLER (caso Ricardo)
model VehicleUsage {
  id            String        @id @default(cuid())
  vehicleId     String
  vehicle       Vehicle       @relation(fields: [vehicleId], references: [id])
  personnelId   String
  personnel     Personnel     @relation(fields: [personnelId], references: [id])
  authorizedBy  String?       // UserId quien autorizó
  purpose       String
  destination   String?
  odometerOut   Int?
  odometerIn    Int?
  checkoutAt    DateTime
  expectedReturn DateTime
  returnAt      DateTime?
  status        UsageStatus   @default(PENDING_RETURN)
  notes         String?
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  @@index([vehicleId])
  @@index([personnelId])
  @@index([checkoutAt])
}

enum UsageStatus {
  PENDING_RETURN
  RETURNED_ON_TIME
  RETURNED_LATE
  OVERDUE
  UNAUTHORIZED
}

// ═══════════════════════════════════════
// INVENTARIO
// ═══════════════════════════════════════

model Category {
  id          String    @id @default(cuid())
  name        String    @unique
  description String?
  parentId    String?
  parent      Category? @relation("CategoryTree", fields: [parentId], references: [id])
  children    Category[] @relation("CategoryTree")
  items       InventoryItem[]
  createdAt   DateTime  @default(now())
}

model Supplier {
  id           String    @id @default(cuid())
  name         String
  contactName  String?
  phone        String?
  email        String?
  address      String?
  ruc          String?   @unique
  website      String?
  paymentTerms String?
  notes        String?
  isImporter   Boolean   @default(false) // Para control de importaciones (caso Ricardo)
  commissionRate Decimal? @db.Decimal(5, 2) // Control de comisiones
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
  deletedAt    DateTime?

  items        InventoryItem[]
  purchases    Purchase[]
  commissions  Commission[]
}

model InventoryItem {
  id             String       @id @default(cuid())
  code           String       @unique
  name           String
  description    String?
  categoryId     String
  category       Category     @relation(fields: [categoryId], references: [id])
  supplierId     String?
  supplier       Supplier?    @relation(fields: [supplierId], references: [id])
  unit           String       @default("unit") // unit, liter, kg, meter
  costPrice      Decimal      @db.Decimal(10, 2)
  salePrice      Decimal      @db.Decimal(10, 2)
  currentStock   Int          @default(0)
  minStock       Int          @default(5)
  maxStock       Int          @default(100)
  location       String?      // Posición física en almacén
  barcode        String?
  isImported     Boolean      @default(false)
  customsCost    Decimal?     @db.Decimal(10, 2)
  photos         String[]
  notes          String?
  isActive       Boolean      @default(true)
  createdAt      DateTime     @default(now())
  updatedAt      DateTime     @updatedAt

  stockMovements StockMovement[]
  workOrderItems WorkOrderItem[]
  purchaseItems  PurchaseItem[]

  @@index([code])
  @@index([categoryId])
  @@index([currentStock])
}

model StockMovement {
  id          String        @id @default(cuid())
  itemId      String
  item        InventoryItem @relation(fields: [itemId], references: [id])
  type        MovementType
  quantity    Int
  costUnit    Decimal?      @db.Decimal(10, 2)
  totalCost   Decimal?      @db.Decimal(10, 2)
  reference   String?       // WorkOrder ID, Purchase ID
  reason      String?
  notes       String?
  userId      String        // Quién registró
  createdAt   DateTime      @default(now())

  @@index([itemId])
  @@index([type])
  @@index([createdAt])
}

enum MovementType {
  IN_PURCHASE
  IN_RETURN
  IN_ADJUSTMENT
  OUT_WORK_ORDER
  OUT_WASTE
  OUT_TRANSFER
  OUT_ADJUSTMENT
  RESERVATION
}

// ═══════════════════════════════════════
// ÓRDENES DE TRABAJO
// ═══════════════════════════════════════

model WorkOrder {
  id              String          @id @default(cuid())
  orderNumber     String          @unique // WO-2026-0001
  clientId        String
  client          Client          @relation(fields: [clientId], references: [id])
  vehicleId       String
  vehicle         Vehicle         @relation(fields: [vehicleId], references: [id])
  status          WorkOrderStatus @default(RECEIVED)
  priority        Priority        @default(NORMAL)
  type            ServiceType     @default(CORRECTIVE)
  description     String
  diagnosis       String?
  recommendation  String?
  odometerIn      Int?
  odometerOut     Int?
  fuelLevel       String?         // E, 1/4, 1/2, 3/4, F
  receivedAt      DateTime        @default(now())
  estimatedAt     DateTime?
  startedAt       DateTime?
  completedAt     DateTime?
  deliveredAt     DateTime?
  laborCost       Decimal         @default(0) @db.Decimal(10, 2)
  partsCost       Decimal         @default(0) @db.Decimal(10, 2)
  totalCost       Decimal         @default(0) @db.Decimal(10, 2)
  discount        Decimal         @default(0) @db.Decimal(10, 2)
  tax             Decimal         @default(0) @db.Decimal(10, 2)
  finalAmount     Decimal         @default(0) @db.Decimal(10, 2)
  paymentStatus   PaymentStatus   @default(PENDING)
  paymentMethod   PaymentMethod?
  customerNotes   String?
  internalNotes   String?
  photos          String[]        // Antes/después
  signature       String?         // URL firma digital cliente
  warrantyDays    Int             @default(0)
  createdBy       String
  updatedBy       String?
  
  // Control de pagos no autorizados (caso Ricardo)
  paymentReceivedBy String?
  paymentYape       String?       // Si usó Yape personal (para auditoría)
  
  assignees       WorkOrderAssignee[]
  items           WorkOrderItem[]
  payments        Payment[]
  invoice         Invoice?
  quote           Quote?
  timeline        WorkOrderEvent[]
  createdAt       DateTime        @default(now())
  updatedAt       DateTime        @updatedAt

  @@index([clientId])
  @@index([vehicleId])
  @@index([status])
  @@index([receivedAt])
  @@index([orderNumber])
}

enum WorkOrderStatus {
  RECEIVED
  DIAGNOSING
  QUOTED
  APPROVED
  IN_PROGRESS
  WAITING_PARTS
  PAUSED
  COMPLETED
  QUALITY_CHECK
  READY_FOR_DELIVERY
  DELIVERED
  CANCELLED
  WARRANTY_CLAIM
}

enum ServiceType {
  CORRECTIVE
  PREVENTIVE
  DIAGNOSTIC
  EMERGENCY
  WARRANTY
  INSPECTION
}

enum Priority {
  LOW
  NORMAL
  HIGH
  URGENT
}

model WorkOrderAssignee {
  id          String    @id @default(cuid())
  workOrderId String
  workOrder   WorkOrder @relation(fields: [workOrderId], references: [id])
  personnelId String
  personnel   Personnel @relation("AssignedTo", fields: [personnelId], references: [id])
  role        String    @default("MECHANIC") // LEAD, MECHANIC, HELPER
  assignedAt  DateTime  @default(now())
  completedAt DateTime?

  @@unique([workOrderId, personnelId])
}

model WorkOrderItem {
  id          String        @id @default(cuid())
  workOrderId String
  workOrder   WorkOrder     @relation(fields: [workOrderId], references: [id])
  type        ItemType      @default(PART)
  itemId      String?
  item        InventoryItem? @relation(fields: [itemId], references: [id])
  description String
  quantity    Decimal       @db.Decimal(10, 3)
  unitPrice   Decimal       @db.Decimal(10, 2)
  totalPrice  Decimal       @db.Decimal(10, 2)
  discount    Decimal       @default(0) @db.Decimal(10, 2)
  notes       String?
  createdAt   DateTime      @default(now())

  @@index([workOrderId])
}

enum ItemType {
  PART
  LABOR
  EXTERNAL_SERVICE
  CONSUMABLE
}

model WorkOrderEvent {
  id          String    @id @default(cuid())
  workOrderId String
  workOrder   WorkOrder @relation(fields: [workOrderId], references: [id])
  event       String
  description String?
  metadata    Json?
  userId      String
  createdAt   DateTime  @default(now())

  @@index([workOrderId])
  @@index([createdAt])
}

// ═══════════════════════════════════════
// FINANZAS
// ═══════════════════════════════════════

model Invoice {
  id           String        @id @default(cuid())
  number       String        @unique // INV-2026-0001
  workOrderId  String?       @unique
  workOrder    WorkOrder?    @relation(fields: [workOrderId], references: [id])
  clientId     String
  client       Client        @relation(fields: [clientId], references: [id])
  type         InvoiceType   @default(BOLETA)
  status       InvoiceStatus @default(DRAFT)
  subtotal     Decimal       @db.Decimal(10, 2)
  tax          Decimal       @db.Decimal(10, 2)
  discount     Decimal       @default(0) @db.Decimal(10, 2)
  total        Decimal       @db.Decimal(10, 2)
  paidAmount   Decimal       @default(0) @db.Decimal(10, 2)
  dueAmount    Decimal       @db.Decimal(10, 2)
  dueDate      DateTime?
  notes        String?
  issuedAt     DateTime?
  paidAt       DateTime?
  cancelledAt  DateTime?
  cancelReason String?
  createdBy    String
  approvedBy   String?
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt

  payments     Payment[]

  @@index([clientId])
  @@index([status])
  @@index([number])
}

enum InvoiceType {
  BOLETA
  FACTURA
  NOTA_CREDITO
  NOTA_DEBITO
  INTERNAL
}

enum InvoiceStatus {
  DRAFT
  ISSUED
  PARTIALLY_PAID
  PAID
  OVERDUE
  CANCELLED
}

model Payment {
  id            String        @id @default(cuid())
  invoiceId     String?
  invoice       Invoice?      @relation(fields: [invoiceId], references: [id])
  workOrderId   String?
  workOrder     WorkOrder?    @relation(fields: [workOrderId], references: [id])
  method        PaymentMethod
  amount        Decimal       @db.Decimal(10, 2)
  reference     String?       // Nro. operación, Yape, etc.
  receivedBy    String        // USER ID que recibió el pago
  verifiedBy    String?       // Quien verificó
  channel       PaymentChannel @default(IN_PERSON)
  
  // Control de Yape personal (caso Ricardo)
  isPersonalYape Boolean      @default(false)
  yapeAccount   String?       // Para auditoría
  
  notes         String?
  receiptUrl    String?
  paidAt        DateTime      @default(now())
  createdAt     DateTime      @default(now())

  @@index([invoiceId])
  @@index([workOrderId])
  @@index([method])
  @@index([paidAt])
}

enum PaymentMethod {
  CASH
  YAPE
  PLIN
  BANK_TRANSFER
  CARD_VISA
  CARD_MASTERCARD
  CREDIT
  CHECK
}

enum PaymentChannel {
  IN_PERSON
  ONLINE
  MOBILE
}

model Expense {
  id           String        @id @default(cuid())
  category     ExpenseCategory
  description  String
  amount       Decimal       @db.Decimal(10, 2)
  supplier     String?
  supplierRuc  String?
  invoiceNumber String?
  paymentMethod PaymentMethod @default(CASH)
  paidBy       String        // USER ID
  approvedBy   String?       // USER ID
  requiresApproval Boolean   @default(false)
  approvalStatus ApprovalStatus @default(PENDING)
  receipt      String?       // URL
  notes        String?
  expenseDate  DateTime      @default(now())
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt

  approval     Approval?

  @@index([category])
  @@index([expenseDate])
  @@index([approvalStatus])
}

enum ExpenseCategory {
  PARTS_PURCHASE
  TOOLS
  RENT
  UTILITIES
  SALARIES
  COMMISSIONS
  MARKETING
  MAINTENANCE
  TRANSPORT
  FOOD
  OTHER
}

enum ApprovalStatus {
  PENDING
  APPROVED
  REJECTED
  CANCELLED
}

// Control de autorizaciones (caso finanzas)
model Approval {
  id            String        @id @default(cuid())
  type          ApprovalType
  status        ApprovalStatus @default(PENDING)
  title         String
  description   String
  amount        Decimal?      @db.Decimal(10, 2)
  requestedById String
  requestedBy   User          @relation("RequestedBy", fields: [requestedById], references: [id])
  approvedById  String?
  approvedBy    User?         @relation("ApprovedBy", fields: [approvedById], references: [id])
  expenseId     String?       @unique
  expense       Expense?      @relation(fields: [expenseId], references: [id])
  reason        String?
  rejectionReason String?
  expiresAt     DateTime?
  approvedAt    DateTime?
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  @@index([status])
  @@index([requestedById])
  @@index([type])
}

enum ApprovalType {
  EXPENSE
  VEHICLE_USAGE
  SALARY_ADVANCE
  CREDIT_NOTE
  DISCOUNT
  IMPORT_COMMISSION
  INVENTORY_WRITE_OFF
}

model Purchase {
  id           String         @id @default(cuid())
  number       String         @unique // PO-2026-0001
  supplierId   String
  supplier     Supplier       @relation(fields: [supplierId], references: [id])
  status       PurchaseStatus @default(DRAFT)
  subtotal     Decimal        @db.Decimal(10, 2)
  tax          Decimal        @db.Decimal(10, 2)
  shipping     Decimal        @default(0) @db.Decimal(10, 2)
  customs      Decimal        @default(0) @db.Decimal(10, 2)
  total        Decimal        @db.Decimal(10, 2)
  currency     String         @default("PEN")
  
  // Control de comisiones en importaciones (caso Ricardo)
  commissionPaid  Boolean     @default(false)
  commissionAmount Decimal?   @db.Decimal(10, 2)
  commissionTo    String?     // A quién se pagó comisión
  isImported      Boolean     @default(false)
  
  notes        String?
  orderedAt    DateTime?
  expectedAt   DateTime?
  receivedAt   DateTime?
  createdBy    String
  approvedBy   String?
  createdAt    DateTime       @default(now())
  updatedAt    DateTime       @updatedAt

  items        PurchaseItem[]
  commissions  Commission[]

  @@index([supplierId])
  @@index([status])
}

enum PurchaseStatus {
  DRAFT
  SENT
  CONFIRMED
  PARTIALLY_RECEIVED
  RECEIVED
  CANCELLED
}

model PurchaseItem {
  id          String        @id @default(cuid())
  purchaseId  String
  purchase    Purchase      @relation(fields: [purchaseId], references: [id])
  itemId      String
  item        InventoryItem @relation(fields: [itemId], references: [id])
  quantity    Int
  unitCost    Decimal       @db.Decimal(10, 2)
  totalCost   Decimal       @db.Decimal(10, 2)
  receivedQty Int           @default(0)
  notes       String?
}

model Commission {
  id          String      @id @default(cuid())
  personnelId String
  personnel   Personnel   @relation(fields: [personnelId], references: [id])
  supplierId  String?
  supplier    Supplier?   @relation(fields: [supplierId], references: [id])
  purchaseId  String?
  purchase    Purchase?   @relation(fields: [purchaseId], references: [id])
  type        String      // IMPORT, REFERRAL, SALES
  amount      Decimal     @db.Decimal(10, 2)
  percentage  Decimal?    @db.Decimal(5, 2)
  status      ApprovalStatus @default(PENDING)
  notes       String?
  paidAt      DateTime?
  createdAt   DateTime    @default(now())
}

model Quote {
  id          String      @id @default(cuid())
  number      String      @unique
  clientId    String
  client      Client      @relation(fields: [clientId], references: [id])
  workOrderId String?     @unique
  workOrder   WorkOrder?  @relation(fields: [workOrderId], references: [id])
  status      QuoteStatus @default(DRAFT)
  validUntil  DateTime
  subtotal    Decimal     @db.Decimal(10, 2)
  tax         Decimal     @db.Decimal(10, 2)
  total       Decimal     @db.Decimal(10, 2)
  notes       String?
  createdBy   String
  approvedAt  DateTime?
  rejectedAt  DateTime?
  rejectionReason String?
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt

  @@index([clientId])
  @@index([status])
}

enum QuoteStatus {
  DRAFT
  SENT
  APPROVED
  REJECTED
  EXPIRED
  CONVERTED
}

// ═══════════════════════════════════════
// AUDITORÍA Y SEGURIDAD
// ═══════════════════════════════════════

model AuditLog {
  id          String    @id @default(cuid())
  userId      String?
  user        User?     @relation(fields: [userId], references: [id])
  action      String    // CREATE, READ, UPDATE, DELETE, LOGIN, LOGOUT, etc.
  entity      String    // WorkOrder, Invoice, Personnel, etc.
  entityId    String?
  changes     Json?     // { before: {...}, after: {...} }
  metadata    Json?     // IP, userAgent, etc.
  ipAddress   String?
  userAgent   String?
  severity    AuditSeverity @default(INFO)
  createdAt   DateTime  @default(now())

  @@index([userId])
  @@index([entity])
  @@index([action])
  @@index([createdAt])
  @@index([severity])
}

enum AuditSeverity {
  INFO
  WARNING
  CRITICAL
  SECURITY_ALERT
}

model Notification {
  id          String    @id @default(cuid())
  userId      String
  type        String
  title       String
  body        String
  data        Json?
  isRead      Boolean   @default(false)
  readAt      DateTime?
  priority    String    @default("NORMAL")
  channel     String    @default("IN_APP") // IN_APP, PUSH, SMS, EMAIL
  createdAt   DateTime  @default(now())

  @@index([userId])
  @@index([isRead])
  @@index([createdAt])
}

model Setting {
  id        String   @id @default(cuid())
  key       String   @unique
  value     String
  category  String   @default("GENERAL")
  isPublic  Boolean  @default(false)
  updatedBy String?
  updatedAt DateTime @updatedAt

  @@index([category])
}
```

> **ACCIÓN:** Compara el schema existente con el de arriba. Extiéndelo sin perder lo ya migrado. Genera una **nueva migración** con `npx prisma migrate dev --name "extend_full_schema"`.

---

### 3. SEED COMPLETO CON DATOS REALISTAS

En `prisma/seed.ts`, genera datos hardcodeados realistas del negocio peruano:

```typescript
// USUARIOS (con passwords hasheadas via bcrypt)
const users = [
  { email: 'edgar@arellanautos.pe',   role: 'OWNER',   name: 'Edgar Arellan' },
  { email: 'juan@arellanautos.pe',    role: 'OWNER',   name: 'Juan Arellan' },
  { email: 'ana@arellanautos.pe',     role: 'MANAGER', name: 'Ana Arellan' },
  { email: 'finanzas@arellanautos.pe',role: 'FINANCE', name: 'Sofía Arellan' },
  { email: 'mecanico1@arellanautos.pe',role: 'MECHANIC',name: 'Carlos Quispe' },
  { email: 'mecanico2@arellanautos.pe',role: 'MECHANIC',name: 'Luis Mamani' },
  { email: 'practicante@arellanautos.pe',role:'APPRENTICE',name:'Miguel Torres'},
];
// Password por defecto en desarrollo: "Arellan2026!"

// CLIENTES (10+ clientes ficticios limeños)
// VEHÍCULOS (2-3 por cliente)
// INVENTARIO (30+ ítems: aceites, filtros, frenos, herramientas)
// ÓRDENES DE TRABAJO (20+ órdenes en diferentes estados)
// PROVEEDORES (5+ proveedores nacionales e importadores)
// GASTOS (20+ del último mes)
// ASISTENCIAS (último mes completo)
// AUDITORÍA (últimas 50 acciones de ejemplo)
// NOTIFICACIONES (10+ no leídas por usuario)
// CONFIGURACIONES del sistema
```

Todos los datos deben usar nombres, placas, DNIs y RUCs con formato peruano real.
Password de desarrollo para todos: `Arellan2026!` (hasheada con bcrypt, 12 rounds).

> **ACCIÓN:** Completa `prisma/seed.ts` y ejecuta `npx prisma db seed`. Verifica que funcione.

---

### 4. COMPLETAR Y REFACTORIZAR TODOS LOS MÓDULOS BACKEND

Para **CADA módulo** (`auth`, `clients`, `vehicles`, `orders`, `inventory`, `finance`, `personnel`, `audit`):

#### 4.1 AUTH MODULE — Prioridad CRÍTICA
```typescript
// DEBE incluir:
- POST /auth/login          → JWT + Refresh Token
- POST /auth/refresh        → Renovar access token
- POST /auth/logout         → Invalidar sesión en Redis
- POST /auth/logout-all     → Cerrar todas las sesiones
- GET  /auth/me             → Usuario actual con perfil
- POST /auth/change-password
- POST /auth/forgot-password (placeholder)
- GET  /auth/sessions       → Lista de sesiones activas
- DELETE /auth/sessions/:id → Revocar sesión específica

// Guards deben funcionar en TODOS los controllers
// Audit interceptor debe loguear TODOS los accesos
// Rate limiting en login: max 5 intentos, bloqueo 15 min
// Refresh tokens en Redis con TTL 7 días
// Access tokens JWT: 15 minutos de vida
```

#### 4.2 CADA MÓDULO DEBE TENER:
```typescript
// Controller con:
@Controller('module-name')
@UseGuards(JwtAuthGuard, RolesGuard)
@UseInterceptors(AuditInterceptor)

// Endpoints CRUD completos:
GET    /            → Lista paginada (page, limit, filters, sort)
GET    /:id         → Detalle completo
POST   /            → Crear
PATCH  /:id         → Actualizar parcialmente
DELETE /:id         → Soft delete

// Endpoints específicos por módulo (ver abajo)

// Service con:
- Lógica de negocio separada del controller
- Manejo de errores con excepciones NestJS
- Transacciones Prisma donde aplique
- Cacheo Redis donde aplique (listas frecuentes)

// DTOs con:
- class-validator decorators en TODOS los campos
- Swagger @ApiProperty en TODOS los campos
- CreateDTO, UpdateDTO (Partial<CreateDTO>), ResponseDTO
```

#### 4.3 ENDPOINTS ESPECÍFICOS CRÍTICOS:

```typescript
// ORDERS MODULE
POST   /orders                          → Crear orden
GET    /orders/:id/timeline             → Historial de eventos
PATCH  /orders/:id/status              → Cambiar estado
POST   /orders/:id/assign               → Asignar mecánico
POST   /orders/:id/items               → Agregar repuesto/labor
POST   /orders/:id/payment             → Registrar pago
GET    /orders/by-status/:status       → Órdenes por estado
GET    /orders/stats/summary           → Dashboard summary
WebSocket: emit 'order:updated' on every change

// INVENTORY MODULE
GET    /inventory/low-stock            → Alertas stock mínimo
POST   /inventory/:id/adjust           → Ajuste manual stock
GET    /inventory/movements            → Movimientos paginados
GET    /inventory/valuation            → Valor total inventario
POST   /inventory/items/:id/reserve   → Reservar para orden

// FINANCE MODULE
GET    /finance/dashboard              → KPIs financieros
GET    /finance/cashflow               → Flujo de caja
POST   /finance/expenses               → Registrar gasto
POST   /finance/expenses/:id/approve   → Aprobar gasto
GET    /finance/commissions            → Registro comisiones
POST   /finance/reports/daily          → Reporte diario
POST   /finance/reports/monthly        → Reporte mensual

// PERSONNEL MODULE
POST   /personnel/:id/attendance/check-in  → Check-in
POST   /personnel/:id/attendance/check-out → Check-out
GET    /personnel/attendance/today         → Asistencia hoy
GET    /personnel/:id/performance          → Productividad
POST   /personnel/vehicle-usage/authorize → Autorizar uso vehículo
GET    /personnel/vehicle-usage/active    → Usos activos

// VEHICLES MODULE
GET    /vehicles/workshop-fleet           → Vehículos del taller
GET    /vehicles/workshop-fleet/in-use    → En uso ahora
GET    /vehicles/:id/history              → Historial completo

// AUDIT MODULE
GET    /audit/logs                        → Logs paginados con filtros
GET    /audit/alerts                      → Alertas de seguridad
GET    /audit/user/:userId/activity       → Actividad por usuario
```

---

### 5. CONECTAR FRONTEND WEB CON BACKEND

**EN `arellan-frontend-web`:**

#### 5.1 Configuración del cliente HTTP:
```typescript
// lib/api-client.ts — Cliente centralizado
// - Base URL desde env: NEXT_PUBLIC_API_URL
// - Interceptor para adjuntar JWT en cada request
// - Interceptor para refresh automático del token (401 → refresh → retry)
// - Interceptor para manejar errores globalmente
// - TypeScript types desde DTOs del backend

// Usar: fetch nativo con wrapper O axios O ky
```

#### 5.2 Variables de entorno requeridas:
```env
# arellan-frontend-web/.env.local
NEXT_PUBLIC_API_URL=http://localhost:3001/api/v1
NEXT_PUBLIC_WS_URL=ws://localhost:3001
NEXT_PUBLIC_APP_NAME="Arellan Autos — Panel"
NEXT_PUBLIC_UPLOAD_URL=http://localhost:3001/uploads
```

```env
# arellan-platform/.env
DATABASE_URL=postgresql://postgres:password@localhost:5432/arellan_db
REDIS_URL=redis://localhost:6379
JWT_SECRET=arellan_super_secret_jwt_2026_development
JWT_REFRESH_SECRET=arellan_refresh_secret_2026_development
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
PORT=3001
NODE_ENV=development
CORS_ORIGINS=http://localhost:3000,http://localhost:3002,http://localhost:3003
```

#### 5.3 Estructura de hooks y servicios en el frontend:
```
arellan-frontend-web/src/
├── lib/
│   ├── api-client.ts           ← Cliente HTTP base
│   ├── auth.ts                 ← Funciones de autenticación
│   └── websocket.ts            ← Cliente WebSocket
├── services/
│   ├── auth.service.ts         ← Login, logout, refresh
│   ├── orders.service.ts       ← CRUD órdenes
│   ├── clients.service.ts      ← CRUD clientes
│   ├── inventory.service.ts    ← CRUD inventario
│   ├── finance.service.ts      ← CRUD finanzas
│   ├── personnel.service.ts    ← CRUD personal
│   └── vehicles.service.ts     ← CRUD vehículos
├── hooks/
│   ├── useAuth.ts
│   ├── useOrders.ts
│   ├── useClients.ts
│   ├── useInventory.ts
│   ├── useFinance.ts
│   ├── usePersonnel.ts
│   └── useRealtime.ts          ← WebSocket hook
└── store/
    └── auth.store.ts           ← Zustand store para auth
```

#### 5.4 Páginas obligatorias que deben funcionar con datos REALES:
```
/login                          → Autenticación real con backend
/dashboard                      → KPIs reales: órdenes hoy, ingresos, inventario bajo stock
/orders                         → Lista real paginada + filtros
/orders/new                     → Formulario conectado a backend
/orders/:id                     → Detalle con timeline real
/clients                        → Lista real de clientes
/clients/new                    → Crear cliente real
/inventory                      → Lista inventario real + alertas stock
/finance/dashboard              → Finanzas reales
/finance/expenses               → Gastos y aprobaciones reales
/personnel                      → Lista personal con asistencias
/vehicles                       → Vehículos y usos reales
/settings                       → Configuraciones del sistema
```

---

### 6. CONECTAR MECHANIC UI CON BACKEND

**EN `arellan-mechanic-ui`** (UI tablet para mecánicos):

```typescript
// Variables de entorno
NEXT_PUBLIC_API_URL=http://localhost:3001/api/v1
NEXT_PUBLIC_WS_URL=ws://localhost:3001
NEXT_PUBLIC_MECHANIC_MODE=true

// Páginas (diseño tablet-first, touch-friendly, sin mouse):
/login              → Login simplificado (email + PIN de 4 dígitos)
/dashboard          → Mis órdenes del día, estado actual
/orders             → Lista de órdenes asignadas a MI usuario
/orders/:id         → Detalle: ver repuestos, actualizar estado, subir fotos, registrar tiempo
/orders/:id/items   → Agregar repuestos de inventario
/check-in           → Registrar mi entrada al turno
/check-out          → Registrar mi salida

// Flujo crítico:
// 1. Mecánico hace check-in → registro en BD
// 2. Ve sus órdenes asignadas
// 3. Actualiza estado de orden (en progreso, esperando pieza, completado)
// 4. Registra repuestos usados → descuenta del inventario
// 5. Sube fotos de evidencia
// 6. Hace check-out
// Todo via WebSocket para actualización en tiempo real en el panel admin
```

---

### 7. CONECTAR CLIENT PORTAL CON BACKEND

**EN `arellan-client-portal`** (portal de seguimiento para clientes):

```typescript
// Rutas públicas (sin autenticación):
/track/:orderNumber  → Rastrear orden por número (como tracking de envío)
/                    → Landing con buscador de orden

// Rutas privadas (clientes con cuenta):
/login               → Login de cliente
/dashboard           → Mis vehículos y órdenes
/orders              → Historial de órdenes
/orders/:id          → Detalle con timeline, fotos, cotización
/quotes/:id          → Ver y aprobar cotización
/vehicles            → Mis vehículos
/profile             → Mi perfil y datos de contacto

// El portal DEBE mostrar estado real de la orden en tiempo real via WebSocket
// Los clientes SOLO ven sus propios datos (filtrado por clientId en backend)
```

---

### 8. DOCKER COMPOSE — ENTORNO LOCAL COMPLETO

Genera o actualiza `docker-compose.yml` en la raíz del proyecto:

```yaml
# docker-compose.yml — Servicios locales para desarrollo
version: '3.9'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: arellan_db
    ports: ['5432:5432']
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ['6379:6379']
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

  # pgAdmin para administración visual de BD
  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@arellan.pe
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports: ['5050:80']
    depends_on: [postgres]

volumes:
  postgres_data:
  redis_data:
```

> **ACCIÓN:** Asegurate de que `docker-compose up -d` levante postgres y redis, y que la app backend pueda conectarse. Documenta en README del backend.

---

### 9. WEBSOCKETS — EVENTOS EN TIEMPO REAL

En el backend, implementa Socket.io con estos eventos:

```typescript
// Gateway: src/common/gateway/realtime.gateway.ts

// EVENTOS QUE EL SERVIDOR EMITE:
'order:created'        → { orderId, orderNumber, clientName, vehiclePlate }
'order:status_changed' → { orderId, oldStatus, newStatus, updatedBy }
'order:assigned'       → { orderId, personnelId, personnelName }
'inventory:low_stock'  → { itemId, itemName, currentStock, minStock }
'payment:received'     → { workOrderId, amount, method, receivedBy }
'personnel:check_in'   → { personnelId, name, timestamp }
'personnel:check_out'  → { personnelId, name, timestamp }
'vehicle:overdue'      → { vehicleId, plate, personnelName, expectedReturn }
'approval:requested'   → { approvalId, type, amount, requestedBy }
'approval:resolved'    → { approvalId, status, resolvedBy }
'alert:security'       → { type, description, severity, userId }

// Salas (rooms) por rol:
// OWNER/MANAGER → reciben todo
// MECHANIC → solo sus órdenes y alertas de inventario
// CLIENT → solo sus órdenes
```

---

### 10. SWAGGER API DOCUMENTATION

En el backend `main.ts`, configura Swagger completo:

```typescript
// Accesible en: http://localhost:3001/api/docs
// Incluye:
- Todos los endpoints documentados
- DTOs con ejemplos
- Autenticación Bearer Token en la UI
- Tags por módulo
- Ejemplos de request/response
```

---

### 11. DESIGN SYSTEM — COMPONENTES COMPARTIDOS

En `arellan-design-system`, crea un paquete npm local compartido:

```typescript
// Exportar como @arellan/ui
// Componentes mínimos:
- Button (variants: primary, secondary, danger, ghost)
- Input, Select, Textarea, Checkbox, DatePicker
- Card, Modal, Drawer, Tooltip, Popover
- Badge (estados: success, warning, error, info)
- Table (con paginación, sorting, filtros)
- StatusBadge (para estados de órdenes, pagos, etc.)
- Avatar
- LoadingSpinner, Skeleton
- Alert, Toast (notificaciones)
- PageHeader (con breadcrumbs)

// Paleta de colores del Design System:
--arellan-red: #C0392B       ← Color primario del taller
--arellan-dark: #1A1A2E      ← Fondo oscuro
--arellan-gray: #2C2C54      ← Sidebar
--arellan-accent: #E94560    ← Acento
--arellan-success: #27AE60
--arellan-warning: #F39C12
--arellan-error: #E74C3C
--arellan-info: #2980B9
```

---

### 12. VALIDACIONES DE NEGOCIO CRÍTICAS

Implementa estas reglas de negocio en el backend:

```typescript
// ANTI-FRAUDE (basado en el caso Ricardo):

// 1. Pagos solo con métodos autorizados:
//    - Si paymentMethod = YAPE, verificar que el Yape sea del taller (configuración)
//    - Si receivedBy ≠ OWNER/MANAGER, crear alerta de auditoría

// 2. Uso de vehículos del taller:
//    - Requiere autorización de OWNER o MANAGER ANTES del uso
//    - Si returnAt > expectedReturn + 30min → alerta automática
//    - Notificación push al dueño si vehículo está fuera más de 2 horas

// 3. Comisiones de importación:
//    - Solo OWNER puede registrar comisiones
//    - Todas las comisiones van a AuditLog con severity CRITICAL
//    - Requiere aprobación de 2 dueños para comisiones > S/ 200

// 4. Gastos:
//    - Gastos < S/ 100: auto-aprobado si es MANAGER
//    - Gastos S/ 100-500: requiere 1 OWNER
//    - Gastos > S/ 500: requiere ambos OWNERS

// 5. Descuentos en órdenes:
//    - Descuentos > 20% requieren aprobación de OWNER
//    - Todo descuento se registra en AuditLog

// 6. Contraseñas y sesiones:
//    - Máximo 3 sesiones simultáneas por usuario
//    - Sesión caduca después de 4 horas de inactividad
//    - Login fallido x5 → bloqueo 15 minutos
```

---

### 13. ESCALABILIDAD — PATRONES A IMPLEMENTAR

Asegúrate de que el código siga estos patrones para soportar millones de usuarios:

```typescript
// 1. PAGINACIÓN en TODOS los endpoints de lista:
{
  data: [...],
  meta: { total, page, limit, totalPages, hasNext, hasPrev }
}

// 2. ÍNDICES en Prisma: ya están en el schema anterior
//    Agregar en schema.prisma según los filtros más usados

// 3. CACHEO con Redis:
//    - Dashboard stats: TTL 60 segundos
//    - Lista de inventario: TTL 30 segundos, invalidar en mutations
//    - Perfil de usuario: TTL 5 minutos

// 4. SOFT DELETES: deletedAt en todos los modelos críticos
//    Middleware Prisma global para excluir deleted records

// 5. RATE LIMITING global en NestJS:
//    - API general: 100 req/min por IP
//    - Login: 5 intentos/15min por email

// 6. OPTIMISTIC UI en el frontend:
//    - Actualizar estado local antes de confirmar con backend
//    - Rollback si el backend falla

// 7. CÓDIGO DE ESTADO DE ÓRDENES como máquina de estados:
//    - Validar transiciones permitidas en el servicio
//    - Ejemplo: RECEIVED → DIAGNOSING ✓, RECEIVED → DELIVERED ✗

// 8. SEPARAR read replicas (preparar para futuro):
//    - Queries de lectura usar readClient
//    - Queries de escritura usar writeClient
```

---

### 14. VARIABLES DE ENTORNO Y CONFIGURACIÓN

Crea `arellan-platform/.env.example` con TODAS las variables necesarias y sus valores de desarrollo. Documentar qué hace cada una.

Crea `arellan-frontend-web/.env.local.example` igual.

---

### 15. TESTING — DATOS SUFICIENTES PARA PROBAR TODO

Después de ejecutar el seed, el sistema debe poder demostrar:

1. **Login** con edgar@arellanautos.pe / Arellan2026! → redirige al dashboard
2. **Dashboard** muestra datos reales: 3+ órdenes activas, KPIs con números
3. **Órdenes** lista con filtros por estado, búsqueda por placa o cliente
4. **Crear orden** con cliente existente, vehículo existente, añadir repuestos del inventario
5. **Inventario** muestra items con stock, 2+ items en stock mínimo (alerta roja)
6. **Finanzas** muestra ingresos del mes, gastos pendientes de aprobación
7. **Personal** asistencia del día, uso de vehículos activo
8. **Login en Mechanic UI** con mecanico1@arellanautos.pe → ve solo sus órdenes
9. **Portal cliente** buscar una orden por número → ver estado en tiempo real

---

### 16. ORDEN DE EJECUCIÓN RECOMENDADO

Sigue este orden para no romper dependencias:

```
1. Revisar y actualizar schema.prisma
2. Generar migración: npx prisma migrate dev --name "extend_full_schema"
3. Completar seed.ts → npx prisma db seed
4. Completar y refactorizar todos los módulos backend
5. Verificar que el backend levanta: npm run start:dev (puerto 3001)
6. Verificar Swagger en http://localhost:3001/api/docs
7. Completar servicios y hooks del arellan-frontend-web
8. Verificar login y dashboard del frontend (puerto 3000)
9. Verificar arellan-mechanic-ui (puerto 3002)
10. Verificar arellan-client-portal (puerto 3003)
11. Probar WebSockets: abrir 2 ventanas, acción en una → actualización en otra
```

---

### 17. RESTRICCIONES Y CALIDAD DE CÓDIGO

- **TypeScript estricto**: `strict: true` en tsconfig. No usar `any`.
- **Sin console.log** en producción: usar NestJS Logger o pino
- **Error handling**: Todos los errores tienen formato consistente `{ statusCode, message, error, timestamp }`
- **Nombres en inglés** para código (variables, funciones, clases). Strings de UI en español.
- **Commits pequeños**: Un commit por módulo o feature completada
- **No borres código existente** sin leerlo primero: puede tener lógica de negocio importante
- **Los archivos `.env`** nunca se commitean. Solo `.env.example`
- **Cada controller** debe tener Swagger decorators (`@ApiTags`, `@ApiOperation`, `@ApiResponse`)

---

### RESULTADO ESPERADO AL TERMINAR

Al terminar esta sesión de trabajo, el sistema debe:

✅ Backend NestJS corriendo en puerto 3001 con todos los módulos funcionales  
✅ PostgreSQL con schema completo + datos seed realistas  
✅ Redis corriendo y siendo usado para sesiones y caché  
✅ Swagger accesible en /api/docs con todos los endpoints  
✅ Frontend web en puerto 3000 autenticándose contra el backend real  
✅ Dashboard mostrando datos reales del seed  
✅ Al menos 3 flujos completos end-to-end funcionando (login → ver órdenes → actualizar estado)  
✅ Mechanic UI en puerto 3002 conectado al backend  
✅ Client portal en puerto 3003 mostrando estado de orden real  
✅ WebSockets: cambio en backend → actualización en tiempo real en frontend  
✅ Auditoría: cada acción queda registrada en AuditLog  
✅ Control de pagos: alerta si se registra Yape no autorizado  
✅ Alertas de inventario: notificación si stock cae bajo mínimo  

---

**Comienza leyendo la estructura actual del proyecto y todos los archivos existentes antes de modificar nada. Reporta qué encontraste en cada producto y cuál es tu plan de acción antes de empezar a escribir código.**
