# Skill: Dependencies

## Quando Ativar
- Adicionar nova dependencia/pacote NPM ao projeto
- Avaliar se uma biblioteca e adequada
- Atualizar dependencias existentes
- Auditar seguranca de dependencias (npm audit)
- Substituir uma dependencia por alternativa melhor
- Configurar polyfills

## Quando NAO Ativar
- Usar APIs nativas do browser ou Node.js
- Logica de negocio pura (use Components/State Management)
- Criar testes (use Testing)

---

## Regras para Adicionar Dependencias

### Criterios de Avaliacao (OBRIGATORIOS antes de adicionar)

1. **Necessidade real**: API nativa do browser ou Node nao resolve? Tem justificativa concreta?
2. **Bundle size**: qual o impacto no bundle? Verifique com `bundlephobia.com`
3. **Manutencao ativa**: ultimo commit < 6 meses, issues respondidas
4. **Adocao**: > 1k stars no GitHub ou parte do ecossistema oficial
5. **Seguranca**: sem vulnerabilidades conhecidas (`npm audit`)
6. **Tree-shaking**: a biblioteca suporta tree-shaking (ESM)?
7. **TypeScript**: possui tipos embutidos ou `@types/` atualizado?
8. **Licenca**: compativel com o projeto (MIT, Apache 2.0, BSD)

### Verificacao de Seguranca

```bash
# Audit de vulnerabilidades
pnpm audit

# Verificar tamanho do bundle
npx bundle-phobia-cli <package-name>

# Verificar licencas
npx license-checker --summary
```

---

## Dependencias Recomendadas por Categoria

### Core

| Categoria | Biblioteca | Justificativa |
|---|---|---|
| Framework | React 18+ | Ecosystem, Concurrent Mode, hooks |
| Build Tool | Vite 5+ | HMR rapido, ESM nativo, config simples |
| Linguagem | TypeScript 5+ | Type safety, DX, refactoring seguro |
| Package Manager | pnpm | Rapido, eficiente em disco, strict por default |

### Data & State

| Categoria | Biblioteca | Justificativa |
|---|---|---|
| Server State | @tanstack/react-query | Cache, mutations, optimistic updates |
| Client State | zustand | Minimalista, sem boilerplate, selectors |
| Form State | react-hook-form | Performance, uncontrolled by default |
| Schema Validation | zod | TypeScript-first, composable, inferencia de tipos |

### UI & Styling

| Categoria | Biblioteca | Justificativa |
|---|---|---|
| CSS Framework | Tailwind CSS | Utility-first, tree-shakable, design system |
| Component Library | Radix UI | Headless, acessivel, composavel |
| Class Merging | tailwind-merge | Merge inteligente de classes Tailwind |
| Conditional Classes | clsx | Leve, tipado, combinavel com tw-merge |
| Animations | framer-motion | Declarativa, layout animations, gestures |
| Icons | lucide-react | Tree-shakable, consistente, MIT |

### Infra & DX

| Categoria | Biblioteca | Justificativa |
|---|---|---|
| Date | date-fns | Tree-shakable, imutavel, modular |
| HTTP Client | fetch nativo | Zero dependency, suportado em todos runtimes |
| Error Tracking | @sentry/react | Source maps, replays, performance monitoring |
| Testing | vitest | Rapido, compativel com Vite, ESM nativo |
| Testing UI | @testing-library/react | Filosofia de teste por comportamento |
| API Mocking | msw | Service Worker, intercepta fetch real |
| E2E Testing | @playwright/test | Multi-browser, auto-wait, trace viewer |
| Linting | eslint | Extensivel, ecossistema rico |
| Formatting | prettier | Opinativo, zero config, consistente |

---

## Funcao Utilitaria cn() (clsx + tailwind-merge)

```typescript
// shared/utils/cn.ts
import { type ClassValue, clsx } from 'clsx'
import { twMerge } from 'tailwind-merge'

function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

export { cn }
```

---

## Dependencias que EVITAR

| Biblioteca | Motivo | Alternativa |
|---|---|---|
| moment.js | Pesado (330kB), mutavel, deprecated | date-fns ou Intl API |
| lodash (full) | Bundle enorme | lodash-es (tree-shakable) ou funcoes nativas |
| axios | Overhead desnecessario com fetch nativo | fetch API |
| jQuery | Desnecessario com React | React + DOM APIs |
| classnames | Nao faz merge de Tailwind | clsx + tailwind-merge |
| styled-components | Runtime CSS-in-JS, performance inferior | Tailwind CSS |
| redux | Boilerplate excessivo para maioria dos casos | Zustand |
| formik | Performance inferior, mais complexo | react-hook-form |
| yup | Sem inferencia de tipos nativa | zod |

---

## Regras de Lock File

- SEMPRE commite `pnpm-lock.yaml` no repositorio
- SEMPRE use `pnpm install --frozen-lockfile` no CI
- NUNCA edite o lock file manualmente
- NUNCA misture package managers (npm, yarn, pnpm) no mesmo projeto

---

## Atualizacao de Dependencias

```bash
# Ver dependencias desatualizadas
pnpm outdated

# Atualizar patch e minor (seguro)
pnpm update

# Atualizar major (requer teste manual)
pnpm update --latest

# Atualizar uma dependencia especifica
pnpm update react@latest
```

### Processo de Atualizacao

1. Verifique changelog da biblioteca para breaking changes
2. Atualize em branch separada
3. Execute todos os testes
4. Teste manualmente fluxos criticos
5. Merge apenas se todos os testes passarem

---

## Checklist de Validacao

1. Dependencia avaliada pelos criterios antes de adicionar?
2. Bundle size verificado no bundlephobia?
3. npm audit sem vulnerabilidades criticas?
4. Licenca verificada e compativel?
5. Biblioteca suporta tree-shaking (ESM)?
6. TypeScript types disponiveis?
7. Lock file commitado no repositorio?
8. Nenhuma dependencia deprecated em uso?
9. Context7 consultado quando conhecimento nao era suficiente?
10. Funcoes nativas preferidas a bibliotecas quando possivel?
