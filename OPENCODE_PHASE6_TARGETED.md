# OPENCODE — FASE 6: CORRECCIONES QUIRÚRGICAS (basado en auditoría UI)
# Solo toca lo que el audit identificó. No modifiques nada más.

---

## CONTEXTO DEL PROYECTO

El ecosistema tiene 100+ archivos de UI ya construidos con datos reales.
El audit identificó exactamente qué falta. Esta sesión corrige solo eso.

**Design system en uso:**
- Paquete: `@arellan-hnos-core-ecosystem/ui` v0.2.2
- Color primario: `#1B3A6B` (navy blue) — NO usar otros colores primarios
- Secundario: `#F59E0B` (amber) | Acento: `#10B981` (emerald)
- Componentes: importar SIEMPRE desde `@arellan-hnos-core-ecosystem/ui`
- CSS tokens: `@arellan-hnos-core-ecosystem/ui/styles.css`
- Patrón: `class-variance-authority + Tailwind preset del DS`
- NO importar: shadcn/ui, Radix, Lucide, Material UI

**Regla absoluta:**
```
Antes de cada cambio → view del archivo primero.
Si el archivo tiene contenido real → str_replace, NUNCA reemplazar completo.
Si es un archivo nuevo → crearlo desde cero siguiendo el DS existente.
```

---

## CRÍTICO 1 — Crear `/orders/new` en frontend-web

Este es el único archivo nuevo importante. El botón "Nueva Orden" en `/orders` navega aquí y da 404.

Lee primero:
- `arellan-frontend-web/src/app/(admin)/orders/page.tsx` → ver cómo llama a este path
- `arellan-frontend-web/src/services/` → ver qué servicios existen para orders y clients
- `arellan-frontend-web/src/hooks/` → ver hooks disponibles (use-orders, use-clients, etc.)
- Un archivo de página existente (ej. `/clients/page.tsx`) → copiar el patrón de imports y estructura

Crea: `arellan-frontend-web/src/app/(admin)/orders/new/page.tsx`

**Wizard de 4 pasos — usar los componentes del DS existente:**

```typescript
'use client';
// Importar componentes desde @arellan-hnos-core-ecosystem/ui
// Usar el mismo patrón de imports que las otras páginas del proyecto
// Usar los hooks y servicios que ya existen en src/hooks/ y src/services/

// PASO 1 — Buscar o crear cliente
//   Input de búsqueda → llama a GET /clients?search=X (usar el hook useClients existente)
//   Resultados en lista: nombre, DNI, teléfono
//   Botón "Crear cliente" → formulario inline con: firstName, lastName, phone, dni
//   POST /clients → usar el servicio de clients existente

// PASO 2 — Seleccionar vehículo
//   Muestra vehículos del cliente: GET /vehicles?clientId=X
//   Cards de vehículo: placa + marca + modelo + año
//   Botón "Agregar vehículo" → modal: plate, brand, model, year, color
//   POST /vehicles

// PASO 3 — Datos de la orden
//   Campos requeridos:
//     description (textarea) — obligatorio
//     type: select ['CORRECTIVE','PREVENTIVE','DIAGNOSTIC','EMERGENCY']
//     priority: select ['NORMAL','HIGH','URGENT']
//     odometerIn: number
//     fuelLevel: select ['E','1/4','1/2','3/4','F']
//     assignedPersonnelId: select con GET /personnel?role=MECHANIC
//   Campos opcionales:
//     customerNotes (textarea)

// PASO 4 — Repuestos iniciales (opcional)
//   Search: GET /inventory?search=X (debounce 300ms)
//   Tabla de items seleccionados con qty y precio
//   Botón "Sin repuestos, continuar"

// SUBMIT:
//   POST /orders con todos los datos del wizard
//   Si éxito → router.push('/orders/' + newOrder.id)
//   Toast de confirmación: "Orden WO-XXXX creada correctamente"

// BARRA DE PROGRESO:
//   Indicador visual de 4 pasos en el header
//   Botones "Anterior" y "Siguiente" entre pasos
//   Validación con zod antes de avanzar (usar react-hook-form como las otras páginas)
```

---

## CRÍTICO 2 — Unificar API client duplicado en frontend-web

El audit detectó dos archivos que hacen lo mismo: `src/lib/api.ts` y `src/services/api-client.ts`.

```bash
# Paso 1: leer ambos archivos completos
view arellan-frontend-web/src/lib/api.ts
view arellan-frontend-web/src/services/api-client.ts

# Paso 2: ver cuál está siendo importado más
grep -r "from.*lib/api\|from.*api-client" \
  arellan-frontend-web/src/hooks/ \
  arellan-frontend-web/src/services/ \
  arellan-frontend-web/src/app/ 2>/dev/null | head -30
```

**Acción:** El que tenga más importaciones = el que se mantiene. El otro se elimina.
- En los archivos que importan el eliminado → `str_replace` para actualizar el import
- Verificar que el que se mantiene tiene: JWT interceptor + refresh automático + manejo de 401

---

## CRÍTICO 3 — Registrar CheckInPage en mechanic-ui

```bash
# Leer el router actual
view arellan-mechanic-ui/src/App.tsx
# (o el archivo de rutas equivalente en ese proyecto Vite)
```

Agregar la ruta con `str_replace`:
```typescript
// Buscar el bloque de rutas existente y agregar:
// import CheckInPage from './pages/CheckInPage';  (verificar la ruta exacta del import)
// <Route path="/check-in" element={<CheckInPage />} />

// También agregar link de navegación al dashboard si no hay forma de llegar a /check-in
// Ver cómo está estructurada la navegación en DashboardPage para seguir el mismo patrón
```

---

## IMPORTANTE 1 — CSS tokens en mechanic-ui

```bash
view arellan-mechanic-ui/src/styles.css
# (o el archivo CSS principal del proyecto)
```

Si no tiene el import del design system, agregar al inicio con `str_replace`:
```css
/* Agregar al inicio del archivo CSS principal */
@import '@arellan-hnos-core-ecosystem/ui/styles.css';
```

Verificar que el paquete está en `package.json` de mechanic-ui. Si no está instalado:
```bash
cd arellan-mechanic-ui && npm install @arellan-hnos-core-ecosystem/ui
```

---

## IMPORTANTE 2 — ThemeProvider y CSS tokens en client-portal

```bash
view arellan-client-portal/src/app/layout.tsx
view arellan-client-portal/src/app/globals.css
```

Si no tiene ThemeProvider ni tokens, agregar con `str_replace`:

```typescript
// En layout.tsx — envolver children con ThemeProvider del DS
// import { ThemeProvider } from '@arellan-hnos-core-ecosystem/ui'
// <ThemeProvider defaultTheme="system" storageKey="arellan-client-theme">
//   {children}
// </ThemeProvider>
```

```css
/* En globals.css — agregar al inicio si no está */
@import '@arellan-hnos-core-ecosystem/ui/styles.css';
```

---

## IMPORTANTE 3 — Unificar API en client-portal (3 patrones → 1)

El audit detectó fetch nativo, axios directo y lib/api.ts mezclados.

```bash
# Ver cuál es el cliente correcto del proyecto
view arellan-client-portal/src/lib/api.ts

# Ver qué archivos usan fetch o axios directo
grep -rn "fetch(\|axios\." arellan-client-portal/src/app/ | grep -v "node_modules"
```

Para cada archivo que use `fetch()` o `axios` directamente → `str_replace` para reemplazar la llamada por el cliente unificado `api.get()` / `api.post()`.

Agregar también manejo de error donde los `.catch` estén vacíos:
```typescript
// Reemplazar .catch(() => {}) por:
.catch((error) => {
  console.error('[client-portal] Error:', error.message);
  // mostrar estado de error al usuario si aplica
})
```

---

## IMPORTANTE 4 — Personnel page: asistencias y uso de vehículos

```bash
view arellan-frontend-web/src/app/(admin)/personnel/page.tsx
# El audit dice que importa hooks pero no los usa
```

La página ya importa los hooks relevantes. Con `str_replace`, activar las secciones:

```typescript
// 1. Activar la sección de asistencias del día:
//    Llamar: GET /personnel/attendance/today (hook ya existe)
//    Mostrar tabla: nombre | cargo | hora entrada | hora salida | estado
//    Estado: PRESENT=verde, ABSENT=rojo, LATE=naranja

// 2. Agregar sección de uso de vehículos del taller:
//    Llamar: GET /personnel/vehicle-usage/active
//    Llamar: GET /personnel/vehicle-usage/overdue
//    Si hay OVERDUE: banner de alerta con los vehículos tardíos
//    Tabla: empleado | vehículo | salida | retorno esperado | estado

// 3. Quitar los imports de react-hook-form y zod si no se usan en esta página
//    (o usarlos si hay un formulario que falta)
```

---

## IMPORTANTE 5 — Finance page: tabs de comisiones y dashboard histórico

```bash
view arellan-frontend-web/src/app/(admin)/finance/page.tsx
# El audit dice que tiene 2/4 tabs. Faltan: comisiones y dashboard histórico
```

Con `str_replace`, agregar los dos tabs que faltan:

```typescript
// TAB COMISIONES:
//   Datos: GET /finance/commissions
//   Tabla: Personal | Proveedor | Tipo | Monto | % | Estado | Fecha
//   Badge especial en rojo para type = 'IMPORT': "Importación"
//   Banner informativo: "Toda comisión de importación requiere aprobación de dos propietarios"
//   Filtros: por estado (PENDING/APPROVED/REJECTED), por personal

// TAB DASHBOARD HISTÓRICO:
//   Datos: GET /finance/cashflow
//   Gráfico de barras: ingresos vs egresos últimos 30 días
//   Usar el componente de gráfico que ya tenga el proyecto (ver si usa recharts, chart.js, etc.)
//   KPIs adicionales: promedio diario, mejor día, peor día

// Arreglar: window.location.href → router.push('/approvals')
// (usar el hook de Next.js router que ya usan otras páginas del proyecto)
```

---

## IMPORTANTE 6 — Alerta de Yape no autorizado

Esta es la alerta más importante del caso Arellan. Debe aparecer en dos lugares.

**En `/orders/[id]/page.tsx`** — la página más grande (493 líneas):
```bash
view arellan-frontend-web/src/app/(admin)/orders/[id]/page.tsx
# Buscar la sección de pagos
grep -n "payment\|Payment\|pago\|Pago\|yape\|Yape" \
  arellan-frontend-web/src/app/(admin)/orders/[id]/page.tsx | head -20
```

Con `str_replace`, agregar alerta en la sección de pagos:
```typescript
// Después de mostrar la lista de pagos, agregar:
// {payments.some(p => p.isPersonalYape) && (
//   <Alert variant="destructive" className="border-red-500 bg-red-50">
//     <AlertTitle>⚠️ Pago en cuenta Yape personal detectado</AlertTitle>
//     <AlertDescription>
//       {payments.filter(p => p.isPersonalYape).map(p => (
//         <div key={p.id}>
//           S/ {p.amount} recibido en cuenta {p.yapeAccount} por {p.receivedByName}
//         </div>
//       ))}
//       Verificar con el equipo de administración.
//     </AlertDescription>
//   </Alert>
// )}

// Usar el componente Alert que ya esté en uso en otras páginas del proyecto
// Mantener el mismo estilo visual que las otras alertas de la app
```

**En `/finance/page.tsx`** — sección de pagos:
```typescript
// En la sección que muestra transacciones o pagos, agregar un banner si hay pagos sospechosos:
// Datos: ya tiene acceso a pagos — verificar si isPersonalYape está en los datos que llegan
// Si no está → agregar ?includePaymentFlags=true al query o ajustar el hook
```

---

## MENOR 1 — Iconos PWA en mobile-app

```bash
ls arellan-mobile-app/public/icons/ 2>/dev/null
view arellan-mobile-app/public/manifest.json
```

Crear los iconos SVG mínimos para PWA (si el manifest los referencia y no existen):

```bash
# El manifest probablemente referencia icon-192x192.png e icon-512x512.png
# Crear SVG placeholder que se puede convertir a PNG:
# Un círculo navy (#1B3A6B) con la letra "A" en blanco en el centro
# Guardar en public/icons/icon-192x192.svg y public/icons/icon-512x512.svg
```

---

## MENOR 2 — Limpiar dead code en frontend-web

```bash
# Verificar antes de borrar que realmente no se usan
grep -r "useRealtime\|use-realtime\|use-order-socket" \
  arellan-frontend-web/src/app/ \
  arellan-frontend-web/src/components/ 2>/dev/null
```

Si los hooks `useRealtime.ts`, `use-realtime.ts` y `use-order-socket.ts` realmente no tienen ningún import:

**Opción A (recomendada):** No borrar — conectarlos al layout para habilitar WebSockets:
```typescript
// En arellan-frontend-web/src/app/(admin)/layout.tsx
// str_replace para agregar:
// import { useRealtime } from '@/hooks/useRealtime'; // o la ruta correcta
// 
// Dentro del layout component:
// useRealtime({
//   'order:status_changed': () => queryClient.invalidateQueries({ queryKey: ['orders'] }),
//   'inventory:low_stock': (data) => toast.warning(`Stock bajo: ${data.itemName}`),
//   'approval:requested': () => queryClient.invalidateQueries({ queryKey: ['approvals'] }),
//   'alert:security': (data) => toast.error(`Alerta: ${data.description}`, { duration: 0 }),
// });
```

**Opción B:** Si la arquitectura del proyecto usa polling y no se quiere cambiar ahora → dejar los hooks, no borrar ni conectar (solo comentar que existen para uso futuro).

---

## MENOR 3 — Swipe en mobile-app

```bash
view arellan-mobile-app/src/app/approvals/page.tsx
# Ver cómo están los botones de aprobar/rechazar actualmente
```

Si los botones ya funcionan (que sí — el audit lo confirma), agregar swipe es enhancement:

```typescript
// Con str_replace, en cada card de aprobación agregar handlers de touch:
// onTouchStart, onTouchEnd para detectar swipe horizontal
// Si swipe > 80px a la derecha → ejecutar la misma función que el botón "Aprobar"
// Si swipe > 80px a la izquierda → ejecutar la misma función que el botón "Rechazar"
// Agregar indicador visual: fondo verde sutil al deslizar derecha, rojo al deslizar izquierda
```

---

## MENOR 4 — Botón "Agregar cliente" en frontend-web

```bash
view arellan-frontend-web/src/app/(admin)/clients/page.tsx
# Ver cómo está el header de la página actualmente
```

Con `str_replace`, agregar al header:
```typescript
// El wizard de /orders/new ya tiene lógica de crear cliente
// En clients/page.tsx, agregar botón que abre modal inline:
// Formulario: firstName, lastName, phone, dni, email (opcional), district
// POST /clients → refetch de la lista
// Usar el mismo patrón de modal que tiene inventory/page.tsx (que sí tiene modal de agregar)
```

---

## VERIFICACIÓN — Solo los ítems tocados

```bash
# 1. Verificar que /orders/new existe y carga
cd arellan-frontend-web && npm run build 2>&1 | grep -E "error|Error|✓" | tail -20

# 2. Verificar que no hay imports rotos después de unificar API client
grep -r "from.*api-client\|from.*lib/api" arellan-frontend-web/src/ | head -10

# 3. Verificar que CheckInPage tiene ruta en mechanic-ui
grep -n "check-in\|CheckIn" arellan-mechanic-ui/src/App.tsx

# 4. Build de todos los productos
for dir in arellan-frontend-web arellan-mechanic-ui arellan-client-portal arellan-mobile-app; do
  echo "=== $dir ==="
  cd ../$dir
  npm run build 2>&1 | tail -3
done
```

---

## REPORTE FINAL

```
FIXES APLICADOS:
  CRÍTICO:
    ✅/❌ /orders/new creada (wizard 4 pasos)
    ✅/❌ API client unificado (api.ts o api-client.ts, no ambos)
    ✅/❌ CheckInPage registrada en App.tsx de mechanic-ui

  IMPORTANTE:
    ✅/❌ CSS tokens en mechanic-ui
    ✅/❌ ThemeProvider + CSS tokens en client-portal
    ✅/❌ API unificada en client-portal (1 patrón)
    ✅/❌ Personnel: asistencias + vehículos del taller
    ✅/❌ Finance: tabs comisiones + dashboard histórico
    ✅/❌ Alerta Yape en orders/[id] y finance

  MENOR:
    ✅/❌ Iconos PWA mobile-app
    ✅/❌ WebSocket conectado en layout (useRealtime)
    ✅/❌ Swipe en aprobaciones mobile
    ✅/❌ Botón agregar cliente

BUILDS:
  frontend-web  → ✅/❌
  mechanic-ui   → ✅/❌
  client-portal → ✅/❌
  mobile-app    → ✅/❌

ARCHIVOS MODIFICADOS: [lista de los archivos tocados con str_replace]
ARCHIVOS CREADOS: [solo los nuevos]
ARCHIVOS ELIMINADOS: [solo el duplicado de API client]
```

**No toques ningún archivo que no esté listado en este prompt.**
