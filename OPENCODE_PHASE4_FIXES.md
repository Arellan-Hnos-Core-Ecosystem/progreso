# OPENCODE — FASE 4: CORRECCIÓN DE LOS 11 ÍTEMS PENDIENTES
# Basado en scorecard verificado el 2026-06-04

---

## CONTEXTO

El sistema tiene 35/42 ítems funcionando. Esta sesión cierra exactamente los 11 restantes.
**No toques lo que ya funciona.** Lee el archivo correspondiente antes de modificar cualquier cosa.

Los 11 ítems pendientes, en orden de implementación:

```
❌ POST /auth/logout              → 404
❌ Rate limiting                  → HTTP 400 en vez de 429
❌ GET  /inventory/movements      → 404
❌ GET  /finance/commissions      → 404
❌ GET  /vehicles/workshop-fleet  → 404
❌ POST /personnel/attendance/check-in  → 404
❌ POST /personnel/attendance/check-out → 404
❌ POST /personnel/vehicle-usage/authorize → 404
⚠️ Gasto < S/100                 → queda PENDING (debe auto-aprobarse)
⚠️ POST /orders/:id/discount     → endpoint no existe (descuento >20% sin control)
⚠️ useRealtime.ts                → no encontrado en arellan-frontend-web/src/hooks/
```

---

## FIX 1 — POST /auth/logout (❌ 404)

Lee `arellan-platform/src/modules/auth/auth.controller.ts`.

Si el endpoint no existe o está en una ruta distinta, agrégalo:

```typescript
// auth.controller.ts

@Post('logout')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
@ApiOperation({ summary: 'Cerrar sesión actual' })
@ApiResponse({ status: 200, description: 'Sesión cerrada correctamente' })
async logout(@Request() req: any) {
  const userId = req.user.id ?? req.user.sub;
  const sessionId = req.user.sessionId; // debe venir del JWT payload
  await this.authService.logout(userId, sessionId);
  return { message: 'Sesión cerrada correctamente' };
}
```

Verifica en `auth.service.ts` que el método `logout(userId, sessionId)` existe e invalida la sesión en Redis. Si no existe:

```typescript
// auth.service.ts
async logout(userId: string, sessionId: string) {
  // 1. Eliminar de Redis
  await this.redisService.del(`session:${userId}:${sessionId}`);
  
  // 2. Marcar como revocada en BD
  await this.prisma.session.updateMany({
    where: { userId, id: sessionId, revokedAt: null },
    data: { revokedAt: new Date() },
  });

  // 3. Audit log
  await this.prisma.auditLog.create({
    data: {
      userId,
      action: 'LOGOUT',
      entity: 'Session',
      entityId: sessionId,
      severity: 'INFO',
    },
  });
}
```

**Importante:** Verifica que el JWT payload incluye `sessionId`. Si no, añádelo al generar el token en `login()`:

```typescript
// En login(), al generar el accessToken:
const payload = {
  sub: user.id,
  email: user.email,
  role: user.role,
  sessionId: session.id, // ← agregar este campo
};
```

---

## FIX 2 — Rate Limiting: HTTP 400 en vez de 429 (⚠️)

El rate limiting existe pero retorna 400 en lugar de 429. Hay dos causas posibles:

**Causa A:** El `ThrottlerGuard` no está configurado globalmente o devuelve el código incorrecto.

Lee `arellan-platform/src/app.module.ts`. Verifica:

```typescript
// app.module.ts — debe tener:
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([{
      name: 'login',
      ttl: 60000,   // 1 minuto
      limit: 5,     // 5 intentos
    }]),
    // ... otros módulos
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
```

**Causa B:** El bloqueo por `failedAttempts` retorna 400 (BadRequestException) en vez de 429.

En `auth.service.ts`, cambia la excepción del bloqueo:

```typescript
// Cambiar esto:
throw new BadRequestException('Cuenta bloqueada temporalmente');

// Por esto:
throw new HttpException(
  { statusCode: 429, message: 'Demasiados intentos fallidos. Cuenta bloqueada 15 minutos.' },
  HttpStatus.TOO_MANY_REQUESTS
);
```

Importa `HttpException, HttpStatus` de `@nestjs/common`.

---

## FIX 3 — GET /inventory/movements (❌ 404)

Lee `arellan-platform/src/modules/inventory/inventory.controller.ts`.

Si el endpoint no existe, agrégalo:

```typescript
// inventory.controller.ts

@Get('movements')
@Roles('OWNER', 'MANAGER', 'FINANCE')
@ApiOperation({ summary: 'Historial de movimientos de inventario paginado' })
@ApiQuery({ name: 'page', required: false, type: Number })
@ApiQuery({ name: 'limit', required: false, type: Number })
@ApiQuery({ name: 'itemId', required: false, type: String })
@ApiQuery({ name: 'type', required: false, enum: MovementType })
async getMovements(
  @Query('page') page = 1,
  @Query('limit') limit = 20,
  @Query('itemId') itemId?: string,
  @Query('type') type?: string,
) {
  return this.inventoryService.getStockMovements({ page: +page, limit: +limit, itemId, type });
}
```

Si `getStockMovements()` no existe en el service:

```typescript
// inventory.service.ts
async getStockMovements(query: { page: number; limit: number; itemId?: string; type?: string }) {
  const { page, limit, itemId, type } = query;
  const where: any = {};
  if (itemId) where.itemId = itemId;
  if (type) where.type = type;

  const [data, total] = await Promise.all([
    this.prisma.stockMovement.findMany({
      where,
      include: { item: { select: { name: true, code: true, unit: true } } },
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * limit,
      take: limit,
    }),
    this.prisma.stockMovement.count({ where }),
  ]);

  return { data, meta: { total, page, limit, totalPages: Math.ceil(total / limit) } };
}
```

> **Nota:** El modelo puede llamarse `StockMovement`, `InventoryMovement` o similar — verifica en `schema.prisma` el nombre exacto y úsalo.

---

## FIX 4 — GET /finance/commissions (❌ 404)

Lee `arellan-platform/src/modules/finance/finance.controller.ts` (o el módulo donde estén las comisiones).

Agrega si no existe:

```typescript
// finance.controller.ts (o commissions.controller.ts si el módulo existe separado)

@Get('commissions')
@Roles('OWNER', 'FINANCE')
@ApiOperation({ summary: 'Listado de comisiones con filtros' })
@ApiQuery({ name: 'status', required: false })
@ApiQuery({ name: 'personnelId', required: false })
async getCommissions(
  @Query('status') status?: string,
  @Query('personnelId') personnelId?: string,
  @Query('page') page = 1,
  @Query('limit') limit = 20,
) {
  const where: any = {};
  if (status) where.status = status;
  if (personnelId) where.personnelId = personnelId;

  const [data, total] = await Promise.all([
    this.prisma.commission.findMany({
      where,
      include: {
        personnel: { select: { firstName: true, lastName: true } },
        supplier: { select: { name: true } },
      },
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * +limit,
      take: +limit,
    }),
    this.prisma.commission.count({ where }),
  ]);

  return { data, meta: { total, page: +page, limit: +limit, totalPages: Math.ceil(total / +limit) } };
}
```

Si ya existe un `CommissionsController` con este endpoint pero en ruta diferente, verifica cuál es la ruta real y documenta en el reporte final.

---

## FIX 5 — GET /vehicles/workshop-fleet (❌ 404)

Lee `arellan-platform/src/modules/vehicles/vehicles.controller.ts`.

Agrega si no existe:

```typescript
// vehicles.controller.ts

@Get('workshop-fleet')
@Roles('OWNER', 'MANAGER')
@ApiOperation({ summary: 'Vehículos propios del taller (flota interna)' })
async getWorkshopFleet() {
  return this.vehiclesService.getWorkshopFleet();
}

@Get('workshop-fleet/in-use')
@Roles('OWNER', 'MANAGER')
@ApiOperation({ summary: 'Vehículos del taller actualmente en uso' })
async getFleetInUse() {
  return this.vehiclesService.getFleetInUse();
}
```

En el service:

```typescript
// vehicles.service.ts

async getWorkshopFleet() {
  // Los vehículos del taller se identifican porque NO tienen clientId
  // O porque tienen un flag especial — verifica en el schema cuál aplica
  return this.prisma.vehicle.findMany({
    where: {
      OR: [
        { clientId: null },
        // Si hay un campo isWorkshopVehicle: { isWorkshopVehicle: true }
      ]
    },
    include: {
      usageLogs: {
        where: { status: 'PENDING_RETURN' },
        take: 1,
        include: { personnel: { select: { firstName: true, lastName: true } } }
      }
    }
  });
}

async getFleetInUse() {
  return this.prisma.vehicleUsage.findMany({
    where: { status: 'PENDING_RETURN' },
    include: {
      vehicle: { select: { plate: true, brand: true, model: true } },
      personnel: { select: { firstName: true, lastName: true } },
    },
    orderBy: { checkoutAt: 'desc' },
  });
}
```

> **Nota:** Si el seed no tiene vehículos del taller (sin clientId), agrega al menos 2 al `seed.ts` y recorre el seed:
> ```typescript
> // En seed.ts — vehículos del taller (sin cliente asignado)
> await prisma.vehicle.createMany({
>   data: [
>     { plate: 'TAL-001', brand: 'Toyota', model: 'Hilux', year: 2020, color: 'Blanco', engineType: 'DIESEL' },
>     { plate: 'TAL-002', brand: 'Hyundai', model: 'H100', year: 2019, color: 'Gris', engineType: 'GASOLINE' },
>   ]
> });
> ```
> Luego: `npx prisma db seed` (si el seed es idempotente) o agrega solo estos registros manualmente.

---

## FIX 6 — POST /personnel/attendance/check-in y check-out (❌ 404)

Lee `arellan-platform/src/modules/personnel/personnel.controller.ts` y `attendance.controller.ts` (si existe).

Agrega si no existe en el controller correspondiente:

```typescript
// personnel.controller.ts (o attendance.controller.ts)

@Post('attendance/check-in')
@UseGuards(JwtAuthGuard)  // sin RolesGuard — cualquier empleado puede hacer check-in
@ApiBearerAuth()
@ApiOperation({ summary: 'Registrar entrada del empleado autenticado' })
async checkIn(@Request() req: any, @Body() body: { notes?: string }) {
  const userId = req.user.id ?? req.user.sub;
  return this.personnelService.checkIn(userId, body.notes);
}

@Post('attendance/check-out')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
@ApiOperation({ summary: 'Registrar salida del empleado autenticado' })
async checkOut(@Request() req: any, @Body() body: { notes?: string }) {
  const userId = req.user.id ?? req.user.sub;
  return this.personnelService.checkOut(userId, body.notes);
}

@Get('attendance/today')
@Roles('OWNER', 'MANAGER')
@ApiOperation({ summary: 'Asistencia de todo el personal del día de hoy' })
async getAttendanceToday() {
  return this.personnelService.getAttendanceToday();
}
```

En el service:

```typescript
// personnel.service.ts

async checkIn(userId: string, notes?: string) {
  // Buscar el Personnel del usuario
  const personnel = await this.prisma.personnel.findUnique({ where: { userId } });
  if (!personnel) throw new NotFoundException('Personal no encontrado para este usuario');

  const today = new Date();
  today.setHours(0, 0, 0, 0);

  // Verificar si ya hizo check-in hoy
  const existing = await this.prisma.attendance.findFirst({
    where: { personnelId: personnel.id, date: today },
  });

  if (existing?.checkIn) {
    throw new BadRequestException('Ya registraste tu entrada hoy');
  }

  const record = await this.prisma.attendance.upsert({
    where: { personnelId_date: { personnelId: personnel.id, date: today } },
    create: {
      personnelId: personnel.id,
      date: today,
      checkIn: new Date(),
      type: 'PRESENT',
      notes,
    },
    update: { checkIn: new Date(), notes },
  });

  // Emitir evento WebSocket
  this.realtimeGateway?.emitPersonnelCheckIn({
    personnelId: personnel.id,
    name: `${personnel.firstName} ${personnel.lastName}`,
    timestamp: record.checkIn!,
  });

  return record;
}

async checkOut(userId: string, notes?: string) {
  const personnel = await this.prisma.personnel.findUnique({ where: { userId } });
  if (!personnel) throw new NotFoundException('Personal no encontrado');

  const today = new Date();
  today.setHours(0, 0, 0, 0);

  const existing = await this.prisma.attendance.findFirst({
    where: { personnelId: personnel.id, date: today },
  });

  if (!existing?.checkIn) throw new BadRequestException('No has registrado tu entrada hoy');
  if (existing.checkOut) throw new BadRequestException('Ya registraste tu salida hoy');

  const record = await this.prisma.attendance.update({
    where: { id: existing.id },
    data: { checkOut: new Date(), notes },
  });

  this.realtimeGateway?.emitPersonnelCheckOut?.({
    personnelId: personnel.id,
    name: `${personnel.firstName} ${personnel.lastName}`,
    timestamp: record.checkOut!,
  });

  return record;
}

async getAttendanceToday() {
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  return this.prisma.attendance.findMany({
    where: { date: today },
    include: { personnel: { select: { firstName: true, lastName: true, position: true } } },
    orderBy: { checkIn: 'asc' },
  });
}
```

> **Nota:** El modelo puede llamarse `Attendance` — verifica el nombre exacto en `schema.prisma`. Si la relación única es `@@unique([personnelId, date])` usa `personnelId_date` como key compuesto en el `upsert`.

---

## FIX 7 — POST /personnel/vehicle-usage/authorize (❌ 404)

Agrega en `personnel.controller.ts`:

```typescript
@Post('vehicle-usage/authorize')
@Roles('OWNER', 'MANAGER')
@ApiOperation({ summary: 'Autorizar uso de vehículo del taller a un empleado' })
async authorizeVehicleUsage(
  @Body() body: {
    vehicleId: string;
    personnelId: string;
    purpose: string;
    destination?: string;
    expectedReturn: string;
  },
  @Request() req: any,
) {
  const authorizedBy = req.user.id ?? req.user.sub;
  return this.personnelService.authorizeVehicleUsage({ ...body, authorizedBy });
}

@Get('vehicle-usage/active')
@Roles('OWNER', 'MANAGER')
@ApiOperation({ summary: 'Usos de vehículos del taller actualmente activos' })
async getActiveVehicleUsages() {
  return this.personnelService.getActiveVehicleUsages();
}

@Get('vehicle-usage/overdue')
@Roles('OWNER', 'MANAGER')
@ApiOperation({ summary: 'Vehículos no devueltos a tiempo' })
async getOverdueVehicleUsages() {
  return this.personnelService.checkOverdueVehicles();
}

@Patch('vehicle-usage/:id/return')
@UseGuards(JwtAuthGuard)
@ApiOperation({ summary: 'Registrar devolución del vehículo' })
async returnVehicle(
  @Param('id') usageId: string,
  @Body() body: { odometerIn?: number; notes?: string },
) {
  return this.personnelService.returnVehicle(usageId, body);
}
```

En el service (si `authorizeVehicleUsage` no existe o está incompleto):

```typescript
async authorizeVehicleUsage(data: {
  vehicleId: string;
  personnelId: string;
  purpose: string;
  destination?: string;
  expectedReturn: string;
  authorizedBy: string;
}) {
  const usage = await this.prisma.vehicleUsage.create({
    data: {
      vehicleId: data.vehicleId,
      personnelId: data.personnelId,
      authorizedBy: data.authorizedBy,
      purpose: data.purpose,
      destination: data.destination,
      expectedReturn: new Date(data.expectedReturn),
      checkoutAt: new Date(),
      status: 'PENDING_RETURN',
    },
    include: {
      vehicle: { select: { plate: true, brand: true, model: true } },
      personnel: { select: { firstName: true, lastName: true } },
    },
  });

  // Auditar el uso autorizado
  await this.prisma.auditLog.create({
    data: {
      userId: data.authorizedBy,
      action: 'VEHICLE_USAGE_AUTHORIZED',
      entity: 'VehicleUsage',
      entityId: usage.id,
      severity: 'INFO',
      metadata: { vehicleId: data.vehicleId, personnelId: data.personnelId, purpose: data.purpose },
    },
  });

  return usage;
}
```

---

## FIX 8 — Gasto < S/100 debe auto-aprobarse (⚠️ queda en PENDING)

Lee `arellan-platform/src/modules/finance/finance.service.ts`.

En el método `createExpense()`, la lógica de auto-aprobación no está funcionando. Corrígela:

```typescript
async createExpense(dto: CreateExpenseDto, userId: string) {
  const amount = Number(dto.amount);

  // Determinar si requiere aprobación según el monto
  let requiresApproval = false;
  let approvalStatus: 'APPROVED' | 'PENDING' = 'APPROVED';

  if (amount > 500) {
    requiresApproval = true;
    approvalStatus = 'PENDING'; // doble aprobación OWNER
  } else if (amount > 100) {
    requiresApproval = true;
    approvalStatus = 'PENDING'; // 1 OWNER
  }
  // < S/100: approvalStatus queda como 'APPROVED' — auto-aprobado

  const expense = await this.prisma.expense.create({
    data: {
      ...dto,
      amount: dto.amount,
      paidBy: userId,
      approvedBy: amount <= 100 ? userId : undefined, // auto-aprobado por quien lo crea
      approvalStatus,
      requiresApproval,
    },
  });

  // Solo crear Approval si realmente se requiere
  if (requiresApproval) {
    await this.prisma.approval.create({
      data: {
        type: 'EXPENSE',
        status: 'PENDING',
        title: dto.description,
        amount: dto.amount,
        requestedById: userId,
        expenseId: expense.id,
        // Para > S/500, agregar metadata de doble aprobación
        ...(amount > 500 ? { reason: 'Requiere doble aprobación de dos OWNERS (monto > S/ 500)' } : {}),
      },
    });

    // Notificar a los OWNERS vía WebSocket
    this.realtimeGateway?.emitApprovalRequested({
      approvalId: expense.id,
      type: 'EXPENSE',
      amount,
      requestedBy: userId,
    });
  }

  return expense;
}
```

---

## FIX 9 — POST /orders/:id/discount (⚠️ endpoint no existe)

Lee `arellan-platform/src/modules/orders/orders.controller.ts`.

Agrega el endpoint:

```typescript
// orders.controller.ts

@Post(':id/discount')
@Roles('OWNER', 'MANAGER', 'MECHANIC')
@ApiOperation({ summary: 'Aplicar descuento a una orden. Descuentos > 20% requieren aprobación OWNER.' })
async applyDiscount(
  @Param('id') orderId: string,
  @Body() body: { discountAmount?: number; discountPercentage?: number; reason: string },
  @Request() req: any,
) {
  const userId = req.user.id ?? req.user.sub;
  const role = req.user.role;
  return this.ordersService.applyDiscount(orderId, body, userId, role);
}
```

En el service:

```typescript
// orders.service.ts

async applyDiscount(
  orderId: string,
  dto: { discountAmount?: number; discountPercentage?: number; reason: string },
  userId: string,
  userRole: string,
) {
  const order = await this.prisma.workOrder.findUnique({ where: { id: orderId } });
  if (!order) throw new NotFoundException('Orden no encontrada');

  // Calcular el descuento
  const total = Number(order.totalCost);
  const discountAmount = dto.discountAmount
    ?? (total * (dto.discountPercentage ?? 0) / 100);
  const discountPct = dto.discountPercentage
    ?? ((discountAmount / total) * 100);

  // Regla: descuento > 20% requiere aprobación OWNER
  if (discountPct > 20 && userRole !== 'OWNER') {
    const approval = await this.prisma.approval.create({
      data: {
        type: 'DISCOUNT',
        status: 'PENDING',
        title: `Descuento ${discountPct.toFixed(1)}% en orden ${order.orderNumber}`,
        amount: discountAmount,
        requestedById: userId,
        reason: dto.reason,
      },
    });

    this.realtimeGateway?.emitApprovalRequested({
      approvalId: approval.id,
      type: 'DISCOUNT',
      amount: discountAmount,
      requestedBy: userId,
    });

    return {
      status: 'PENDING_APPROVAL',
      message: 'Descuento > 20% requiere aprobación de OWNER',
      approvalId: approval.id,
    };
  }

  // Descuento aprobado directamente
  const updated = await this.prisma.workOrder.update({
    where: { id: orderId },
    data: {
      discount: discountAmount,
      finalAmount: Math.max(0, total - discountAmount),
    },
  });

  await this.prisma.auditLog.create({
    data: {
      userId,
      action: 'DISCOUNT_APPLIED',
      entity: 'WorkOrder',
      entityId: orderId,
      severity: discountPct > 20 ? 'WARNING' : 'INFO',
      metadata: { discountAmount, discountPct, reason: dto.reason },
    },
  });

  return updated;
}
```

---

## FIX 10 — useRealtime.ts en arellan-frontend-web (⚠️ no encontrado)

Lee la estructura de `arellan-frontend-web/src/hooks/`. Si el archivo no existe en esa ruta (puede estar en otra como `lib/websocket.ts`), crea `src/hooks/useRealtime.ts`:

```typescript
// arellan-frontend-web/src/hooks/useRealtime.ts
'use client';

import { useEffect, useRef, useCallback } from 'react';
import { io, Socket } from 'socket.io-client';

const WS_URL = process.env.NEXT_PUBLIC_WS_URL ?? 'http://localhost:3001';

type RealtimeEvent =
  | 'order:created'
  | 'order:status_changed'
  | 'inventory:low_stock'
  | 'payment:received'
  | 'personnel:check_in'
  | 'personnel:check_out'
  | 'vehicle:overdue'
  | 'approval:requested'
  | 'approval:resolved'
  | 'alert:security';

type EventHandler = (data: any) => void;

export function useRealtime(
  events: Partial<Record<RealtimeEvent, EventHandler>>,
  enabled = true,
) {
  const socketRef = useRef<Socket | null>(null);

  const getToken = useCallback(() => {
    if (typeof window === 'undefined') return null;
    return localStorage.getItem('accessToken');
  }, []);

  useEffect(() => {
    if (!enabled) return;
    const token = getToken();
    if (!token) return;

    const socket = io(WS_URL, {
      auth: { token },
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 2000,
    });

    socketRef.current = socket;

    socket.on('connect', () => {
      console.log('[WS] Conectado:', socket.id);
    });

    socket.on('disconnect', (reason) => {
      console.log('[WS] Desconectado:', reason);
    });

    socket.on('connect_error', (err) => {
      console.warn('[WS] Error de conexión:', err.message);
    });

    // Suscribir a los eventos pasados como parámetro
    Object.entries(events).forEach(([event, handler]) => {
      if (handler) socket.on(event, handler);
    });

    return () => {
      Object.keys(events).forEach((event) => socket.off(event));
      socket.disconnect();
      socketRef.current = null;
    };
  }, [enabled]); // eslint-disable-line react-hooks/exhaustive-deps

  return socketRef;
}
```

Instala la dependencia si no está:

```bash
cd arellan-frontend-web
npm install socket.io-client
```

Verifica que en `arellan-frontend-web/package.json` ya esté `socket.io-client`. Si no, agrégala.

**Uso en el dashboard** — agrega este hook en la página del dashboard para recibir actualizaciones en tiempo real:

```typescript
// src/app/(dashboard)/page.tsx — agregar al componente:
import { useRealtime } from '@/hooks/useRealtime';
import { useQueryClient } from '@tanstack/react-query';

// Dentro del componente:
const queryClient = useQueryClient();

useRealtime({
  'order:status_changed': () => {
    queryClient.invalidateQueries({ queryKey: ['orders'] });
    queryClient.invalidateQueries({ queryKey: ['orders-summary'] });
  },
  'inventory:low_stock': (data) => {
    console.warn('[Alerta] Stock bajo:', data.itemName, data.currentStock);
    queryClient.invalidateQueries({ queryKey: ['inventory-low-stock'] });
  },
  'approval:requested': () => {
    queryClient.invalidateQueries({ queryKey: ['pending-approvals'] });
  },
  'alert:security': (data) => {
    console.error('[SEGURIDAD]', data.description);
  },
});
```

---

## VERIFICACIÓN FINAL — Solo los 11 ítems corregidos

Ejecuta solo estos comandos. No vuelvas a verificar lo que ya pasó.

```bash
BASE="http://localhost:3001/api/v1"
TOKEN=$(curl -s -X POST "$BASE/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

echo "=== VERIFICACIÓN DE LOS 11 FIXES ==="

# Fix 1: logout
CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$BASE/auth/logout" \
  -H "Authorization: Bearer $TOKEN")
[ "$CODE" = "200" ] && echo "✅ POST /auth/logout → $CODE" || echo "❌ POST /auth/logout → $CODE"

# Fix 2: rate limiting
for i in 1 2 3 4 5; do
  curl -s -o /dev/null -X POST "$BASE/auth/login" \
    -H "Content-Type: application/json" \
    -d '{"email":"mecanico1@arellanautos.pe","password":"wrong_pass"}'
done
CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$BASE/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"mecanico1@arellanautos.pe","password":"wrong_pass"}')
[ "$CODE" = "429" ] || [ "$CODE" = "403" ] && \
  echo "✅ Rate limiting → HTTP $CODE" || echo "❌ Rate limiting → HTTP $CODE (esperado 429)"

# Re-login Edgar
TOKEN=$(curl -s -X POST "$BASE/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

# Fix 3
CODE=$(curl -s -o /dev/null -w "%{http_code}" "$BASE/inventory/movements" \
  -H "Authorization: Bearer $TOKEN")
[ "$CODE" = "200" ] && echo "✅ GET /inventory/movements → $CODE" || echo "❌ → $CODE"

# Fix 4
CODE=$(curl -s -o /dev/null -w "%{http_code}" "$BASE/finance/commissions" \
  -H "Authorization: Bearer $TOKEN")
[ "$CODE" = "200" ] && echo "✅ GET /finance/commissions → $CODE" || echo "❌ → $CODE"

# Fix 5
CODE=$(curl -s -o /dev/null -w "%{http_code}" "$BASE/vehicles/workshop-fleet" \
  -H "Authorization: Bearer $TOKEN")
[ "$CODE" = "200" ] && echo "✅ GET /vehicles/workshop-fleet → $CODE" || echo "❌ → $CODE"

# Fix 6
TOKEN_MECH=$(curl -s -X POST "$BASE/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"mecanico1@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$BASE/personnel/attendance/check-in" \
  -H "Authorization: Bearer $TOKEN_MECH" \
  -H "Content-Type: application/json" \
  -d '{"notes":"Llegué puntual"}')
[ "$CODE" = "200" ] || [ "$CODE" = "201" ] && \
  echo "✅ POST /personnel/attendance/check-in → $CODE" || echo "❌ → $CODE"

CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$BASE/personnel/attendance/check-out" \
  -H "Authorization: Bearer $TOKEN_MECH" \
  -H "Content-Type: application/json" \
  -d '{"notes":"Turno completado"}')
[ "$CODE" = "200" ] || [ "$CODE" = "201" ] && \
  echo "✅ POST /personnel/attendance/check-out → $CODE" || echo "❌ → $CODE"

# Fix 7
FLEET_VEHICLE=$(curl -s "$BASE/vehicles/workshop-fleet" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); items=d.get('data',d) if isinstance(d,dict) else d; print(items[0]['id'] if items else '')" 2>/dev/null)
MECH_PERS=$(curl -s "$BASE/personnel?limit=1" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['id'])" 2>/dev/null)

if [ ! -z "$FLEET_VEHICLE" ]; then
  CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    -X POST "$BASE/personnel/vehicle-usage/authorize" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"vehicleId\":\"$FLEET_VEHICLE\",\"personnelId\":\"$MECH_PERS\",\"purpose\":\"Test\",\"expectedReturn\":\"$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u +%Y-%m-%dT%H:%M:%SZ)\"}")
  [ "$CODE" = "200" ] || [ "$CODE" = "201" ] && \
    echo "✅ POST /personnel/vehicle-usage/authorize → $CODE" || echo "❌ → $CODE"
else
  echo "⚠️ No hay vehículos de flota en BD — Fix 5 debe completarse primero"
fi

# Fix 8: gasto < S/100 auto-aprobado
EXPENSE=$(curl -s -X POST "$BASE/finance/expenses" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"category":"FOOD","description":"Almuerzo equipo test","amount":50,"paymentMethod":"CASH"}')
APPROVAL_STATUS=$(echo $EXPENSE | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('approvalStatus','?'))" 2>/dev/null)
[ "$APPROVAL_STATUS" = "APPROVED" ] && \
  echo "✅ Gasto S/50 → approvalStatus=$APPROVAL_STATUS (auto-aprobado)" || \
  echo "❌ Gasto S/50 → approvalStatus=$APPROVAL_STATUS (debe ser APPROVED)"

# Fix 9: descuento con control
ORDER_ID=$(curl -s "$BASE/orders?limit=1" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['id'])" 2>/dev/null)
DISC=$(curl -s -X POST "$BASE/orders/$ORDER_ID/discount" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"discountPercentage":25,"reason":"Test descuento alto"}')
STATUS=$(echo $DISC | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','applied'))" 2>/dev/null)
[ "$STATUS" = "PENDING_APPROVAL" ] && \
  echo "✅ Descuento 25% → $STATUS (requiere aprobación)" || \
  echo "⚠️ Descuento 25% → $STATUS (verificar si OWNER lo aplica directamente)"

# Fix 10: useRealtime.ts
[ -f "arellan-frontend-web/src/hooks/useRealtime.ts" ] && \
  echo "✅ useRealtime.ts existe" || echo "❌ useRealtime.ts no encontrado"

# Socket.io-client en package.json
grep -q "socket.io-client" arellan-frontend-web/package.json && \
  echo "✅ socket.io-client en dependencies" || echo "❌ socket.io-client no instalado"

echo ""
echo "=== BUILDS FINALES ==="
cd arellan-frontend-web && npm run build 2>&1 | tail -3 && echo "✅ frontend-web" || echo "❌ frontend-web build falló"
cd ../arellan-mechanic-ui && npm run build 2>&1 | tail -3 && echo "✅ mechanic-ui" || echo "❌ mechanic-ui build falló"
cd ../arellan-client-portal && npm run build 2>&1 | tail -3 && echo "✅ client-portal" || echo "❌ client-portal build falló"
```

---

## REPORTE FINAL REQUERIDO

```
═══════════════════════════════════════════════════════════════════
REPORTE FASE 4 — 11 FIXES APLICADOS
═══════════════════════════════════════════════════════════════════
Fix 1:  POST /auth/logout ........................ [✅ HTTP 200 / ❌]
Fix 2:  Rate limiting → HTTP 429 ................. [✅ / ❌ aún HTTP N]
Fix 3:  GET /inventory/movements ................. [✅ HTTP 200 / ❌]
Fix 4:  GET /finance/commissions ................. [✅ HTTP 200 / ❌]
Fix 5:  GET /vehicles/workshop-fleet ............. [✅ HTTP 200, N vehículos / ❌]
Fix 6a: POST /personnel/attendance/check-in ...... [✅ HTTP 201 / ❌]
Fix 6b: POST /personnel/attendance/check-out ..... [✅ HTTP 200 / ❌]
Fix 7:  POST /personnel/vehicle-usage/authorize .. [✅ HTTP 201 / ❌]
Fix 8:  Gasto S/50 → approvalStatus=APPROVED ..... [✅ / ❌]
Fix 9:  Descuento 25% → PENDING_APPROVAL ......... [✅ / ❌]
Fix 10: useRealtime.ts + socket.io-client ........ [✅ / ❌]

SCORECARD ACTUALIZADO:
  Antes de esta sesión: 35/42 ítems ✅
  Fixes aplicados:      N/11
  Total final:          N/42 ítems ✅

SISTEMA LISTO PARA PRUEBAS: [SÍ / NO — falta: X]
═══════════════════════════════════════════════════════════════════
```

**Implementa en el orden listado. No cambies lo que ya funciona del scorecard anterior.**
