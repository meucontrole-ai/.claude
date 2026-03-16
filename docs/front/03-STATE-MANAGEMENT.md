# Skill: State Management

## Quando Ativar
- Gerenciar estado global da aplicacao (auth, enterprise, config, sidebar)
- Criar ou estender slices Redux com `createSlice`
- Implementar logica de rollback / desfazer
- Implementar estado derivado ou computado via selectors
- Decidir entre estado local vs global vs server state
- Gerenciar estado de URL (search params)

## Quando NAO Ativar
- Buscar dados do servidor (use Data Fetching — RTK Query / createAsyncThunk)
- Estado local simples de um componente (`useState` basta)
- Criar componentes de UI (use Components)

---

## Hierarquia de Estado

| Tipo | Descricao | Solucao |
|---|---|---|
| **Local** | Estado de um unico componente (toggle, input) | `useState` |
| **Compartilhado** | Entre poucos componentes proximos | Lifting state / Context |
| **Global** | Acessado em toda a aplicacao (auth, enterprise, config) | Redux Toolkit (`createSlice`) |
| **Server** | Dados do servidor com cache (dashboard, reports) | RTK Query (`createApi`) |
| **URL** | Filtros, paginacao, busca refletidos na URL | `useSearchParams` (react-router-dom) |
| **Form** | Estado de formularios complexos | React Hook Form |

Regra: use a solucao **MAIS SIMPLES** que resolve o problema. Nao use Redux para o que `useState` resolve.

---

## Redux Toolkit — Arquitetura do Projeto

O projeto segue um padrao de 4 camadas por feature:

```
src/redux-store/features/<feature>/
  ├── <feature>.slice.ts          # Estado, reducers sincronos, extraReducers
  ├── <feature>.module.ts         # Conecta UseCases com AsyncThunks
  ├── reducer/
  │   ├── <feature>.reducer.ts        # Reducers sincronos (fabrica)
  │   └── <feature>-extra.reducer.ts  # AsyncThunks (createAsyncThunk)
  └── use-cases/
      ├── index.ts                    # Classe base abstrata
      ├── init.usecases.ts            # UseCase fulfilled/pending/rejected
      └── save.usecases.ts
```

---

## Tipo Base: ReduxState\<T\>

```typescript
// src/models/redux/type.ts
export type ReduxStateStatus = 'idle' | 'loading' | 'succeeded' | 'failed'

export type ReduxState<T> = {
  old: T | null       // snapshot para rollback
  data: T
  error: string
  status: ReduxStateStatus
}
```

---

## 1. Slice

```typescript
// src/redux-store/features/example/example.slice.ts
import { createSlice } from '@reduxjs/toolkit'
import { ReduxState } from '@/models/redux/type'
import { ExampleModule } from './example.module'
import { ExampleReducer } from './reducer/example.reducer'

export type StateExample = {
  items: ExampleItem[]
  selectedId: string | null
}

export const initialStateExample: ReduxState<StateExample> = {
  old: null,
  error: '',
  status: 'idle',
  data: {
    items: [],
    selectedId: null,
  },
}

export const exampleSlice = createSlice({
  name: 'example',
  initialState: initialStateExample,
  reducers: ExampleReducer,
  extraReducers: ExampleModule,
})

export const { setExample, rollbackExample } = exampleSlice.actions
```

### combineSlices (features com sub-dominios)

```typescript
// src/redux-store/features/config/config.slice.ts
import { combineSlices } from '@reduxjs/toolkit'

export const configSlice = combineSlices(
  geralSlice,
  themeSlice,
  paymentSlice,
  openingHoursSlice,
)
```

---

## 2. Reducers Sincronos (fabrica)

```typescript
// src/redux-store/features/example/reducer/example.reducer.ts
import { PayloadAction } from '@reduxjs/toolkit'
import { StateExample } from '../example.slice'
import { ReduxState } from '@/models/redux/type'

export const ExampleReducer = () => {
  const setExample = (
    state: ReduxState<StateExample>,
    action: PayloadAction<Partial<StateExample>>,
  ) => {
    if (!state.old) state.old = { ...state.data }  // snapshot para rollback
    state.data = { ...state.data, ...action.payload }
  }

  const rollbackExample = (state: ReduxState<StateExample>) => {
    if (state.old) state.data = state.old
    state.old = null
  }

  return { setExample, rollbackExample }
}
```

**Padrao rollback**: salve `state.old` antes de qualquer mutacao quando a operacao precisa de confirmacao. Use `rollback*` para desfazer.

---

## 3. AsyncThunks (ExtraReducers)

```typescript
// src/redux-store/features/example/reducer/example-extra.reducer.ts
import { createAsyncThunk } from '@reduxjs/toolkit'
import { merchantApi } from '@/api'
import { RootState } from '@/redux-store/store'

export class ExampleExtraReducers {
  static init = createAsyncThunk('example/init', async () => {
    const { data } = await merchantApi.get('/example')
    return data
  })

  static save = createAsyncThunk('example/save', async (_, { getState }) => {
    const state = getState() as RootState
    const payload = state.example.data
    await merchantApi.patch('/example', payload)
  })
}
```

---

## 4. UseCases (transicoes de estado)

```typescript
// src/redux-store/features/example/use-cases/index.ts
import { ReduxUseCases } from '@/models/redux/type'

export abstract class ExampleUseCases extends ReduxUseCases {
  pending(state: any): void {
    state.status = 'loading'
    state.error = ''
  }
  rejected(state: any, action: any): void {
    state.status = 'failed'
    state.error = action.error?.message ?? 'Erro desconhecido'
  }
  abstract fulfilled(state: any, action: any): void
}

// src/redux-store/features/example/use-cases/init.usecases.ts
export class InitExampleUseCase extends ExampleUseCases {
  fulfilled(state: any, action: any): void {
    state.status = 'succeeded'
    state.error = ''
    state.data = action.payload
  }
}
```

---

## 5. Module (conecta tudo)

```typescript
// src/redux-store/features/example/example.module.ts
import { ReduxModule } from '@/models/redux/type'
import { combineUseCasesWithExtraReducers } from '@/models/redux/utils'
import { ExampleExtraReducers } from './reducer/example-extra.reducer'
import { InitExampleUseCase } from './use-cases/init.usecases'

export const initExample = ExampleExtraReducers.init
export const saveExample = ExampleExtraReducers.save

const init = combineUseCasesWithExtraReducers({
  useCase: new InitExampleUseCase(),
  extraReducers: initExample,
})

export const ExampleModule: ReduxModule<any> = (builder) => {
  builder.addCase(init.fulfilled.extra, init.fulfilled.useCase)
  builder.addCase(init.pending.extra, init.pending.useCase)
  builder.addCase(init.rejected.extra, init.rejected.useCase)
}
```

---

## 6. Registrar no Store

```typescript
// src/redux-store/root-reducer.ts
import { exampleSlice } from './features/example/example.slice'

export const rootReducer = {
  // ... outros
  example: exampleSlice.reducer,
}
```

---

## Uso nos Componentes

```tsx
import { useAppDispatch, useAppSelector } from '@/redux-store/hooks'
import { initExample, saveExample } from '@/redux-store/features/example/example.module'
import { rollbackExample } from '@/redux-store/features/example/example.slice'

function ExamplePage() {
  const dispatch = useAppDispatch()
  const { data, status, error } = useAppSelector((state) => state.example)

  useEffect(() => {
    dispatch(initExample())
  }, [dispatch])

  const handleSave = () => dispatch(saveExample())
  const handleCancel = () => dispatch(rollbackExample())

  if (status === 'loading') return <Spinner />
  if (status === 'failed') return <ErrorState message={error} />

  return <ExampleForm data={data} onSave={handleSave} onCancel={handleCancel} />
}
```

**SEMPRE use `useAppDispatch` e `useAppSelector`** — nunca os hooks genericos do react-redux.

---

## useState (Estado Local)

```tsx
function PasswordInput() {
  const [isVisible, setIsVisible] = useState(false)

  return (
    <div>
      <input type={isVisible ? 'text' : 'password'} />
      <button type="button" onClick={() => setIsVisible((prev) => !prev)}>
        {isVisible ? 'Ocultar' : 'Mostrar'}
      </button>
    </div>
  )
}
```

- SEMPRE use callback form para updates baseados em estado anterior: `setState(prev => !prev)`
- NUNCA mute estado diretamente
- Separe estados independentes em variaveis distintas

---

## Context (Estado Compartilhado com Escopo)

Use para estado que precisa ser compartilhado em uma subarvore especifica (nao global):

```tsx
import { createContext, useContext, useState, useMemo, type ReactNode } from 'react'

interface ThemeContextValue {
  theme: 'light' | 'dark'
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextValue | null>(null)

function useTheme() {
  const context = useContext(ThemeContext)
  if (!context) throw new Error('useTheme must be used within ThemeProvider')
  return context
}

function ThemeProvider({ children, defaultTheme = 'light' }: { children: ReactNode; defaultTheme?: 'light' | 'dark' }) {
  const [theme, setTheme] = useState<'light' | 'dark'>(defaultTheme)

  const value = useMemo<ThemeContextValue>(
    () => ({ theme, toggleTheme: () => setTheme((prev) => (prev === 'light' ? 'dark' : 'light')) }),
    [theme],
  )

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
}
```

- **SEMPRE** retorne erro explicito se Context for usado fora do Provider
- **SEMPRE** memoize o value com `useMemo`
- **NUNCA** use Context para estado que muda frequentemente (use Redux)

---

## Estado de URL (Search Params)

Para estado que deve ser refletido na URL (filtros, paginacao, busca):

```tsx
import { useSearchParams, useNavigate, useLocation } from 'react-router-dom'
import { useCallback } from 'react'

function useQueryParams() {
  const [searchParams] = useSearchParams()
  const navigate = useNavigate()
  const { pathname } = useLocation()

  const setQueryParam = useCallback(
    (key: string, value: string | null) => {
      const params = new URLSearchParams(searchParams.toString())
      if (value === null) {
        params.delete(key)
      } else {
        params.set(key, value)
      }
      navigate(`${pathname}?${params.toString()}`, { replace: true })
    },
    [searchParams, navigate, pathname],
  )

  return {
    searchParams,
    setQueryParam,
    getParam: (key: string) => searchParams.get(key),
  }
}
```

Use estado de URL quando:
- O estado deve ser compartilhavel via link (filtros, busca)
- O estado deve sobreviver a navegacao back/forward

---

## Quando Usar o Que

| Cenario | Solucao |
|---|---|
| Toggle de modal/dropdown | `useState` |
| Formulario com validacao | React Hook Form |
| Autenticacao (user, token) | Redux slice (`authSlice`) |
| Empresa selecionada | Redux slice (`enterpriseSlice`) |
| Configuracoes da loja | Redux slice (`configSlice`) com `combineSlices` |
| Dados do servidor com cache | RTK Query (`createApi`) |
| Filtros de listagem | URL search params (`useSearchParams`) |
| Tema por subarvore | Context |
| Estado local efemero | `useState` |

---

## Regras de State Management

1. **USE** Redux Toolkit (`createSlice`) para estado global — nao Jotai ou outras libs (legado em `src/store/`)
2. **SIGA** a arquitetura de 4 camadas: Slice → Module → UseCase → ExtraReducer
3. **USE** `ReduxState<T>` como wrapper padrao do estado da slice
4. **IMPLEMENTE** o padrao `old`/rollback sempre que a mutacao precisar de confirmacao
5. **NUNCA** armazene server state em slices manuais quando RTK Query gerencia o cache
6. **SEMPRE** use `useAppDispatch` e `useAppSelector` — nunca os hooks genericos
7. **SEPARE** reducers sincronos (fabrica de funcoes) dos thunks (ExtraReducers)
8. **USE** `combineSlices` para features com sub-dominios relacionados
9. **NUNCA** mute estado diretamente — Immer (incluido no RTK) ja trata isso no `createSlice`
10. **USE** Context apenas para estado compartilhado dentro de uma subarvore, nunca global

---

## Checklist de Validacao

1. Tipo de estado identificado corretamente (local/global/server/URL)?
2. Solucao mais simples escolhida?
3. `ReduxState<T>` usado como wrapper na slice?
4. Padrao rollback implementado (`state.old`) quando necessario?
5. Reducers sincronos em arquivo separado (`<feature>.reducer.ts`)?
6. AsyncThunks em `<feature>-extra.reducer.ts`?
7. UseCases implementam `fulfilled`, `pending`, `rejected`?
8. Module conecta UseCases com thunks via `combineUseCasesWithExtraReducers`?
9. Slice registrada no `root-reducer.ts`?
10. `useAppDispatch`/`useAppSelector` usados (nunca os genericos)?
11. Server state gerenciado com RTK Query (nao em slice manual)?
12. Estado de URL com `useSearchParams` do react-router-dom?
