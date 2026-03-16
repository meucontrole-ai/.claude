# Skill: Seguranca Frontend

## Quando Ativar
- Implementar autenticacao (JWT, OAuth2, cookies httpOnly)
- Implementar autorizacao (RBAC em rotas/componentes)
- Proteger contra XSS, CSRF, clickjacking
- Configurar Content Security Policy (CSP)
- Sanitizar input de usuario
- Gerenciar tokens e sessoes
- Configurar HTTPS e headers de seguranca

## Quando NAO Ativar
- Logica de negocio pura (use Components/State Management)
- Criar testes (use Testing)
- Configurar build/deploy (use Build & Deploy)

---

## Autenticacao com JWT

### Token Storage

NUNCA armazene tokens JWT em localStorage -- vulneravel a XSS. Prefira cookies httpOnly quando possivel.

| Metodo | XSS Safe | CSRF Safe | Recomendado |
|---|---|---|---|
| Cookie httpOnly + Secure + SameSite | Sim | Sim (SameSite=Strict) | Sim |
| Memoria (variavel/Zustand) | Sim | N/A | Sim (SPA puro) |
| localStorage | Nao | N/A | Nao |
| sessionStorage | Nao | N/A | Nao |

### Implementacao com Token em Memoria (SPA)

Em uma SPA com React + Vite, o approach mais comum e armazenar o token em memoria (Zustand com persist) e envia-lo via header Authorization:

```typescript
// features/auth/stores/authStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface AuthState {
  token: string | null
  user: User | null
  isAuthenticated: boolean
}

interface AuthActions {
  setAuth: (user: User, token: string) => void
  logout: () => void
}

const useAuthStore = create<AuthState & AuthActions>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      isAuthenticated: false,

      setAuth: (user, token) =>
        set({ user, token, isAuthenticated: true }),

      logout: () =>
        set({ user: null, token: null, isAuthenticated: false }),
    }),
    {
      name: 'auth-storage',
      partialize: (state) => ({
        token: state.token,
        user: state.user,
        isAuthenticated: state.isAuthenticated,
      }),
    }
  )
)

export { useAuthStore }
```

### Route Guard (React Router)

```tsx
// shared/components/auth/ProtectedRoute.tsx
import { Navigate, useLocation } from 'react-router-dom'
import { useAuthStore } from '@/features/auth/stores/authStore'
import type { ReactNode } from 'react'

const publicRoutes = ['/login', '/register', '/forgot-password']

interface ProtectedRouteProps {
  children: ReactNode
}

function ProtectedRoute({ children }: ProtectedRouteProps) {
  const isAuthenticated = useAuthStore((state) => state.isAuthenticated)
  const { pathname } = useLocation()

  if (!isAuthenticated && !publicRoutes.includes(pathname)) {
    return <Navigate to={`/login?redirect=${pathname}`} replace />
  }

  return children
}

export { ProtectedRoute }
```

Uso no router:

```tsx
// app/router.tsx
import { createBrowserRouter } from 'react-router-dom'
import { ProtectedRoute } from '@/shared/components/auth/ProtectedRoute'

const router = createBrowserRouter([
  {
    path: '/login',
    element: <LoginPage />,
  },
  {
    path: '/dashboard',
    element: (
      <ProtectedRoute>
        <DashboardLayout />
      </ProtectedRoute>
    ),
    children: [
      { index: true, element: <OverviewPage /> },
      { path: 'settings', element: <SettingsPage /> },
    ],
  },
])

export { router }
```

---

## Autorizacao (RBAC)

### Hook de Permissoes

```typescript
// features/auth/hooks/usePermission.ts
import { useAuthStore } from '@/features/auth/stores/authStore'

type Permission = 'users:read' | 'users:write' | 'payments:read' | 'payments:write' | 'admin:access'

const rolePermissions: Record<string, Permission[]> = {
  admin: ['users:read', 'users:write', 'payments:read', 'payments:write', 'admin:access'],
  manager: ['users:read', 'users:write', 'payments:read'],
  viewer: ['users:read', 'payments:read'],
}

function usePermission() {
  const role = useAuthStore((state) => state.user?.role)

  function hasPermission(permission: Permission): boolean {
    if (!role) return false
    return rolePermissions[role]?.includes(permission) ?? false
  }

  function hasAnyPermission(permissions: Permission[]): boolean {
    return permissions.some(hasPermission)
  }

  return { hasPermission, hasAnyPermission, role }
}

export { usePermission }
export type { Permission }
```

### Guard Component

```tsx
// shared/components/auth/PermissionGate.tsx
import type { ReactNode } from 'react'
import { usePermission, type Permission } from '@/features/auth/hooks/usePermission'

interface PermissionGateProps {
  permission: Permission | Permission[]
  fallback?: ReactNode
  children: ReactNode
}

function PermissionGate({ permission, fallback = null, children }: PermissionGateProps) {
  const { hasPermission, hasAnyPermission } = usePermission()

  const isAllowed = Array.isArray(permission)
    ? hasAnyPermission(permission)
    : hasPermission(permission)

  if (!isAllowed) return fallback

  return children
}

export { PermissionGate }
```

Uso:
```tsx
<PermissionGate permission="users:write">
  <Button onClick={handleDelete}>Excluir Usuario</Button>
</PermissionGate>
```

---

## Protecao contra XSS

### Regras Obrigatorias

1. **NUNCA** use `dangerouslySetInnerHTML` sem sanitizacao
2. **NUNCA** construa HTML com string concatenation/template literals
3. **SEMPRE** use DOMPurify para sanitizar HTML externo
4. **SEMPRE** escape dados do usuario antes de renderizar

### Sanitizacao com DOMPurify

```typescript
// shared/utils/sanitize.ts
import DOMPurify from 'dompurify'

function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li'],
    ALLOWED_ATTR: ['href', 'target', 'rel'],
  })
}

export { sanitizeHtml }
```

Uso:
```tsx
function RichContent({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: sanitizeHtml(html) }} />
}
```

---

## Content Security Policy (CSP)

Em uma SPA com Vite, configure os headers de seguranca no servidor que serve a aplicacao (Nginx, Caddy, etc.) ou via plugin Vite para desenvolvimento:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    headers: {
      'X-Frame-Options': 'DENY',
      'X-Content-Type-Options': 'nosniff',
      'Referrer-Policy': 'strict-origin-when-cross-origin',
      'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',
    },
  },
})
```

Para producao, configure no servidor (exemplo Nginx):

```nginx
# nginx.conf
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' blob: data: https:; font-src 'self'; connect-src 'self' https://api.project.com; frame-ancestors 'none'; form-action 'self'; base-uri 'self'; object-src 'none';" always;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

---

## Input Validation (Zod)

SEMPRE valide input no frontend E no backend. A validacao no frontend e para UX, a do backend e para seguranca.

```typescript
// features/users/schemas/userSchema.ts
import { z } from 'zod'

const createUserSchema = z.object({
  name: z
    .string()
    .min(2, 'Nome deve ter pelo menos 2 caracteres')
    .max(100, 'Nome deve ter no maximo 100 caracteres')
    .regex(/^[a-zA-ZÀ-ÿ\s]+$/, 'Nome deve conter apenas letras'),
  email: z
    .string()
    .email('Email invalido')
    .max(255),
  password: z
    .string()
    .min(8, 'Senha deve ter pelo menos 8 caracteres')
    .regex(/[A-Z]/, 'Senha deve conter pelo menos uma letra maiuscula')
    .regex(/[0-9]/, 'Senha deve conter pelo menos um numero')
    .regex(/[^A-Za-z0-9]/, 'Senha deve conter pelo menos um caractere especial'),
})

type CreateUserFormData = z.infer<typeof createUserSchema>

export { createUserSchema }
export type { CreateUserFormData }
```

---

## Protecao contra CSRF

Em uma SPA pura com React + Vite, a protecao contra CSRF e feita por:

1. Use `SameSite=Strict` nos cookies de sessao (se usar cookies)
2. Use header `Authorization: Bearer <token>` para autenticacao (CSRF-safe por padrao)
3. Valide o header `Origin` no backend para requests sensíveis

Se a API usa cookies para autenticacao, implemente CSRF token no backend e envie-o via header customizado:

```typescript
// shared/lib/apiClient.ts (trecho adicional)
async function getCsrfToken(): Promise<string> {
  const response = await fetch('/api/csrf-token', { credentials: 'include' })
  const data = await response.json()
  return data.token
}
```

---

## Environment Variables de Seguranca

```
# .env -- NUNCA commitar
VITE_API_URL=https://api.project.com        # Exposta ao browser (cuidado)
VITE_APP_ENV=production                       # Exposta ao browser (cuidado)
```

Regras:
- Prefixo `VITE_` expoe a variavel ao browser -- use APENAS para dados nao sensiveis
- Secrets (API keys, JWT secret, DB credentials) NUNCA devem estar no frontend
- Em uma SPA, TODO codigo e exposto ao browser -- nunca inclua secrets no bundle
- NUNCA commite `.env` ou `.env.production` no repositorio

---

## OWASP Top 10 Frontend

| Vulnerabilidade | Mitigacao |
|---|---|
| XSS (Cross-Site Scripting) | React escapa por default, DOMPurify para HTML externo, CSP |
| CSRF | Bearer token via header (CSRF-safe), SameSite cookies |
| Clickjacking | X-Frame-Options: DENY, frame-ancestors 'none' |
| Sensitive Data Exposure | Sem secrets no client, HTTPS only |
| Broken Auth | Token em memoria/Zustand, route guards, redirect correto |
| Injection | Input validation com Zod, parameterized queries no backend |
| Insecure Dependencies | npm audit, Dependabot, Snyk |
| Open Redirects | Validar URLs de redirect contra allowlist |

---

## Checklist de Validacao

1. Tokens armazenados de forma segura (memoria/Zustand, nao localStorage)?
2. Route guards protegendo rotas privadas?
3. RBAC implementado com PermissionGate e usePermission?
4. CSP configurado no servidor de producao?
5. Headers de seguranca (X-Frame-Options, HSTS, etc.) configurados?
6. Input validado com Zod em formularios?
7. Sem `dangerouslySetInnerHTML` sem sanitizacao?
8. Sem secrets no bundle do frontend?
9. `.env*` no .gitignore?
10. npm audit sem vulnerabilidades criticas?
