# Correcciones realizadas - Clinica Automotriz Arellan Hnos

Fecha: 2026-06-03

## Resumen
Se analizaron los 7 productos del ecosistema (backend + 4 frontends + design system + infrastructure). Se corrigieron problemas de compatibilidad entre el design system (`arellan-design-system`) y los frontends (`arellan-frontend-web`, `arellan-mechanic-ui`, `arellan-mobile-app`, `arellan-client-portal`).

---

## 1. Backend (`arellan-platform`)

### 1.1 DTOs faltantes
- **`src/modules/personnel/dto/personnel.dto.ts`** (creado): `PersonnelFilterDto`, `UpdateRoleDto`, `UpdateStatusDto`
- **`src/modules/audit/dto/audit.dto.ts`** (creado): `AuditFilterDto`
- **Controllers actualizados**: `personnel.controller.ts` y `audit.controller.ts` para usar los nuevos DTOs con `@Query()` y `@Body()` validados

### 1.2 Estado: OK
- PostgreSQL + Redis: dependen de Docker Desktop funcionando
- Migraciones: ejecutadas (`prisma migrate deploy`)
- Seed: datos de prueba cargados
- TypeScript compila sin errores

---

## 2. Design System (`arellan-design-system`)

### 2.1 Vite multi-entry para tailwind
- **Archivo**: `packages/ui/vite.config.ts`
- **Problema**: `dist/tailwind/index.js` y `dist/tailwind/arellan-preset.js` no se generaban
- **Solución**: Cambio de entry único a entry múltiple en `build.lib.entry`

### 2.2 CSS no se generaba en dist
- **Archivo**: `packages/ui/src/index.ts`
- **Problema**: `dist/styles.css` no existía porque el CSS no se importaba en ningún entry point
- **Solución**: Agregado `import './styles.css'` en `src/index.ts` + `cssFileName: 'styles'` en vite config
- **Nota**: El archivo se genera como `dist/style.css` (singular), el package.json apunta correctamente

### 2.3 OrderStatusBadge valores incorrectos
- **Archivo**: `packages/ui/src/components/business/OrderStatusBadge.tsx`
- **Problema**: Usaba nombres en español (`RECIBIDO`, `EN_DIAGNOSTICO`) que no coinciden con lo que retorna el backend (Prisma enum: `RECEIVED`, `IN_DIAGNOSIS`)
- **Solución**: Alineado con Prisma enum values del backend

### 2.4 @types/react version mismatch
- **Archivo**: `packages/ui/package.json`
- **Problema**: `@types/react@18.3.29` (design system) vs `18.3.30` (frontend) causaba errores de tipo `ReactNode`
- **Solución**: Actualizado a `^18.3.30`

### 2.5 Deprecation de `version` en docker-compose.yml
- **Archivo**: `arellan-platform/docker-compose.yml`
- **Problema**: `version: "3.8"` es obsoleto
- **Solución**: Eliminada la línea

---

## 3. Admin Panel (`arellan-frontend-web`)

### 3.1 Variant types inválidos
- `variant="destructive"` → cambiado a `variant="error"` (Alert, Badge) o `variant="danger"` (Button, ConfirmDialog)
- `variant="secondary"` Badge → `variant="brand"`
- `variant="default"` Button → `variant="primary"`
- `size="icon"` Button → `size="sm"`

### 3.2 Select con children en lugar de options
- **Páginas**: audit, orders/[id], personnel, orders
- **Solución**: Convertido `<Select><option>...</option></Select>` a `<Select options={[...]} />`

### 3.3 DataTable sin keyExtractor
- **Páginas**: audit, personnel, clients, inventory, orders
- **Solución**: Agregado `keyExtractor={(row) => row.id}`

### 3.4 Pagination props incorrectos
- `currentPage` → `page` (en los 5 usos de Pagination)

### 3.5 OrderStatus valores incorrectos
- `src/types/index.ts`: Alineado con backend (RECEIVED, IN_DIAGNOSIS, etc.)
- `orders/page.tsx`: statusOptions actualizados
- `orders/[id]/page.tsx`: statusLabels y statusTransitions actualizados

### 3.6 ThemeProvider props incorrectos
- `providers.tsx`: Eliminados props `attribute`, `defaultTheme`, `enableSystem`, `disableTransitionOnChange`
- Usar `defaultScheme="system"` que es lo que el design system soporta

---

## 4. Mechanic UI (`arellan-mechanic-ui`)

### 4.1 OrderStatus valores incorrectos
- `src/types/index.ts`: Alineado con backend
- `OrderDetailPage.tsx`: STATUS_ACTIONS, flow map actualizados al nuevo flujo RECEIVED→IN_DIAGNOSIS→BUDGETED→IN_PROGRESS→IN_REVIEW→READY→DELIVERED
- `PartsRequestPage.tsx`: Filtro de órdenes activas actualizado (`COMPLETED` ya no existe)

### 4.2 StatusIndicator variant inválido
- `"online"` → `"active"`, `"offline"` → `"idle"` (o se mantiene `"offline"` donde ya es válido)
- Agregado `label` prop donde faltaba

### 4.3 Badge variant `"outline"` inválido → `"neutral"`

### 4.4 ConfirmDialog props incorrectos
- `message` → `description`
- `confirmText`/`cancelText` → `confirmLabel`/`cancelLabel`
- `onCancel` → `onClose`
- Agregado `open={true}`

### 4.5 Select children → options
- `PartsRequestPage.tsx` (2 Selects)
- `VehicleIntakePage.tsx` (1 Select)

### 4.6 IDB offline store
- **`stores/offline.ts`**: `db` era la función `getDb`, no la instancia. Corregido a `const db = await getDb()`

### 4.7 hooks/use-orders.ts
- `mechanicId` y `orderId` con valor `undefined` causaban error en array keys. Agregado `?? ""`

---

## 5. Mobile App (`arellan-mobile-app`)

### 5.1 Push notifications Uint8Array type
- **`notifications/push.ts`**: `new Uint8Array(rawData.length)` cambiado a `new Uint8Array(new ArrayBuffer(rawData.length))` para compatibilidad con TS 5.x

### 5.2 Estado: OK - TypeScript compila limpio

---

## 6. Client Portal (`arellan-client-portal`)

### 6.1 EmptyState action props
- `actionLabel` + `actionHref` → `action={<Link href="...">...</Link>}` (3 ocurrencias en `lookup/page.tsx`)

### 6.2 Estado: OK - TypeScript compila limpio

---

## 7. Advertencias de npm

### 7.1 `npm warn Unknown env config "devdir"`
- Es una configuración global de npm (`devdir = "D:\\IA\\node-gyp"`)
- No afecta al proyecto. Para eliminar: `npm config delete devdir`

### 7.2 Vulnerabilidades de seguridad
- `arellan-frontend-web`: 10 vulnerabilities (5 moderate, 4 high, 1 critical)
- `arellan-mechanic-ui`: 5 vulnerabilities (4 moderate, 1 critical)
- `arellan-mobile-app`: 7 vulnerabilities (1 moderate, 5 high, 1 critical)
- `arellan-client-portal`: 5 vulnerabilities (1 moderate, 4 high)
- Para resolver: `npm audit fix` en cada proyecto (puede requerir `--force`)

---

## 8. Pendientes / Problemas de entorno

### 8.1 Docker Desktop no inicia
- El daemon de Docker no arranca en esta máquina. PostgreSQL y Redis dependen de Docker.
- **Acción**: Iniciar Docker Desktop manualmente antes de levantar el backend.

### 8.2 Directorio artefacto en backend
- `src/common/{decorators,filters,guards,interceptors,pipes}/` - nombre con llaves literales, posible artefacto de creación

### 8.3 PWA icons mobile-app
- `public/icons/icon-192x192.png` y `icon-512x512.png` referenciados en manifest.json pero no encontrados en disco

---

## Comandos para levantar

```bash
# 1. Iniciar Docker Desktop manualmente

# 2. Infraestructura
cd arellan-platform
docker compose up -d

# 3. Backend (puerto 3000)
cd arellan-platform
npm run dev

# 4. Admin Panel (puerto 3001)
cd arellan-frontend-web
npm run dev

# 5. Mechanic UI (puerto 3002)
cd arellan-mechanic-ui
npm run dev

# 6. Mobile App (puerto 3003)
cd arellan-mobile-app
npm run dev

# 7. Client Portal (puerto 3004)
cd arellan-client-portal
npm run dev
```

## URLs locales
| Servicio | URL |
|---|---|
| Backend API | http://localhost:3000/api/v1 |
| Admin Panel | http://localhost:3001 |
| Mechanic UI | http://localhost:3002 |
| Mobile App | http://localhost:3003 |
| Client Portal | http://localhost:3004 |

## Usuarios de prueba
| Usuario | Email | Password | Rol |
|---|---|---|---|
| Edgar | edgar@arellan.pe | Arellan2026! | OWNER |
| Juan | juan@arellan.pe | Arellan2026! | OWNER |
| Ana | ana@arellan.pe | Arellan2026! | ADMIN |
| Finanzas | finanzas@arellan.pe | Arellan2026! | FINANCE |
| Mecanico | mecanico@arellan.pe | Arellan2026! | MECHANIC |
