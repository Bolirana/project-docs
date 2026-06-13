# 🎨 Skill — Frontend WIN24 (Next.js)

> **Proyecto:** Bolirana WIN24 — Plataforma de apuestas deportivas  
> **Stack:** Next.js 16 · React 19 · TypeScript · Tailwind v4 · SWR  
> **Directorio:** `project-frontend/`

---

## 1. Arquitectura

El frontend sigue la arquitectura **App Router de Next.js** con separación clara entre Server Components y Client Components, y una estructura de carpetas orientada a responsabilidades:

```
project-frontend/
├── app/                    # Rutas (App Router)
│   ├── layout.tsx          # Root layout — fuentes, metadata, AppShell
│   ├── globals.css         # Tema Tailwind v4 (@theme inline), paleta WIN24
│   ├── page.tsx            # "/" Dashboard
│   ├── eventos/page.tsx    # "/eventos"
│   ├── apuestas/page.tsx   # "/apuestas"
│   ├── mercados/page.tsx   # "/mercados"
│   ├── movimientos/page.tsx# "/movimientos"
│   └── usuarios/page.tsx   # "/usuarios"
│
├── components/
│   ├── eventos/            # Componentes del dominio (MatchListing, Paises, PromoBanners)
│   ├── layout/             # Shell de la app (AppShell, Sidebar, TopNavbar, SubNavbar)
│   ├── ui/                 # Componentes genéricos (Button — shadcn base)
│   └── win/                # Átomos de dominio WIN24 (Modal, OddsButton, shared)
│
└── lib/
    ├── api.ts              # BASE_URL, fetcher, postJSON, helpers de formato
    ├── types.ts            # Tipos TypeScript del dominio (alineados al backend)
    └── utils.ts            # cn() (clsx + tailwind-merge)
```

### Flujo de datos

```
Backend Spring Boot :8080
         │
         │  HTTP (JSON)
         ▼
next.config.mjs rewrites()    ← proxy /api/* → localhost:8080/api/*
         │
         ▼
lib/api.ts  (fetcher / postJSON)
         │
         ▼
SWR cache  (useSWR por página)
         │
         ▼
Client Components  →  render UI
```

---

## 2. Páginas y componentes

### Mapa de rutas

| Ruta | Componente raíz | Endpoints consumidos |
|---|---|---|
| `/` | `app/page.tsx` | `GET /api/eventos` (via MatchListing) |
| `/eventos` | `app/eventos/page.tsx` | `GET /api/eventos`, `POST /api/eventos` |
| `/apuestas` | `app/apuestas/page.tsx` | `GET /api/apuestas`, `GET /api/usuarios`, `POST /api/apuestas` |
| `/mercados` | `app/mercados/page.tsx` | `GET /api/mercados`, `GET /api/eventos`, `POST /api/mercados` |
| `/movimientos` | `app/movimientos/page.tsx` | `GET /api/usuarios`, `GET /api/movimientos[/usuario/{id}]`, `POST /api/movimientos` |
| `/usuarios` | `app/usuarios/page.tsx` | `GET /api/usuarios`, `POST /api/usuarios` |

### Componentes principales

| Componente | Tipo | Descripción |
|---|---|---|
| `AppShell` | Server | Layout raíz: compone TopNavbar + SubNavbar + Sidebar + `<main>` |
| `Sidebar` | Client | Navegación + lista de eventos en vivo (useSWR) |
| `MatchListing` | Client | Tabla de partidos con botones de cuotas 1/X/2 |
| `Modal` | Client | Overlay reutilizable para todos los formularios CRUD |
| `shared.tsx` | — | Átomos: `PageHeader`, `FloatingAddButton`, `EmptyState`, `SkeletonRows`, `StatusBadge` |
| `OddsButton` | Client | Botón de cuota con estado seleccionado/hover |

---

## 3. Comunicación con el backend (lib/api.ts)

### Proxy (sin CORS en desarrollo)

```js
// next.config.mjs
async rewrites() {
  return [{
    source: '/api/:path*',
    destination: 'http://localhost:8080/api/:path*',
  }]
}
```

### API helpers

```ts
const BASE_URL = '/api'  // proxeado por Next.js → localhost:8080

// GET genérico (para SWR)
export async function fetcher<T>(url: string): Promise<T> {
  const res = await fetch(url)
  if (!res.ok) throw new Error(`Error ${res.status}`)
  return res.json()
}

// POST
export async function postJSON<T>(url: string, body: unknown): Promise<T> {
  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  })
  if (!res.ok) throw new Error(`Error ${res.status}`)
  return res.json()
}
```

### Patrón SWR en páginas CRUD

```tsx
'use client'
export default function EventosPage() {
  const { data, isLoading, mutate } = useSWR<Evento[]>('/api/eventos', fetcher)

  const handleCreate = async (form: Partial<Evento>) => {
    await postJSON('/api/eventos', form)
    mutate()  // revalida cache tras POST
  }

  if (isLoading) return <SkeletonRows />
  if (!data?.length) return <EmptyState />
  return <TablaEventos data={data} />
}
```

---

## 4. Tipos TypeScript (lib/types.ts)

Alineados 1:1 con las entidades del backend Spring Boot:

```ts
export type Evento = {
  id: number
  nombre: string
  deporte: string
  estado: string  // "ABIERTO" | "CERRADO" | "FINALIZADO"
}

export type Mercado = {
  id: number
  nombre: string
  descripcion: string
  evento: Evento
}

export type Usuario = {
  id: number
  nombre: string
  correo: string
  saldo: number
}

export type OpcionApuesta = {
  id: number
  cuotaActual: number
}

export type Apuesta = {
  id: number
  apostador: Usuario
  opcionApuesta: OpcionApuesta
  cuotaCongelada: number
  monto: number
  estado: string  // "PENDIENTE" | "GANADA" | "PERDIDA"
}

export type MovimientoSaldo = {
  id: number
  usuario: Usuario
  tipo: string    // "DEPOSITO" | "RETIRO"
  monto: number
}
```

---

## 5. Diseño y paleta WIN24

### Tema (globals.css — Tailwind v4)

```css
@theme inline {
  --color-win-bg:       #0d1117;   /* fondo principal */
  --color-win-card:     #1e2d3d;   /* tarjetas y paneles */
  --color-win-sidebar:  #111827;   /* sidebar */
  --color-win-navbar:   #1a2332;   /* top navbar */
  --color-win-border:   #2a3f55;   /* bordes */
  --color-win-gold:     #f5a623;   /* acento primario (CTAs, activos) */
  --color-win-cyan:     #00bfff;   /* acento secundario (highlights) */
  --color-win-text:     #ffffff;   /* texto principal */
  --color-win-muted:    #8b9ab0;   /* texto secundario */
}
```

### StatusBadge — colores por estado

```tsx
const STATUS_STYLES: Record<string, string> = {
  ABIERTO:    'bg-cyan-500/10   text-cyan-400',
  CERRADO:    'bg-gray-500/10   text-gray-400',
  FINALIZADO: 'bg-orange-500/10 text-orange-400',
  PENDIENTE:  'bg-yellow-500/10 text-yellow-400',
  GANADA:     'bg-green-500/10  text-green-400',
  PERDIDA:    'bg-red-500/10    text-red-400',
  DEPOSITO:   'bg-green-500/10  text-green-400',
  RETIRO:     'bg-red-500/10    text-red-400',
}
```

### Formato de dinero

```ts
export const formatCOP = (value: number): string =>
  new Intl.NumberFormat('es-CO', {
    style: 'currency', currency: 'COP', minimumFractionDigits: 0
  }).format(value)
// → "$ 250.000"
```

---

## 6. Convenciones de código

### Server vs Client Components

| Tipo | Cuándo usarlo | Ejemplos |
|---|---|---|
| Server Component (default) | Sin interactividad, sin hooks, sin estado | `AppShell`, `app/page.tsx`, `PromoBanners` |
| `'use client'` | Hooks, eventos, SWR, formularios | Todas las páginas CRUD, `Sidebar`, `MatchListing` |

### Naming

| Elemento | Convención | Ejemplo |
|---|---|---|
| Archivos | kebab-case | `match-listing.tsx`, `app-shell.tsx` |
| Componentes | PascalCase | `MatchListing`, `FloatingAddButton` |
| Helpers/funciones | camelCase | `fetcher`, `formatCOP`, `splitTeams` |
| Constantes de config | UPPER_CASE | `POPULAR`, `GESTION`, `STATUS_STYLES` |
| Tipos de dominio | PascalCase | `Evento`, `Apuesta`, `MovimientoSaldo` |

### Patrón loading / empty / data

```tsx
// Patrón fijo en todas las páginas CRUD:
if (isLoading)        return <SkeletonRows />
if (error)            return <ErrorState message={error.message} />
if (!data?.length)    return <EmptyState />
return                <TablaContenido data={data} />
```

### Composición de componentes privados

```tsx
// Subcomponentes no exportados dentro del mismo archivo (co-location)
function MatchRow({ evento }: { evento: Evento }) { ... }
function EventoItem({ evento }: { evento: Evento }) { ... }

export default function MatchListing() {
  return data.map(e => <MatchRow key={e.id} evento={e} />)
}
```

---

## 7. MUST have ✅ / SHOULD have 🔶

### MUST have — Reglas obligatorias

- [x] `'use client'` solo en componentes que realmente lo necesitan (hooks, eventos)
- [x] Tipos TypeScript para todas las entidades del dominio (`lib/types.ts`)
- [x] Un único punto de acceso a la API (`lib/api.ts`) — sin `fetch` directo en componentes
- [x] `fetcher` + `postJSON` con manejo de errores (`!res.ok → throw Error`)
- [x] Revalidación de cache tras mutaciones (`mutate()` después de `postJSON`)
- [x] Estado de carga con skeletons (`SkeletonRows`) — nunca pantalla en blanco
- [x] Estado vacío explícito (`EmptyState`) — nunca tabla vacía sin mensaje
- [x] Proxy en `next.config.mjs` para evitar CORS en desarrollo
- [x] Tipos de dominio en español alineados al backend
- [x] Componentes atómicos reutilizables en `components/win/shared.tsx`
- [x] Formateo de dinero centralizado en `formatCOP()` — nunca inline

### SHOULD have — Buenas prácticas recomendadas

- [ ] Usar variables CSS `--color-win-*` en lugar de hex literales (`bg-[#1e2d3d]`)
- [ ] Eliminar `typescript.ignoreBuildErrors: true` del `next.config.mjs`
- [ ] Extraer `formatHora` / `formatFecha` duplicadas a `lib/utils.ts`
- [ ] Reemplazar cuotas mock (`oddsFor`, `goalsOdds`) con datos reales de `OpcionApuesta`
- [ ] Agregar `loading.tsx` por ruta (App Router) para suspense automático
- [ ] `error.tsx` por ruta para manejo de errores granular
- [ ] Separar constantes de UI (`STATUS_STYLES`, `BANNERS`) a `lib/constants.ts`

---

## 8. Cómo ejecutar

```bash
# Requisitos: Node.js 18+, backend corriendo en :8080

cd project-frontend

# Instalar dependencias:
npm install

# Modo desarrollo:
npm run dev
# → http://localhost:3000

# Build de producción:
npm run build

# Verificar tipos:
npx tsc --noEmit
# → 0 errores
```

### Variables de entorno (opcional)

```bash
# .env.local — para apuntar a otro backend:
NEXT_PUBLIC_API_URL=http://localhost:8080
```
