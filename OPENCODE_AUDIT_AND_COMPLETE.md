# OPENCODE — AUDITORÍA Y COMPLETADO DEL ECOSISTEMA ARELLAN
# Fase 2: Verificar lo hecho, implementar lo que falta

---

## MISIÓN

Antes de escribir una sola línea de código, debes leer y auditar **cada archivo relevante** del proyecto. Tu trabajo en esta sesión tiene dos fases secuenciales e inamovibles:

**FASE 1 → AUDITORÍA:** Leer archivos reales, verificar contra el checklist y reportar el estado exacto de cada ítem.
**FASE 2 → IMPLEMENTACIÓN:** Todo ítem marcado como ❌ o ⚠️ debe ser implementado o completado, en el orden de prioridad establecido.

**Regla absoluta:** No saltes a la Fase 2 sin terminar el reporte de la Fase 1. No asumas que algo está hecho sin leerlo.

---

## FASE 1 — AUDITORÍA COMPLETA

### INSTRUCCIONES DE LECTURA

Lee los siguientes archivos en este orden exacto. Para cada uno, reporta: ruta leída, líneas totales, y hallazgo crítico en una línea.

---

### BLOQUE A — BASE DE DATOS

```
LEER: arellan-platform/prisma/schema.prisma
VERIFICAR:
  [ ] Modelo User      — tiene campos: mfaEnabled, failedAttempts, lockedUntil, deletedAt
  [ ] Modelo Session   — existe con campos: token, deviceInfo, ipAddress, revokedAt
  [ ] Modelo Personnel — tiene campos: dni, nationality, contractType, salary, salaryType
  [ ] Modelo Attendance — existe con enum AttendanceType
  [ ] Modelo VehicleUsage — existe con enum UsageStatus (incluye UNAUTHORIZED)
  [ ] Modelo WorkOrder — tiene campos: paymentYape, paymentReceivedBy
  [ ] Modelo Payment   — tiene campos: isPersonalYape, yapeAccount
  [ ] Modelo Approval  — existe con relación a User (requestedBy, approvedBy)
  [ ] Modelo Commission — existe con relación a Personnel y Supplier
  [ ] Modelo AuditLog  — tiene enum AuditSeverity con valor SECURITY_ALERT
  [ ] Modelo Notification — existe
  [ ] Modelo Setting   — existe
  RESULTADO ESPERADO: 32 modelos mínimo

LEER: arellan-platform/prisma/seed.ts
VERIFICAR:
  [ ] Usuarios: 7 registros con bcrypt hash real (no texto plano)
  [ ] Roles cubiertos: OWNER x2, MANAGER x1, FINANCE x1, MECHANIC x2, APPRENTICE x1
  [ ] Clientes: 10+ registros con formato peruano (DNI 8 dígitos, RUC 11 dígitos)
  [ ] Vehículos: 20+ con placas formato peruano (ABC-123 o AB-1234)
  [ ] Inventario: 30+ ítems, al menos 4 con currentStock < minStock
  [ ] Órdenes de trabajo: 15+ en estados distintos (RECEIVED, IN_PROGRESS, COMPLETED, etc.)
  [ ] Personal: 7 registros con datos de Personnel vinculados a Users
  [ ] Gastos/Expenses: 10+ registros con distintas categorías
  [ ] Notificaciones: al menos 8 no leídas
  [ ] Configuraciones (Setting): al menos 10 claves del sistema

LEER: arellan-platform/prisma/migrations/ (listar carpetas)
VERIFICAR:
  [ ] Existe migración 20260602003437_init (o similar)
  [ ] Existe migración posterior para el schema extendido
  [ ] No hay migraciones en estado de error (migration_lock.toml)
```

---

### BLOQUE B — BACKEND (arellan-platform/src)

```
LEER: arellan-platform/src/app.module.ts
VERIFICAR:
  [ ] Importa AuthModule
  [ ] Importa ClientsModule
  [ ] Importa VehiclesModule
  [ ] Importa OrdersModule
  [ ] Importa InventoryModule
  [ ] Importa FinanceModule
  [ ] Importa PersonnelModule
  [ ] Importa AuditModule
  [ ] Importa AttendanceModule    ← nuevo
  [ ] Importa PurchasesModule     ← nuevo
  [ ] Importa CommissionsModule   ← nuevo
  [ ] Importa QuotesModule        ← nuevo
  [ ] Importa InvoicesModule      ← nuevo
  [ ] Importa PaymentsModule      ← nuevo
  [ ] Importa SettingsModule      ← nuevo
  [ ] Importa RealtimeGateway o RealtimeModule ← WebSockets
  [ ] ThrottlerModule configurado (rate limiting)
  [ ] BullModule o similar (si hay colas)

LEER: arellan-platform/src/modules/auth/auth.service.ts
VERIFICAR:
  [ ] Método login() retorna accessToken + refreshToken (no solo accessToken)
  [ ] Método refreshToken() existe y valida desde Redis
  [ ] Método logout() invalida token en Redis
  [ ] Método logoutAll() invalida TODAS las sesiones del usuario en Redis
  [ ] Conteo de intentos fallidos: incrementa failedAttempts en BD
  [ ] Bloqueo automático: si failedAttempts >= 5 → setea lockedUntil = now + 15min
  [ ] Revisa lockedUntil antes de autenticar

LEER: arellan-platform/src/modules/auth/auth.controller.ts
VERIFICAR:
  [ ] POST /auth/login
  [ ] POST /auth/refresh
  [ ] POST /auth/logout
  [ ] POST /auth/logout-all
  [ ] GET  /auth/me
  [ ] POST /auth/change-password
  [ ] GET  /auth/sessions
  [ ] DELETE /auth/sessions/:id

LEER: arellan-platform/src/modules/orders/orders.service.ts
VERIFICAR:
  [ ] findAll() tiene paginación: { data, meta: { total, page, limit, totalPages } }
  [ ] findByStatus(status) existe
  [ ] getSummaryStats() existe (órdenes hoy, ingresos del día, etc.)
  [ ] updateStatus() valida transiciones permitidas (máquina de estados)
  [ ] updateStatus() emite evento WebSocket 'order:status_changed'
  [ ] addItem() descuenta del inventario via StockMovement
  [ ] registerPayment() verifica si paymentMethod es YAPE y si isPersonalYape → alerta

LEER: arellan-platform/src/modules/inventory/inventory.service.ts
VERIFICAR:
  [ ] getLowStock() existe — retorna items donde currentStock < minStock
  [ ] getStockMovements() paginado con filtros
  [ ] getValuation() — retorna valor total del inventario
  [ ] adjustStock() crea StockMovement de tipo IN_ADJUSTMENT o OUT_ADJUSTMENT
  [ ] reserveForOrder() existe

LEER: arellan-platform/src/modules/finance/finance.service.ts
VERIFICAR:
  [ ] getDashboard() existe — KPIs: ingresos hoy/semana/mes, gastos, balance
  [ ] getCashflow() existe
  [ ] approveExpense() valida regla: OWNER aprueba si > S/100, dos OWNERS si > S/500
  [ ] getCommissions() existe y filtra comisiones por personal

LEER: arellan-platform/src/modules/personnel/personnel.service.ts
VERIFICAR:
  [ ] authorizeVehicleUsage() existe — crea VehicleUsage con status PENDING_RETURN
  [ ] getActiveVehicleUsages() retorna usos no devueltos
  [ ] checkOverdueVehicles() — detecta vehículos con returnAt > expectedReturn + 30min
  [ ] getAttendanceToday() existe

LEER: arellan-platform/src/common/gateway/ (o similar)
VERIFICAR si existe: realtime.gateway.ts
  [ ] Decorado con @WebSocketGateway
  [ ] Usa @WebSocketServer()
  [ ] Método emitOrderUpdated() o similar
  [ ] Manejo de salas (rooms) por userId o role
  [ ] Autenticación de socket via JWT en handshake

LEER: arellan-platform/src/common/interceptors/audit.interceptor.ts
VERIFICAR:
  [ ] Captura userId del request
  [ ] Registra action, entity, entityId, ipAddress, userAgent
  [ ] Diferencia severity según el tipo de acción
  [ ] Se aplica globalmente en app.module.ts

LEER: arellan-platform/src/main.ts
VERIFICAR:
  [ ] Puerto: 3001 (no 3000 para evitar conflicto con frontend)
  [ ] CORS habilitado con origins: localhost:3000, 3002, 3003
  [ ] ValidationPipe global activado
  [ ] Swagger DocumentBuilder configurado
  [ ] Prefix global: /api/v1
  [ ] Adaptador para WebSockets si hay Socket.io (IoAdapter)
```

---

### BLOQUE C — FRONTEND WEB (arellan-frontend-web/src)

```
LEER: arellan-frontend-web/src/lib/api-client.ts (o services/api-client.ts)
VERIFICAR:
  [ ] BASE_URL apunta a process.env.NEXT_PUBLIC_API_URL
  [ ] Adjunta Authorization: Bearer {token} en cada request
  [ ] Intercepta respuesta 401 → llama a /auth/refresh → reintenta la request original
  [ ] Si el refresh falla → redirige a /login
  [ ] Maneja errores de red con mensaje consistente

LEER: arellan-frontend-web/src/store/auth.store.ts (o similar)
VERIFICAR:
  [ ] Zustand store (o Context) con: user, accessToken, isAuthenticated
  [ ] Función login() llama al backend y guarda tokens
  [ ] Función logout() llama a POST /auth/logout y limpia estado
  [ ] Persiste token en httpOnly cookie O localStorage (preferir cookie)
  [ ] Al recargar la página, intenta restaurar sesión via /auth/me

LEER: arellan-frontend-web/src/app/ (listar páginas existentes)
VERIFICAR que EXISTEN estos archivos:
  [ ] app/(auth)/login/page.tsx
  [ ] app/(dashboard)/page.tsx           ← dashboard principal
  [ ] app/(dashboard)/orders/page.tsx
  [ ] app/(dashboard)/orders/new/page.tsx
  [ ] app/(dashboard)/orders/[id]/page.tsx
  [ ] app/(dashboard)/clients/page.tsx
  [ ] app/(dashboard)/inventory/page.tsx
  [ ] app/(dashboard)/finance/page.tsx
  [ ] app/(dashboard)/personnel/page.tsx
  [ ] app/(dashboard)/vehicles/page.tsx

PARA CADA PÁGINA EXISTENTE, verificar:
  [ ] Usa datos del API (useQuery, fetch, SWR o similar) — NO arrays hardcodeados en el componente
  [ ] Tiene estado de loading y error
  [ ] Está protegida por autenticación (redirect si no hay token)

LEER: arellan-frontend-web/src/hooks/useRealtime.ts
VERIFICAR:
  [ ] Conecta a NEXT_PUBLIC_WS_URL via socket.io-client
  [ ] Se autenticacion en handshake con el token JWT
  [ ] Suscribe a eventos: 'order:status_changed', 'inventory:low_stock', etc.
  [ ] Limpia la conexión en useEffect cleanup
```

---

### BLOQUE D — MECHANIC UI (arellan-mechanic-ui/src)

```
LEER: arellan-mechanic-ui/ (listar estructura)
VERIFICAR:
  [ ] Existe archivo de configuración de API (apunta al backend)
  [ ] Existe página de login
  [ ] Existe página de órdenes del mecánico
  [ ] Existe mecanismo de check-in / check-out
  [ ] Las páginas llaman al backend real (no mocks)
  [ ] Diseño tablet-first (touch targets > 44px, sin hover-only interactions)
```

---

### BLOQUE E — CLIENT PORTAL (arellan-client-portal/src)

```
LEER: arellan-client-portal/ (listar estructura)
VERIFICAR:
  [ ] Existe página pública de tracking (/track/[orderNumber])
  [ ] El tracking llama a un endpoint público del backend
  [ ] Existe login de cliente
  [ ] Existe vista de historial de órdenes del cliente
  [ ] Existe vista para aprobar/rechazar cotización
```

---

### BLOQUE F — DESIGN SYSTEM (arellan-design-system)

```
LEER: arellan-design-system/ (listar estructura completa)
VERIFICAR:
  [ ] Existe package.json con name: "@arellan/ui"
  [ ] Existen componentes: Button, Input, Badge, Card, Table, Modal
  [ ] Existe tokens de color Arellan (--arellan-red: #C0392B, etc.)
  [ ] Los componentes son importables desde los otros productos
```

---

### BLOQUE G — INFRAESTRUCTURA

```
LEER: docker-compose.yml (raíz del proyecto)
VERIFICAR:
  [ ] Servicio postgres: imagen postgres:16-alpine, puerto 5432
  [ ] Servicio redis: imagen redis:7-alpine, puerto 6379
  [ ] Servicio pgadmin: puerto 5050
  [ ] healthcheck en postgres
  [ ] Volúmenes persistentes definidos

LEER: arellan-platform/.env.example
VERIFICAR variables presentes:
  [ ] DATABASE_URL
  [ ] REDIS_URL
  [ ] JWT_SECRET
  [ ] JWT_REFRESH_SECRET
  [ ] JWT_EXPIRES_IN (debe ser "15m")
  [ ] JWT_REFRESH_EXPIRES_IN (debe ser "7d")
  [ ] PORT (debe ser 3001)
  [ ] CORS_ORIGINS
  [ ] NODE_ENV
```

---

### FORMATO DEL REPORTE QUE DEBES GENERAR

Al terminar la Fase 1, presenta exactamente este formato antes de escribir código:

```
═══════════════════════════════════════════════════════
REPORTE DE AUDITORÍA — ECOSISTEMA ARELLAN HNOS
Fecha: [fecha actual]
═══════════════════════════════════════════════════════

BLOQUE A — BASE DE DATOS
  schema.prisma .......... [✅ 32 modelos / ⚠️ N modelos faltantes / ❌ no existe]
  seed.ts ............... [✅ completo / ⚠️ datos incompletos / ❌ no existe]
  Migración extendida .... [✅ existe / ❌ no existe]

BLOQUE B — BACKEND
  app.module.ts ......... [✅ N módulos / ⚠️ faltan: X,Y,Z]
  auth.service.ts ....... [✅ refresh+sesiones / ⚠️ falta: X / ❌ incompleto]
  orders.service.ts ..... [✅ / ⚠️ falta: stats,websocket / ❌]
  inventory.service.ts .. [✅ / ⚠️ falta: low-stock,valuation / ❌]
  finance.service.ts .... [✅ / ⚠️ falta: dashboard / ❌]
  personnel.service.ts .. [✅ / ⚠️ falta: vehicle-usage / ❌]
  realtime.gateway.ts ... [✅ existe / ❌ NO EXISTE]
  audit.interceptor.ts .. [✅ / ⚠️ / ❌]
  main.ts ............... [✅ puerto 3001, CORS, Swagger / ⚠️ / ❌]

BLOQUE C — FRONTEND WEB
  api-client.ts ......... [✅ JWT+refresh / ⚠️ sin refresh / ❌ no existe]
  auth.store.ts ......... [✅ / ❌ no existe]
  Páginas (N/10) ........ [✅ todas / ⚠️ existen N, faltan X]
  Páginas con datos reales [✅ / ⚠️ N con mocks / ❌]
  useRealtime.ts ........ [✅ / ❌ no existe]

BLOQUE D — MECHANIC UI
  Conexión backend ...... [✅ / ⚠️ parcial / ❌ sin conexión]
  Páginas funcionales ... [✅ N/5 / ❌]

BLOQUE E — CLIENT PORTAL
  Tracking público ...... [✅ / ❌ no existe]
  Login cliente ......... [✅ / ❌]
  Historial órdenes ..... [✅ / ❌]

BLOQUE F — DESIGN SYSTEM
  Paquete @arellan/ui ... [✅ / ⚠️ incompleto / ❌ no existe]
  Componentes base ...... [✅ N/6 / ❌]

BLOQUE G — INFRAESTRUCTURA
  docker-compose.yml .... [✅ 3 servicios / ⚠️ / ❌]
  .env.example .......... [✅ N vars / ⚠️ faltan: X / ❌]

───────────────────────────────────────────────────────
ÍTEMS PENDIENTES (a implementar en Fase 2):
  CRÍTICO: [lista]
  IMPORTANTE: [lista]
  MENOR: [lista]

TIEMPO ESTIMADO FASE 2: [N horas]
═══════════════════════════════════════════════════════
```

---

## FASE 2 — IMPLEMENTACIÓN EN ORDEN DE PRIORIDAD

Una vez entregado el reporte, procede a implementar **todo lo que marcaste como faltante o incompleto**, siguiendo exactamente este orden de prioridad. No pases al siguiente nivel sin terminar el anterior.

---

### PRIORIDAD 1 — CRÍTICO (el sistema no funciona sin esto)

#### P1.1 — Auth module completo

Si el módulo de autenticación no tiene refresh tokens y sesiones, impleméntalo así:

**`auth.service.ts`** — debe tener estos métodos funcionando:

```typescript
// login() — genera AMBOS tokens
async login(dto: LoginDto, ip: string, userAgent: string) {
  // 1. Buscar usuario por email
  // 2. Verificar si está bloqueado (lockedUntil > now)
  // 3. Comparar password con bcrypt
  // 4. Si falla: incrementar failedAttempts, si >= 5 → lockedUntil = now + 15min
  // 5. Si ok: resetear failedAttempts = 0, actualizar lastLoginAt y lastLoginIp
  // 6. Generar accessToken (JWT, 15min) y refreshToken (JWT, 7d)
  // 7. Guardar refreshToken hasheado en Redis: SET session:{userId}:{sessionId} {tokenHash} EX 604800
  // 8. Crear registro Session en BD
  // 9. Crear AuditLog con action: 'LOGIN', severity: INFO
  // 10. Retornar { accessToken, refreshToken, user: {...} }
}

// refreshToken() — renueva el access token
async refreshToken(token: string) {
  // 1. Verificar JWT del refresh token
  // 2. Buscar en Redis: GET session:{userId}:{sessionId}
  // 3. Comparar hash guardado con token recibido
  // 4. Si ok: generar nuevo accessToken
  // 5. Retornar { accessToken }
}

// logout() — invalida sesión específica
async logout(userId: string, sessionId: string) {
  // 1. DEL session:{userId}:{sessionId} en Redis
  // 2. Marcar Session como revokedAt = now en BD
  // 3. AuditLog action: 'LOGOUT'
}

// logoutAll() — invalida todas las sesiones
async logoutAll(userId: string) {
  // 1. Buscar todas las keys: SCAN session:{userId}:*
  // 2. DEL todas
  // 3. Update all Sessions: revokedAt = now
  // 4. AuditLog action: 'LOGOUT_ALL', severity: WARNING
}
```

**`auth.controller.ts`** — endpoints completos:
```typescript
@Post('login')          // público
@Post('refresh')        // público — body: { refreshToken }
@Post('logout')         // autenticado
@Post('logout-all')     // autenticado
@Get('me')             // autenticado — retorna user + personnel
@Post('change-password') // autenticado
@Get('sessions')        // autenticado — lista sesiones activas
@Delete('sessions/:id') // autenticado — revocar sesión específica
```

---

#### P1.2 — main.ts en el puerto correcto con todo configurado

```typescript
// arellan-platform/src/main.ts — debe quedar así:
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, Logger } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { IoAdapter } from '@nestjs/platform-socket.io';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const logger = new Logger('Bootstrap');

  // Prefijo global
  app.setGlobalPrefix('api/v1');

  // CORS
  app.enableCors({
    origin: process.env.CORS_ORIGINS?.split(',') ?? ['http://localhost:3000'],
    credentials: true,
  });

  // Validación global
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    transform: true,
    forbidNonWhitelisted: true,
  }));

  // WebSockets
  app.useWebSocketAdapter(new IoAdapter(app));

  // Swagger
  const config = new DocumentBuilder()
    .setTitle('Arellan Platform API')
    .setDescription('API del ecosistema digital Clínica Automotriz Arellan Hnos')
    .setVersion('1.0')
    .addBearerAuth()
    .addTag('auth', 'Autenticación y sesiones')
    .addTag('clients', 'Gestión de clientes')
    .addTag('vehicles', 'Gestión de vehículos')
    .addTag('orders', 'Órdenes de trabajo')
    .addTag('inventory', 'Control de inventario')
    .addTag('finance', 'Finanzas y aprobaciones')
    .addTag('personnel', 'Personal y asistencias')
    .addTag('attendance', 'Registro de asistencia')
    .addTag('purchases', 'Compras a proveedores')
    .addTag('commissions', 'Control de comisiones')
    .addTag('quotes', 'Cotizaciones')
    .addTag('invoices', 'Facturación')
    .addTag('payments', 'Registro de pagos')
    .addTag('settings', 'Configuración del sistema')
    .addTag('audit', 'Auditoría y seguridad')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);

  const port = process.env.PORT ?? 3001;
  await app.listen(port);
  logger.log(`Backend corriendo en http://localhost:${port}`);
  logger.log(`Swagger docs: http://localhost:${port}/api/docs`);
}
bootstrap();
```

---

#### P1.3 — WebSocket Gateway (si no existe)

Crea **`src/common/gateway/realtime.gateway.ts`**:

```typescript
import {
  WebSocketGateway, WebSocketServer,
  OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect,
  SubscribeMessage, MessageBody, ConnectedSocket,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Logger } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@WebSocketGateway({
  cors: { origin: process.env.CORS_ORIGINS?.split(','), credentials: true },
  namespace: '/',
})
export class RealtimeGateway implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;
  private logger = new Logger('RealtimeGateway');

  constructor(private jwtService: JwtService) {}

  afterInit() { this.logger.log('WebSocket gateway iniciado'); }

  async handleConnection(client: Socket) {
    try {
      const token = client.handshake.auth?.token || client.handshake.headers?.authorization?.replace('Bearer ', '');
      const payload = this.jwtService.verify(token, { secret: process.env.JWT_SECRET });
      client.data.userId = payload.sub;
      client.data.role = payload.role;
      // Unir a sala por rol para broadcasts segmentados
      client.join(`role:${payload.role}`);
      client.join(`user:${payload.sub}`);
      this.logger.log(`Cliente conectado: ${payload.sub} (${payload.role})`);
    } catch {
      client.disconnect();
    }
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Cliente desconectado: ${client.data?.userId}`);
  }

  // ── Métodos que otros servicios llaman para emitir eventos ──

  emitOrderCreated(data: { orderId: string; orderNumber: string; clientName: string; vehiclePlate: string }) {
    this.server.to('role:OWNER').to('role:MANAGER').emit('order:created', data);
  }

  emitOrderStatusChanged(data: { orderId: string; oldStatus: string; newStatus: string; updatedBy: string }) {
    this.server.emit('order:status_changed', data); // todos ven cambios de estado
  }

  emitInventoryLowStock(data: { itemId: string; itemName: string; currentStock: number; minStock: number }) {
    this.server.to('role:OWNER').to('role:MANAGER').to('role:FINANCE').emit('inventory:low_stock', data);
  }

  emitPaymentReceived(data: { workOrderId: string; amount: number; method: string; receivedBy: string; isAlert: boolean }) {
    this.server.to('role:OWNER').to('role:FINANCE').emit('payment:received', data);
  }

  emitPersonnelCheckIn(data: { personnelId: string; name: string; timestamp: Date }) {
    this.server.to('role:OWNER').to('role:MANAGER').emit('personnel:check_in', data);
  }

  emitVehicleOverdue(data: { vehicleId: string; plate: string; personnelName: string; minutesOverdue: number }) {
    this.server.to('role:OWNER').to('role:MANAGER').emit('vehicle:overdue', { ...data, severity: 'CRITICAL' });
  }

  emitApprovalRequested(data: { approvalId: string; type: string; amount: number; requestedBy: string }) {
    this.server.to('role:OWNER').emit('approval:requested', data);
  }

  emitSecurityAlert(data: { type: string; description: string; severity: string; userId?: string }) {
    this.server.to('role:OWNER').emit('alert:security', data);
  }
}
```

Crea **`src/common/gateway/realtime.module.ts`** e importa `RealtimeGateway` en `AppModule`.

---

#### P1.4 — Reglas anti-fraude en OrdersService y PaymentsService

Si no existen estas validaciones, impleméntalas:

```typescript
// En payments.service.ts — al registrar un pago:
async createPayment(dto: CreatePaymentDto, userId: string) {
  // Obtener configuración del Yape oficial del taller
  const officialYape = await this.settingsService.get('TALLER_YAPE_NUMBER');

  // Si el pago es por Yape, verificar que sea el oficial
  if (dto.method === 'YAPE') {
    const isPersonalYape = dto.yapeAccount !== officialYape;
    if (isPersonalYape) {
      // Registrar alerta de seguridad
      await this.auditService.log({
        userId,
        action: 'PAYMENT_UNAUTHORIZED_YAPE',
        entity: 'Payment',
        severity: 'SECURITY_ALERT',
        metadata: { yapeAccount: dto.yapeAccount, officialYape, workOrderId: dto.workOrderId }
      });
      // Emitir alerta via WebSocket a los dueños
      this.realtimeGateway.emitSecurityAlert({
        type: 'UNAUTHORIZED_YAPE',
        description: `Pago recibido en Yape personal: ${dto.yapeAccount}. Orden: ${dto.workOrderId}`,
        severity: 'CRITICAL',
        userId
      });
      // Guardar el pago marcado como personal (para auditoría) pero NO cancelarlo
      dto = { ...dto, isPersonalYape: true };
    }
  }

  // Crear el pago
  return this.prisma.payment.create({ data: { ...dto, receivedBy: userId } });
}

// En orders.service.ts — al aplicar descuento:
async applyDiscount(orderId: string, discount: number, userId: string) {
  const order = await this.prisma.workOrder.findUnique({ where: { id: orderId } });
  const discountPercentage = (discount / order.totalCost) * 100;

  if (discountPercentage > 20) {
    // Requiere aprobación de OWNER
    return this.approvalsService.createApproval({
      type: 'DISCOUNT',
      title: `Descuento > 20% en orden ${order.orderNumber}`,
      amount: discount,
      requestedById: userId,
    });
  }
  // Si es <= 20%, aplicar directo con log
  return this.updateDiscount(orderId, discount, userId);
}

// En finance.service.ts — al crear un gasto:
async createExpense(dto: CreateExpenseDto, userId: string) {
  const requiresApproval = dto.amount > 100; // S/ 100 = umbral mínimo

  if (dto.amount > 500) {
    // Requiere 2 OWNERs — se implementa como doble aprobación
    await this.approvalsService.createApproval({
      type: 'EXPENSE',
      title: dto.description,
      amount: dto.amount,
      requestedById: userId,
      metadata: { requiresDoubleApproval: true }
    });
  } else if (dto.amount > 100) {
    // Requiere 1 OWNER
    await this.approvalsService.createApproval({
      type: 'EXPENSE',
      title: dto.description,
      amount: dto.amount,
      requestedById: userId,
    });
  }

  return this.prisma.expense.create({
    data: { ...dto, paidBy: userId, approvalStatus: requiresApproval ? 'PENDING' : 'APPROVED' }
  });
}
```

---

### PRIORIDAD 2 — IMPORTANTE (sin esto el frontend está desconectado del backend)

#### P2.1 — Frontend Web: api-client.ts completo

Si no existe o está incompleto, crea **`arellan-frontend-web/src/lib/api-client.ts`**:

```typescript
const BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3001/api/v1';

class ApiClient {
  private getToken(): string | null {
    if (typeof window === 'undefined') return null;
    return localStorage.getItem('accessToken');
  }

  private async refreshAccessToken(): Promise<string | null> {
    const refreshToken = localStorage.getItem('refreshToken');
    if (!refreshToken) return null;

    const res = await fetch(`${BASE_URL}/auth/refresh`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken }),
    });

    if (!res.ok) {
      localStorage.removeItem('accessToken');
      localStorage.removeItem('refreshToken');
      window.location.href = '/login';
      return null;
    }

    const { accessToken } = await res.json();
    localStorage.setItem('accessToken', accessToken);
    return accessToken;
  }

  async request<T>(path: string, options: RequestInit = {}): Promise<T> {
    let token = this.getToken();

    const makeRequest = (tkn: string | null) =>
      fetch(`${BASE_URL}${path}`, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          ...(tkn ? { Authorization: `Bearer ${tkn}` } : {}),
          ...(options.headers ?? {}),
        },
      });

    let res = await makeRequest(token);

    // Token expirado → refrescar y reintentar UNA vez
    if (res.status === 401) {
      token = await this.refreshAccessToken();
      if (!token) throw new Error('Session expired');
      res = await makeRequest(token);
    }

    if (!res.ok) {
      const error = await res.json().catch(() => ({ message: 'Error de red' }));
      throw new Error(error.message ?? `HTTP ${res.status}`);
    }

    return res.json();
  }

  get<T>(path: string) { return this.request<T>(path, { method: 'GET' }); }
  post<T>(path: string, body: unknown) { return this.request<T>(path, { method: 'POST', body: JSON.stringify(body) }); }
  patch<T>(path: string, body: unknown) { return this.request<T>(path, { method: 'PATCH', body: JSON.stringify(body) }); }
  delete<T>(path: string) { return this.request<T>(path, { method: 'DELETE' }); }
}

export const api = new ApiClient();
```

---

#### P2.2 — Auth store (Zustand)

Si no existe, crea **`src/store/auth.store.ts`**:

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { api } from '@/lib/api-client';

interface User { id: string; email: string; role: string; }
interface AuthState {
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  loadSession: () => Promise<void>;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      accessToken: null,
      isAuthenticated: false,
      isLoading: false,

      login: async (email, password) => {
        set({ isLoading: true });
        const data = await api.post<{ accessToken: string; refreshToken: string; user: User }>(
          '/auth/login', { email, password }
        );
        localStorage.setItem('accessToken', data.accessToken);
        localStorage.setItem('refreshToken', data.refreshToken);
        set({ user: data.user, accessToken: data.accessToken, isAuthenticated: true, isLoading: false });
      },

      logout: async () => {
        await api.post('/auth/logout', {}).catch(() => {});
        localStorage.removeItem('accessToken');
        localStorage.removeItem('refreshToken');
        set({ user: null, accessToken: null, isAuthenticated: false });
      },

      loadSession: async () => {
        const token = localStorage.getItem('accessToken');
        if (!token) return;
        const user = await api.get<User>('/auth/me').catch(() => null);
        if (user) set({ user, isAuthenticated: true });
      },
    }),
    { name: 'arellan-auth', partialize: (s) => ({ user: s.user }) }
  )
);
```

---

#### P2.3 — Páginas del dashboard con datos reales

Para CADA página que uses mocks o arrays hardcodeados, conviértela a datos reales. El patrón base es:

```typescript
// app/(dashboard)/orders/page.tsx
'use client';
import { useEffect, useState } from 'react';
import { api } from '@/lib/api-client';

interface PaginatedOrders { data: Order[]; meta: { total: number; page: number; totalPages: number } }

export default function OrdersPage() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [meta, setMeta] = useState({ total: 0, page: 1, totalPages: 1 });
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    api.get<PaginatedOrders>('/orders?page=1&limit=20')
      .then(res => { setOrders(res.data); setMeta(res.meta); })
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <div>Cargando órdenes...</div>;
  if (error) return <div>Error: {error}</div>;
  return ( /* tu JSX con orders reales */ );
}
```

Aplica este patrón a: `/orders`, `/clients`, `/inventory`, `/finance`, `/personnel`, `/vehicles`, `/dashboard`.

---

#### P2.4 — Endpoints faltantes en los servicios del backend

Si estos endpoints no existen, créalos:

```typescript
// orders.controller.ts
@Get('stats/summary')      // KPIs del dashboard
@Get('by-status/:status')  // Lista filtrada por estado

// inventory.controller.ts
@Get('low-stock')          // Items bajo mínimo → alerta roja en dashboard
@Get('valuation')          // Valor total del inventario
@Get('movements')          // Movimientos paginados

// finance.controller.ts
@Get('dashboard')          // KPIs financieros: ingresos hoy/semana/mes
@Get('cashflow')           // Flujo de caja

// personnel.controller.ts
@Get('attendance/today')           // Lista de asistencias del día
@Post('vehicle-usage/authorize')   // Autorizar uso de vehículo del taller
@Get('vehicle-usage/active')       // Vehículos del taller actualmente en uso
@Get('vehicle-usage/overdue')      // Vehículos con devolución tardía
```

---

### PRIORIDAD 3 — MECHANIC UI (conectar al backend)

Si `arellan-mechanic-ui` no tiene conexión con el backend, implementa:

```
ARCHIVOS A CREAR:
src/lib/api-client.ts     ← mismo patrón que el frontend web
src/lib/auth.ts           ← login simplificado
src/app/login/page.tsx    ← formulario email + contraseña (sin PIN por ahora)
src/app/page.tsx          ← dashboard del mecánico: mis órdenes del día
src/app/orders/page.tsx   ← lista de órdenes asignadas a MI usuario
src/app/orders/[id]/page.tsx ← detalle con botón de cambiar estado
src/app/check-in/page.tsx ← botón grande de check-in / check-out

REGLAS TABLET-FIRST:
- Todos los botones: min-height 56px, font-size 18px
- Sin hover-only interactions (es táctil)
- Colores de estado: RECEIVED=#3498db, IN_PROGRESS=#f39c12, COMPLETED=#27ae60
- Layout de 1 columna en < 768px, 2 columnas en >= 768px
```

---

### PRIORIDAD 4 — CLIENT PORTAL (tracking público)

Si `arellan-client-portal` no tiene tracking funcional:

```
ARCHIVOS A CREAR:
src/app/page.tsx                  ← buscador de orden (campo de texto + botón)
src/app/track/[orderNumber]/page.tsx ← resultado del tracking

ENDPOINT PÚBLICO REQUERIDO EN EL BACKEND:
GET /public/orders/:orderNumber/status
→ Sin autenticación
→ Retorna SOLO: orderNumber, status, vehiclePlate, brand, model,
                clientFirstName, estimatedAt, completedAt, timeline (eventos)
→ NO expone: montos, datos internos, datos del personal

CREAR: arellan-platform/src/common/public/public.controller.ts
  @Get('orders/:orderNumber/status')
  @Public() // sin guard
  async getOrderStatus(@Param('orderNumber') orderNumber: string) { ... }
```

---

### PRIORIDAD 5 — DESIGN SYSTEM (si no existe)

Si `arellan-design-system` no tiene componentes exportables:

```
CREAR package.json con:
  "name": "@arellan/ui"
  "main": "./src/index.ts"
  "version": "0.1.0"

CREAR src/tokens/colors.ts:
  export const colors = {
    red: '#C0392B',
    dark: '#1A1A2E',
    gray: '#2C2C54',
    accent: '#E94560',
    success: '#27AE60',
    warning: '#F39C12',
    error: '#E74C3C',
    info: '#2980B9',
  }

CREAR src/components/ con componentes básicos:
  Button.tsx, Badge.tsx, Card.tsx, StatusBadge.tsx, LoadingSpinner.tsx, Alert.tsx

CREAR src/index.ts que exporta todo

NO se requiere Storybook en esta fase — solo que sea importable.
```

---

## VERIFICACIÓN FINAL OBLIGATORIA

Después de implementar todo, ejecuta los siguientes comandos en orden y reporta el resultado:

```bash
# 1. Levantar infraestructura
docker compose -f docker-compose.yml up -d postgres redis pgadmin
# Esperar 5 segundos

# 2. Backend
cd arellan-platform
npx prisma migrate deploy
npx prisma db seed
npm run start:dev
# Verificar: "Backend corriendo en http://localhost:3001" en los logs
# Verificar: sin errores de módulo no encontrado

# 3. Prueba de endpoints mínimos (reportar código HTTP de cada uno):
curl http://localhost:3001/api/v1/health
curl -X POST http://localhost:3001/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}'
# Verificar: retorna accessToken y refreshToken
# Guardar el accessToken en TOKEN=...

curl http://localhost:3001/api/v1/orders/stats/summary \
  -H "Authorization: Bearer $TOKEN"
# Verificar: retorna JSON con KPIs (no 404, no 500)

curl http://localhost:3001/api/v1/inventory/low-stock \
  -H "Authorization: Bearer $TOKEN"
# Verificar: retorna array con al menos 4 ítems (del seed)

# 4. Frontend web
cd ../arellan-frontend-web
npm run dev
# Verificar: arranca en puerto 3000 sin errores de build

# 5. Verificación funcional mínima:
# Abrir http://localhost:3000/login
# Login con edgar@arellanautos.pe / Arellan2026!
# Verificar: redirige al dashboard
# Verificar: el dashboard muestra datos reales (nombres, números del seed)
# Verificar: la página /orders carga la lista de órdenes

# 6. Mechanic UI
cd ../arellan-mechanic-ui
npm run dev
# Verificar: arranca en puerto 3002

# 7. Client Portal
cd ../arellan-client-portal
npm run dev
# Verificar: arranca en puerto 3003
# Verificar: /track/WO-2026-0001 muestra datos (usar número de orden del seed)
```

---

## ENTREGA FINAL

Al terminar la Fase 2, entrega un reporte en este formato:

```
═══════════════════════════════════════════════════════
REPORTE FINAL — IMPLEMENTACIÓN COMPLETADA
═══════════════════════════════════════════════════════

IMPLEMENTADO EN ESTA SESIÓN:
  [lista de lo que hiciste]

ESTADO DE CADA PRODUCTO:
  arellan-platform    → [✅ funcionando en :3001 / ⚠️ parcial / ❌]
  arellan-frontend-web→ [✅ :3000 con datos reales / ⚠️ / ❌]
  arellan-mechanic-ui → [✅ :3002 conectado / ⚠️ / ❌]
  arellan-client-portal→[✅ :3003 tracking ok / ⚠️ / ❌]
  arellan-design-system→[✅ @arellan/ui exportable / ⚠️ / ❌]

VERIFICACIÓN DE COMANDOS:
  docker compose up      → [✅ / ❌ error: X]
  prisma migrate deploy  → [✅ / ❌ error: X]
  prisma db seed         → [✅ N registros / ❌ error: X]
  backend start:dev      → [✅ :3001 / ❌ error: X]
  POST /auth/login       → [✅ HTTP 200, tokens ok / ❌ HTTP N]
  GET  /orders/stats     → [✅ HTTP 200 / ❌ HTTP N]
  GET  /inventory/low-stock → [✅ N items / ❌]
  frontend npm run dev   → [✅ :3000 / ❌ error: X]
  /login funcional       → [✅ redirige al dashboard / ❌]
  /dashboard con datos   → [✅ datos reales del seed / ⚠️ mocks / ❌]

AÚN PENDIENTE (para siguiente sesión):
  [lista honesta de lo que no alcanzaste]
═══════════════════════════════════════════════════════
```

---

**Empieza por la Fase 1. No escribas ningún código hasta tener el reporte completo.**
