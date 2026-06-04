# OPENCODE — FASE 3: COMPLETADO FINAL Y VERIFICACIÓN E2E
# Cierre del ecosistema Arellan Hnos

---

## CONTEXTO

El ecosistema tiene dos fases previas completadas:
- Schema Prisma (32 modelos), seed (499 líneas), Docker Compose → ✅
- Auth completo (refresh tokens, sesiones Redis, rate limiting) → ✅
- 16 módulos backend registrados en app.module.ts → ✅
- RealtimeGateway en src/common/gateway/ con 10 eventos → ✅
- payments.service con validación anti-Yape personal → ✅
- orders.service con getSummaryStats(), addItem() con descuento de inventario → ✅
- Frontend web (api-client con JWT+refresh, Zustand auth store, 11 páginas) → ✅
- Mechanic UI (6 páginas incluyendo CheckInPage.tsx) → ✅
- Design system (@arellan-hnos-core-ecosystem/ui, 33 componentes) → ✅

**Esta sesión tiene un alcance específico y acotado.** No refactorices lo que ya funciona. Lee los archivos antes de modificar cualquier cosa.

---

## LO QUE DEBES IMPLEMENTAR — EN ESTE ORDEN EXACTO

---

### PASO 1 — inventory.controller.ts: exponer los endpoints que faltan

Lee primero `arellan-platform/src/modules/inventory/inventory.service.ts` y `inventory.controller.ts`.

Los métodos `getLowStock()` y `getValuation()` ya existen en el service. Solo hay que exponerlos en el controller si no están.

Agrega estos endpoints si no existen:

```typescript
// inventory.controller.ts

@Get('low-stock')
@ApiOperation({ summary: 'Items con stock bajo el mínimo configurado' })
@ApiResponse({ status: 200, description: 'Lista de items en alerta de stock' })
async getLowStock() {
  return this.inventoryService.getLowStock();
}

@Get('valuation')
@ApiOperation({ summary: 'Valor total del inventario actual' })
@ApiResponse({ status: 200, description: '{ totalItems, totalCostValue, totalSaleValue, lowStockCount }' })
async getValuation() {
  return this.inventoryService.getValuation();
}
```

Si `getValuation()` no existe en el service, impleméntalo:

```typescript
// inventory.service.ts
async getValuation() {
  const items = await this.prisma.inventoryItem.findMany({
    where: { isActive: true },
    select: { currentStock: true, costPrice: true, salePrice: true, minStock: true }
  });

  const totalItems = items.length;
  const totalCostValue = items.reduce((sum, i) => sum + Number(i.costPrice) * i.currentStock, 0);
  const totalSaleValue = items.reduce((sum, i) => sum + Number(i.salePrice) * i.currentStock, 0);
  const lowStockCount = items.filter(i => i.currentStock < i.minStock).length;

  return { totalItems, totalCostValue, totalSaleValue, lowStockCount };
}
```

---

### PASO 2 — finance.controller.ts: exponer los endpoints que faltan

Lee primero `arellan-platform/src/modules/finance/finance.service.ts` y `finance.controller.ts`.

`getDashboard()` y `getCashflow()` ya existen en el service. Expón si no están:

```typescript
// finance.controller.ts

@Get('dashboard')
@Roles('OWNER', 'MANAGER', 'FINANCE')
@ApiOperation({ summary: 'KPIs financieros del período actual' })
async getDashboard(@Query('period') period: 'day' | 'week' | 'month' = 'month') {
  return this.financeService.getDashboard(period);
}

@Get('cashflow')
@Roles('OWNER', 'FINANCE')
@ApiOperation({ summary: 'Flujo de caja: ingresos vs egresos por período' })
async getCashflow(
  @Query('from') from: string,
  @Query('to') to: string,
) {
  return this.financeService.getCashflow(
    from ? new Date(from) : undefined,
    to ? new Date(to) : undefined
  );
}
```

Si `getDashboard()` o `getCashflow()` no existen o son stubs vacíos en el service, impleméntalos:

```typescript
// finance.service.ts

async getDashboard(period: 'day' | 'week' | 'month' = 'month') {
  const now = new Date();
  const from = period === 'day'
    ? new Date(now.setHours(0, 0, 0, 0))
    : period === 'week'
    ? new Date(Date.now() - 7 * 24 * 60 * 60 * 1000)
    : new Date(now.getFullYear(), now.getMonth(), 1);

  const [payments, expenses, orders, pendingApprovals] = await Promise.all([
    this.prisma.payment.aggregate({
      where: { paidAt: { gte: from } },
      _sum: { amount: true },
      _count: true,
    }),
    this.prisma.expense.aggregate({
      where: { expenseDate: { gte: from }, approvalStatus: 'APPROVED' },
      _sum: { amount: true },
    }),
    this.prisma.workOrder.count({
      where: { status: { in: ['IN_PROGRESS', 'WAITING_PARTS', 'DIAGNOSING'] } }
    }),
    this.prisma.approval.count({ where: { status: 'PENDING' } }),
  ]);

  const totalRevenue = Number(payments._sum.amount ?? 0);
  const totalExpenses = Number(expenses._sum.amount ?? 0);

  return {
    period,
    from,
    revenue: totalRevenue,
    expenses: totalExpenses,
    profit: totalRevenue - totalExpenses,
    transactionCount: payments._count,
    activeOrders: orders,
    pendingApprovals,
  };
}

async getCashflow(from?: Date, to?: Date) {
  const start = from ?? new Date(new Date().getFullYear(), new Date().getMonth(), 1);
  const end = to ?? new Date();

  const [inflows, outflows] = await Promise.all([
    this.prisma.payment.groupBy({
      by: ['method'],
      where: { paidAt: { gte: start, lte: end } },
      _sum: { amount: true },
    }),
    this.prisma.expense.groupBy({
      by: ['category'],
      where: { expenseDate: { gte: start, lte: end }, approvalStatus: 'APPROVED' },
      _sum: { amount: true },
    }),
  ]);

  const totalIn = inflows.reduce((s, r) => s + Number(r._sum.amount ?? 0), 0);
  const totalOut = outflows.reduce((s, r) => s + Number(r._sum.amount ?? 0), 0);

  return { from: start, to: end, inflows, outflows, totalIn, totalOut, net: totalIn - totalOut };
}
```

---

### PASO 3 — inventory.service.ts: implementar reserveForOrder()

Si no existe, agrégalo:

```typescript
// inventory.service.ts
async reserveForOrder(itemId: string, quantity: number, workOrderId: string, userId: string) {
  return this.prisma.$transaction(async (tx) => {
    const item = await tx.inventoryItem.findUnique({ where: { id: itemId } });
    if (!item) throw new NotFoundException(`Item ${itemId} no encontrado`);
    if (item.currentStock < quantity) {
      throw new BadRequestException(
        `Stock insuficiente. Disponible: ${item.currentStock}, solicitado: ${quantity}`
      );
    }

    // Descontar el stock
    await tx.inventoryItem.update({
      where: { id: itemId },
      data: { currentStock: { decrement: quantity } },
    });

    // Registrar movimiento
    await tx.stockMovement.create({
      data: {
        itemId,
        type: 'RESERVATION',
        quantity: -quantity,
        reference: workOrderId,
        reason: `Reservado para orden ${workOrderId}`,
        userId,
      },
    });

    // Verificar si quedó bajo mínimo → alerta
    const updated = await tx.inventoryItem.findUnique({ where: { id: itemId } });
    if (updated && updated.currentStock < updated.minStock) {
      // Emitir alerta WebSocket (inyectar RealtimeGateway en el service si no está)
      this.realtimeGateway?.emitInventoryLowStock({
        itemId,
        itemName: item.name,
        currentStock: updated.currentStock,
        minStock: updated.minStock,
      });
    }

    return { success: true, remainingStock: updated?.currentStock };
  });
}
```

Expón el endpoint en el controller:

```typescript
@Post(':id/reserve')
@Roles('OWNER', 'MANAGER', 'MECHANIC')
@ApiOperation({ summary: 'Reservar stock para una orden de trabajo' })
async reserveForOrder(
  @Param('id') itemId: string,
  @Body() body: { quantity: number; workOrderId: string },
  @CurrentUser() user: any,
) {
  return this.inventoryService.reserveForOrder(itemId, body.quantity, body.workOrderId, user.id);
}
```

---

### PASO 4 — finance.service.ts: doble aprobación para gastos > S/ 500

Lee el método `approveExpense()` en `finance.service.ts`. Si no tiene la lógica de doble aprobación, agrégala:

```typescript
async approveExpense(approvalId: string, approverId: string, role: string) {
  const approval = await this.prisma.approval.findUnique({
    where: { id: approvalId },
    include: { expense: true }
  });

  if (!approval || !approval.expense) {
    throw new NotFoundException('Aprobación no encontrada');
  }

  const amount = Number(approval.expense.amount);

  // Gastos > S/ 500: requieren 2 approvers (usar metadata para contar)
  if (amount > 500) {
    const metadata = (approval.expense as any).metadata ?? {};
    const approvers: string[] = metadata.approvers ?? [];

    if (approvers.includes(approverId)) {
      throw new BadRequestException('Ya aprobaste este gasto. Se requiere un segundo OWNER.');
    }

    approvers.push(approverId);

    if (approvers.length < 2) {
      // Primera aprobación: guardar en metadata y esperar la segunda
      await this.prisma.expense.update({
        where: { id: approval.expense.id },
        data: { notes: `Aprobado por ${approverId} (1/2). Esperando segundo OWNER.` }
      });
      return { status: 'PARTIAL_APPROVAL', message: 'Primera aprobación registrada. Se requiere un segundo OWNER.' };
    }
    // Segunda aprobación: aprobar definitivamente
  }

  // Para gastos <= S/ 500: aprobación directa de 1 OWNER o MANAGER
  return this.prisma.$transaction([
    this.prisma.approval.update({
      where: { id: approvalId },
      data: { status: 'APPROVED', approvedById: approverId, approvedAt: new Date() }
    }),
    this.prisma.expense.update({
      where: { id: approval.expense!.id },
      data: { approvalStatus: 'APPROVED', approvedBy: approverId }
    }),
  ]);
}
```

---

### PASO 5 — Unificar OrdersGateway + RealtimeGateway

Lee ambos archivos: `src/gateways/orders.gateway.ts` y `src/common/gateway/realtime.gateway.ts`.

**Estrategia:** Mueve toda la lógica de `orders.gateway.ts` al `RealtimeGateway`. Luego elimina `orders.gateway.ts` y actualiza cualquier importación en los services que lo usen.

Pasos:
1. Leer el contenido de `orders.gateway.ts`
2. Copiar los métodos que NO existan ya en `realtime.gateway.ts`
3. En cada service que importe `OrdersGateway`, reemplazar por `RealtimeGateway`
4. Actualizar `app.module.ts` y `gateways.module.ts` para quitar `OrdersGateway`
5. Eliminar `orders.gateway.ts`

Verifica que `RealtimeGateway` sea inyectable en los services (debe estar en `exports` del `RealtimeModule`).

---

### PASO 6 — mechanic-ui: verificar que CheckInPage llama al backend real

Lee `arellan-mechanic-ui/src/app/check-in/page.tsx`.

Verifica que el botón de check-in llame a:
```typescript
// POST /api/v1/personnel/attendance/check-in
await api.post('/personnel/attendance/check-in', { notes: '' });
```

Y el check-out a:
```typescript
// POST /api/v1/personnel/attendance/check-out
await api.post('/personnel/attendance/check-out', { notes: '' });
```

Si la página hace llamadas a mocks o arrays locales, conviértela a llamadas reales. El usuario autenticado se obtiene del token JWT en el backend (el endpoint debe extraer el `personnelId` del `userId` de la sesión).

Verifica también que en `arellan-platform/src/modules/personnel/personnel.controller.ts` existan:
```typescript
@Post('attendance/check-in')
@Post('attendance/check-out')
```

Si no existen, créalos llamando al `attendanceService.checkIn(userId)` / `checkOut(userId)`.

---

### PASO 7 — client-portal: login de cliente + historial + endpoint público backend

#### 7a — Endpoint público en el backend

Lee `arellan-platform/src/common/public/public.controller.ts`.

Si no existe el endpoint de tracking, agrégalo:

```typescript
// public.controller.ts — sin guards, sin JWT

@Get('orders/:orderNumber/status')
@ApiOperation({ summary: 'Tracking público de orden por número (sin autenticación)' })
async getOrderStatus(@Param('orderNumber') orderNumber: string) {
  const order = await this.prisma.workOrder.findUnique({
    where: { orderNumber },
    select: {
      orderNumber: true,
      status: true,
      estimatedAt: true,
      completedAt: true,
      deliveredAt: true,
      vehicle: {
        select: { plate: true, brand: true, model: true, year: true }
      },
      client: {
        select: { firstName: true }
      },
      timeline: {
        select: { event: true, description: true, createdAt: true },
        orderBy: { createdAt: 'asc' }
      }
      // NO exponer: montos, datos del personal, datos financieros
    }
  });

  if (!order) throw new NotFoundException('Orden no encontrada');
  return order;
}
```

Verifica que el módulo público esté decorado con `@Public()` en todos sus endpoints o que el guard global lo excluya por ruta.

#### 7b — client-portal: páginas de login e historial

Lee la estructura actual de `arellan-client-portal/src/app/`.

Si no existen, crea:

**`src/app/login/page.tsx`** — formulario de login para clientes:
```typescript
// Llama a POST /api/v1/auth/login con las credenciales del cliente
// El rol del usuario debe ser 'CLIENT' para poder acceder al portal
// Si el login retorna un rol distinto, mostrar error: "Este portal es solo para clientes"
// Al autenticar, guardar token en localStorage y redirigir a /dashboard
```

**`src/app/dashboard/page.tsx`** — vista principal del cliente autenticado:
```typescript
// Llama a GET /api/v1/vehicles?myVehicles=true  (vehículos del cliente)
// Llama a GET /api/v1/orders?clientId=[miId]    (mis órdenes)
// Muestra: nombre, mis vehículos, órdenes recientes con su estado
// Solo datos del cliente autenticado (el backend filtra por clientId del JWT)
```

**`src/app/orders/page.tsx`** — historial de órdenes del cliente:
```typescript
// Llama a GET /api/v1/orders — el backend filtra automáticamente por el cliente del JWT
// Lista paginada con: número de orden, estado, vehículo, fecha, monto total
// Estado visual: badge de color según WorkOrderStatus
```

**`src/app/orders/[id]/page.tsx`** — detalle de orden del cliente:
```typescript
// Llama a GET /api/v1/orders/:id
// Muestra: timeline de eventos, repuestos usados, cotización si existe, monto
// Botón "Aprobar cotización" si el estado es QUOTED
```

Para el backend: verifica que en `orders.service.ts`, cuando el usuario tiene rol `CLIENT`, el `findAll()` filtre automáticamente por `clientId` del usuario. Si no está, agrégalo:

```typescript
// En orders.service.ts — findAll()
async findAll(query: any, userId: string, role: string) {
  const where: any = { deletedAt: null };

  // Clientes solo ven sus propias órdenes
  if (role === 'CLIENT') {
    const client = await this.prisma.client.findFirst({ where: { userId } });
    if (!client) throw new ForbiddenException('No tienes órdenes registradas');
    where.clientId = client.id;
  }

  // ... resto de los filtros
}
```

---

## VERIFICACIÓN E2E OBLIGATORIA

Una vez implementados todos los pasos anteriores, ejecuta estos comandos en orden. **Reporta el output exacto de cada uno.**

### Paso A — Infraestructura

```bash
# Desde la raíz del proyecto
docker compose up -d postgres redis pgadmin
sleep 5
docker compose ps
# Esperado: postgres, redis, pgadmin → State: running
```

### Paso B — Backend: migración y seed

```bash
cd arellan-platform
npx prisma migrate deploy
# Esperado: "All migrations have been successfully applied"

npx prisma db seed
# Esperado: "Seed completado: N usuarios, N clientes, N órdenes..."
# Sin errores de foreign key ni unique constraint

npm run start:dev &
sleep 10
# Esperado: "Backend corriendo en http://localhost:3001"
# Esperado: "WebSocket gateway iniciado"
# Sin errores de módulo no encontrado
```

### Paso C — Prueba de endpoints (reportar HTTP status de cada uno)

```bash
# Login
TOKEN=$(curl -s -X POST http://localhost:3001/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

echo "Token obtenido: ${TOKEN:0:20}..."

# Health
curl -s http://localhost:3001/api/v1/health | head -c 100
# Esperado: {"status":"ok",...}

# Orders stats
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3001/api/v1/orders/stats/summary \
  -H "Authorization: Bearer $TOKEN"
# Esperado: 200

# Inventory low-stock
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3001/api/v1/inventory/low-stock \
  -H "Authorization: Bearer $TOKEN"
# Esperado: 200

# Finance dashboard
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3001/api/v1/finance/dashboard \
  -H "Authorization: Bearer $TOKEN"
# Esperado: 200

# Inventory valuation
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3001/api/v1/inventory/valuation \
  -H "Authorization: Bearer $TOKEN"
# Esperado: 200

# Tracking público (sin token) — usa número de orden del seed
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3001/api/v1/public/orders/WO-2026-0001/status
# Esperado: 200 (sin auth)
```

### Paso D — Frontends: arranque sin errores

```bash
# Frontend web
cd ../arellan-frontend-web && npm run build 2>&1 | tail -5
# Esperado: sin errores de TypeScript ni de import

# Mechanic UI
cd ../arellan-mechanic-ui && npm run build 2>&1 | tail -5

# Client Portal
cd ../arellan-client-portal && npm run build 2>&1 | tail -5

# Mobile App
cd ../arellan-mobile-app && npm run build 2>&1 | tail -5
```

### Paso E — npm audit fix

```bash
for dir in arellan-frontend-web arellan-mechanic-ui arellan-client-portal arellan-mobile-app; do
  echo "=== $dir ==="
  cd ../$dir
  npm audit fix --force 2>&1 | tail -3
done
```

---

## REPORTE FINAL REQUERIDO

Al terminar, entrega este reporte:

```
═══════════════════════════════════════════════════════
REPORTE FASE 3 — COMPLETADO
═══════════════════════════════════════════════════════

IMPLEMENTADO:
  [ ] inventory.controller — GET /low-stock, GET /valuation
  [ ] inventory.service — reserveForOrder()
  [ ] finance.controller — GET /dashboard, GET /cashflow
  [ ] finance.service — doble aprobación > S/ 500
  [ ] OrdersGateway unificado en RealtimeGateway
  [ ] mechanic-ui — CheckInPage conectada al backend real
  [ ] client-portal — login, dashboard, historial
  [ ] backend — GET /public/orders/:num/status (sin auth)

VERIFICACIÓN E2E:
  docker compose ps         → [✅ 3 servicios running / ❌ error: X]
  prisma migrate deploy     → [✅ / ❌ error: X]
  prisma db seed            → [✅ / ❌ error: X]
  backend :3001             → [✅ / ❌ error: X]
  POST /auth/login          → HTTP [200 / N] — token: [✅ / ❌]
  GET /orders/stats/summary → HTTP [200 / N]
  GET /inventory/low-stock  → HTTP [200 / N] — items: N
  GET /finance/dashboard    → HTTP [200 / N]
  GET /inventory/valuation  → HTTP [200 / N]
  GET /public/orders/:num   → HTTP [200 / N] (sin auth)
  frontend-web build        → [✅ / ❌ error: X]
  mechanic-ui build         → [✅ / ❌ error: X]
  client-portal build       → [✅ / ❌ error: X]
  npm audit fix             → [✅ 0 críticos / ⚠️ N warnings]

AÚN PENDIENTE (si quedó algo):
  [lista honesta]

SISTEMA LISTO PARA PRUEBAS MANUALES: [SÍ / NO — falta: X]
═══════════════════════════════════════════════════════
```

---

**Empieza leyendo los archivos de cada paso antes de modificar. No toques lo que ya funciona.**
