# OPENCODE — AUDITORÍA DE UI (SOLO LECTURA)
# No modificar ningún archivo. Solo leer, analizar y reportar.

---

## MISIÓN

Lee todos los archivos de los 4 productos frontend. **No crees, no modifiques, no borres ningún archivo.**  
Tu único objetivo es generar un mapa exacto del estado actual del código de UI.

---

## REGLAS ABSOLUTAS DE ESTA SESIÓN

```
❌ PROHIBIDO crear nuevos archivos
❌ PROHIBIDO usar str_replace
❌ PROHIBIDO ejecutar npm install o npm run build
❌ PROHIBIDO modificar package.json
✅ SOLO puedes leer archivos (view, cat, ls, find, grep)
✅ SOLO puedes generar el reporte al final
```

---

## QUÉ LEER Y QUÉ DETECTAR EN CADA ARCHIVO

Para cada archivo `.tsx` o `.ts` que encuentres, responde estas preguntas:

**¿Tiene datos reales o mocks?**
- Real: usa un hook (`useQuery`, `useEffect` + `api.get`, `useSomething()`)
- Mock: tiene un array hardcodeado en el componente (ej. `const orders = [{ id: 1, ... }]`)
- Stub: el componente existe pero retorna `<div>TODO</div>` o similar

**¿Tiene diseño propio?**
- Sí: usa clases de Tailwind / CSS Modules / styled-components con estilos específicos
- Parcial: tiene estructura básica pero sin estilo visual definido
- No: solo estructura HTML sin estilos

**¿Está conectado al backend?**
- Sí: importa y usa el api-client o los hooks del proyecto
- No: hace fetch directo sin el cliente centralizado, o no hace fetch en absoluto

---

## LECTURA POR PRODUCTO

### PRODUCTO A — arellan-frontend-web

```bash
# Paso 1: ver estructura completa
find arellan-frontend-web/src -type f -name "*.tsx" -o -name "*.ts" | sort
find arellan-frontend-web/src/app -type f | sort

# Paso 2: detectar el sistema de diseño en uso
cat arellan-frontend-web/tailwind.config.ts 2>/dev/null || \
cat arellan-frontend-web/tailwind.config.js 2>/dev/null
cat arellan-frontend-web/src/app/globals.css 2>/dev/null | head -80

# Paso 3: ver qué colores/tokens usa el proyecto
grep -r "primary\|arellan\|--color\|theme\|brand" \
  arellan-frontend-web/src/app/globals.css \
  arellan-frontend-web/tailwind.config.ts 2>/dev/null | head -30

# Paso 4: leer cada página existente
# Para cada archivo .tsx en src/app/, leer y evaluar
```

Lee cada uno de estos archivos si existe y toma nota:

```
ARCHIVOS A REVISAR (frontend-web):
- src/app/(auth)/login/page.tsx          ó  src/app/login/page.tsx
- src/app/(dashboard)/layout.tsx         ó  src/app/layout.tsx
- src/app/(dashboard)/page.tsx           ó  src/app/dashboard/page.tsx
- src/app/(dashboard)/orders/page.tsx    ó  src/app/orders/page.tsx
- src/app/(dashboard)/orders/new/page.tsx
- src/app/(dashboard)/orders/[id]/page.tsx
- src/app/(dashboard)/clients/page.tsx
- src/app/(dashboard)/inventory/page.tsx
- src/app/(dashboard)/finance/page.tsx
- src/app/(dashboard)/personnel/page.tsx
- src/app/(dashboard)/audit/page.tsx
- src/components/ (listar todos los componentes propios)
- src/hooks/ (listar todos los hooks)
- src/services/ (listar todos los servicios)
- src/store/ (auth store)
- src/lib/api-client.ts
```

Para cada archivo encontrado, nota:
1. Cuántas líneas tiene
2. Si tiene datos reales o mocks
3. Si tiene diseño visual (Tailwind classes, colores, layout)
4. Si tiene estado de loading/error
5. Fragmento de las primeras 5 líneas del componente para ver el estilo de código

---

### PRODUCTO B — arellan-mobile-app

```bash
find arellan-mobile-app/src -type f -name "*.tsx" -o -name "*.ts" | sort 2>/dev/null
find arellan-mobile-app/app -type f | sort 2>/dev/null
ls arellan-mobile-app/public/ 2>/dev/null
cat arellan-mobile-app/package.json | grep -E '"name"|"dependencies"' 2>/dev/null | head -20
```

Archivos a revisar:
```
- src/app/login/page.tsx        ó  app/login/page.tsx
- src/app/page.tsx              (home — ¿qué muestra?)
- src/app/approvals/page.tsx    (aprobaciones pendientes)
- src/app/alerts/page.tsx
- manifest.json o manifest.webmanifest  (¿configurado como PWA?)
- public/icons/  (¿tiene iconos de app?)
```

---

### PRODUCTO C — arellan-mechanic-ui

```bash
find arellan-mechanic-ui/src -type f -name "*.tsx" -o -name "*.ts" | sort 2>/dev/null
```

Archivos a revisar:
```
- src/app/login/page.tsx
- src/app/page.tsx              (lista de órdenes del mecánico)
- src/app/orders/[id]/page.tsx  (detalle de orden)
- src/app/check-in/page.tsx     (check-in / check-out)
- ¿Es tablet-first? → revisar si usa clases de tamaño (text-xl, min-h-16, etc.)
```

---

### PRODUCTO D — arellan-client-portal

```bash
find arellan-client-portal/src -type f -name "*.tsx" -o -name "*.ts" | sort 2>/dev/null
```

Archivos a revisar:
```
- src/app/page.tsx                      (landing / buscador de orden)
- src/app/track/[orderNumber]/page.tsx  (resultado del tracking)
- src/app/login/page.tsx
- src/app/(client)/dashboard/page.tsx
- src/app/(client)/orders/page.tsx
```

---

### COMPONENTES PROPIOS DEL PROYECTO

```bash
# Detectar si el usuario tiene componentes propios (no del design system)
find arellan-frontend-web/src/components -type f 2>/dev/null | sort
find arellan-frontend-web/src/ui -type f 2>/dev/null | sort

# Detectar qué design system está siendo importado
grep -r "from '@arellan\|from '@/components\|from 'shadcn\|from '@radix" \
  arellan-frontend-web/src/app/ 2>/dev/null | head -20
```

---

## FORMATO DEL REPORTE

Al terminar la lectura, genera este reporte **exacto**. No escribas código. No hagas cambios.

```
════════════════════════════════════════════════════════════════════
AUDITORÍA DE UI — ECOSISTEMA ARELLAN HNOS
Fecha: [fecha]
════════════════════════════════════════════════════════════════════

SISTEMA DE DISEÑO EN USO:
  Librería UI principal: [Tailwind / shadcn/ui / design-system propio / otra]
  Colores definidos en: [globals.css / tailwind.config / tokens.ts / otra]
  Color primario detectado: [#XXXXX o nombre de token]
  Componentes propios en: [src/components/ / src/ui/ / no hay]
  Design system Arellan (@arellan-hnos): [Importado y usado / Instalado pero no usado / No instalado]

────────────────────────────────────────────────────────────────────
FRONTEND-WEB (Panel Admin)
────────────────────────────────────────────────────────────────────

LAYOUT Y AUTH:
  Layout dashboard: [✅ Existe (N líneas) / ❌ No existe / ⚠️ Stub]
    → Sidebar: [✅ Diseño propio / ⚠️ Básico / ❌ No hay]
    → Topbar: [✅ / ⚠️ / ❌]
    → Auth guard (redirect si no hay token): [✅ / ❌]
    → WebSocket inicializado en layout: [✅ / ❌]
  Login page: [✅ Existe (N líneas) / ❌ No existe / ⚠️ Stub]
    → Diseño visual: [✅ Completo y propio / ⚠️ Básico / ❌ Sin estilos]
    → Conectado a backend real: [✅ / ❌]
    → Manejo de error de credenciales: [✅ / ❌]

PÁGINAS PRINCIPALES:
  /dashboard (page.tsx):
    Estado: [✅ Completa / ⚠️ Parcial / ❌ Stub / ❌ No existe]
    Datos: [Real (api) / Mock (array hardcodeado) / Vacío]
    KPIs: [N KPIs visibles / Sin KPIs]
    Diseño: [Completo / Básico / Sin estilos]
    Loading/Error states: [✅ / ❌]
    Detalle: [descripción breve de qué muestra]

  /orders (page.tsx):
    Estado: [✅ / ⚠️ / ❌]
    Datos: [Real / Mock / Vacío]
    Filtros: [✅ Tiene / ❌ No tiene]
    Paginación: [✅ / ❌]
    Diseño: [Completo / Básico / Sin estilos]
    Detalle: [descripción breve]

  /orders/new (wizard):
    Estado: [✅ / ⚠️ / ❌]
    Pasos: [N pasos implementados]
    Conectado a backend: [✅ / ❌]
    Detalle: [descripción breve]

  /orders/[id] (detalle):
    Estado: [✅ / ⚠️ / ❌]
    Timeline visible: [✅ / ❌]
    Pagos visibles: [✅ / ❌]
    Alerta Yape personal: [✅ / ❌]
    Detalle: [descripción breve]

  /clients:
    Estado: [✅ / ⚠️ / ❌]
    Datos: [Real / Mock / Vacío]

  /inventory:
    Estado: [✅ / ⚠️ / ❌]
    Alertas de stock bajo visibles: [✅ / ❌]

  /finance:
    Estado: [✅ / ⚠️ / ❌]
    Tabs (dashboard/gastos/aprobaciones/comisiones): [N/4 implementados]
    Alerta de Yape no autorizado: [✅ / ❌]
    Control de aprobaciones: [✅ / ❌]

  /personnel:
    Estado: [✅ / ⚠️ / ❌]
    Asistencias del día: [✅ / ❌]
    Uso de vehículos del taller: [✅ / ❌]

  /audit:
    Estado: [✅ / ⚠️ / ❌]

────────────────────────────────────────────────────────────────────
MOBILE APP (PWA Gerencial)
────────────────────────────────────────────────────────────────────
  Configurada como PWA: [✅ manifest.json / ❌ No]
  Login: [✅ / ⚠️ / ❌]
  Home / Dashboard móvil: [✅ / ⚠️ / ❌]
  Pantalla de aprobaciones: [✅ / ⚠️ / ❌ No existe]
    → Swipe para aprobar/rechazar: [✅ / ❌]
  Pantalla de alertas: [✅ / ⚠️ / ❌ No existe]
  Diseño mobile-first (touch targets >= 56px): [✅ / ⚠️ / ❌]
  Detalle: [descripción breve de lo que existe]

────────────────────────────────────────────────────────────────────
MECHANIC UI (Tablet)
────────────────────────────────────────────────────────────────────
  Login: [✅ / ⚠️ / ❌]
  Lista mis órdenes: [✅ / ⚠️ / ❌]
    → Datos reales (solo órdenes del mecánico): [✅ / ❌ / mock]
  Detalle de orden: [✅ / ⚠️ / ❌]
    → Cambio de estado desde UI: [✅ / ❌]
    → Agregar repuestos: [✅ / ❌]
  Check-in / check-out: [✅ conectado a backend / ⚠️ existe pero sin conexión / ❌]
  Diseño tablet-first: [✅ / ⚠️ / ❌]
  Detalle: [descripción breve]

────────────────────────────────────────────────────────────────────
CLIENT PORTAL
────────────────────────────────────────────────────────────────────
  Tracking público (sin login): [✅ / ⚠️ / ❌]
    → Muestra barra de progreso de estados: [✅ / ❌]
    → Sin datos sensibles: [✅ verificado / ❌ / no revisado]
  Login de cliente: [✅ / ❌]
  Historial de órdenes del cliente: [✅ / ❌]
  Detalle: [descripción breve]

────────────────────────────────────────────────────────────────────
RESUMEN DE HUECOS (lo que realmente falta)
────────────────────────────────────────────────────────────────────

CRÍTICO (sin esto el sistema no es usable):
  1. [describir qué falta exactamente]
  2. ...

IMPORTANTE (funcionalidad del negocio Arellan):
  1. [describir]
  ...

MENOR (nice-to-have):
  1. [describir]
  ...

NOTA DE COMPATIBILIDAD DE DISEÑO:
  "El proyecto usa [X sistema de diseño]. Cualquier adición debe:
   - Usar las clases/tokens: [listar los detectados]
   - Respetar el color primario: [color detectado]
   - Seguir la convención de componentes: [patrón detectado]
   - NO importar librerías nuevas no usadas ya en el proyecto"

════════════════════════════════════════════════════════════════════
NO SE MODIFICÓ NINGÚN ARCHIVO EN ESTA SESIÓN.
════════════════════════════════════════════════════════════════════
```

---

## INSTRUCCIÓN FINAL

Después de entregar el reporte, **detente completamente**. No propongas soluciones. No empieces a implementar. El siguiente paso lo decide el usuario basado en este reporte.
