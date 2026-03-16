# Arquitetura de Projeto (Feature-Based Architecture)

## Quando Ativar

- Criar novo projeto ou aplicacao do zero
- Definir ou reestruturar a organizacao de pastas
- Refatorar para arquitetura modular
- Adicionar nova feature ou bounded context
- Configurar monorepo com workspaces

## Quando NAO Ativar

- Modificar apenas um componente existente sem impacto estrutural
- Adicionar uma pagina simples em estrutura ja existente
- Corrigir bug pontual

---

## Estrutura de Projeto (React + Vite)

```
project-root/
|-- src/
|   |-- app/                              # Rotas e layouts
|   |   |-- routes/
|   |   |   |-- auth/
|   |   |   |   |-- LoginPage.tsx
|   |   |   |   |-- RegisterPage.tsx
|   |   |   |-- dashboard/
|   |   |   |   |-- OverviewPage.tsx
|   |   |   |   |-- SettingsPage.tsx
|   |   |-- layouts/
|   |   |   |-- AuthLayout.tsx
|   |   |   |-- DashboardLayout.tsx
|   |   |   |-- RootLayout.tsx
|   |   |-- router.tsx                    # Configuracao de rotas (React Router)
|   |   |-- App.tsx                       # Componente raiz
|   |-- features/                         # Features por dominio
|   |   |-- auth/
|   |   |   |-- components/
|   |   |   |   |-- LoginForm.tsx
|   |   |   |   |-- RegisterForm.tsx
|   |   |   |-- hooks/
|   |   |   |   |-- useAuth.ts
|   |   |   |   |-- useSession.ts
|   |   |   |-- services/
|   |   |   |   |-- authService.ts
|   |   |   |-- types/
|   |   |   |   |-- auth.ts
|   |   |   |-- utils/
|   |   |   |   |-- tokenStorage.ts
|   |   |   |-- index.ts                  # Barrel export (API publica da feature)
|   |   |-- payments/
|   |   |   |-- components/
|   |   |   |-- hooks/
|   |   |   |-- services/
|   |   |   |-- types/
|   |   |   |-- index.ts
|   |   |-- users/
|   |   |   |-- components/
|   |   |   |-- hooks/
|   |   |   |-- services/
|   |   |   |-- types/
|   |   |   |-- index.ts
|   |-- shared/                           # Codigo compartilhado entre features
|   |   |-- components/                   # Componentes genericos reutilizaveis
|   |   |   |-- ui/                       # Primitivos de UI (Button, Input, Modal)
|   |   |   |   |-- Button.tsx
|   |   |   |   |-- Input.tsx
|   |   |   |   |-- Modal.tsx
|   |   |   |   |-- index.ts
|   |   |   |-- layout/                   # Componentes de layout
|   |   |   |   |-- Header.tsx
|   |   |   |   |-- Sidebar.tsx
|   |   |   |   |-- Footer.tsx
|   |   |   |-- feedback/                 # Toast, Alert, Skeleton
|   |   |   |   |-- Toast.tsx
|   |   |   |   |-- Skeleton.tsx
|   |   |-- hooks/                        # Hooks genericos
|   |   |   |-- useDebounce.ts
|   |   |   |-- useLocalStorage.ts
|   |   |   |-- useMediaQuery.ts
|   |   |-- lib/                          # Configuracoes de bibliotecas
|   |   |   |-- apiClient.ts
|   |   |   |-- queryClient.ts
|   |   |-- types/                        # Tipos globais
|   |   |   |-- api.ts
|   |   |   |-- common.ts
|   |   |-- utils/                        # Funcoes utilitarias puras
|   |   |   |-- formatCurrency.ts
|   |   |   |-- formatDate.ts
|   |   |   |-- validators.ts
|   |   |-- constants/                    # Constantes globais
|   |   |   |-- routes.ts
|   |   |   |-- config.ts
|   |-- styles/                           # Estilos globais e tokens
|   |   |-- globals.css
|   |   |-- tokens.css
|   |-- main.tsx                          # Entry point da aplicacao
|-- public/
|   |-- fonts/
|   |-- images/
|-- tests/                                # Testes E2E (Playwright/Cypress)
|   |-- e2e/
|   |   |-- auth.spec.ts
|   |   |-- payments.spec.ts
|-- .env
|-- .env.production
|-- vite.config.ts
|-- tsconfig.json
|-- tailwind.config.ts
|-- package.json
```

---

## Regras de Dependencia

```
app/ (rotas) --> features/ (dominios) --> shared/ (codigo compartilhado)
```

As dependencias SEMPRE apontam para dentro (de mais especifico para mais generico). Codigo compartilhado nunca conhece features especificas.

- **shared/** (nucleo compartilhado): ZERO dependencias de features. Contem componentes de UI genericos, hooks reutilizaveis, utilitarios puros, tipos globais e configuracoes de bibliotecas.
- **features/** (dominios de negocio): Importam apenas de `shared/`. Contem componentes, hooks, services, tipos e utilitarios especificos de um dominio. Cada feature e autocontida e exporta sua API publica via barrel export (`index.ts`).
- **app/** (rotas e layouts): Importam de `features/` e `shared/`. Responsavel apenas por composicao de pagina, definicao de rotas e layouts. Sem logica de negocio.

NUNCA uma feature importa de outra feature diretamente. Se duas features precisam compartilhar logica, extraia para `shared/`.

### Anti-Patterns (EVITAR)

- Componente de pagina com logica de negocio (deve delegar para feature)
- Feature que importa de outra feature (acoplamento horizontal)
- Componente em `shared/` com logica de dominio especifico
- Barrel exports profundos (re-exportar tudo de subpastas aninhadas)
- Circular dependency entre modulos
- Arquivo `utils.ts` gigante com funcoes nao relacionadas

---

## Barrel Exports (API Publica da Feature)

Cada feature expoe apenas o que e necessario para consumo externo:

```typescript
// features/auth/index.ts
export { LoginForm } from './components/LoginForm'
export { RegisterForm } from './components/RegisterForm'
export { useAuth } from './hooks/useAuth'
export { useSession } from './hooks/useSession'
export type { User, AuthState, LoginCredentials } from './types/auth'
```

Componentes internos, hooks auxiliares e utilitarios privados NAO sao exportados. Isso cria um contrato claro entre features e consumidores.

---

## Colocation Pattern

Mantenha arquivos relacionados proximos ao componente:

```
features/payments/
|-- components/
|   |-- PaymentForm/
|   |   |-- PaymentForm.tsx        # Componente
|   |   |-- PaymentForm.test.tsx   # Teste
|   |   |-- PaymentForm.skeleton.tsx # Loading state
|   |   |-- index.ts               # Re-export
```

Para componentes simples (sem testes ou variantes), um unico arquivo basta. Use a estrutura de pasta apenas quando ha 3+ arquivos relacionados.

---

## Configuracao TypeScript

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/features/*": ["./src/features/*"],
      "@/shared/*": ["./src/shared/*"]
    },
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": false,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*", "tests/**/*"],
  "exclude": ["node_modules"]
}
```

### Path Aliases Obrigatorios

- `@/` para raiz do `src/`
- `@/features/` para features
- `@/shared/` para codigo compartilhado

NUNCA use imports relativos que subam mais de um nivel (`../../`). Use path aliases.

Os path aliases devem ser configurados tambem no `vite.config.ts`:

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
})
```

---

## Environment Variables

```typescript
// src/shared/constants/config.ts
const requiredEnvVars = [
  'VITE_API_URL',
  'VITE_APP_ENV',
] as const

type EnvVar = (typeof requiredEnvVars)[number]

function getEnvVar(name: EnvVar): string {
  const value = import.meta.env[name]
  if (!value) {
    throw new Error(`Missing required environment variable: ${name}`)
  }
  return value
}

export const config = {
  apiUrl: getEnvVar('VITE_API_URL'),
  appEnv: getEnvVar('VITE_APP_ENV'),
  isDev: import.meta.env.DEV,
  isProd: import.meta.env.PROD,
} as const
```

NUNCA acesse `import.meta.env` diretamente nos componentes. Centralize em `config.ts`.

---

## Quando Criar uma Nova Feature

Crie um novo diretorio em `features/` quando:
- O dominio tem entidades e regras de negocio proprias
- O dominio tem componentes, hooks e services especificos que nao sao compartilhados
- Dois dominios usam o mesmo termo com significados diferentes (ex: `User` em auth vs `User` em billing)

NAO crie uma nova feature quando:
- E apenas um componente de UI generico (use `shared/components/`)
- A separacao criaria dependencia circular entre features
- O codigo e utilizado por 3+ features (extraia para `shared/`)

---

## Checklist de Validacao

1. Features nao importam de outras features diretamente?
2. `shared/` nao contem logica de dominio especifico?
3. `app/` contem apenas composicao de pagina e definicao de rotas?
4. Barrel exports expoe apenas a API publica de cada feature?
5. Path aliases configurados e usados consistentemente?
6. Nenhum arquivo excede 250 linhas?
7. TypeScript strict mode habilitado?
8. Environment variables centralizadas em config.ts?
9. Colocation aplicada (testes e tipos proximos ao componente)?
10. Nenhuma dependencia circular entre modulos?
