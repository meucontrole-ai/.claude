# Skill: Testing (Unit, Integration, E2E)

## Quando Ativar
- Criar/modificar testes unitarios (hooks, utils, componentes)
- Criar/modificar testes de integracao (componentes com interacao)
- Criar/modificar testes E2E (Playwright)
- Criar mocks, fixtures e test utilities
- Aumentar coverage
- Configurar test environment

## Quando NAO Ativar
- Escrever codigo de producao sem relacao direta com testes
- Configurar CI/CD (use Build & Deploy)

---

## Piramide de Testes

```
        /    E2E     \          <- Poucos, lentos, alto custo (Playwright)
       /--------------\
      /  Integracao     \       <- Moderados, testam interacao (Testing Library)
     /--------------------\
    /     Unitarios        \    <- Muitos, rapidos, isolados (Vitest)
   /-------------------------\
```

| Tipo | Coverage Target | Scope | Velocidade |
|---|---|---|---|
| Unitario | 80%+ hooks/utils | Funcao/hook isolado | < 5s total |
| Integracao | Componentes criticos | Componente + interacao do usuario | < 30s total |
| E2E | Fluxos criticos | Pagina completa no browser | < 2min total |

---

## Stack de Testes

| Ferramenta | Uso |
|---|---|
| Vitest | Test runner, assertions, mocking |
| Testing Library | Renderizacao e interacao com componentes |
| MSW (Mock Service Worker) | Mock de APIs em testes de integracao |
| Playwright | Testes E2E no browser |

---

## Convencoes de Naming

- Arquivo de teste: mesmo nome do arquivo testado com sufixo `.test.ts(x)`
- Funcao de teste: `describe` com nome do componente/hook, `it` com descricao do cenario
- Subtests: descricao em portugues, descrevendo cenario e resultado esperado
  - Bom: `'deve exibir erro quando email e invalido'`
  - Ruim: `'test1'`, `'funciona'`, `'erro'`

---

## Testes Unitarios

### Hook Test

```typescript
// features/auth/hooks/useAuth.test.ts
import { renderHook, act } from '@testing-library/react'
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { useAuth } from './useAuth'

describe('useAuth', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('deve retornar usuario null quando nao autenticado', () => {
    const { result } = renderHook(() => useAuth())

    expect(result.current.user).toBeNull()
    expect(result.current.isAuthenticated).toBe(false)
  })

  it('deve autenticar usuario com credenciais validas', async () => {
    const { result } = renderHook(() => useAuth())

    await act(async () => {
      await result.current.login({ email: 'user@test.com', password: 'password123' })
    })

    expect(result.current.isAuthenticated).toBe(true)
    expect(result.current.user).toEqual(
      expect.objectContaining({ email: 'user@test.com' })
    )
  })

  it('deve limpar estado ao fazer logout', async () => {
    const { result } = renderHook(() => useAuth())

    await act(async () => {
      await result.current.login({ email: 'user@test.com', password: 'password123' })
    })

    act(() => {
      result.current.logout()
    })

    expect(result.current.user).toBeNull()
    expect(result.current.isAuthenticated).toBe(false)
  })
})
```

### Utility Function Test

```typescript
// shared/utils/formatCurrency.test.ts
import { describe, it, expect } from 'vitest'
import { formatCurrency } from './formatCurrency'

describe('formatCurrency', () => {
  it('deve formatar valor em BRL', () => {
    expect(formatCurrency(1500, 'BRL')).toBe('R$ 15,00')
  })

  it('deve formatar valor zero', () => {
    expect(formatCurrency(0, 'BRL')).toBe('R$ 0,00')
  })

  it('deve formatar centavos corretamente', () => {
    expect(formatCurrency(1099, 'BRL')).toBe('R$ 10,99')
  })

  it('deve formatar valores negativos', () => {
    expect(formatCurrency(-500, 'BRL')).toBe('-R$ 5,00')
  })
})
```

---

## Testes de Integracao (Componentes)

### Componente com Interacao

```tsx
// features/auth/components/LoginForm.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, vi } from 'vitest'
import { LoginForm } from './LoginForm'

describe('LoginForm', () => {
  it('deve chamar onSubmit com dados do formulario', async () => {
    const user = userEvent.setup()
    const handleSubmit = vi.fn()

    render(<LoginForm onSubmit={handleSubmit} />)

    await user.type(screen.getByLabelText('Email'), 'user@test.com')
    await user.type(screen.getByLabelText('Senha'), 'password123')
    await user.click(screen.getByRole('button', { name: 'Entrar' }))

    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'user@test.com',
      password: 'password123',
    })
  })

  it('deve exibir erro de validacao para email invalido', async () => {
    const user = userEvent.setup()

    render(<LoginForm onSubmit={vi.fn()} />)

    await user.type(screen.getByLabelText('Email'), 'invalid')
    await user.click(screen.getByRole('button', { name: 'Entrar' }))

    expect(screen.getByRole('alert')).toHaveTextContent('Email invalido')
  })

  it('deve desabilitar botao durante submissao', async () => {
    const user = userEvent.setup()
    const handleSubmit = vi.fn(() => new Promise(() => {}))

    render(<LoginForm onSubmit={handleSubmit} />)

    await user.type(screen.getByLabelText('Email'), 'user@test.com')
    await user.type(screen.getByLabelText('Senha'), 'password123')
    await user.click(screen.getByRole('button', { name: 'Entrar' }))

    expect(screen.getByRole('button', { name: /criando|carregando|entrar/i })).toBeDisabled()
  })
})
```

### Regras do Testing Library

1. **SEMPRE** busque elementos por role, label ou text -- nunca por classe CSS ou test-id (exceto ultimo recurso)
2. **Prioridade de queries**: `getByRole` > `getByLabelText` > `getByText` > `getByPlaceholderText` > `getByTestId`
3. **SEMPRE** use `userEvent` ao inves de `fireEvent` para simular interacao real
4. **SEMPRE** use `userEvent.setup()` antes de cada teste
5. **NUNCA** teste detalhes de implementacao (estado interno, refs, lifecycle)

---

## MSW (Mock Service Worker)

### Setup

```typescript
// tests/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

const handlers = [
  http.get('/api/v1/users', () => {
    return HttpResponse.json({
      data: [
        { id: '1', name: 'John Doe', email: 'john@example.com' },
        { id: '2', name: 'Jane Doe', email: 'jane@example.com' },
      ],
      page: 1,
      pageSize: 20,
      totalItems: 2,
      totalPages: 1,
      hasNext: false,
    })
  }),

  http.post('/api/v1/users', async ({ request }) => {
    const body = await request.json()
    return HttpResponse.json(
      { id: '3', ...body },
      { status: 201 }
    )
  }),

  http.get('/api/v1/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: params.id,
      name: 'John Doe',
      email: 'john@example.com',
    })
  }),
]

export { handlers }
```

### Setup do Server

```typescript
// tests/mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

const server = setupServer(...handlers)

export { server }
```

### Vitest Setup

```typescript
// tests/setup.ts
import { beforeAll, afterAll, afterEach } from 'vitest'
import { server } from './mocks/server'
import '@testing-library/jest-dom/vitest'

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterAll(() => server.close())
afterEach(() => server.resetHandlers())
```

### Teste com MSW

```tsx
// features/users/components/UserList.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import { describe, it, expect } from 'vitest'
import { http, HttpResponse } from 'msw'
import { server } from '@/tests/mocks/server'
import { UserList } from './UserList'
import { TestProviders } from '@/tests/utils/TestProviders'

describe('UserList', () => {
  it('deve exibir lista de usuarios', async () => {
    render(<UserList />, { wrapper: TestProviders })

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument()
      expect(screen.getByText('Jane Doe')).toBeInTheDocument()
    })
  })

  it('deve exibir mensagem de erro quando API falha', async () => {
    server.use(
      http.get('/api/v1/users', () => {
        return HttpResponse.json(
          { code: 'INTERNAL', message: 'Server error' },
          { status: 500 }
        )
      })
    )

    render(<UserList />, { wrapper: TestProviders })

    await waitFor(() => {
      expect(screen.getByText(/erro/i)).toBeInTheDocument()
    })
  })

  it('deve exibir estado vazio quando nao ha usuarios', async () => {
    server.use(
      http.get('/api/v1/users', () => {
        return HttpResponse.json({
          data: [],
          page: 1,
          pageSize: 20,
          totalItems: 0,
          totalPages: 0,
          hasNext: false,
        })
      })
    )

    render(<UserList />, { wrapper: TestProviders })

    await waitFor(() => {
      expect(screen.getByText(/nenhum usuario/i)).toBeInTheDocument()
    })
  })
})
```

### Test Providers

```tsx
// tests/utils/TestProviders.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { ReactNode } from 'react'

function TestProviders({ children }: { children: ReactNode }) {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: 0,
      },
    },
  })

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

export { TestProviders }
```

---

## Testes E2E (Playwright)

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Autenticacao', () => {
  test('deve fazer login com sucesso', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('user@test.com')
    await page.getByLabel('Senha').fill('password123')
    await page.getByRole('button', { name: 'Entrar' }).click()

    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByText('Bem-vindo')).toBeVisible()
  })

  test('deve exibir erro com credenciais invalidas', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('invalid@test.com')
    await page.getByLabel('Senha').fill('wrong')
    await page.getByRole('button', { name: 'Entrar' }).click()

    await expect(page.getByRole('alert')).toContainText('Credenciais invalidas')
  })

  test('deve redirecionar para login quando nao autenticado', async ({ page }) => {
    await page.goto('/dashboard')

    await expect(page).toHaveURL('/login')
  })
})
```

### Playwright Config

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'mobile', use: { ...devices['Pixel 5'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

---

## Comandos de Execucao

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}
```

---

## Regras de Testes

1. **Testes devem ser deterministicos**: mesmo input, mesmo resultado
2. **Testes devem ser isolados**: cada teste cria seu proprio estado
3. **Teste comportamento, nao implementacao**: verifique o que o usuario ve, nao estado interno
4. **NUNCA** teste detalhes de implementacao (state, refs, lifecycle interno)
5. **SEMPRE** use MSW para mockar APIs em testes de integracao
6. **SEMPRE** trate os tres estados: sucesso, erro e vazio
7. **NUNCA** use `getByTestId` quando `getByRole` ou `getByLabelText` resolvem

---

## Checklist de Validacao

1. Testes unitarios para hooks e funcoes utilitarias?
2. Testes de integracao para componentes com interacao?
3. MSW configurado para mockar APIs?
4. TestProviders com QueryClient configurado para testes?
5. Coverage >= 80% em hooks e utils?
6. Playwright configurado para E2E?
7. Testes deterministicos e isolados?
8. Queries do Testing Library seguem prioridade (role > label > text > testId)?
9. userEvent.setup() usado em cada teste de interacao?
10. Loading, error e empty states testados em componentes com data fetching?
