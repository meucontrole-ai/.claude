# Skill: Performance

## Quando Ativar
- Otimizar bundle size (code splitting, tree-shaking)
- Implementar lazy loading de componentes e rotas
- Otimizar re-renders desnecessarios
- Otimizar carregamento de imagens e fonts
- Melhorar Web Vitals (LCP, CLS, INP)
- Implementar virtualizacao de listas longas
- Profiling de performance

## Quando NAO Ativar
- Criar componentes sem problemas de performance
- Otimizacao prematura sem metricas que justifiquem

---

## Code Splitting

### Lazy Import de Componentes

```tsx
import { lazy, Suspense } from 'react'

const HeavyChart = lazy(() => import('@/features/analytics/components/HeavyChart'))

function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart />
      </Suspense>
    </div>
  )
}
```

### Lazy Import Condicional

```tsx
import { lazy, Suspense, useState } from 'react'

const AdminPanel = lazy(() => import('@/features/admin/components/AdminPanel'))

function Dashboard() {
  const [showAdmin, setShowAdmin] = useState(false)

  return (
    <div>
      <button onClick={() => setShowAdmin(true)}>Abrir Admin</button>
      {showAdmin && (
        <Suspense fallback={<AdminPanelSkeleton />}>
          <AdminPanel />
        </Suspense>
      )}
    </div>
  )
}
```

### Lazy Loading de Rotas (React Router)

```tsx
// app/router.tsx
import { lazy, Suspense } from 'react'
import { createBrowserRouter } from 'react-router-dom'

const DashboardPage = lazy(() => import('./routes/dashboard/OverviewPage'))
const SettingsPage = lazy(() => import('./routes/dashboard/SettingsPage'))

const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <DashboardLayout />,
    children: [
      {
        index: true,
        element: (
          <Suspense fallback={<PageSkeleton />}>
            <DashboardPage />
          </Suspense>
        ),
      },
      {
        path: 'settings',
        element: (
          <Suspense fallback={<PageSkeleton />}>
            <SettingsPage />
          </Suspense>
        ),
      },
    ],
  },
])
```

---

## Memoizacao

### React.memo

Use APENAS quando:
- O componente renderiza frequentemente com as mesmas props
- O componente e pesado (muitos filhos ou calculos)
- Profiling confirmou re-renders desnecessarios

```tsx
import { memo } from 'react'

interface UserCardProps {
  name: string
  email: string
  avatarUrl: string
}

const UserCard = memo(function UserCard({ name, email, avatarUrl }: UserCardProps) {
  return (
    <div>
      <img src={avatarUrl} alt={name} />
      <h3>{name}</h3>
      <p>{email}</p>
    </div>
  )
})

export { UserCard }
```

### useMemo

Use para calculos pesados que dependem de poucos valores:

```tsx
import { useMemo } from 'react'

function OrderSummary({ items }: { items: OrderItem[] }) {
  const totals = useMemo(() => {
    const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0)
    const tax = subtotal * 0.1
    const total = subtotal + tax
    return { subtotal, tax, total }
  }, [items])

  return (
    <div>
      <p>Subtotal: {formatCurrency(totals.subtotal)}</p>
      <p>Imposto: {formatCurrency(totals.tax)}</p>
      <p>Total: {formatCurrency(totals.total)}</p>
    </div>
  )
}
```

### useCallback

Use APENAS quando a funcao e passada como prop para componente memoizado ou como dependencia de useEffect:

```tsx
import { useCallback } from 'react'

function ParentComponent() {
  const handleSelect = useCallback((id: string) => {
    // logica de selecao
  }, [])

  return <MemoizedList onSelect={handleSelect} />
}
```

### Quando NAO usar memoizacao

- Componentes simples que renderizam rapido
- Valores primitivos (strings, numbers, booleans) -- React ja otimiza
- Funcoes que nao sao passadas como props para componentes memoizados
- Sem evidencia de problema de performance (profiling)

---

## Otimizacao de Imagens

Em uma SPA com Vite, use o elemento `<img>` nativo com boas praticas:

```tsx
function ProductCard({ product }: { product: Product }) {
  return (
    <div>
      <img
        src={product.imageUrl}
        alt={product.name}
        width={400}
        height={300}
        loading="lazy"
        decoding="async"
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      />
      <h3>{product.name}</h3>
    </div>
  )
}
```

### Regras de Imagens

1. **SEMPRE** use `loading="lazy"` para imagens abaixo da dobra
2. **SEMPRE** defina `width` e `height` para evitar layout shift (CLS)
3. **SEMPRE** defina `sizes` para responsive images com `srcSet`
4. **REMOVA** `loading="lazy"` de imagens above-the-fold (LCP)
5. **NUNCA** carregue imagens maiores que o necessario
6. **USE** formatos modernos (WebP, AVIF) quando possivel

---

## Otimizacao de Fonts

```css
/* styles/globals.css */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-weight: 100 900;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, U+02C6, U+02DA, U+02DC, U+2000-206F;
}
```

Regras:
- **SEMPRE** use `font-display: swap` para evitar FOIT (Flash of Invisible Text)
- **SEMPRE** use `woff2` como formato preferencial
- **SEMPRE** faca preload de fonts criticas no `index.html`
- **SEMPRE** especifique `unicode-range` para carregar apenas caracteres necessarios

```html
<!-- index.html -->
<link rel="preload" href="/fonts/inter-var.woff2" as="font" type="font/woff2" crossorigin />
```

---

## Virtualizacao de Listas

Para listas com 100+ itens, use virtualizacao:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'
import { useRef } from 'react'

interface VirtualListProps<T> {
  items: T[]
  estimateSize: number
  renderItem: (item: T, index: number) => React.ReactNode
}

function VirtualList<T>({ items, estimateSize, renderItem }: VirtualListProps<T>) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => estimateSize,
    overscan: 5,
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {renderItem(items[virtualItem.index]!, virtualItem.index)}
          </div>
        ))}
      </div>
    </div>
  )
}

export { VirtualList }
```

---

## Debounce para Inputs de Busca

```typescript
// shared/hooks/useDebounce.ts
import { useEffect, useState } from 'react'

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])

  return debouncedValue
}

export { useDebounce }
```

Uso:
```tsx
function SearchInput() {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebounce(query, 300)

  const { data } = useSearch(debouncedQuery)

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />
}
```

---

## Prefetching

```tsx
// Prefetch de rota ao hover com React Router
import { Link } from 'react-router-dom'

function Navigation() {
  return (
    <nav>
      <Link to="/dashboard">Dashboard</Link>
      <Link to="/settings">Settings</Link>
    </nav>
  )
}
```

```typescript
// Prefetch de dados com TanStack Query
import { useQueryClient } from '@tanstack/react-query'
import { userKeys } from './userKeys'
import { userService } from '../services/userService'

function useUserPrefetch() {
  const queryClient = useQueryClient()

  function prefetchUser(id: string) {
    queryClient.prefetchQuery({
      queryKey: userKeys.detail(id),
      queryFn: () => userService.getById(id),
      staleTime: 60 * 1000,
    })
  }

  return { prefetchUser }
}
```

---

## Bundle Analysis

```bash
# Vite bundle analyzer
npx vite-bundle-visualizer
```

Ou instale como dependencia de desenvolvimento:

```bash
pnpm add -D rollup-plugin-visualizer
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true, gzipSize: true }),
  ],
})
```

---

## Regras de Performance

1. **NUNCA** otimize prematuramente -- meça primeiro com profiling
2. **SEMPRE** use `lazy()` + `Suspense` para code splitting de rotas e componentes pesados
3. **SEMPRE** use `loading="lazy"` para imagens abaixo da dobra
4. **SEMPRE** use `font-display: swap` e preload para fonts
5. **EVITE** re-renders desnecessarios com selectors (Zustand) e memo
6. **USE** code splitting para componentes pesados que nao sao above-the-fold
7. **USE** virtualizacao para listas com 100+ itens
8. **USE** debounce para inputs de busca que fazem API calls
9. **USE** prefetching para navegacao previsivel
10. **MONITORE** Web Vitals continuamente (LCP < 2.5s, CLS < 0.1, INP < 200ms)

---

## Checklist de Validacao

1. Bundle size analisado e sem dependencias desnecessarias?
2. Code splitting implementado para rotas e componentes pesados?
3. Imagens com loading="lazy", width/height definidos?
4. Fonts com font-display: swap e preload?
5. Listas longas virtualizadas?
6. Inputs de busca com debounce?
7. Re-renders desnecessarios eliminados (verificado com React DevTools Profiler)?
8. Web Vitals dentro dos thresholds (LCP < 2.5s, CLS < 0.1, INP < 200ms)?
9. Prefetching implementado para navegacao previsivel?
10. Vite bundle visualizer sem dependencias excessivas?

---

## React 18+: APIs de concorrência

### `startTransition` — marcar updates como não-urgentes

Search, filtros, mudança de abas — updates que o usuário aceita ver "depois" enquanto a UI fica responsiva:

```tsx
import { useTransition } from "react";

function Search() {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <input
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);              // update urgente (input)
          startTransition(() => {
            setResults(expensiveFilter(e.target.value));  // não-urgente
          });
        }}
      />
      {isPending && <Spinner />}
    </>
  );
}
```

### `useDeferredValue` — versão "atrasada" de um valor

Alternativa a `startTransition` quando o valor vem de props ou state remoto:

```tsx
const deferredQuery = useDeferredValue(query);
const results = useMemo(() => expensiveFilter(deferredQuery), [deferredQuery]);
// input responde imediato; results recomputam com leve atraso
```

### `Suspense` + code splitting por rota

```tsx
const Dashboard = lazy(() => import("./Dashboard"));

<Suspense fallback={<Skeleton />}>
  <Dashboard />
</Suspense>
```

Combine com `startTransition` na navegação pra o fallback não piscar em rotas que carregam rápido.

### `useSyncExternalStore` pra stores externos

Zustand, Redux vanilla, observables — em vez de subscribe manual em `useEffect` (que roda updates depois do render), `useSyncExternalStore` força render consistente:

```tsx
const selectedTab = useSyncExternalStore(
  tabStore.subscribe,
  tabStore.getSnapshot,
);
```

---

## Virtualização — quando acionar

Liste 100+ itens OU DOM altura fixa de 40+px cada = candidato pra virtualização (TanStack Virtual, `react-window`).

```tsx
const rowVirtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 52,
  overscan: 5,
});
```

Sem virtualização, listar 500 itens renderiza 500 DOM nodes toda vez que qualquer coisa muda — INP disparado.

---

## Memoização — só quando mede

- `useMemo` / `useCallback` têm custo (comparação de deps + hold de referência).
- Só aplique em callbacks que são **dependência de `useEffect` de filhos memorizados** ou valores computados que custam > 1ms.
- NÃO memoize tudo preventivamente — se a dep quebrar (referência mudando), o useMemo vira overhead sem ganho.

Ferramentas pra decidir:
- **React DevTools Profiler** → veja quais componentes renderizam mais do que deviam.
- **why-did-you-render** em dev → pega re-renders desnecessários.
