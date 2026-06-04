# Clinica Automotriz Arellan Hnos -- Ecosystem Architecture v2.0

## Componentes del Monorepo

| Componente | Puerto | Tecnologia | Funcion |
|---|---|---|---|
| `arellan-platform` | 3001 | NestJS + Prisma + Redis + BullMQ | API REST + WebSockets + Colas |
| `arellan-frontend-web` | 3000 | Next.js 15 + Tailwind | Dashboard gerencial (OWNER/ADMIN) |
| `arellan-mechanic-ui` | 3002 | Vite + React 19 | Tablets de mecanicos |
| `arellan-client-portal` | 3003 | Next.js | Portal de clientes |
| `arellan-mobile-app` | 3004 | React Native / Expo | App movil de duenos |
| `arellan-sdk-core` | N/A | TypeScript | SDK compartido (API client, WS hooks) |
| `arellan-design-system` | 6006 | shadcn/ui + Tailwind | Sistema de diseno |
| `arellan-infrastructure` | N/A | Docker Compose | PostgreSQL, Redis, dependencias |
| `arellan-security-compliance` | N/A | Auditoria + Compliance | Politicas y escaneos |
| `arellan-data-intelligence` | N/A | PostgreSQL Views + Prisma | Vistas materializadas y reportes |

---

## Stack de Infraestructura

```
┌──────────────────────────────────────────────────────────┐
│                     Docker Compose                        │
│  ┌──────────┐  ┌──────────┐  ┌─────────────────────────┐ │
│  │PostgreSQL│  │  Redis   │  │ arellan-platform (Node)  │ │
│  │  :5432   │  │  :6379   │  │  :3001 (API + WS)       │ │
│  └──────────┘  └──────────┘  └─────────────────────────┘ │
│       ↑              ↑                    ↑               │
│  Prisma ORM    ioredis/pub     BullMQ / Cache-Aside       │
└──────────────────────────────────────────────────────────┘
```

---

## Flujos de Datos Asincronos

### 1. Ciclo de Vida de una Orden de Trabajo

```
[Recepcionista] → POST /api/v1/orders
    → AuditInterceptor registra (SHA-256 hash inmutable)
    → RealtimeGateway.emitOrderCreated()
        → dashboard (OWNER/ADMIN reciben "order:created")
    
[Mecanico Tablet] → PATCH /api/v1/orders/:id/status
    → OrdersService.updateStatus()
        → Si status=DELIVERED → validateVehicleDelivery()
            → Verifica: laborCost + partsCost == totalPaid
            → Si hay Yape >= 500 → log de alerta
        → RealtimeGateway.emitOrderStatusChanged()
            → dashboard: "order:status_changed"
            → order:{id}: "order:status_changed"
            → client:{clientId}: "order:status_changed"
    
[Mecanico Tablet] → WS emit "mechanic:progress"
    → RealtimeGateway.emitMechanicProgress()
        → dashboard: "mechanic:progress"
        → order:{id}: "mechanic:progress"
    
[Vehiculo Entregado] → emitVehicleDelivered()
    → client:{clientId}: "vehicle:delivered"
```

### 2. Flujo de Aprobacion de Gastos (BullMQ)

```
[FINANCE] → POST /api/v1/finance/expenses
    → FinanceService.createExpense()
        → Si monto >= 200 o categoria sospechosa:
            → BullMQ ALERT_DISPATCHER: "suspicious-expense"
        → Siempre:
            → BullMQ ALERT_DISPATCHER: "expense-approval-required"
    → RealtimeGateway.emitApprovalRequested()

[ADMIN/OWNER] → POST /api/v1/finance/expenses/:id/approve
    → FinanceService.approveExpense()
        → Si DUAL_OWNER → approveExpenseWithDualApproval()
        → BullMQ: "expense-disbursed"
    → RealtimeGateway.emitApprovalResolved()
```

### 3. Cache-Aside para Inventario y Vehiculos

```
[Lectura] → InventoryService.findAll()
    → redis.get("inventory:catalog:...")
        → HIT: retorna desde Redis (TTL 120s)
        → MISS: Prisma query → redis.set() → retorna

[Escritura] → InventoryService.addMovement()
    → Prisma transaction (movement + stock update)
    → redis.del("inventory:item:{id}")
    → invalidateCatalogCache() → SCAN + DEL "inventory:catalog:*"
    → redis.del("inventory:lowstock:*")
    → redis.del("inventory:critical")
    → redis.del("inventory:valuation")

[Busqueda de Placa] → VehiclesService.findByPlate()
    → redis.get("vehicles:plate:{PLATE}") → TTL 300s
```

### 4. WebSocket Rooms y Canales

| Room/Canal | Suscriptores | Eventos |
|---|---|---|
| `dashboard` | OWNER, ADMIN, FINANCE | `order:created`, `order:status_changed`, `mechanic:progress`, `payment:received`, `alert:security` |
| `order:{id}` | Clientes, mecanicos asignados | `order:updated`, `order:status_changed`, `mechanic:progress` |
| `mechanic:{id}` | Mecanico especifico | `mechanic:progress` |
| `client:{id}` | Cliente especifico | `vehicle:delivered`, `order:status_changed` |
| `role:{role}` | Todos los usuarios de un rol | Eventos broadcast por rol |

---

## Interceptores y Middlewares de Seguridad

### AuditInterceptor (Global)
- Captura POST, PATCH, DELETE en todas las rutas
- Genera hash SHA-256 inmutable via `IntegrityHashService`
- Fallback: `crypto.createHmac("sha256")` si el servicio falla
- Clasifica severidad: INFO, WARNING, CRITICAL, SECURITY_ALERT
- Sanitiza campos sensibles (password, token, mfaSecret)

### AntiFraudMiddleware (Global)
- Detecta pagos en Yape personal (`isPersonalYape`)
- Detecta ajustes de inventario (`ADJUSTMENT`, "merma", "perdida")
- Detecta mutaciones de alto valor (> S/ 500)
- Detecta accesos a endpoints restringidos
- Persiste alertas en `audit_logs` con `integrityHash`

### CDN / Cache Policy
- `noeviction`: Redis configurado para nunca expulsar por memoria
- Inventory items: TTL 120s (catalogo), 300s (item individual)
- Vehicles: TTL 60s (listas), 300s (busqueda por placa, detalle)
- Finance dashboard: TTL 30s

---

## Endpoints Criticos Interconectados

### Ordenes → Finanzas → Inventario

```
GET    /api/v1/orders                     → Lista con filtros
POST   /api/v1/orders                     → Crear OT (requiere vehicleId, clientId, mechanicId)
PATCH  /api/v1/orders/:id                 → Actualizar diagnostico/costos
POST   /api/v1/orders/:id/status          → Transicion de estado (valida pago al DELIVERED)
POST   /api/v1/orders/:id/items           → Agregar repuesto a OT (consume inventario)

GET    /api/v1/finance/cashbox/today      → Estado de caja actual
POST   /api/v1/finance/cashbox/open       → Abrir caja (MFA requerido)
POST   /api/v1/finance/cashbox/close      → Cerrar caja (detecta discrepancia > 50)
POST   /api/v1/finance/expenses           → Crear gasto (dispara alertas BullMQ)
POST   /api/v1/finance/expenses/:id/approve → Aprobar/rechazar gasto

GET    /api/v1/inventory                  → Catalogo con cache Redis
POST   /api/v1/inventory/:id/movements    → Movimiento de stock (invalida cache)
POST   /api/v1/inventory/:id/reserve      → Reservar para OT

GET    /api/v1/vehicles/lookup/:plate     → Busqueda por placa (cache Redis)
```

### Autenticacion y Seguridad

```
POST   /api/v1/auth/login                 → Login con JWT + Refresh Token
POST   /api/v1/auth/refresh               → Rotacion de Refresh Token
POST   /api/v1/auth/mfa/setup             → Configurar MFA (TOTP)
POST   /api/v1/auth/mfa/verify            → Verificar MFA
GET    /api/v1/audit                      → Consultar logs de auditoria
```

---

## Flujo de Autenticacion desde Frontends

```
1. Cliente envia POST /auth/login con email + password
2. Servidor retorna { accessToken, refreshToken }
3. SDK (ArellanApiClient) almacena tokens en store
4. Interceptor axios inyecta `Authorization: Bearer {accessToken}`
5. Al recibir 401, interceptor usa refreshToken para rotar
6. Si refresh falla → logout + redireccion a login
7. WebSocket: useRealtime() hook se conecta con token en handshake.auth
```

## Politica de Cache y Concurrencia

- **Redis noeviction**: nunca expulsa keys por memoria
- **TTLs diferenciados**: finanzas (30s), inventario (120s), vehiculos (300s)
- **Invalidacion por escritura**: cada mutacion limpia las keys relacionadas
- **BullMQ**: cola ALERT_DISPATCHER para notificaciones async
- **Circuit Breaker**: SDK incluye proteccion con 5 fallos consecutivos
