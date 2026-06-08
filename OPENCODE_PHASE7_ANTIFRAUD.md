# OPENCODE — FASE 7: REGLAS DE NEGOCIO ANTI-FRAUDE
# Las 9 reglas críticas que justifican la existencia del sistema

---

## CONTEXTO

La infraestructura está construida (42/42 endpoints, 4 UIs funcionales). Pero las **reglas de negocio que detienen el fraude activo** — los S/.2,000-5,000/mes en pérdidas — no existen aún. Este prompt las implementa.

**Referencia principal:** Los archivos de documentación del proyecto:
- `cash-management-flow.md` — flujo de QR y caja
- `expense-authorization.md` — máquina de estados de gastos
- `vehicle-intake-flow.md` — fotos + SHA-256 + geofencing
- `arellan_backend_api.md` — reglas de inventario + audit log
- `arellan_workers.md` — 5 BullMQ workers
- `arellan_auth_service.md` — MFA TOTP

**Regla de oro de esta sesión:**
```
Antes de modificar cualquier archivo → view + leer
Para archivos existentes → str_replace
Para archivos nuevos → create_file
Respetar el design system: @arellan-hnos-core-ecosystem/ui, color primario #1B3A6B
```

---

## REGLA 1 — QR Dinámico de Cobro (CRÍTICO MÁXIMO)

Este es el mecanismo anti-fraude principal. Ricardo desviaba cobros a su Yape personal porque no había sistema. Esta regla lo hace imposible.

**Leer primero:**
```
view arellan-platform/src/modules/finance/finance.controller.ts
view arellan-platform/src/modules/finance/finance.service.ts
view arellan-platform/src/modules/orders/orders.service.ts
```

### Backend: POST /finance/qr/generate

```typescript
// finance.controller.ts — agregar con str_replace
@Post('qr/generate')
@Roles('OWNER', 'MANAGER', 'FINANCE')
@ApiOperation({ summary: 'Genera QR dinámico de cobro para una OT. Expira en 7 minutos.' })
async generatePaymentQR(
  @Body() body: { workOrderId: string },
  @Request() req: any,
) {
  return this.financeService.generatePaymentQR(body.workOrderId, req.user.id);
}
```

```typescript
// finance.service.ts — implementar generatePaymentQR()

async generatePaymentQR(workOrderId: string, userId: string) {
  const order = await this.prisma.workOrder.findUnique({
    where: { id: workOrderId },
    include: { client: { select: { firstName: true, lastName: true } } }
  });
  if (!order) throw new NotFoundException('Orden no encontrada');
  if (order.paymentStatus === 'PAID') {
    throw new BadRequestException('Esta orden ya está pagada');
  }

  // Generar token único de idempotencia para este QR
  const qrToken = crypto.randomUUID();
  const expiresAt = new Date(Date.now() + 7 * 60 * 1000); // 7 minutos

  // Guardar en Redis con TTL de 7 minutos
  await this.redisService.setex(
    `qr:payment:${qrToken}`,
    420, // 7 minutos en segundos
    JSON.stringify({
      workOrderId,
      amount: Number(order.finalAmount || order.totalCost),
      orderId: order.orderNumber,
      createdBy: userId,
    })
  );

  // Registrar en audit log
  await this.prisma.auditLog.create({
    data: {
      userId,
      action: 'PAYMENT_QR_GENERATED',
      entity: 'WorkOrder',
      entityId: workOrderId,
      severity: 'INFO',
      metadata: { qrToken, amount: order.finalAmount, expiresAt },
    }
  });

  // NOTA: En MVP sin Culqi real, retornar el QR token como URL de pago simulada
  // En producción: llamar a Culqi API para generar QR real
  const qrUrl = `${process.env.APP_URL}/pay/${qrToken}`;

  return {
    qrToken,
    qrUrl,
    amount: Number(order.finalAmount || order.totalCost),
    orderId: order.orderNumber,
    expiresAt,
    message: 'QR válido por 7 minutos. No compartir Yape personal.',
  };
}
```

### Backend: POST /webhooks/payment/confirm (simulado para MVP)

```typescript
// Crear: arellan-platform/src/modules/finance/payment-webhook.controller.ts

@Controller('webhooks')
export class PaymentWebhookController {
  constructor(
    private readonly financeService: FinanceService,
    private readonly redisService: RedisService,
  ) {}

  // Endpoint para confirmar pago manualmente en MVP (sustituye Culqi webhook)
  // En producción: POST /webhooks/culqi/payment con verificación HMAC
  @Post('payment/confirm')
  @Public() // Sin auth — viene del webhook de la pasarela
  async confirmPayment(@Body() body: { qrToken: string; paymentMethod: string; reference?: string }) {
    // Verificar idempotencia: ¿ya procesamos este qrToken?
    const existing = await this.redisService.get(`payment:processed:${body.qrToken}`);
    if (existing) return { status: 'already_processed' };

    // Obtener datos del QR de Redis
    const qrData = await this.redisService.get(`qr:payment:${body.qrToken}`);
    if (!qrData) throw new BadRequestException('QR expirado o inválido');

    const { workOrderId, amount } = JSON.parse(qrData);

    // Marcar como procesado (idempotencia)
    await this.redisService.setex(`payment:processed:${body.qrToken}`, 86400, 'processed');

    // Registrar el pago
    const payment = await this.financeService.recordConfirmedPayment({
      workOrderId,
      amount,
      method: body.paymentMethod || 'DYNAMIC_QR',
      reference: body.qrToken,
      isPersonalYape: false, // Siempre false — viene del sistema
    });

    // Limpiar el QR usado
    await this.redisService.del(`qr:payment:${body.qrToken}`);

    return { status: 'confirmed', payment };
  }
}
```

### Frontend: botón "Cobrar" en /orders/:id

```typescript
// En arellan-frontend-web, en la página de detalle de orden
// Buscar donde está el botón de registrar pago y agregar el flujo QR

// NUEVO FLUJO (str_replace en /orders/[id]/page.tsx):
// 1. Botón "Generar QR de Cobro" → POST /finance/qr/generate
// 2. Modal con el QR visible (usar qrUrl) + countdown de 7 minutos
// 3. Botón "Confirmar pago recibido" → POST /webhooks/payment/confirm
// 4. Al confirmar: toast verde "Pago registrado ✓", actualizar estado de orden

// El modal debe mostrar:
// - QR code (usar librería qrcode.react si está disponible, o imagen del qrUrl)
// - Monto en GRANDE: "S/ XXX.XX"
// - Countdown: "Expira en 06:45"
// - ADVERTENCIA visible: "Este es el QR oficial del taller. No usar Yape personal."
// - Método de pago: [Yape QR] [POS Tarjeta] [Efectivo] [Transferencia]
```

Agrega `qrcode.react` SOLO si no está en package.json. Verificar primero:
```bash
grep "qrcode" arellan-frontend-web/package.json
```

---

## REGLA 2 — Apertura y Cierre de Caja con Discrepancia

**Leer primero:**
```
view arellan-platform/src/modules/finance/finance.service.ts
view arellan-platform/prisma/schema.prisma | grep -A20 "CashboxSession"
```

Si el modelo `CashboxSession` no existe en el schema, agrégalo con str_replace:

```prisma
// schema.prisma — agregar modelo CashboxSession si no existe
model CashboxSession {
  id              String    @id @default(cuid())
  openedBy        String    // userId
  closedBy        String?
  openingAmount   Decimal   @db.Decimal(10, 2)
  closingAmount   Decimal?  @db.Decimal(10, 2)
  expectedAmount  Decimal?  @db.Decimal(10, 2)
  discrepancy     Decimal?  @db.Decimal(10, 2)
  status          CashboxStatus @default(OPEN)
  justification   String?   // Obligatorio si discrepancy entre S/.5 y S/.50
  openedAt        DateTime  @default(now())
  closedAt        DateTime?
  reportSentAt    DateTime?
  metadata        Json?

  @@index([status])
  @@index([openedAt])
}

enum CashboxStatus {
  OPEN
  CLOSED_NORMAL
  CLOSED_WITH_DISCREPANCY
  CLOSED_BLOCKED   // > S/.50 — requiere override del owner
}
```

Si se agrega el modelo: `npx prisma migrate dev --name "add_cashbox_session"`

### Endpoints:

```typescript
// finance.controller.ts — agregar:

@Post('cashbox/open')
@Roles('OWNER', 'MANAGER', 'FINANCE')
@ApiOperation({ summary: 'Apertura de caja — registrar monto inicial en efectivo' })
async openCashbox(@Body() body: { openingAmount: number }, @Request() req: any) {
  return this.financeService.openCashbox(body.openingAmount, req.user.id);
}

@Post('cashbox/close')
@Roles('OWNER', 'MANAGER', 'FINANCE')
@ApiOperation({ summary: 'Cierre de caja — declarar efectivo físico final' })
async closeCashbox(
  @Body() body: { actualCash: number; justification?: string },
  @Request() req: any,
) {
  return this.financeService.closeCashbox(body.actualCash, body.justification, req.user.id);
}

@Get('cashbox/current')
@Roles('OWNER', 'MANAGER', 'FINANCE')
async getCurrentCashbox() {
  return this.financeService.getCurrentCashboxSession();
}

@Get('cashbox/history')
@Roles('OWNER', 'FINANCE')
async getCashboxHistory(@Query('limit') limit = 30) {
  return this.financeService.getCashboxHistory(+limit);
}
```

### Lógica de cierre con los 3 niveles:

```typescript
// finance.service.ts — implementar openCashbox() y closeCashbox()

async openCashbox(openingAmount: number, userId: string) {
  // Verificar que no hay una caja abierta
  const existing = await this.prisma.cashboxSession.findFirst({
    where: { status: 'OPEN' }
  });
  if (existing) throw new BadRequestException('Ya hay una caja abierta. Ciérrala primero.');

  const session = await this.prisma.cashboxSession.create({
    data: { openedBy: userId, openingAmount, status: 'OPEN' }
  });

  await this.prisma.auditLog.create({
    data: { userId, action: 'CASHBOX_OPENED', entity: 'CashboxSession',
      entityId: session.id, severity: 'INFO', metadata: { openingAmount } }
  });

  return session;
}

async closeCashbox(actualCash: number, justification: string | undefined, userId: string) {
  const session = await this.prisma.cashboxSession.findFirst({ where: { status: 'OPEN' } });
  if (!session) throw new BadRequestException('No hay caja abierta');

  // Calcular el monto esperado: apertura + ingresos del día - gastos aprobados
  const [ingresos, gastos] = await Promise.all([
    this.prisma.payment.aggregate({
      where: { paidAt: { gte: session.openedAt } },
      _sum: { amount: true }
    }),
    this.prisma.expense.aggregate({
      where: { expenseDate: { gte: session.openedAt }, approvalStatus: 'APPROVED', paymentMethod: 'CASH' },
      _sum: { amount: true }
    })
  ]);

  const totalIngresos = Number(ingresos._sum.amount ?? 0);
  const totalGastos = Number(gastos._sum.amount ?? 0);
  const expectedAmount = Number(session.openingAmount) + totalIngresos - totalGastos;
  const discrepancy = actualCash - expectedAmount;
  const absDiscrepancy = Math.abs(discrepancy);

  // NIVEL 1: < S/.5 — Cierre normal
  if (absDiscrepancy < 5) {
    const closed = await this.prisma.cashboxSession.update({
      where: { id: session.id },
      data: { closedBy: userId, closingAmount: actualCash, expectedAmount,
        discrepancy, status: 'CLOSED_NORMAL', closedAt: new Date() }
    });
    await this.triggerDailyCashboxReport(closed);
    return { status: 'CLOSED_NORMAL', discrepancy, message: 'Cierre sin diferencias' };
  }

  // NIVEL 2: S/.5 a S/.50 — Requiere justificación
  if (absDiscrepancy >= 5 && absDiscrepancy <= 50) {
    if (!justification || justification.trim().length < 10) {
      throw new BadRequestException(
        `Diferencia de S/.${absDiscrepancy.toFixed(2)} detectada. Debes ingresar una justificación (mínimo 10 caracteres).`
      );
    }
    const closed = await this.prisma.cashboxSession.update({
      where: { id: session.id },
      data: { closedBy: userId, closingAmount: actualCash, expectedAmount, discrepancy,
        status: 'CLOSED_WITH_DISCREPANCY', justification, closedAt: new Date() }
    });
    // Notificar a owners (informativo)
    this.realtimeGateway?.emitSecurityAlert({
      type: 'CASHBOX_DISCREPANCY_MINOR',
      description: `Cierre de caja con diferencia S/.${discrepancy.toFixed(2)}. Justificación: ${justification}`,
      severity: 'WARNING',
    });
    await this.triggerDailyCashboxReport(closed);
    return { status: 'CLOSED_WITH_DISCREPANCY', discrepancy };
  }

  // NIVEL 3: > S/.50 — BLOQUEADO — Alerta forense P1
  // Registrar el intento de cierre en audit log ANTES del bloqueo
  await this.prisma.auditLog.create({
    data: {
      userId, action: 'CASHBOX_CLOSE_BLOCKED', entity: 'CashboxSession',
      entityId: session.id, severity: 'SECURITY_ALERT',
      metadata: {
        actualCash, expectedAmount, discrepancy,
        timestamp: new Date().toISOString(),
        // El timestamp exacto sirve para cruzar con cámaras de seguridad
        note: 'Cierre bloqueado por diferencia > S/.50. Revisar cámaras de seguridad en esta franja horaria.',
      }
    }
  });
  // Alerta P1 a owners — enviar a todos los OWNER
  this.realtimeGateway?.emitSecurityAlert({
    type: 'CASHBOX_DISCREPANCY_CRITICAL',
    description: `🚨 Cierre de caja BLOQUEADO. Diferencia: S/.${Math.abs(discrepancy).toFixed(2)}. Revisar cámaras.`,
    severity: 'CRITICAL',
  });
  throw new HttpException(
    {
      statusCode: 422,
      error: 'CASHBOX_DISCREPANCY_CRITICAL',
      message: `Diferencia de S/.${Math.abs(discrepancy).toFixed(2)} supera S/.50. Cierre bloqueado. Se notificó a los propietarios. Se requiere override de OWNER.`,
      discrepancy,
      expectedAmount,
      actualCash,
    },
    HttpStatus.UNPROCESSABLE_ENTITY
  );
}

private async triggerDailyCashboxReport(session: any) {
  // Encolar reporte diario para envío a owners (implementado en Workers)
  // Por ahora: registrar en audit log que el reporte debe enviarse
  await this.prisma.auditLog.create({
    data: {
      action: 'CASHBOX_REPORT_QUEUED', entity: 'CashboxSession',
      entityId: session.id, severity: 'INFO',
      metadata: { queuedAt: new Date() }
    }
  });
}
```

---

## REGLA 3 — Audit Log: Inmutabilidad PostgreSQL

**Esta es la única regla que requiere modificar directamente la base de datos.**

Crear una nueva migración:

```bash
# Crear el archivo de migración manualmente:
# arellan-platform/prisma/migrations/20260605_audit_log_immutability/migration.sql
```

Contenido del archivo `migration.sql`:

```sql
-- Reglas PostgreSQL que previenen UPDATE y DELETE en audit_logs
-- Incluso con acceso directo a la BD, estas reglas aplican
-- Basado en: arellan_backend_api.md y arellan_e2e_tests.md

CREATE RULE no_update_audit_logs AS
  ON UPDATE TO "AuditLog"
  DO INSTEAD NOTHING;

CREATE RULE no_delete_audit_logs AS
  ON DELETE TO "AuditLog"
  DO INSTEAD NOTHING;

-- Verificar que las reglas se crearon correctamente:
-- SELECT rulename FROM pg_rules WHERE tablename = 'AuditLog';
```

Para aplicar manualmente (ya que es SQL raw, no Prisma):

```bash
cd arellan-platform
npx prisma db execute --file ./prisma/migrations/20260605_audit_log_immutability/migration.sql
```

Verificar:
```bash
npx prisma db execute --stdin << 'EOF'
SELECT rulename, event, definition FROM pg_rules WHERE tablename = 'AuditLog';
EOF
# ESPERADO: 2 filas — no_update_audit_logs y no_delete_audit_logs
```

---

## REGLA 4 — RBAC Data Masking para Mecánicos

**Leer primero:**
```
view arellan-platform/src/common/interceptors/audit.interceptor.ts
view arellan-platform/src/app.module.ts
```

Crear `arellan-platform/src/common/interceptors/data-masking.interceptor.ts`:

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

// Campos sensibles del cliente que mecánicos NO deben ver
const MASKED_FIELDS_FOR_MECHANIC = [
  'phone', 'email', 'address', 'dni', 'ruc', 'phone2',
  'creditLimit', 'creditBalance',
];

const MECHANIC_ROLES = ['MECHANIC', 'APPRENTICE'];

function maskObject(obj: any): any {
  if (!obj || typeof obj !== 'object') return obj;
  if (Array.isArray(obj)) return obj.map(maskObject);

  const masked = { ...obj };
  // Si el objeto tiene campos de cliente, enmascararlos
  if ('client' in masked && masked.client) {
    masked.client = { ...masked.client };
    MASKED_FIELDS_FOR_MECHANIC.forEach(field => {
      if (field in masked.client) delete masked.client[field];
    });
  }
  // Recursivo para objetos anidados
  Object.keys(masked).forEach(key => {
    if (masked[key] && typeof masked[key] === 'object') {
      masked[key] = maskObject(masked[key]);
    }
  });
  return masked;
}

@Injectable()
export class DataMaskingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const userRole = request.user?.role;

    if (!MECHANIC_ROLES.includes(userRole)) {
      return next.handle(); // Roles no mecánicos ven todo
    }

    return next.handle().pipe(
      map(data => maskObject(data))
    );
  }
}
```

Registrar globalmente en `app.module.ts` con str_replace:

```typescript
// En providers del AppModule, agregar:
{
  provide: APP_INTERCEPTOR,
  useClass: DataMaskingInterceptor,
},
```

---

## REGLA 5 — Inventario: HTTP 422 sin OT activa

**Leer primero:**
```
view arellan-platform/src/modules/inventory/inventory.service.ts
```

En el método que registra salidas de inventario (OUT_WORK_ORDER), agregar la validación:

```typescript
// inventory.service.ts — en el método que crea movimientos de salida
// Busca el método adjustStock(), createMovement() o similar

// Agregar ANTES de procesar cualquier salida de inventario:
async validateWorkOrderForInventoryOut(workOrderId: string | undefined) {
  if (!workOrderId) {
    throw new HttpException(
      {
        statusCode: 422,
        error: 'INVENTORY_NO_ACTIVE_ORDER',
        message: 'No se puede retirar inventario sin una Orden de Trabajo activa vinculada.',
        hint: 'Crea o selecciona una OT antes de solicitar repuestos.',
      },
      HttpStatus.UNPROCESSABLE_ENTITY
    );
  }

  const order = await this.prisma.workOrder.findUnique({
    where: { id: workOrderId },
    select: { id: true, status: true, orderNumber: true }
  });

  if (!order) throw new NotFoundException(`OT ${workOrderId} no encontrada`);

  // Estados que permiten salida de inventario
  const ALLOWED_STATUSES = ['RECEIVED', 'DIAGNOSING', 'IN_PROGRESS', 'WAITING_PARTS', 'QA'];
  if (!ALLOWED_STATUSES.includes(order.status)) {
    throw new HttpException(
      {
        statusCode: 422,
        error: 'INVENTORY_ORDER_INVALID_STATUS',
        message: `La OT ${order.orderNumber} está en estado ${order.status} y no permite salida de inventario.`,
      },
      HttpStatus.UNPROCESSABLE_ENTITY
    );
  }

  return order;
}
```

Llama a `validateWorkOrderForInventoryOut(dto.workOrderId)` en CUALQUIER endpoint que procese una salida de inventario (`OUT_WORK_ORDER`, `RESERVATION`).

---

## REGLA 6 — BullMQ Workers (5 workers)

**Leer primero:**
```
view arellan-platform/src/app.module.ts
cat arellan-platform/package.json | grep "bullmq\|bull"
```

Verificar si BullMQ está instalado. Si no:
```bash
cd arellan-platform && npm install bullmq @nestjs/bull
```

Crear `arellan-platform/src/workers/workers.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { ScheduleModule } from '@nestjs/schedule';
import { InventoryAlertWorker } from './inventory-alert.worker';
import { CashboxReportWorker } from './cashbox-report.worker';
import { AuditAnomalyWorker } from './audit-anomaly.worker';
import { BackupWorker } from './backup.worker';
import { MonthlyDiscrepancyWorker } from './monthly-discrepancy.worker';

@Module({
  imports: [
    ScheduleModule.forRoot(),
    BullModule.registerQueue(
      { name: 'notifications' },
      { name: 'reports' },
      { name: 'inventory-alerts' },
      { name: 'audit-exports' },
    ),
  ],
  providers: [
    InventoryAlertWorker,
    CashboxReportWorker,
    AuditAnomalyWorker,
    BackupWorker,
    MonthlyDiscrepancyWorker,
  ],
  exports: [BullModule],
})
export class WorkersModule {}
```

Crear cada worker en `arellan-platform/src/workers/`:

### Worker 1: InventoryAlertWorker (cada 6 horas)

```typescript
// src/workers/inventory-alert.worker.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { PrismaService } from '../common/prisma/prisma.service';
import { RealtimeGateway } from '../common/gateway/realtime.gateway';

@Injectable()
export class InventoryAlertWorker {
  private readonly logger = new Logger(InventoryAlertWorker.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly realtimeGateway: RealtimeGateway,
  ) {}

  @Cron('0 */6 * * *') // Cada 6 horas
  async checkInventoryLevels() {
    this.logger.log('Checking inventory levels...');
    const criticalItems = await this.prisma.inventoryItem.findMany({
      where: { isActive: true, currentStock: { lte: this.prisma.inventoryItem.fields.minStock } }
    });
    // Alternativa si la sintaxis anterior no funciona:
    // where: { isActive: true }
    // luego filtrar: criticalItems.filter(i => i.currentStock <= i.minStock)

    for (const item of criticalItems) {
      if (item.currentStock <= item.minStock) {
        this.realtimeGateway.emitInventoryLowStock({
          itemId: item.id,
          itemName: item.name,
          currentStock: item.currentStock,
          minStock: item.minStock,
        });
        this.logger.warn(`Stock crítico: ${item.name} (${item.currentStock}/${item.minStock})`);
      }
    }
    this.logger.log(`Inventory check complete. ${criticalItems.length} critical items.`);
  }
}
```

### Worker 2: CashboxReportWorker (Lunes-Sábado 8PM)

```typescript
// src/workers/cashbox-report.worker.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { PrismaService } from '../common/prisma/prisma.service';
import { RealtimeGateway } from '../common/gateway/realtime.gateway';

@Injectable()
export class CashboxReportWorker {
  private readonly logger = new Logger(CashboxReportWorker.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly realtimeGateway: RealtimeGateway,
  ) {}

  @Cron('0 20 * * 1-6') // Lunes a sábado a las 8 PM
  async sendDailyCashboxReport() {
    this.logger.log('Generating daily cashbox report...');
    const today = new Date();
    today.setHours(0, 0, 0, 0);

    const [payments, expenses, session, orders] = await Promise.all([
      this.prisma.payment.aggregate({
        where: { paidAt: { gte: today } },
        _sum: { amount: true }, _count: true
      }),
      this.prisma.expense.aggregate({
        where: { expenseDate: { gte: today }, approvalStatus: 'APPROVED' },
        _sum: { amount: true }
      }),
      this.prisma.cashboxSession.findFirst({
        where: { openedAt: { gte: today } },
        orderBy: { openedAt: 'desc' }
      }),
      this.prisma.workOrder.count({ where: { status: { in: ['IN_PROGRESS', 'COMPLETED'] } } })
    ]);

    const report = {
      date: today.toISOString().split('T')[0],
      ingresos: Number(payments._sum.amount ?? 0),
      egresos: Number(expenses._sum.amount ?? 0),
      saldoNeto: Number(payments._sum.amount ?? 0) - Number(expenses._sum.amount ?? 0),
      transacciones: payments._count,
      otsActivas: orders,
      discrepancia: session?.discrepancy ?? 0,
      estadoCaja: session?.status ?? 'NO_SESSION',
    };

    // Emitir a owners via WebSocket
    this.realtimeGateway.emitSecurityAlert({
      type: 'CASHBOX_DAILY_REPORT',
      description: `Reporte de caja: Ingresos S/.${report.ingresos.toFixed(2)} | Neto S/.${report.saldoNeto.toFixed(2)}`,
      severity: 'INFO',
    });

    // Guardar en audit log
    await this.prisma.auditLog.create({
      data: {
        action: 'CASHBOX_DAILY_REPORT_SENT', entity: 'CashboxSession',
        severity: 'INFO', metadata: report
      }
    });

    this.logger.log(`Daily cashbox report sent. Ingresos: S/.${report.ingresos}`);
  }
}
```

### Worker 3: AuditAnomalyWorker (cada hora)

```typescript
// src/workers/audit-anomaly.worker.ts
@Injectable()
export class AuditAnomalyWorker {
  @Cron('0 * * * *') // Cada hora
  async detectAnomalies() {
    const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);

    // 1. Detectar accesos financieros fuera de horario laboral (antes de 7AM o después de 9PM)
    const hour = new Date().getHours();
    if (hour < 7 || hour > 21) {
      const offHoursAccess = await this.prisma.auditLog.findMany({
        where: {
          createdAt: { gte: oneHourAgo },
          entity: { in: ['Finance', 'CashboxSession', 'Payment', 'Invoice'] },
        },
        include: { user: { select: { email: true, role: true } } }
      });
      if (offHoursAccess.length > 0) {
        this.realtimeGateway.emitSecurityAlert({
          type: 'OFF_HOURS_FINANCE_ACCESS',
          description: `${offHoursAccess.length} acceso(s) al módulo financiero fuera de horario laboral`,
          severity: 'WARNING',
        });
      }
    }

    // 2. Detectar múltiples cambios financieros rápidos (> 5 en 5 minutos)
    const fiveMinAgo = new Date(Date.now() - 5 * 60 * 1000);
    const rapidChanges = await this.prisma.auditLog.groupBy({
      by: ['userId'],
      where: {
        createdAt: { gte: fiveMinAgo },
        action: { in: ['PAYMENT_CREATED', 'EXPENSE_CREATED', 'CASHBOX_CLOSED'] }
      },
      _count: true,
      having: { userId: { _count: { gt: 5 } } }
    });
    for (const change of rapidChanges) {
      this.realtimeGateway.emitSecurityAlert({
        type: 'RAPID_FINANCE_CHANGES',
        description: `Usuario ${change.userId} realizó ${change._count} cambios financieros en 5 minutos`,
        severity: 'SECURITY_ALERT',
        userId: change.userId,
      });
    }
  }
}
```

### Worker 4: BackupWorker (2AM diario)

```typescript
// src/workers/backup.worker.ts
@Injectable()
export class BackupWorker {
  @Cron('0 2 * * *') // 2AM todos los días
  async dailyBackup() {
    this.logger.log('Daily backup job triggered');
    // En MVP: Supabase/Railway manejan backups automáticamente
    // Registrar el trigger en audit log como evidencia
    await this.prisma.auditLog.create({
      data: {
        action: 'BACKUP_TRIGGERED', entity: 'System',
        severity: 'INFO',
        metadata: { timestamp: new Date().toISOString(), type: 'DAILY_AUTOMATIC' }
      }
    });
    this.logger.log('Backup log registered. Cloud provider handles actual backup.');
  }
}
```

### Worker 5: MonthlyDiscrepancyWorker (día 1 de cada mes, 9AM)

```typescript
// src/workers/monthly-discrepancy.worker.ts
@Injectable()
export class MonthlyDiscrepancyWorker {
  @Cron('0 9 1 * *') // Primer día de cada mes a las 9AM
  async monthlyInventoryAudit() {
    const lastMonth = new Date();
    lastMonth.setMonth(lastMonth.getMonth() - 1);
    lastMonth.setDate(1);
    lastMonth.setHours(0, 0, 0, 0);
    const thisMonthStart = new Date();
    thisMonthStart.setDate(1);
    thisMonthStart.setHours(0, 0, 0, 0);

    const movements = await this.prisma.stockMovement.groupBy({
      by: ['itemId', 'type'],
      where: { createdAt: { gte: lastMonth, lt: thisMonthStart } },
      _sum: { quantity: true }
    });

    await this.prisma.auditLog.create({
      data: {
        action: 'MONTHLY_INVENTORY_AUDIT', entity: 'Inventory',
        severity: 'INFO',
        metadata: { month: lastMonth.toISOString().slice(0, 7), itemsAudited: movements.length }
      }
    });

    this.logger.log(`Monthly inventory audit: ${movements.length} items audited`);
  }
}
```

**Registrar WorkersModule en app.module.ts:**
```typescript
// str_replace en app.module.ts para agregar WorkersModule a imports
```

---

## REGLA 7 — Máquina de Estados de Gastos con Botón Bloqueado

**Leer primero:**
```
view arellan-frontend-web/src/app/(admin)/finance/page.tsx
view arellan-platform/src/modules/finance/finance.service.ts
```

### Backend: nuevos estados y endpoint DISBURSED

```typescript
// En finance.service.ts — agregar endpoint de desembolso
async disburseExpense(expenseId: string, userId: string) {
  const expense = await this.prisma.expense.findUnique({ where: { id: expenseId } });
  if (!expense) throw new NotFoundException('Gasto no encontrado');
  if (expense.approvalStatus !== 'APPROVED') {
    throw new BadRequestException('Solo se pueden desembolsar gastos APROBADOS');
  }

  return this.prisma.expense.update({
    where: { id: expenseId },
    data: {
      approvalStatus: 'DISBURSED',
      // disbursedAt: new Date(), // agregar al schema si no existe
      updatedBy: userId,
    }
  });
}

// Endpoint POST /finance/expenses/:id/disburse
```

### Frontend: botón de pago deshabilitado

```typescript
// En finance/page.tsx, en la lista de gastos pendientes,
// agregar con str_replace el condicional en el botón "Pagar":

// El botón "Pagar" / "Desembolsar" debe estar:
// - DESHABILITADO si approvalStatus === 'PENDING' o 'REGISTERED'
// - HABILITADO solo si approvalStatus === 'APPROVED'
// - TEXTO diferente por estado: "Pendiente de aprobación" | "Pagar" | "Pagado" | "Verificar"

// Visualización del estado en la tabla:
// REGISTERED → badge gris "Registrado"
// PENDING → badge naranja "Pendiente aprobación" + botón Pagar DISABLED
// APPROVED → badge verde "Aprobado" + botón Pagar ENABLED (color success)
// DISBURSED → badge azul "Desembolsado" + botón "Adjuntar comprobante"
// VERIFIED → badge teal "Verificado SUNAT" (Fase 2)
```

---

## REGLA 8 — Recepción de Vehículo: 5 Fotos Obligatorias + SHA-256

**Leer primero:**
```
view arellan-mechanic-ui/src/pages/VehicleIntakePage.tsx
view arellan-platform/src/modules/vehicles/vehicles.service.ts
```

### Backend: endpoint de fotos con SHA-256

```typescript
// vehicles.controller.ts — agregar:
@Post('intake/photos')
@Roles('OWNER', 'MANAGER', 'MECHANIC', 'APPRENTICE')
@UseInterceptors(FileFieldsInterceptor([
  { name: 'front', maxCount: 1 },
  { name: 'rear', maxCount: 1 },
  { name: 'left', maxCount: 1 },
  { name: 'right', maxCount: 1 },
  { name: 'dashboard', maxCount: 1 },
]))
@ApiOperation({ summary: 'Subir 5 fotos obligatorias de recepción con hash SHA-256 inmutable' })
async uploadIntakePhotos(
  @UploadedFiles() files: { front?, rear?, left?, right?, dashboard? },
  @Body() body: { workOrderId: string },
) {
  const angles = ['front', 'rear', 'left', 'right', 'dashboard'];
  const missing = angles.filter(angle => !files[angle]?.[0]);
  if (missing.length > 0) {
    throw new BadRequestException(
      `Faltan fotos obligatorias: ${missing.join(', ')}. Se requieren las 5 fotos.`
    );
  }
  return this.vehiclesService.saveIntakePhotos(body.workOrderId, files);
}
```

```typescript
// vehicles.service.ts — método saveIntakePhotos()
async saveIntakePhotos(workOrderId: string, files: any) {
  const crypto = require('crypto');
  const photos = [];

  for (const [angle, fileArray] of Object.entries(files)) {
    const file = (fileArray as any)[0];
    const buffer = file.buffer;

    // Calcular SHA-256 del archivo original
    const sha256Hash = crypto.createHash('sha256').update(buffer).digest('hex');

    // En MVP: guardar como base64 en BD o simular URL de S3
    // En producción: subir a AWS S3 y guardar la URL
    const s3Key = `vehicles/${workOrderId}/${angle}-${Date.now()}.jpg`;

    photos.push({
      workOrderId,
      angle: angle.toUpperCase() as any,
      s3Key,
      sha256Hash, // INMUTABLE — si alguien reemplaza la foto, el hash no coincidirá
    });
  }

  // Guardar fotos en BD
  // Usar el modelo WorkOrderPhoto o similar que exista en el schema
  // Si no existe: crear en schema.prisma → migrar
  return { success: true, photos: photos.length, message: '5 fotos de recepción guardadas con hash SHA-256' };
}
```

### Frontend mechanic-ui: forzar las 5 fotos

```typescript
// En VehicleIntakePage.tsx, verificar que el botón "Continuar"
// está DESHABILITADO si faltan fotos.
// Si actualmente permite continuar sin todas las fotos → str_replace para bloquearlo:

// const allPhotosRequired = ['front', 'rear', 'left', 'right', 'dashboard'];
// const canContinue = allPhotosRequired.every(angle => capturedPhotos[angle]);
// El botón "Continuar": disabled={!canContinue}
// Mensaje de error si falta alguna: "Faltan fotos: [lista de ángulos pendientes]"
```

---

## REGLA 9 — MFA TOTP Obligatorio para OWNER/ADMIN/FINANCE

**Leer primero:**
```
view arellan-platform/src/modules/auth/auth.service.ts
view arellan-platform/src/modules/auth/auth.controller.ts
view arellan-platform/src/common/guards/mfa-required.guard.ts
```

Si el guard de MFA existe pero no está implementado, completarlo:

```typescript
// auth.service.ts — métodos de MFA:

async setupMFA(userId: string) {
  const { authenticator } = await import('otplib');
  const secret = authenticator.generateSecret();
  const user = await this.prisma.user.findUnique({ where: { id: userId } });
  const otpAuthUrl = authenticator.keyuri(user.email, 'Arellan Autos', secret);

  // Guardar secret (AES-256 en producción, plain en MVP)
  await this.prisma.user.update({
    where: { id: userId },
    data: { mfaSecret: secret } // Idealmente cifrado con AES-256
  });

  return { secret, otpAuthUrl, qrCodeUrl: `https://chart.googleapis.com/chart?chs=200x200&chld=M|0&cht=qr&chl=${encodeURIComponent(otpAuthUrl)}` };
}

async verifyMFACode(userId: string, code: string): Promise<boolean> {
  const { authenticator } = await import('otplib');
  const user = await this.prisma.user.findUnique({ where: { id: userId }, select: { mfaSecret: true } });
  if (!user?.mfaSecret) return false;
  return authenticator.verify({ token: code, secret: user.mfaSecret });
}

// En login(): si el rol requiere MFA y mfaEnabled = true
// → NO retornar tokens aún
// → retornar { mfaPending: true, userId }
// El cliente llama a POST /auth/mfa/verify con userId + totpCode
// → si válido: generar y retornar tokens
```

```typescript
// auth.controller.ts — agregar:
@Post('mfa/setup')
@UseGuards(JwtAuthGuard)
async setupMFA(@Request() req: any) {
  return this.authService.setupMFA(req.user.id);
}

@Post('mfa/verify')
@Public()
async verifyMFA(@Body() body: { userId: string; totpCode: string }) {
  const isValid = await this.authService.verifyMFACode(body.userId, body.totpCode);
  if (!isValid) throw new UnauthorizedException('Código MFA inválido');
  return this.authService.generateTokensForUser(body.userId);
}
```

Instalar otplib si no está:
```bash
cd arellan-platform
grep "otplib" package.json || npm install otplib
```

---

## VERIFICACIÓN FINAL — Reglas 1-9

```bash
BASE="http://localhost:3001/api/v1"
TOKEN=$(curl -s -X POST "$BASE/auth/login" -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

echo "=== REGLA 1: QR Dinámico ==="
ORDER_ID=$(curl -s "$BASE/orders?limit=1" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data'][0]['id'])" 2>/dev/null)
QR=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$BASE/finance/qr/generate" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"workOrderId\":\"$ORDER_ID\"}")
[ "$QR" = "200" ] || [ "$QR" = "201" ] && echo "✅ QR generado" || echo "❌ QR → HTTP $QR"

echo "=== REGLA 2: Apertura de Caja ==="
CAJA=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$BASE/finance/cashbox/open" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"openingAmount": 500}')
[ "$CAJA" = "200" ] || [ "$CAJA" = "201" ] && echo "✅ Caja abierta" || echo "❌ Caja → HTTP $CAJA"

echo "=== REGLA 3: Audit Log Inmutable ==="
RULES=$(curl -s -X POST "$BASE/audit/test-immutability" -H "Authorization: Bearer $TOKEN" 2>/dev/null)
# Alternativa directa:
npx prisma db execute --stdin << 'EOF' 2>/dev/null || true
SELECT rulename FROM pg_rules WHERE tablename = 'AuditLog';
EOF

echo "=== REGLA 4: Data Masking ==="
TOKEN_MECH=$(curl -s -X POST "$BASE/auth/login" -H "Content-Type: application/json" \
  -d '{"email":"mecanico1@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
MASKED=$(curl -s "$BASE/orders?limit=1" -H "Authorization: Bearer $TOKEN_MECH")
echo $MASKED | python3 -c "
import sys,json
d=json.load(sys.stdin)
items=d.get('data',d) if isinstance(d,dict) else d
if items:
  o=items[0]
  c=o.get('client',{})
  if not c.get('phone') and not c.get('email'):
    print('✅ Data masking activo (phone/email ocultos para mecánico)')
  else:
    print('❌ Data masking FALLA — mecánico ve datos del cliente')
" 2>/dev/null

echo "=== REGLA 5: 422 sin OT activa ==="
ITEM_ID=$(curl -s "$BASE/inventory?limit=1" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data'][0]['id'])" 2>/dev/null)
NO_OT=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$BASE/inventory/$ITEM_ID/reserve" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"quantity": 1}') # Sin workOrderId
[ "$NO_OT" = "422" ] && echo "✅ HTTP 422 INVENTORY_NO_ACTIVE_ORDER" || echo "❌ → HTTP $NO_OT (debe ser 422)"

echo "=== REGLA 6: Workers BullMQ ==="
grep -r "InventoryAlertWorker\|CashboxReportWorker\|AuditAnomalyWorker" \
  arellan-platform/src/ --include="*.ts" -l | head -5
echo "Workers encontrados en los archivos anteriores"

echo "=== REGLA 7: Gasto PENDING → botón bloqueado ==="
EXPENSE=$(curl -s -X POST "$BASE/finance/expenses" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"category":"FOOD","description":"Test botón bloqueado","amount":300,"paymentMethod":"CASH"}')
STATUS=$(echo $EXPENSE | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('approvalStatus','?'))" 2>/dev/null)
[ "$STATUS" = "PENDING" ] || [ "$STATUS" = "PENDING_APPROVAL" ] && \
  echo "✅ Gasto S/300 en PENDING (botón pagar debe estar disabled)" || \
  echo "❌ approvalStatus=$STATUS (debe ser PENDING)"

echo "=== REGLA 9: MFA setup disponible ==="
MFA=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$BASE/auth/mfa/setup" \
  -H "Authorization: Bearer $TOKEN")
[ "$MFA" = "200" ] || [ "$MFA" = "201" ] && echo "✅ POST /auth/mfa/setup disponible" || echo "❌ → HTTP $MFA"
```

---

## REPORTE FINAL REQUERIDO

```
═══════════════════════════════════════════════════════════
FASE 7 — REGLAS ANTI-FRAUDE IMPLEMENTADAS
═══════════════════════════════════════════════════════════
REGLA 1: QR Dinámico de cobro ............. [✅ / ❌]
REGLA 2: Apertura/cierre caja + 3 niveles . [✅ / ❌]
REGLA 3: Audit log PostgreSQL inmutable .... [✅ 2 reglas / ❌]
REGLA 4: Data masking mecánicos ........... [✅ / ❌]
REGLA 5: HTTP 422 sin OT activa ........... [✅ / ❌]
REGLA 6: 5 BullMQ Workers registrados ..... [✅ N/5 / ❌]
REGLA 7: Estado gastos + botón bloqueado .. [✅ / ❌]
REGLA 8: 5 fotos SHA-256 en recepción ..... [✅ / ❌]
REGLA 9: MFA TOTP setup/verify ............ [✅ / ❌]

MIGRACIÓN NUEVA:
  audit_log_immutability ................. [✅ aplicada / ❌]

SISTEMA PUEDE DETENER EL FRAUDE ACTIVO: [SÍ / NO — falta: X]
═══════════════════════════════════════════════════════════
```

**Empieza por la Regla 1 (QR). Es la que más impacto tiene en el negocio real.**
