# Skill: Routing

## Quando Ativar
- Configurar rotas e paginas (React Router)
- Implementar layouts compartilhados e aninhados
- Implementar loading e error states por rota
- Implementar protecao de rotas (auth guards)
- Configurar rotas lazy-loaded

## Quando NAO Ativar
- Logica de negocio (use Components/State Management)
- Data fetching sem relacao com rotas (use Data Fetching)
- Estilizacao (use Styling)

---

## Estrutura de Rotas (React Router)

### Configuracao do Router

```tsx
// app/router.tsx
import { createBrowserRouter } from 'react-router-dom'
import { lazy, Suspense } from 'react'
import { RootLayout } from './layouts/RootLayout'
import { DashboardLayout } from './layouts/DashboardLayout'
import { AuthLayout } from './layouts/AuthLayout'
import { ProtectedRoute } from '@/shared/components/auth/ProtectedRoute'
import { ErrorPage } from './routes/ErrorPage'
import { PageSkeleton } from '@/shared/components/feedback/Skeleton'

const LoginPage = lazy(() => import('./routes/auth/LoginPage'))
const RegisterPage = lazy(() => import('./routes/auth/RegisterPage'))
const OverviewPage = lazy(() => import('./routes/dashboard/OverviewPage'))
const SettingsPage = lazy(() => import('./routes/dashboard/SettingsPage'))
const UsersPage = lazy(() => import('./routes/dashboard/UsersPage'))
const UserDetailPage = lazy(() => import('./routes/dashboard/UserDetailPage'))

function withSuspense(Component: React.LazyExoticComponent<() => JSX.Element>) {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Component />
    </Suspense>
  )
}

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      {
        element: <AuthLayout />,
        children: [
          { path: 'login', element: withSuspense(LoginPage) },
          { path: 'register', element: withSuspense(RegisterPage) },
        ],
      },
      {
        element: (
          <ProtectedRoute>
            <DashboardLayout />
          </ProtectedRoute>
        ),
        children: [
          { path: 'overview', element: withSuspense(OverviewPage) },
          { path: 'settings', element: withSuspense(SettingsPage) },
          { path: 'users', element: withSuspense(UsersPage) },
          { path: 'users/:id', element: withSuspense(UserDetailPage) },
        ],
      },
    ],
  },
])

export { router }
```

---

## Layouts

### Root Layout

```tsx
// app/layouts/RootLayout.tsx
import { Outlet } from 'react-router-dom'
import { SkipLink } from '@/shared/components/layout/SkipLink'

function RootLayout() {
  return (
    <div className="bg-surface text-content antialiased">
      <SkipLink />
      <Outlet />
    </div>
  )
}

export { RootLayout }
```

### Layout de Dashboard (Aninhado)

```tsx
// app/layouts/DashboardLayout.tsx
import { Outlet } from 'react-router-dom'
import { Header } from '@/shared/components/layout/Header'
import { Sidebar } from '@/shared/components/layout/Sidebar'

function DashboardLayout() {
  return (
    <div className="flex min-h-screen">
      <Sidebar />
      <div className="flex flex-1 flex-col">
        <Header />
        <main id="main-content" className="flex-1 p-6">
          <Outlet />
        </main>
      </div>
    </div>
  )
}

export { DashboardLayout }
```

### Auth Layout

```tsx
// app/layouts/AuthLayout.tsx
import { Outlet } from 'react-router-dom'

function AuthLayout() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="w-full max-w-md">
        <Outlet />
      </div>
    </div>
  )
}

export { AuthLayout }
```

---

## Rotas Dinamicas

```tsx
// app/routes/dashboard/UserDetailPage.tsx
import { useParams } from 'react-router-dom'
import { useUser } from '@/features/users/hooks/useUsers'

function UserDetailPage() {
  const { id } = useParams<{ id: string }>()
  const { data: user, isLoading, error } = useUser(id!)

  if (isLoading) return <UserDetailSkeleton />
  if (error) return <ErrorMessage error={error} />
  if (!user) return <NotFoundMessage />

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}

export default UserDetailPage
```

---

## Protecao de Rotas

```tsx
// shared/components/auth/ProtectedRoute.tsx
import { Navigate, useLocation } from 'react-router-dom'
import { useAuthStore } from '@/features/auth/stores/authStore'
import type { ReactNode } from 'react'

interface ProtectedRouteProps {
  children: ReactNode
}

function ProtectedRoute({ children }: ProtectedRouteProps) {
  const isAuthenticated = useAuthStore((state) => state.isAuthenticated)
  const { pathname } = useLocation()

  if (!isAuthenticated) {
    return <Navigate to={`/login?redirect=${pathname}`} replace />
  }

  return children
}

export { ProtectedRoute }
```

### Guest Route (redireciona autenticados)

```tsx
// shared/components/auth/GuestRoute.tsx
import { Navigate } from 'react-router-dom'
import { useAuthStore } from '@/features/auth/stores/authStore'
import type { ReactNode } from 'react'

function GuestRoute({ children }: { children: ReactNode }) {
  const isAuthenticated = useAuthStore((state) => state.isAuthenticated)

  if (isAuthenticated) {
    return <Navigate to="/overview" replace />
  }

  return children
}

export { GuestRoute }
```

---

## Navegacao Programatica

```tsx
import { useNavigate } from 'react-router-dom'

function CreateUserButton() {
  const navigate = useNavigate()

  async function handleCreate() {
    const user = await createUser(data)
    navigate(`/users/${user.id}`)
  }

  return <button onClick={handleCreate}>Criar</button>
}
```

### Regras de Navegacao

1. **SEMPRE** use `<Link>` para navegacao declarativa
2. **USE** `navigate()` apenas para navegacao programatica (apos acao)
3. **USE** `navigate(path, { replace: true })` para redirect sem adicionar ao historico
4. **NUNCA** use `window.location.href` para navegacao interna

---

## Pagina de Erro

```tsx
// app/routes/ErrorPage.tsx
import { useRouteError, isRouteErrorResponse, Link } from 'react-router-dom'

function ErrorPage() {
  const error = useRouteError()

  if (isRouteErrorResponse(error) && error.status === 404) {
    return (
      <main>
        <h1>Pagina Nao Encontrada</h1>
        <p>A pagina que voce esta procurando nao existe.</p>
        <Link to="/">Voltar ao Inicio</Link>
      </main>
    )
  }

  return (
    <main>
      <h1>Erro Inesperado</h1>
      <p>Ocorreu um erro ao carregar esta pagina.</p>
      <button onClick={() => window.location.reload()}>Tentar Novamente</button>
    </main>
  )
}

export { ErrorPage }
```

---

## Constantes de Rotas

```typescript
// shared/constants/routes.ts
const ROUTES = {
  HOME: '/',
  LOGIN: '/login',
  REGISTER: '/register',
  DASHBOARD: {
    OVERVIEW: '/overview',
    SETTINGS: '/settings',
  },
  USERS: {
    LIST: '/users',
    DETAIL: (id: string) => `/users/${id}` as const,
    CREATE: '/users/new',
  },
  PAYMENTS: {
    LIST: '/payments',
    DETAIL: (id: string) => `/payments/${id}` as const,
  },
} as const

export { ROUTES }
```

Uso:
```tsx
import { ROUTES } from '@/shared/constants/routes'

<Link to={ROUTES.USERS.DETAIL(user.id)}>{user.name}</Link>
```

---

## Checklist de Validacao

1. Layouts aninhados com Outlet para compartilhar UI entre rotas?
2. Rotas lazy-loaded com Suspense para code splitting?
3. errorElement configurado no router?
4. ProtectedRoute protegendo rotas privadas?
5. GuestRoute redirecionando usuarios autenticados?
6. Constantes de rotas centralizadas?
7. Link usado para navegacao declarativa (nao navigate)?
8. Redirect preservando pathname original para pos-login?
9. useParams tipado para rotas dinamicas?
10. Pagina de 404 tratada no errorElement?
