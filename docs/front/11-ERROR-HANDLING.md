# Skill: Error Handling

## Quando Ativar
- Implementar Error Boundaries
- Definir estrategia global de tratamento de erros
- Mapear erros de API para feedback ao usuario
- Implementar fallbacks e recovery
- Configurar paginas de erro (404, erro generico)

## Quando NAO Ativar
- Logica de negocio sem tratamento de erro especifico
- Erros ja cobertos pelo padrao existente

---

## Error Boundary (React)

### Error Boundary Generico

```tsx
// shared/components/feedback/ErrorBoundary.tsx
import { Component, type ErrorInfo, type ReactNode } from 'react'

interface ErrorBoundaryProps {
  children: ReactNode
  fallback?: ReactNode
  onError?: (error: Error, errorInfo: ErrorInfo) => void
}

interface ErrorBoundaryState {
  hasError: boolean
  error: Error | null
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props)
    this.state = { hasError: false, error: null }
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    this.props.onError?.(error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <DefaultErrorFallback error={this.state.error} />
    }
    return this.props.children
  }
}

function DefaultErrorFallback({ error }: { error: Error | null }) {
  return (
    <div role="alert">
      <h2>Algo deu errado</h2>
      <p>Ocorreu um erro inesperado. Tente recarregar a pagina.</p>
      <button onClick={() => window.location.reload()}>Recarregar</button>
    </div>
  )
}

export { ErrorBoundary }
```

### Uso em Subarvores

```tsx
<ErrorBoundary
  fallback={<PaymentErrorFallback />}
  onError={(error) => Sentry.captureException(error)}
>
  <PaymentForm />
</ErrorBoundary>
```

---

## Paginas de Erro (React Router)

### Pagina de Erro Generico

```tsx
// app/routes/ErrorPage.tsx
import { useRouteError, isRouteErrorResponse } from 'react-router-dom'

function ErrorPage() {
  const error = useRouteError()

  if (isRouteErrorResponse(error) && error.status === 404) {
    return (
      <main>
        <h1>Pagina Nao Encontrada</h1>
        <p>A pagina que voce esta procurando nao existe.</p>
        <a href="/">Voltar ao Inicio</a>
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

### Configuracao no Router

```tsx
// app/router.tsx
import { createBrowserRouter } from 'react-router-dom'
import { ErrorPage } from './routes/ErrorPage'

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      // ... rotas
    ],
  },
])
```

---

## Erros de API

### Tipos de Erro

```typescript
// shared/types/errors.ts

class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly status: number
  ) {
    super(message)
    this.name = 'AppError'
  }
}

class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly fields: Array<{ field: string; message: string }>
  ) {
    super(message, 'VALIDATION', 400)
    this.name = 'ValidationError'
  }
}

class NotFoundError extends AppError {
  constructor(message: string = 'Recurso nao encontrado') {
    super(message, 'NOT_FOUND', 404)
    this.name = 'NotFoundError'
  }
}

class UnauthorizedError extends AppError {
  constructor(message: string = 'Nao autenticado') {
    super(message, 'UNAUTHORIZED', 401)
    this.name = 'UnauthorizedError'
  }
}

class ForbiddenError extends AppError {
  constructor(message: string = 'Sem permissao') {
    super(message, 'FORBIDDEN', 403)
    this.name = 'ForbiddenError'
  }
}

export { AppError, ValidationError, NotFoundError, UnauthorizedError, ForbiddenError }
```

### Mapeamento de Erro da API

```typescript
// shared/lib/apiClient.ts (trecho)
import { AppError, ValidationError, NotFoundError, UnauthorizedError, ForbiddenError } from '@/shared/types/errors'

function mapApiError(status: number, body: { code: string; message: string; fields?: Array<{ field: string; message: string }> }): AppError {
  switch (status) {
    case 400:
      if (body.fields) {
        return new ValidationError(body.message, body.fields)
      }
      return new AppError(body.message, body.code, 400)
    case 401:
      return new UnauthorizedError(body.message)
    case 403:
      return new ForbiddenError(body.message)
    case 404:
      return new NotFoundError(body.message)
    default:
      return new AppError(body.message, body.code, status)
  }
}
```

---

## Toast de Erro Global

### Hook de Toast

```typescript
// shared/hooks/useToast.ts
import { create } from 'zustand'

interface Toast {
  id: string
  type: 'success' | 'error' | 'warning' | 'info'
  message: string
}

interface ToastStore {
  toasts: Toast[]
  addToast: (toast: Omit<Toast, 'id'>) => void
  removeToast: (id: string) => void
}

const useToastStore = create<ToastStore>((set) => ({
  toasts: [],
  addToast: (toast) => {
    const id = crypto.randomUUID()
    set((state) => ({ toasts: [...state.toasts, { ...toast, id }] }))
    setTimeout(() => {
      set((state) => ({ toasts: state.toasts.filter((t) => t.id !== id) }))
    }, 5000)
  },
  removeToast: (id) =>
    set((state) => ({ toasts: state.toasts.filter((t) => t.id !== id) })),
}))

export { useToastStore }
```

### Tratamento de Erro em Mutations

```typescript
import { useMutation } from '@tanstack/react-query'
import { useToastStore } from '@/shared/hooks/useToast'
import { UnauthorizedError } from '@/shared/types/errors'

function useCreateUser() {
  const addToast = useToastStore((state) => state.addToast)

  return useMutation({
    mutationFn: (data: CreateUserRequest) => userService.create(data),
    onError: (error) => {
      if (error instanceof UnauthorizedError) {
        window.location.href = '/login'
        return
      }

      addToast({
        type: 'error',
        message: error instanceof AppError
          ? error.message
          : 'Ocorreu um erro inesperado',
      })
    },
    onSuccess: () => {
      addToast({ type: 'success', message: 'Usuario criado com sucesso' })
    },
  })
}
```

---

## Fluxo de Propagacao de Erros

```
API Response (status + body) -> apiClient (mapeia para AppError) -> Hook/Mutation (trata com toast/redirect) -> Componente (exibe feedback)
```

1. **API Client**: recebe erro HTTP, mapeia para `AppError` semantico
2. **Hook/Mutation**: decide a acao (toast, redirect, retry)
3. **Componente**: exibe feedback visual ao usuario
4. **Error Boundary**: captura erros nao tratados em renderizacao

---

## Regras de Error Handling

1. **NUNCA** ignore erros silenciosamente (catch vazio)
2. **SEMPRE** exiba feedback ao usuario quando uma operacao falha
3. **SEMPRE** use Error Boundary em subarvores criticas
4. **NUNCA** exponha stack traces ou detalhes internos ao usuario
5. **SEMPRE** logue o erro completo (Sentry), retorne mensagem segura ao usuario
6. **SEMPRE** trate loading, error e empty states em componentes com data
7. **USE** tipos de erro semanticos (ValidationError, NotFoundError, etc.)
8. **USE** toast para erros de operacao, inline errors para validacao de formulario
9. **REDIRECIONE** para login em erros 401
10. **NUNCA** use try/catch em render -- use Error Boundary

---

## Checklist de Validacao

1. Error Boundary em subarvores criticas?
2. errorElement configurado nas rotas do React Router?
3. Tipos de erro semanticos definidos?
4. API errors mapeados para AppError?
5. Toast de erro para feedback ao usuario?
6. 401 redireciona para login?
7. Validation errors exibidos inline nos formularios?
8. Sem catch vazio ou erros ignorados?
9. Sentry capturando erros em Error Boundaries?
10. Pagina de 404 customizada no router?
