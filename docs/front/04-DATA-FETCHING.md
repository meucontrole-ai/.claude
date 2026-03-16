# Skill: Data Fetching

## Quando Ativar
- Integrar com APIs REST
- Configurar RTK Query (`createApi`, `fetchBaseQuery`)
- Implementar queries e mutations via RTK Query
- Implementar `createAsyncThunk` para operacoes imperativas (ex: sockets, fluxos nao-padrao)
- Implementar cache, tags de invalidacao, prefetching

## Quando NAO Ativar
- Gerenciar estado local/global (use State Management)
- Criar componentes de UI (use Components)
- Navegacao entre rotas (use react-router-dom)

---

## API Client (Axios)

```typescript
// src/api/index.ts (instancia axios existente)
import axios from 'axios'

export const merchantApi = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
})

merchantApi.interceptors.request.use((config) => {
  const token = getAuthToken()
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})
```

---

## RTK Query — Setup

### Base Query Reutilizavel

```typescript
// src/redux-store/services/base-query-config.ts
import { fetchBaseQuery } from '@reduxjs/toolkit/query/react'
import { getAuthToken, getBaseURL } from './api-utils'

export const createBaseQuery = () =>
  fetchBaseQuery({
    baseUrl: getBaseURL(),
    prepareHeaders: (headers) => {
      const token = getAuthToken()
      if (token) headers.set('Authorization', `Bearer ${token}`)
      headers.set('Content-Type', 'application/json')
      return headers
    },
  })
```

### Criar um API Slice

```typescript
// src/redux-store/services/example-api.ts
import { createApi } from '@reduxjs/toolkit/query/react'
import { createBaseQuery } from './base-query-config'

export const exampleApi = createApi({
  reducerPath: 'exampleApi',
  baseQuery: createBaseQuery(),
  tagTypes: ['Example'],
  endpoints: (builder) => ({
    getItems: builder.query<Item[], void>({
      query: () => `/${getEnterpriseId()}/items`,
      providesTags: ['Example'],
    }),
    createItem: builder.mutation<Item, CreateItemRequest>({
      query: (body) => ({
        url: `/${getEnterpriseId()}/items`,
        method: 'POST',
        body,
      }),
      invalidatesTags: ['Example'],
    }),
  }),
})

export const { useGetItemsQuery, useCreateItemMutation } = exampleApi
```

### Registrar no Store

```typescript
// src/redux-store/root-reducer.ts
import { exampleApi } from './services/example-api'

export const rootReducer = {
  // ... outros slices
  [exampleApi.reducerPath]: exampleApi.reducer,
}

// src/redux-store/store.ts
export const makeStore = () =>
  configureStore({
    reducer: rootReducer,
    middleware: (getDefaultMiddleware) =>
      getDefaultMiddleware().concat(exampleApi.middleware),
  })
```

---

## RTK Query — Uso nos Componentes

```tsx
import { useGetItemsQuery, useCreateItemMutation } from '@/redux-store/services/example-api'

function ItemsList() {
  const { data, isLoading, isError } = useGetItemsQuery()
  const [createItem, { isLoading: isCreating }] = useCreateItemMutation()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorState />
  if (!data?.length) return <EmptyState />

  return (
    <>
      {data.map((item) => <ItemCard key={item.id} item={item} />)}
    </>
  )
}
```

### Lazy Query (disparar manualmente)

```tsx
const [trigger, { data, isLoading }] = useLazyGetItemsQuery()

// Disparar quando necessario
trigger()
```

### Query com `queryFn` (multiplas requests ou logica customizada)

```typescript
getOrdersReport: builder.query<Result, Args>({
  keepUnusedDataFor: 3600,
  async queryFn(args, _api, _extraOptions, fetchWithBQ) {
    try {
      const result1 = await fetchWithBQ({ url: '...', method: 'GET' })
      if (result1.error) return { error: result1.error }

      const result2 = await fetchWithBQ({ url: '...', method: 'POST', body: {} })
      if (result2.error) return { error: result2.error }

      return { data: { ...result1.data, ...result2.data } }
    } catch {
      return { error: { status: 'CUSTOM_ERROR', error: 'Erro ao buscar dados' } }
    }
  },
  providesTags: ['Orders'],
}),
```

---

## createAsyncThunk — Operacoes Imperativas

Use `createAsyncThunk` para fluxos que **nao se encaixam** em RTK Query: sockets, logica de side-effects, operacoes sequenciais complexas.

```typescript
// src/redux-store/features/order/reducer/order-extra.reducer.ts
import { createAsyncThunk } from '@reduxjs/toolkit'
import { merchantApi } from '@/api'

export class OrderExtraReducers {
  static getOrders = createAsyncThunk('order/getOrders', async () => {
    const response = await merchantApi.get('/orders')
    return response.data
  })

  static putOrder = createAsyncThunk(
    'order/putOrder',
    async (order: { id: string; status: string }, { dispatch }) => {
      const { data } = await merchantApi.put(`/orders/${order.id}/status`, {
        status: order.status,
      })
      dispatch(updateOrder({ order: data }))
    },
  )
}
```

### Tratar no Slice

```typescript
// src/redux-store/features/order/order.slice.ts
import { createSlice } from '@reduxjs/toolkit'
import { OrderExtraReducers } from './reducer/order-extra.reducer'

export const orderSlice = createSlice({
  name: 'order',
  initialState,
  reducers: OrderReducer,
  extraReducers: (builder) => {
    builder
      .addCase(OrderExtraReducers.getOrders.pending, (state) => {
        state.status = 'loading'
      })
      .addCase(OrderExtraReducers.getOrders.fulfilled, (state, action) => {
        state.status = 'succeeded'
        state.data.mapOrders = action.payload
      })
      .addCase(OrderExtraReducers.getOrders.rejected, (state, action) => {
        state.status = 'failed'
        state.error = action.error.message ?? 'Erro desconhecido'
      })
  },
})
```

### Dispatch no Componente

```tsx
import { useAppDispatch, useAppSelector } from '@/redux-store/hooks'
import { OrderExtraReducers } from '@/redux-store/features/order/reducer/order-extra.reducer'

function OrdersPage() {
  const dispatch = useAppDispatch()
  const { status, data } = useAppSelector((state) => state.order)

  useEffect(() => {
    dispatch(OrderExtraReducers.getOrders())
  }, [dispatch])

  if (status === 'loading') return <Spinner />
  if (status === 'failed') return <ErrorState />

  return <OrderList orders={data.orders} />
}
```

---

## Typed Hooks

```typescript
// src/redux-store/hooks/index.ts
import { useDispatch, useSelector, useStore } from 'react-redux'
import type { RootState, AppDispatch, AppStore } from '../store'

export const useAppDispatch = useDispatch.withTypes<AppDispatch>()
export const useAppSelector = useSelector.withTypes<RootState>()
export const useAppStore = useStore.withTypes<AppStore>()
```

**SEMPRE use `useAppDispatch` e `useAppSelector`** — nunca os hooks genericos do react-redux diretamente.

---

## Store Provider

```tsx
// src/redux-store/StoreProvider.tsx
'use client'
import { useRef } from 'react'
import { Provider } from 'react-redux'
import { makeStore, AppStore } from './store'

export function StoreProvider({ children }: { children: React.ReactNode }) {
  const storeRef = useRef<AppStore | null>(null)
  if (!storeRef.current) storeRef.current = makeStore()
  return <Provider store={storeRef.current}>{children}</Provider>
}
```

---

## Tipos Padronizados

```typescript
// src/models/redux/type.ts
export type ReduxStateStatus = 'idle' | 'loading' | 'succeeded' | 'failed'

export type ReduxState<T> = {
  old: T | null
  data: T
  error: string
  status: ReduxStateStatus
}
```

---

## Regras de Data Fetching

1. **USE** RTK Query (`createApi`) como padrao para queries e mutations REST
2. **USE** `createAsyncThunk` apenas para fluxos imperativos (sockets, side-effects compostos)
3. **NUNCA** faca fetch diretamente em componentes — use hooks gerados pelo RTK Query ou dispatch de thunks
4. **SEMPRE** registre o reducer e middleware de cada `createApi` no store
5. **SEMPRE** use `providesTags` / `invalidatesTags` para manter o cache consistente
6. **SEMPRE** trate `isLoading`, `isError` e estado vazio nos componentes
7. **NUNCA** duplique server state em slices manuais quando RTK Query ja gerencia o cache
8. **SEMPRE** use `useAppDispatch` e `useAppSelector` — nunca os hooks genericos
9. **USE** `keepUnusedDataFor` para configurar TTL de cache por endpoint
10. **USE** `queryFn` customizado para requests multiplos ou logica condicional complexa

---

## Checklist de Validacao

1. `createApi` com `reducerPath`, `baseQuery` e `tagTypes` definidos?
2. Reducer e middleware registrados em `root-reducer.ts` e `store.ts`?
3. Hooks gerados exportados do arquivo de servico?
4. `isLoading`, `isError` e empty state tratados no componente?
5. Mutations usam `invalidatesTags` para invalidar cache relacionado?
6. Thunks com `createAsyncThunk` tratados no `extraReducers` do slice?
7. `useAppDispatch` / `useAppSelector` usados (nunca os genericos)?
8. `keepUnusedDataFor` configurado para endpoints com cache longo?
9. `queryFn` retorna `{ data }` ou `{ error }` — nunca lanca excecao diretamente?
10. `StoreProvider` envolve a arvore de componentes no ponto correto?
