# OPENCODE — VERIFICACIÓN INTEGRAL FASES 1, 2 Y 3
# Ecosistema Digital "Clínica Automotriz Arellan Hnos"

---

## MISIÓN

Esta sesión es exclusivamente de **verificación y corrección**. Tu trabajo:

1. Instalar dependencias y levantar la infraestructura completa
2. Verificar que cada entregable de las fases 1, 2 y 3 existe y funciona
3. Si algo falla o no existe → **corregirlo en el momento**, documentar el fix
4. Ejecutar los 12 escenarios de negocio del caso Arellan
5. Entregar el scorecard final con evidencia real (HTTP codes, outputs de terminal)

**Regla absoluta:** No puedes marcar algo como ✅ sin evidencia de terminal. Cada verificación requiere el output real del comando que la prueba.

---

## BLOQUE 0 — SETUP DE INFRAESTRUCTURA

Ejecuta esto antes de cualquier verificación. Si algo falla aquí, corrígelo antes de continuar.

```bash
# 0.1 — Verificar que Docker esté disponible
docker --version && docker compose version
# Si falla: instalar Docker Desktop desde https://www.docker.com/products/docker-desktop

# 0.2 — Levantar servicios de base de datos
cd <RAÍZ_DEL_PROYECTO>
docker compose up -d postgres redis pgadmin
sleep 8
docker compose ps
# ESPERADO: postgres (healthy), redis (running), pgadmin (running)

# 0.3 — Instalar dependencias del backend
cd arellan-platform
npm install
# Si hay errores de peer deps: npm install --legacy-peer-deps

# 0.4 — Generar cliente Prisma
npx prisma generate
# ESPERADO: "Generated Prisma Client"

# 0.5 — Aplicar migraciones
npx prisma migrate deploy
# ESPERADO: "All migrations have been successfully applied"
# Si hay errores de migración: npx prisma migrate reset --force (solo en dev)

# 0.6 — Correr el seed
npx prisma db seed
# ESPERADO: seed corre sin errores. Guardar el output completo.

# 0.7 — Instalar dependencias de los frontends
cd ../arellan-frontend-web && npm install
cd ../arellan-mechanic-ui && npm install
cd ../arellan-client-portal && npm install
cd ../arellan-mobile-app && npm install 2>/dev/null || echo "mobile-app: sin package.json o error"
cd ../arellan-design-system && npm install 2>/dev/null || echo "design-system: revisar"

# 0.8 — Levantar el backend en background
cd ../arellan-platform
npm run start:dev > /tmp/arellan-backend.log 2>&1 &
BACKEND_PID=$!
sleep 15

# 0.9 — Verificar que el backend está corriendo
curl -s http://localhost:3001/api/v1/health | python3 -m json.tool 2>/dev/null || \
curl -s http://localhost:3001/api/v1/health
# ESPERADO: {"status":"ok",...}

tail -20 /tmp/arellan-backend.log
# ESPERADO: "Backend corriendo en http://localhost:3001" SIN errores de módulo
```

**Si el backend no levanta en :3001**, lee los últimos 50 líneas del log, identifica el error, corrígelo y repite el paso 0.8.

---

## BLOQUE 1 — VERIFICACIÓN BASE DE DATOS (Fase 1)

```bash
# 1.1 — Contar modelos en schema.prisma
grep -c "^model " arellan-platform/prisma/schema.prisma
# ESPERADO: 32 (o más)

# 1.2 — Verificar modelos críticos del caso Arellan
for model in User Session Personnel Attendance VehicleUsage WorkOrder WorkOrderItem \
  WorkOrderEvent WorkOrderAssignee InventoryItem StockMovement Category Supplier \
  Client Vehicle Invoice Payment Expense Approval Purchase PurchaseItem Commission \
  Quote AuditLog Notification Setting; do
  grep -q "^model $model " arellan-platform/prisma/schema.prisma && \
    echo "✅ $model" || echo "❌ FALTA: $model"
done

# 1.3 — Verificar campos anti-fraude en el schema
echo "=== Campos anti-fraude ==="
grep -n "isPersonalYape\|yapeAccount\|paymentYape\|UNAUTHORIZED\|lockedUntil\|failedAttempts\|SECURITY_ALERT" \
  arellan-platform/prisma/schema.prisma
# ESPERADO: al menos 5 líneas encontradas

# 1.4 — Verificar datos del seed en la base de datos
cd arellan-platform
npx prisma db execute --stdin << 'EOF'
SELECT 'users' as tabla, COUNT(*) as total FROM "User"
UNION ALL SELECT 'personnel', COUNT(*) FROM "Personnel"
UNION ALL SELECT 'clients', COUNT(*) FROM "Client"
UNION ALL SELECT 'vehicles', COUNT(*) FROM "Vehicle"
UNION ALL SELECT 'inventory_items', COUNT(*) FROM "InventoryItem"
UNION ALL SELECT 'work_orders', COUNT(*) FROM "WorkOrder"
UNION ALL SELECT 'work_order_events', COUNT(*) FROM "WorkOrderEvent"
UNION ALL SELECT 'notifications', COUNT(*) FROM "Notification"
UNION ALL SELECT 'settings', COUNT(*) FROM "Setting";
EOF
# ESPERADO:
# users >= 7
# personnel >= 7
# clients >= 10
# vehicles >= 20
# inventory_items >= 30
# work_orders >= 20
# work_order_events >= 60
# notifications >= 10
# settings >= 10

# 1.5 — Verificar que hay items bajo stock mínimo (para testear alertas)
npx prisma db execute --stdin << 'EOF'
SELECT name, "currentStock", "minStock"
FROM "InventoryItem"
WHERE "currentStock" < "minStock"
LIMIT 10;
EOF
# ESPERADO: al menos 4 filas

# 1.6 — Verificar que los passwords están hasheados (no texto plano)
npx prisma db execute --stdin << 'EOF'
SELECT email, LEFT(password, 7) as hash_prefix FROM "User" LIMIT 3;
EOF
# ESPERADO: hash_prefix = "$2b$12" (bcrypt) en todos — NUNCA texto plano

# 1.7 — Verificar migraciones aplicadas
npx prisma migrate status
# ESPERADO: todas las migraciones marcadas como "Applied"
```

**Si algún modelo falta** → agrégalo al schema y corre `npx prisma migrate dev --name "fix_missing_models"`.
**Si los datos del seed están incompletos** → corregir seed.ts y reejecutar `npx prisma db seed`.

---

## BLOQUE 2 — VERIFICACIÓN DE AUTENTICACIÓN (Fase 2)

```bash
# Obtener token de Edgar (OWNER)
echo "=== LOGIN EDGAR (OWNER) ==="
RESPONSE=$(curl -s -X POST http://localhost:3001/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}')
echo $RESPONSE | python3 -m json.tool 2>/dev/null || echo $RESPONSE

# Extraer tokens
TOKEN_EDGAR=$(echo $RESPONSE | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
REFRESH_EDGAR=$(echo $RESPONSE | grep -o '"refreshToken":"[^"]*"' | cut -d'"' -f4)

echo "Access Token: ${TOKEN_EDGAR:0:30}..."
echo "Refresh Token: ${REFRESH_EDGAR:0:30}..."
# ESPERADO: ambos tokens presentes con formato JWT (xxx.yyy.zzz)

# 2.2 — Verificar /auth/me
echo "=== GET /auth/me ==="
curl -s http://localhost:3001/api/v1/auth/me \
  -H "Authorization: Bearer $TOKEN_EDGAR" | python3 -m json.tool 2>/dev/null
# ESPERADO: { id, email, role: "OWNER", profile: {...} }

# 2.3 — Refresh token
echo "=== REFRESH TOKEN ==="
NEW_TOKEN=$(curl -s -X POST http://localhost:3001/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH_EDGAR\"}")
echo $NEW_TOKEN | python3 -m json.tool 2>/dev/null || echo $NEW_TOKEN
# ESPERADO: { accessToken: "nuevo_token..." }

# 2.4 — Sesiones activas
echo "=== GET /auth/sessions ==="
HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3001/api/v1/auth/sessions \
  -H "Authorization: Bearer $TOKEN_EDGAR")
echo "HTTP: $HTTP"
# ESPERADO: 200

# 2.5 — Probar rate limiting (5 intentos fallidos → bloqueo)
echo "=== RATE LIMITING TEST ==="
for i in 1 2 3 4 5; do
  CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:3001/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email":"mecanico1@arellanautos.pe","password":"password_wrong"}')
  echo "Intento $i: HTTP $CODE"
done
# Intento 6 — debe estar bloqueado:
CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:3001/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"mecanico1@arellanautos.pe","password":"password_wrong"}')
echo "Intento 6 (esperado 429 o 403): HTTP $CODE"
# ESPERADO: 429 (Too Many Requests) o 403 (cuenta bloqueada)

# 2.6 — Login como mecánico (rol MECHANIC)
echo "=== LOGIN MECÁNICO ==="
TOKEN_MECH=$(curl -s -X POST http://localhost:3001/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"mecanico1@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
echo "Token mecánico: ${TOKEN_MECH:0:30}..."

# 2.7 — Logout
echo "=== LOGOUT ==="
HTTP=$(curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:3001/api/v1/auth/logout \
  -H "Authorization: Bearer $TOKEN_MECH")
echo "HTTP logout: $HTTP"
# ESPERADO: 200

# Re-obtener token Edgar para siguientes pruebas
TOKEN=$(curl -s -X POST http://localhost:3001/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
```

---

## BLOQUE 3 — VERIFICACIÓN DE ENDPOINTS BACKEND (Fases 2 y 3)

```bash
BASE="http://localhost:3001/api/v1"
AUTH="-H \"Authorization: Bearer $TOKEN\""

run_check() {
  local desc="$1"
  local url="$2"
  local extra="${3:-}"
  CODE=$(curl -s -o /dev/null -w "%{http_code}" $url -H "Authorization: Bearer $TOKEN" $extra)
  if [ "$CODE" = "200" ] || [ "$CODE" = "201" ]; then
    echo "✅ $desc → HTTP $CODE"
  else
    echo "❌ $desc → HTTP $CODE (FALLA)"
  fi
}

echo ""
echo "=== MÓDULO: ORDERS ==="
run_check "GET /orders (lista paginada)"       "$BASE/orders"
run_check "GET /orders/stats/summary"          "$BASE/orders/stats/summary"
run_check "GET /orders/by-status/IN_PROGRESS"  "$BASE/orders/by-status/IN_PROGRESS"

# Obtener un ID de orden del seed
ORDER_ID=$(curl -s "$BASE/orders?limit=1" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['id'])" 2>/dev/null)
echo "Order ID de prueba: $ORDER_ID"

run_check "GET /orders/:id (detalle)"          "$BASE/orders/$ORDER_ID"
run_check "GET /orders/:id/timeline"           "$BASE/orders/$ORDER_ID/timeline"

echo ""
echo "=== MÓDULO: INVENTORY ==="
run_check "GET /inventory (lista)"             "$BASE/inventory"
run_check "GET /inventory/low-stock"           "$BASE/inventory/low-stock"
run_check "GET /inventory/valuation"           "$BASE/inventory/valuation"
run_check "GET /inventory/movements"           "$BASE/inventory/movements"

echo ""
echo "=== MÓDULO: FINANCE ==="
run_check "GET /finance/dashboard"             "$BASE/finance/dashboard"
run_check "GET /finance/cashflow"              "$BASE/finance/cashflow"
run_check "GET /finance/expenses"              "$BASE/finance/expenses"
run_check "GET /finance/commissions"           "$BASE/finance/commissions"

echo ""
echo "=== MÓDULO: PERSONNEL ==="
run_check "GET /personnel (lista)"             "$BASE/personnel"
run_check "GET /personnel/attendance/today"    "$BASE/personnel/attendance/today"
run_check "GET /personnel/vehicle-usage/active" "$BASE/personnel/vehicle-usage/active"
run_check "GET /personnel/vehicle-usage/overdue" "$BASE/personnel/vehicle-usage/overdue"

echo ""
echo "=== MÓDULO: CLIENTS ==="
run_check "GET /clients (lista)"               "$BASE/clients"

CLIENT_ID=$(curl -s "$BASE/clients?limit=1" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['id'])" 2>/dev/null)
echo "Client ID de prueba: $CLIENT_ID"
run_check "GET /clients/:id"                   "$BASE/clients/$CLIENT_ID"

echo ""
echo "=== MÓDULO: VEHICLES ==="
run_check "GET /vehicles (lista)"              "$BASE/vehicles"
run_check "GET /vehicles/workshop-fleet"       "$BASE/vehicles/workshop-fleet"

echo ""
echo "=== MÓDULO: AUDIT ==="
run_check "GET /audit/logs"                    "$BASE/audit/logs"

echo ""
echo "=== MÓDULO: SETTINGS ==="
run_check "GET /settings (lista)"              "$BASE/settings"

echo ""
echo "=== ENDPOINT PÚBLICO (sin auth) ==="
# Obtener número de una orden real del seed
ORDER_NUM=$(curl -s "$BASE/orders?limit=1" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['orderNumber'])" 2>/dev/null)
echo "Número de orden: $ORDER_NUM"

PUB_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  "$BASE/public/orders/number/$ORDER_NUM/status")
if [ "$PUB_CODE" = "200" ]; then
  echo "✅ GET /public/orders/number/:num/status → HTTP $PUB_CODE (sin auth)"
  # Verificar que no expone datos financieros
  PUB_RESP=$(curl -s "$BASE/public/orders/number/$ORDER_NUM/status")
  echo "Datos retornados: $PUB_RESP" | python3 -m json.tool 2>/dev/null | head -30
else
  echo "❌ GET /public/orders → HTTP $PUB_CODE (FALLA)"
fi

echo ""
echo "=== SWAGGER ==="
SW_CODE=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:3001/api/docs")
echo "Swagger UI: HTTP $SW_CODE"
# ESPERADO: 200

# Contar endpoints documentados
curl -s "http://localhost:3001/api/docs-json" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); paths=d.get('paths',{}); print(f'Endpoints en Swagger: {sum(len(v) for v in paths.values())}')" 2>/dev/null
# ESPERADO: 40+ endpoints
```

---

## BLOQUE 4 — VERIFICACIÓN DE REGLAS DE NEGOCIO ANTI-FRAUDE (Fase 2)

Estos son los controles críticos del caso Ricardo. Cada uno debe generar el comportamiento esperado.

```bash
BASE="http://localhost:3001/api/v1"

echo ""
echo "=== ANTI-FRAUDE 1: Pago con Yape personal ==="
# Registrar un pago con Yape personal (no el del taller) → debe crear AuditLog SECURITY_ALERT

# Primero obtener una orden en estado que permita pagos
ORDER_ID=$(curl -s "$BASE/orders?status=IN_PROGRESS&limit=1" -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['id'])" 2>/dev/null)

PAYMENT_RESP=$(curl -s -X POST "$BASE/payments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"workOrderId\": \"$ORDER_ID\",
    \"method\": \"YAPE\",
    \"amount\": 150,
    \"yapeAccount\": \"987654321\",
    \"notes\": \"Test pago Yape personal\"
  }")
echo "Respuesta pago Yape: $PAYMENT_RESP" | python3 -m json.tool 2>/dev/null | head -20

# Verificar que se creó un AuditLog de tipo SECURITY_ALERT
sleep 1
AUDIT_ALERT=$(curl -s "$BASE/audit/logs?severity=SECURITY_ALERT&limit=5" \
  -H "Authorization: Bearer $TOKEN")
echo "Alertas de seguridad: $AUDIT_ALERT" | python3 -m json.tool 2>/dev/null | head -20
# ESPERADO: al menos 1 registro con action: "PAYMENT_UNAUTHORIZED_YAPE"

echo ""
echo "=== ANTI-FRAUDE 2: Gasto > S/500 requiere doble aprobación ==="
# Crear un gasto de S/600 → debe quedar en PENDING, no aprobarse automáticamente
EXPENSE_RESP=$(curl -s -X POST "$BASE/finance/expenses" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "category": "PARTS_PURCHASE",
    "description": "Compra de repuestos importados test",
    "amount": 600,
    "paymentMethod": "BANK_TRANSFER"
  }')
echo "Gasto S/600: $EXPENSE_RESP" | python3 -m json.tool 2>/dev/null | head -15
# ESPERADO: approvalStatus = "PENDING", NO = "APPROVED"

echo ""
echo "=== ANTI-FRAUDE 3: Gasto < S/100 aprobación directa ==="
EXPENSE_SMALL=$(curl -s -X POST "$BASE/finance/expenses" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "category": "FOOD",
    "description": "Almuerzo equipo",
    "amount": 45,
    "paymentMethod": "CASH"
  }')
echo "Gasto S/45: $EXPENSE_SMALL" | python3 -m json.tool 2>/dev/null | head -10
# ESPERADO: approvalStatus = "APPROVED" (aprobación automática para montos bajos)

echo ""
echo "=== ANTI-FRAUDE 4: Descuento > 20% requiere aprobación ==="
# Intentar aplicar 30% de descuento en una orden → debe crear una Approval en PENDING
DISCOUNT_RESP=$(curl -s -X POST "$BASE/orders/$ORDER_ID/discount" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"discountPercentage": 30, "reason": "Cliente frecuente test"}')
echo "Descuento 30%: $DISCOUNT_RESP" | python3 -m json.tool 2>/dev/null | head -10
# ESPERADO: {"status":"PENDING_APPROVAL"} o la orden queda en estado que requiere aprobación

echo ""
echo "=== ANTI-FRAUDE 5: Mecánico no puede ver finanzas ==="
TOKEN_MECH=$(curl -s -X POST "$BASE/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"mecanico1@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

FIN_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  "$BASE/finance/dashboard" \
  -H "Authorization: Bearer $TOKEN_MECH")
echo "Mecánico accediendo a /finance/dashboard: HTTP $FIN_CODE"
# ESPERADO: 403 (Forbidden)

AUDIT_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  "$BASE/audit/logs" \
  -H "Authorization: Bearer $TOKEN_MECH")
echo "Mecánico accediendo a /audit/logs: HTTP $AUDIT_CODE"
# ESPERADO: 403 (Forbidden)
```

---

## BLOQUE 5 — VERIFICACIÓN DE WEBSOCKETS (Fase 2)

```bash
echo ""
echo "=== WEBSOCKET: verificar que el gateway responde ==="

# Verificar que Socket.io está montado en el backend
WS_RESP=$(curl -s "http://localhost:3001/socket.io/?EIO=4&transport=polling")
echo "Socket.io handshake: ${WS_RESP:0:50}"
# ESPERADO: {"sid":"...","upgrades":["websocket"],...}

# Verificar que useRealtime.ts existe en el frontend
ls arellan-frontend-web/src/hooks/useRealtime.ts 2>/dev/null && \
  echo "✅ useRealtime.ts existe" || echo "❌ useRealtime.ts NO existe"

# Verificar que RealtimeGateway tiene los métodos de emisión críticos
echo ""
echo "=== MÉTODOS DEL REALTIME GATEWAY ==="
for method in emitOrderCreated emitOrderStatusChanged emitInventoryLowStock \
  emitPaymentReceived emitPersonnelCheckIn emitVehicleOverdue \
  emitApprovalRequested emitSecurityAlert; do
  grep -q "$method" arellan-platform/src/common/gateway/realtime.gateway.ts && \
    echo "✅ $method" || echo "❌ FALTA: $method"
done

# Verificar que orders.gateway.ts fue eliminado (unificación)
ls arellan-platform/src/gateways/orders.gateway.ts 2>/dev/null && \
  echo "⚠️ orders.gateway.ts aún existe (debe eliminarse)" || \
  echo "✅ orders.gateway.ts correctamente eliminado"
```

---

## BLOQUE 6 — ESCENARIOS DE NEGOCIO COMPLETOS (caso Arellan real)

Estos son los 12 flujos del negocio real. Cada uno debe completarse sin errores.

```bash
BASE="http://localhost:3001/api/v1"
# Re-obtener token Edgar
TOKEN=$(curl -s -X POST "$BASE/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

echo "============================================"
echo "ESCENARIO 1: Ingreso de vehículo al taller"
echo "============================================"
# Crear cliente nuevo
CLIENT=$(curl -s -X POST "$BASE/clients" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Roberto",
    "lastName": "Flores",
    "phone": "987123456",
    "dni": "45678901",
    "district": "Miraflores"
  }')
CLIENT_ID=$(echo $CLIENT | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])" 2>/dev/null)
echo "Cliente creado: $CLIENT_ID"
# ESPERADO: ID válido (cuid)

# Registrar vehículo del cliente
VEHICLE=$(curl -s -X POST "$BASE/vehicles" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"clientId\": \"$CLIENT_ID\",
    \"plate\": \"ABC-999\",
    \"brand\": \"Toyota\",
    \"model\": \"Corolla\",
    \"year\": 2019,
    \"color\": \"Blanco\",
    \"engineType\": \"GASOLINE\",
    \"mileage\": 45000
  }")
VEHICLE_ID=$(echo $VEHICLE | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])" 2>/dev/null)
echo "Vehículo creado: $VEHICLE_ID"
# ESPERADO: ID válido

echo ""
echo "============================================"
echo "ESCENARIO 2: Crear Orden de Trabajo"
echo "============================================"
ORDER=$(curl -s -X POST "$BASE/orders" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"clientId\": \"$CLIENT_ID\",
    \"vehicleId\": \"$VEHICLE_ID\",
    \"description\": \"Cambio de aceite y revisión de frenos\",
    \"type\": \"PREVENTIVE\",
    \"priority\": \"NORMAL\",
    \"odometerIn\": 45000,
    \"fuelLevel\": \"3/4\"
  }")
NEW_ORDER_ID=$(echo $ORDER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])" 2>/dev/null)
NEW_ORDER_NUM=$(echo $ORDER | python3 -c "import sys,json; print(json.load(sys.stdin)['orderNumber'])" 2>/dev/null)
echo "Orden creada: $NEW_ORDER_NUM (ID: $NEW_ORDER_ID)"
# ESPERADO: orderNumber formato WO-2026-XXXX

echo ""
echo "============================================"
echo "ESCENARIO 3: Asignar mecánico a la orden"
echo "============================================"
# Obtener ID del mecánico del seed
MECH_PERSONNEL_ID=$(curl -s "$BASE/personnel?role=MECHANIC&limit=1" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['id'])" 2>/dev/null)
echo "Mecánico ID: $MECH_PERSONNEL_ID"

ASSIGN_RESP=$(curl -s -X POST "$BASE/orders/$NEW_ORDER_ID/assign" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"personnelId\": \"$MECH_PERSONNEL_ID\", \"role\": \"LEAD\"}")
echo "Asignación: $(echo $ASSIGN_RESP | python3 -m json.tool 2>/dev/null | head -5)"
# ESPERADO: sin error, orden actualizada con asignado

echo ""
echo "============================================"
echo "ESCENARIO 4: Agregar repuesto y descontar inventario"
echo "============================================"
# Obtener un item de inventario del seed
ITEM_ID=$(curl -s "$BASE/inventory?limit=1" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['id'])" 2>/dev/null)
ITEM_STOCK_BEFORE=$(curl -s "$BASE/inventory/$ITEM_ID" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['currentStock'])" 2>/dev/null)
echo "Stock antes: $ITEM_STOCK_BEFORE"

ADD_ITEM=$(curl -s -X POST "$BASE/orders/$NEW_ORDER_ID/items" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"itemId\": \"$ITEM_ID\",
    \"type\": \"PART\",
    \"quantity\": 1,
    \"description\": \"Aceite 5W30 sintético\"
  }")
echo "Item agregado: $(echo $ADD_ITEM | python3 -m json.tool 2>/dev/null | head -8)"

ITEM_STOCK_AFTER=$(curl -s "$BASE/inventory/$ITEM_ID" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['currentStock'])" 2>/dev/null)
echo "Stock después: $ITEM_STOCK_AFTER"
# ESPERADO: stock_after = stock_before - 1

echo ""
echo "============================================"
echo "ESCENARIO 5: Cambiar estado de la orden"
echo "============================================"
STATUS_RESP=$(curl -s -X PATCH "$BASE/orders/$NEW_ORDER_ID/status" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "IN_PROGRESS", "notes": "Iniciando diagnóstico"}')
echo "Cambio a IN_PROGRESS: $(echo $STATUS_RESP | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','ERROR'))" 2>/dev/null)"
# ESPERADO: status = "IN_PROGRESS"

# Completar la orden
COMPLETE_RESP=$(curl -s -X PATCH "$BASE/orders/$NEW_ORDER_ID/status" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "COMPLETED", "notes": "Servicio completado exitosamente"}')
echo "Cambio a COMPLETED: $(echo $COMPLETE_RESP | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','ERROR'))" 2>/dev/null)"

echo ""
echo "============================================"
echo "ESCENARIO 6: Registrar pago (Yape del taller)"
echo "============================================"
# Obtener el Yape oficial del taller desde Settings
YAPE_OFICIAL=$(curl -s "$BASE/settings" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); items=d.get('data',d) if isinstance(d,dict) else d; s=[x for x in items if x.get('key')=='TALLER_YAPE_NUMBER']; print(s[0]['value'] if s else 'no_encontrado')" 2>/dev/null)
echo "Yape oficial del taller: $YAPE_OFICIAL"

PAYMENT=$(curl -s -X POST "$BASE/payments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"workOrderId\": \"$NEW_ORDER_ID\",
    \"method\": \"YAPE\",
    \"amount\": 250,
    \"yapeAccount\": \"$YAPE_OFICIAL\",
    \"notes\": \"Pago por Yape del taller\"
  }")
echo "Pago registrado: $(echo $PAYMENT | python3 -m json.tool 2>/dev/null | head -10)"
# ESPERADO: isPersonalYape = false, sin alerta de seguridad

echo ""
echo "============================================"
echo "ESCENARIO 7: Check-in de mecánico (desde Mechanic UI)"
echo "============================================"
# Login como mecánico
TOKEN_MECH=$(curl -s -X POST "$BASE/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"mecanico1@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

CHECKIN=$(curl -s -X POST "$BASE/personnel/attendance/check-in" \
  -H "Authorization: Bearer $TOKEN_MECH" \
  -H "Content-Type: application/json" \
  -d '{"notes": "Llegué a tiempo"}')
echo "Check-in mecánico: $CHECKIN" | python3 -m json.tool 2>/dev/null | head -10
# ESPERADO: registro de asistencia creado con checkIn timestamp

echo ""
echo "============================================"
echo "ESCENARIO 8: Alerta de stock bajo en inventario"
echo "============================================"
LOW_STOCK=$(curl -s "$BASE/inventory/low-stock" \
  -H "Authorization: Bearer $TOKEN")
LOW_COUNT=$(echo $LOW_STOCK | python3 -c "import sys,json; d=json.load(sys.stdin); items=d.get('data',d) if isinstance(d,dict) else d; print(len(items))" 2>/dev/null)
echo "Items bajo stock mínimo: $LOW_COUNT"
# ESPERADO: >= 4 (del seed)
echo "Primeros 2: $(echo $LOW_STOCK | python3 -c "import sys,json; d=json.load(sys.stdin); items=d.get('data',d) if isinstance(d,dict) else d; [print(i['name'], 'stock:', i['currentStock'], 'min:', i['minStock']) for i in items[:2]]" 2>/dev/null)"

echo ""
echo "============================================"
echo "ESCENARIO 9: Tracking público desde portal cliente"
echo "============================================"
PUB_TRACKING=$(curl -s "$BASE/public/orders/number/$NEW_ORDER_NUM/status")
echo "Tracking público ($NEW_ORDER_NUM):"
echo $PUB_TRACKING | python3 -m json.tool 2>/dev/null | head -25
# ESPERADO: status, timeline, datos del vehículo — SIN montos ni datos del personal

# Verificar que NO incluye datos sensibles
echo $PUB_TRACKING | grep -q '"amount"\|"salary"\|"password"\|"receivedBy"\|"personnelId"' && \
  echo "⚠️ ALERTA: respuesta pública expone datos sensibles" || \
  echo "✅ Respuesta pública no expone datos sensibles"

echo ""
echo "============================================"
echo "ESCENARIO 10: Dashboard financiero del OWNER"
echo "============================================"
DASHBOARD=$(curl -s "$BASE/finance/dashboard" \
  -H "Authorization: Bearer $TOKEN")
echo "Dashboard financiero:"
echo $DASHBOARD | python3 -m json.tool 2>/dev/null | head -20
# ESPERADO: { revenue, expenses, profit, transactionCount, activeOrders, pendingApprovals }

echo ""
echo "============================================"
echo "ESCENARIO 11: Autorizar uso de vehículo del taller"
echo "============================================"
# Buscar un vehículo del taller (workshop fleet)
FLEET_VEHICLE=$(curl -s "$BASE/vehicles/workshop-fleet&limit=1" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); items=d.get('data',d) if isinstance(d,dict) else d; print(items[0]['id'] if items else 'no_fleet')" 2>/dev/null)
echo "Vehículo de flota: $FLEET_VEHICLE"

if [ "$FLEET_VEHICLE" != "no_fleet" ] && [ ! -z "$FLEET_VEHICLE" ]; then
  USAGE=$(curl -s -X POST "$BASE/personnel/vehicle-usage/authorize" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{
      \"vehicleId\": \"$FLEET_VEHICLE\",
      \"personnelId\": \"$MECH_PERSONNEL_ID\",
      \"purpose\": \"Recojo de repuestos en almacén\",
      \"destination\": \"La Victoria\",
      \"expectedReturn\": \"$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v+2H +%Y-%m-%dT%H:%M:%SZ)\"
    }")
  echo "Uso autorizado: $(echo $USAGE | python3 -m json.tool 2>/dev/null | head -8)"
  # ESPERADO: uso creado con status PENDING_RETURN y authorizedBy del OWNER
else
  echo "⚠️ No hay vehículos de flota en el seed — verificar si el seed incluye vehículos del taller"
fi

echo ""
echo "============================================"
echo "ESCENARIO 12: Inventario — Valorización total"
echo "============================================"
VALUATION=$(curl -s "$BASE/inventory/valuation" \
  -H "Authorization: Bearer $TOKEN")
echo "Valorización del inventario:"
echo $VALUATION | python3 -m json.tool 2>/dev/null
# ESPERADO: { totalItems, totalCostValue, totalSaleValue, lowStockCount }
# totalItems >= 30, totalCostValue > 0
```

---

## BLOQUE 7 — VERIFICACIÓN DE FRONTENDS

```bash
echo ""
echo "=== BUILD DE TODOS LOS FRONTENDS ==="

check_build() {
  local dir="$1"
  local port="$2"
  echo ""
  echo "--- $dir ---"
  cd ../$dir
  
  # Verificar .env.local
  if [ ! -f ".env.local" ] && [ -f ".env.local.example" ]; then
    cp .env.local.example .env.local
    echo "⚠️ .env.local creado desde .env.local.example"
  fi
  
  # Build
  BUILD_OUTPUT=$(npm run build 2>&1)
  BUILD_EXIT=$?
  
  if [ $BUILD_EXIT -eq 0 ]; then
    echo "✅ Build exitoso"
  else
    echo "❌ Build falló:"
    echo "$BUILD_OUTPUT" | tail -20
    echo ""
    echo "→ Corrigiendo errores de build..."
    # OpenCode debe corregir aquí los errores de TypeScript o import que aparezcan
  fi
}

cd arellan-platform  # reset
check_build "arellan-frontend-web" "3000"
check_build "arellan-mechanic-ui" "3002"
check_build "arellan-client-portal" "3004"

echo ""
echo "=== VERIFICACIÓN DE PÁGINAS CLAVE EN FRONTEND WEB ==="
cd ../arellan-frontend-web

for page in \
  "src/app/login/page.tsx" \
  "src/app/(dashboard)/page.tsx|src/app/dashboard/page.tsx" \
  "src/app/(dashboard)/orders/page.tsx|src/app/orders/page.tsx" \
  "src/app/(dashboard)/orders/new/page.tsx|src/app/orders/new/page.tsx" \
  "src/app/(dashboard)/inventory/page.tsx|src/app/inventory/page.tsx" \
  "src/app/(dashboard)/finance/page.tsx|src/app/finance/page.tsx" \
  "src/app/(dashboard)/personnel/page.tsx|src/app/personnel/page.tsx"; do
  
  # Verificar cualquiera de los paths posibles
  FOUND=false
  for p in $(echo $page | tr '|' ' '); do
    [ -f "$p" ] && FOUND=true && echo "✅ $p" && break
  done
  $FOUND || echo "❌ FALTA: $page"
done

echo ""
echo "=== VERIFICAR QUE PÁGINAS USAN API REAL (no mocks) ==="
# Buscar arrays hardcodeados en páginas del dashboard
HARDCODED=$(grep -r "const.*=.*\[\{" src/app/ 2>/dev/null | \
  grep -v "node_modules\|test\|spec\|\.d\.ts" | \
  grep -i "order\|client\|vehicle\|item\|invoice" | head -10)

if [ -z "$HARDCODED" ]; then
  echo "✅ No se detectaron arrays hardcodeados en páginas principales"
else
  echo "⚠️ Posibles datos hardcodeados detectados:"
  echo "$HARDCODED"
fi

echo ""
echo "=== VERIFICACIÓN MECHANIC UI ==="
cd ../arellan-mechanic-ui

for page in \
  "src/app/login/page.tsx" \
  "src/app/page.tsx" \
  "src/app/orders/page.tsx" \
  "src/app/check-in/page.tsx"; do
  [ -f "$page" ] && echo "✅ $page" || echo "❌ FALTA: $page"
done

# Verificar que CheckInPage llama al backend real
if [ -f "src/app/check-in/page.tsx" ]; then
  grep -q "api.post\|fetch\|axios" src/app/check-in/page.tsx && \
    echo "✅ CheckInPage hace llamada HTTP real" || \
    echo "❌ CheckInPage no hace llamada HTTP al backend"
  
  grep -q "check-in\|attendance" src/app/check-in/page.tsx && \
    echo "✅ CheckInPage referencia endpoint de attendance" || \
    echo "⚠️ CheckInPage puede no llamar al endpoint correcto"
fi

echo ""
echo "=== VERIFICACIÓN CLIENT PORTAL ==="
cd ../arellan-client-portal

for page in \
  "src/app/page.tsx" \
  "src/app/track/[orderNumber]/page.tsx|src/app/track/[number]/page.tsx" \
  "src/app/login/page.tsx" \
  "src/app/dashboard/page.tsx|src/app/(client)/dashboard/page.tsx" \
  "src/app/orders/page.tsx|src/app/(client)/orders/page.tsx"; do
  
  FOUND=false
  for p in $(echo $page | tr '|' ' '); do
    [ -f "$p" ] && FOUND=true && echo "✅ $p" && break
  done
  $FOUND || echo "❌ FALTA: $page"
done

echo ""
echo "=== NPM AUDIT FIX ==="
for dir in arellan-frontend-web arellan-mechanic-ui arellan-client-portal; do
  echo "--- $dir ---"
  cd ../$dir
  AUDIT=$(npm audit 2>&1 | tail -5)
  echo "$AUDIT"
  npm audit fix --force 2>&1 | tail -3
done
```

---

## BLOQUE 8 — VERIFICACIÓN DEL DESIGN SYSTEM

```bash
echo ""
echo "=== DESIGN SYSTEM ==="
cd ../arellan-design-system

# Verificar package.json
cat package.json | python3 -c "import sys,json; d=json.load(sys.stdin); print('Nombre:', d.get('name','❌ sin nombre')); print('Versión:', d.get('version','❌'))" 2>/dev/null
# ESPERADO: "@arellan-hnos-core-ecosystem/ui" o "@arellan/ui"

# Verificar componentes
echo ""
echo "Componentes disponibles:"
find . -name "*.tsx" -not -path "*/node_modules/*" -not -path "*/.next/*" | \
  grep -i "component\|Button\|Card\|Badge\|Input\|Modal\|Table\|Spinner\|Alert" | \
  head -20

# Verificar que es importable
cat src/index.ts 2>/dev/null | head -20 || \
  cat packages/ui/src/index.ts 2>/dev/null | head -20 || \
  echo "❌ No se encontró archivo index.ts de exportaciones"
```

---

## BLOQUE 9 — VERIFICAR AUDIT LOG (trazabilidad completa)

```bash
echo ""
echo "=== AUDITORÍA: verificar que todas las acciones quedan registradas ==="
TOKEN=$(curl -s -X POST http://localhost:3001/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"edgar@arellanautos.pe","password":"Arellan2026!"}' \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

AUDIT_LOGS=$(curl -s "$BASE/audit/logs?limit=20" \
  -H "Authorization: Bearer $TOKEN")

echo "Total de logs de auditoría:"
echo $AUDIT_LOGS | python3 -c "import sys,json; d=json.load(sys.stdin); items=d.get('data',d) if isinstance(d,dict) else d; print(f'Total: {len(items)} en esta página')" 2>/dev/null

echo ""
echo "Últimas 5 acciones auditadas:"
echo $AUDIT_LOGS | python3 -c "
import sys, json
d = json.load(sys.stdin)
items = d.get('data', d) if isinstance(d, dict) else d
for item in items[:5]:
    print(f\"  {item.get('createdAt','?')[:19]} | {item.get('action','?')} | {item.get('entity','?')} | {item.get('severity','INFO')}\")
" 2>/dev/null
# ESPERADO: ver acciones de login, creación de orden, pago, etc.

echo ""
echo "Alertas de seguridad generadas:"
SECURITY=$(curl -s "$BASE/audit/logs?severity=SECURITY_ALERT" \
  -H "Authorization: Bearer $TOKEN")
echo $SECURITY | python3 -c "
import sys, json
d = json.load(sys.stdin)
items = d.get('data', d) if isinstance(d, dict) else d
print(f'Total alertas de seguridad: {len(items)}')
for item in items[:3]:
    print(f\"  {item.get('action','?')}: {str(item.get('metadata',''))[:80]}\")
" 2>/dev/null
# ESPERADO: al menos 1 alerta por el pago de Yape personal del Escenario 4
```

---

## BLOQUE 10 — SCORECARD FINAL

Al terminar todos los bloques, genera este reporte:

```
═══════════════════════════════════════════════════════════════════
SCORECARD FINAL — ECOSISTEMA ARELLAN HNOS
Verificación integral Fases 1, 2 y 3
Fecha: [fecha de ejecución]
═══════════════════════════════════════════════════════════════════

INFRAESTRUCTURA
  Docker (postgres + redis + pgadmin) .... [✅ / ❌]
  Backend en :3001 ...................... [✅ / ❌ error: X]
  Swagger en /api/docs .................. [✅ N endpoints / ❌]

BASE DE DATOS (Fase 1)
  Modelos Prisma ........................ [✅ N/32 / ❌ faltan: X]
  Campos anti-fraude .................... [✅ / ❌]
  Migración aplicada .................... [✅ / ❌]
  Seed: usuarios ........................ [✅ N/7 / ❌]
  Seed: clientes ........................ [✅ N/10+ / ❌]
  Seed: vehículos ....................... [✅ N/20+ / ❌]
  Seed: inventario ...................... [✅ N/30+, N bajo stock / ❌]
  Seed: órdenes ......................... [✅ N/20+ / ❌]
  Passwords hasheados (bcrypt) .......... [✅ / ❌ CRÍTICO]

AUTENTICACIÓN (Fase 2)
  Login retorna access + refresh tokens . [✅ / ❌]
  Refresh token funciona ................ [✅ / ❌]
  Rate limiting 5 intentos .............. [✅ HTTP 429/403 / ❌]
  Sesiones en Redis ..................... [✅ / ❌]
  /auth/sessions (listar) ............... [✅ / ❌]
  /auth/logout-all ...................... [✅ / ❌]

ENDPOINTS BACKEND
  /orders/stats/summary ................. [✅ HTTP 200 / ❌ HTTP N]
  /orders/by-status/:status ............. [✅ / ❌]
  /inventory/low-stock .................. [✅ HTTP 200, N items / ❌]
  /inventory/valuation .................. [✅ HTTP 200 / ❌]
  /inventory/:id/reserve ................ [✅ / ❌]
  /finance/dashboard .................... [✅ HTTP 200 / ❌]
  /finance/cashflow ..................... [✅ / ❌]
  /personnel/attendance/check-in ........ [✅ / ❌]
  /personnel/vehicle-usage/authorize .... [✅ / ❌]
  /public/orders/number/:num/status ..... [✅ sin auth / ❌]

REGLAS ANTI-FRAUDE
  Yape personal → SECURITY_ALERT ........ [✅ / ❌]
  Gasto >S/500 → doble aprobación ....... [✅ PENDING / ❌ auto-aprobado]
  Gasto <S/100 → auto-aprobado .......... [✅ / ❌]
  Descuento >20% → aprobación OWNER ..... [✅ / ❌]
  Mecánico no accede a finanzas ......... [✅ HTTP 403 / ❌ HTTP 200]

WEBSOCKETS
  Socket.io handshake ................... [✅ / ❌]
  RealtimeGateway con 8+ métodos ........ [✅ N/8 / ❌]
  OrdersGateway eliminado (unificado) ... [✅ / ⚠️ aún existe]
  useRealtime.ts en frontend web ........ [✅ / ❌]

ESCENARIOS DE NEGOCIO
  1. Crear cliente + vehículo ........... [✅ / ❌]
  2. Crear orden de trabajo ............. [✅ WO-XXXX / ❌]
  3. Asignar mecánico ................... [✅ / ❌]
  4. Agregar repuesto + descuento inventario [✅ stock -1 / ❌]
  5. Cambiar estado de orden ............ [✅ IN_PROGRESS → COMPLETED / ❌]
  6. Registrar pago Yape del taller ..... [✅ sin alerta / ❌]
  7. Check-in de mecánico ............... [✅ / ❌]
  8. Alerta stock bajo .................. [✅ N items / ❌]
  9. Tracking público sin auth .......... [✅ sin datos sensibles / ❌]
  10. Dashboard financiero OWNER ......... [✅ KPIs con datos / ❌]
  11. Autorizar uso vehículo taller ...... [✅ / ❌ / ⚠️ sin flota en seed]
  12. Valorización de inventario ......... [✅ totalCostValue > 0 / ❌]

FRONTENDS
  frontend-web build ................... [✅ / ❌ error: X]
  frontend-web páginas con datos reales . [✅ N/11 / ⚠️ N con mocks]
  mechanic-ui build .................... [✅ / ❌]
  mechanic-ui check-in conectado ........ [✅ / ❌]
  client-portal build .................. [✅ / ❌]
  client-portal tracking public ......... [✅ / ❌]
  client-portal login + historial ....... [✅ / ❌]

AUDITORÍA
  AuditLog registra acciones ........... [✅ N logs / ❌]
  Alertas SECURITY_ALERT generadas ...... [✅ N alertas / ❌]

───────────────────────────────────────────────────────────────────
RESULTADO:
  ✅ Completado: N/50 ítems
  ❌ Falló:      N ítems
  ⚠️ Parcial:    N ítems

SISTEMA OPERATIVO PARA PRODUCCIÓN: [SÍ / NO — falta: X]

FIXES APLICADOS EN ESTA SESIÓN:
  1. [descripción del fix]
  2. ...
═══════════════════════════════════════════════════════════════════
```

---

**Empieza por el Bloque 0. Cada error que encuentres, corrígelo en el momento antes de pasar al siguiente bloque. El objetivo es terminar con el scorecard completo y el sistema funcionando.**
