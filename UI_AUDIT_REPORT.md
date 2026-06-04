═══════════════════════════════════════════════════════════════════
AUDITORÍA DE UI — ECOSISTEMA ARELLAN HNOS
Fecha: 2026-06-04
═══════════════════════════════════════════════════════════════════

SISTEMA DE DISEÑO EN USO:
  Librería UI principal: Custom — @arellan-hnos-core-ecosystem/ui v0.2.2
  Colores definidos en: CSS tokens + Tailwind preset + TypeScript tokens
  Color primario detectado: #1B3A6B (navy blue)
  Secundario: #F59E0B (amber) | Acento: #10B981 (emerald)
  Componentes propios en: No hay src/components/ (todo viene del design system)
  Design system Arellan: Importado y usado en los 4 productos

───────────────────────────────────────────────────────────────────
FRONTEND-WEB (Panel Admin) — Next.js 14, 43 archivos fuente
───────────────────────────────────────────────────────────────────

LAYOUT Y AUTH:
  Layout dashboard: ✅ Existe en (admin)/layout.tsx (con sidebar, topbar, auth guard)
    -> Sidebar: ✅ Diseño propio con navegación
    -> Topbar: ✅ Barra superior con usuario
    -> Auth guard (redirect si no hay token): ✅
    -> WebSocket inicializado en layout: ❌
  Login page: ✅ Existe (227 líneas)
    -> Diseño visual: ✅ Completo (react-hook-form + zod, MFA flow)
    -> Conectado a backend real: ✅
    -> Manejo de error de credenciales: ✅

PÁGINAS PRINCIPALES:
  /dashboard (page.tsx): 276 líneas
    Estado: ✅ Completa
    Datos: Real (useDashboardStats, useCriticalStock, usePendingExpenses, useAuditLogs)
    KPIs: 4 KPIs visibles (órdenes activas, caja, aprobaciones, stock crítico)
    Diseño: Completo (grid + cards + tables + badges)
    Loading/Error states: ✅ (Spinner, Alert variant="error")

  /orders (page.tsx): 199 líneas
    Estado: ✅ Completa
    Datos: Real (useOrders -> GET /orders)
    Filtros: ✅ (status dropdown, search)
    Paginación: ✅ (DataTable + Pagination)
    Diseño: Completo
    ⚠️ BUG: Link a /orders/new pero la página no existe

  /orders/new (wizard): ❌ NO EXISTE
    El botón "Nueva Orden" en /orders navega a /orders/new pero la ruta no está creada

  /orders/[id] (detalle): 493 líneas (página más grande)
    Estado: ✅ Completa
    Timeline visible: ✅
    Pagos visibles: ✅
    Alerta Yape personal: ❌
    Diseño: Completo (3-columnas, status transitions modal, cancelación ConfirmDialog)

  /clients: 143 líneas
    Estado: ✅ Completa
    Datos: Real (useClients -> GET /clients)
    ⚠️ Sin botón "Agregar cliente" ni formulario de creación

  /inventory: 372 líneas
    Estado: ✅ Completa
    Datos: Real (useInventory + useCreatePart)
    Alertas de stock bajo visibles: ✅ (badges color-coded)
    Modal de agregar item: ✅ (react-hook-form + zod)

  /finance: 378 líneas
    Estado: ✅ Completa
    Datos: Real (caja, gastos, aprobaciones)
    Tabs: 2/4 (caja hoy + gastos pendientes, falta comisiones y dashboard histórico)
    Alerta de Yape no autorizado: ❌
    Control de aprobaciones: ✅ (link a /approvals)
    ⚠️ Usa window.location.href en vez de router.push para /approvals

  /personnel: 195 líneas
    Estado: ✅ Completa
    Datos: Real (usePersonnel -> GET /personnel)
    Asistencias del día: ❌ (importa hooks pero no los usa)
    Uso de vehículos del taller: ❌
    ⚠️ Importa react-hook-form y zod pero no los usa

  /audit: 226 líneas
    Estado: ✅ Completa
    Datos: Real (useAuditLogs -> GET /audit)
    Filtros: ✅ (action + entity)

COMPONENTES Y HOOKS:
  Componentes propios: Solo ToastContainer en src/app/components/
  Hooks: 14 hooks (use-orders, use-clients, use-inventory, use-finance, use-audit, use-personnel, 
         use-low-stock, use-vehicle-history, use-client-vehicles, use-finance-dashboard,
         use-order-socket, use-approval-socket, use-realtime, useRealtime)
  Servicios: 8 servicios en src/services/
  Stores: auth.ts (Zustand + persist), ui.ts (sidebar, theme, toast)
  ⚠️ API client DUPLICADO: api-client.ts (services) y api.ts (lib) — ambos hacen lo mismo
  ⚠️ useRealtime.ts y use-realtime.ts existen pero NUNCA se importan (dead code)

BUGS DETECTADOS:
  1. /orders/new no existe — link roto desde orders
  2. Dos instancias de Axios idénticas (api-client.ts y api.ts)
  3. Hooks de WebSocket definidos pero sin usar (useRealtime, use-order-socket)
  4. Personnel page importa form sin usarlo
  5. Finance usa window.location.href para navegación interna

───────────────────────────────────────────────────────────────────
MOBILE APP (PWA Gerencial) — Next.js 14, 20 archivos fuente
───────────────────────────────────────────────────────────────────
  Configurada como PWA: ✅ manifest.json + next-pwa config
    ⚠️ public/icons/ vacío — los iconos referenciados no existen en source
  Login: ✅ 186 líneas, MFA flow completo, conectado a backend
  Home / Dashboard móvil: ✅ 137 líneas, resumen ejecutivo (ingresos, OT activas, caja)
    Datos: Real (useExecutiveSummary -> /dashboard/summary, refetch 60s)
    KPIs: revenue + variación %, active OT, pending approvals, cashbox status
  Pantalla de aprobaciones: ✅ 188 líneas, approve/reject con mutaciones, 30s refetch
    -> Swipe para aprobar/rechazar: ❌ (no implementado)
  Pantalla de alertas: ✅ 153 líneas, severity badges, acknowledge button
  Perfil: ✅ 145 líneas, user info + logout con confirmación
  Bottom nav: ✅ 4 tabs (Dashboard, Aprobaciones, Alertas, Perfil)
  Diseño mobile-first: ✅ portrait-primary, safe-area-bottom, overscroll-none
  Touch targets: ✅ > 56px min-height
  ⚠️ useRealtime NO usado — polling en vez de WebSockets
  ⚠️ Dos sets de hooks duplicados (use-dashboard y use-approvals definen usePendingExpenses)
  ⚠️ Sin íconos de app en public/

───────────────────────────────────────────────────────────────────
MECHANIC UI (Tablet) — Vite + React SPA, 22 archivos fuente
───────────────────────────────────────────────────────────────────
  Login: ✅ 172 líneas, PIN de 6 dígitos (POST /auth/mechanic/login)
  Lista mis órdenes: ✅ 172 líneas (DashboardPage)
    -> Datos reales (solo órdenes del mecánico): ✅ (GET /orders/my, refetch 30s)
    -> Offline queue: ✅ (IndexedDB + sync service)
  Detalle de orden: ✅ 331 líneas (OrderDetailPage)
    -> Cambio de estado desde UI: ✅ (status flow: RECEIVED->IN_DIAGNOSIS->...->DELIVERED)
    -> ConfirmDialog antes de cambios: ✅
  Vehicle intake: ✅ 344 líneas (fotos + OCR simulado)
    ⚠️ OCR es fake (siempre "no detectada")
  Parts request: ✅ 266 líneas (catalogo + request con offline queue)
  Check-in / check-out: ✅ 215 líneas (CheckInPage)
    -> Conectado a backend: ✅ (POST /personnel/{id}/attendance/{type})
    -> ⚠️ PÁGINA HUÉRFANA: no está registrada en App.tsx (sin ruta)
  Mechanic progress: ✅ 251 líneas (WebSocket Socket.io real-time)
  Diseño tablet-first: ✅ (56px min touch targets, coarse-pointer media query, PWA config)
  Offline support: ✅ (IndexedDB queue, status/parts offline, sync on reconnect)
  ⚠️ CSS tokens del design system NO importados (styles.css)
  ⚠️ useRealtime NO usado (tiene useMechanicProgress con Socket.io)

───────────────────────────────────────────────────────────────────
CLIENT PORTAL — Next.js 14, 15 archivos fuente
───────────────────────────────────────────────────────────────────
  Tracking público (sin login): ✅ lookup, track/[orderNumber], status/[id]
    -> Muestra barra de progreso: ✅ TimelineProgress (7 pasos)
    -> Sin datos sensibles: ✅ verificado (solo placa, marca, modelo, estado)
    -> SSR optimizado: ✅ (status/[id] con getOrderStatus server-side + metadata dinámico)
  Login de cliente: ✅ 82 líneas (valida rol CLIENT)
    -> ⚠️ Usa axios directo, no lib/api.ts
  Dashboard cliente: ✅ 120 líneas (lista vehículos + últimas órdenes)
    -> ⚠️ Usa fetch() nativo en vez de lib/api.ts
  Historial de órdenes: ✅ 71 líneas (lista completa)
    -> ⚠️ Usa fetch() nativo
  Detalle de orden: ✅ 105 líneas con TimelineProgress
  ⚠️ 3 patrones de API diferentes (lib/api.ts, axios directo, fetch nativo)
  ⚠️ Sin ThemeProvider (sin dark mode)
  ⚠️ CSS tokens del design system NO importados
  ⚠️ Sin manejo de errores en páginas autenticadas (.catch vacío)
  ⚠️ useRealtime NO usado
  ⚠️ useOrderStatus definido pero nunca usado (dead code con polling 30s)

───────────────────────────────────────────────────────────────────
DISEÑO: Adopción del Design System por producto
───────────────────────────────────────────────────────────────────
  Capa                 | frontend-web | mechanic-ui | client-portal | mobile-app
  ---------------------|-------------|-------------|---------------|------------
  Tailwind preset      | ✅          | ✅          | ✅            | ✅
  CSS tokens           | ✅          | ❌          | ❌            | ✅
  ThemeProvider        | ✅ system   | ✅ system   | ❌            | ✅ light
  Componentes usados   | 20/24       | 17/24       | 9/24          | 14/24

───────────────────────────────────────────────────────────────────
RESUMEN DE HUECOS (lo que realmente falta)
───────────────────────────────────────────────────────────────────

CRÍTICO (sin esto el sistema no es usable):
  1. Página /orders/new NO EXISTE en frontend-web (botón roto)
  2. API client duplicado en frontend-web (2 instancias Axios idénticas)
  3. CheckInPage del mechanic-ui es huérfana (sin ruta en App.tsx)

IMPORTANTE (funcionalidad del negocio Arellan):
  1. CSS tokens faltan en mechanic-ui y client-portal (--color-brand-primary, etc)
  2. Sin dark mode en client-portal (falta ThemeProvider)
  3. 3 patrones de API inconsistentes en client-portal (fetch/axios/lib)
  4. Personnel page no muestra asistencias ni vehículos del taller
  5. Finance page no tiene tabs de comisiones ni dashboard histórico
  6. Sin alerta de Yape no autorizado en frontend-web ni finance page
  7. OCR de placas en mechanic-ui es fake (simulado)

MENOR (nice-to-have):
  1. public/icons/ vacío en mobile-app (íconos PWA no existen)
  2. Dead code: useRealtime.ts, use-realtime.ts, use-order-socket.ts sin usar
  3. Swipe para aprobar/rechazar no implementado en mobile-app
  4. Sin "Agregar cliente" en clients page del frontend-web
  5. Hardcodeo de font-family: system-ui en mechanic-ui (debiera usar Inter del DS)

NOTA DE COMPATIBILIDAD DE DISEÑO:
  "El proyecto usa un design system custom (@arellan-hnos-core-ecosystem/ui v0.2.2).
   Cualquier adición debe:
   - Usar las clases/tokens: brand-primary (#1B3A6B), brand-secondary (#F59E0B), 
     brand-accent (#10B981), status colors, neutral gray scale
   - Respetar el color primario: #1B3A6B (navy blue)
   - Seguir la convención de componentes: class-variance-authority + Tailwind
   - Importar componentes desde @arellan-hnos-core-ecosystem/ui
   - NO importar shadcn/ui, Radix, o Lucide (no están en el ecosistema)
   - Usar el Tailwind preset: @arellan-hnos-core-ecosystem/ui/tailwind
   - Importar CSS tokens: @arellan-hnos-core-ecosystem/ui/styles.css
   - Envolver con ThemeProvider para soporte de temas"

═══════════════════════════════════════════════════════════════════
NO SE MODIFICÓ NINGÚN ARCHIVO EN ESTA SESIÓN.
═══════════════════════════════════════════════════════════════════
