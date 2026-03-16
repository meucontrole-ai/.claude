# Skill: Accessibility (a11y)

## Quando Ativar
- Criar/modificar componentes interativos (modais, dropdowns, tabs)
- Implementar navegacao por teclado
- Adicionar ARIA attributes
- Auditar acessibilidade (axe, Lighthouse)
- Implementar skip links e focus management
- Garantir contraste de cores

## Quando NAO Ativar
- Logica de negocio pura (use Components/State Management)
- Styling sem impacto em acessibilidade

---

## WCAG 2.1 - Niveis de Conformidade

| Nivel | Requisito | Obrigatorio |
|---|---|---|
| **A** | Minimo funcional (alternativas de texto, navegacao por teclado) | Sim |
| **AA** | Padrao de industria (contraste 4.5:1, resize, focus visible) | Sim |
| **AAA** | Maximo (contraste 7:1, sign language, simplificacao de texto) | Nao |

**Target obrigatorio: WCAG 2.1 Nivel AA**

---

## Semantica HTML

### Use elementos HTML semanticos ao inves de divs genericas:

```tsx
// Ruim
<div onClick={handleClick}>Clique aqui</div>
<div className="header">...</div>
<div className="nav">...</div>

// Bom
<button onClick={handleClick}>Clique aqui</button>
<header>...</header>
<nav>...</nav>
```

### Elementos Semanticos por Contexto

| Contexto | Elemento |
|---|---|
| Navegacao principal | `<nav>` |
| Cabecalho da pagina | `<header>` |
| Conteudo principal | `<main>` |
| Rodape | `<footer>` |
| Secao de conteudo | `<section>` com `aria-labelledby` |
| Artigo independente | `<article>` |
| Conteudo lateral | `<aside>` |
| Lista de itens | `<ul>` / `<ol>` |
| Tabela de dados | `<table>` com `<thead>`, `<tbody>`, `<th scope>` |

---

## ARIA Attributes

### Regras de ARIA

1. **Prefira HTML semantico a ARIA**: um `<button>` nao precisa de `role="button"`
2. **Nao altere semantica nativa**: nunca use `<h1 role="button">`
3. **Elementos interativos DEVEM ter nome acessivel**: `aria-label`, `aria-labelledby` ou conteudo textual
4. **ARIA nao adiciona comportamento**: `role="button"` nao torna um `<div>` focavel ou ativavel por teclado

### Padroes Comuns

```tsx
// Botao com icone (sem texto visivel)
<button aria-label="Fechar modal" onClick={onClose}>
  <XIcon />
</button>

// Regiao rotulada
<section aria-labelledby="recent-orders-title">
  <h2 id="recent-orders-title">Pedidos Recentes</h2>
  {/* conteudo */}
</section>

// Status dinamico
<div role="status" aria-live="polite">
  {isLoading ? 'Carregando...' : `${items.length} resultados encontrados`}
</div>

// Erro de formulario
<input
  id="email"
  aria-invalid={!!error}
  aria-describedby={error ? 'email-error' : undefined}
/>
{error && <span id="email-error" role="alert">{error}</span>}

// Navegacao com breadcrumb
<nav aria-label="Breadcrumb">
  <ol>
    <li><a href="/">Inicio</a></li>
    <li><a href="/users" aria-current="page">Usuarios</a></li>
  </ol>
</nav>
```

---

## Navegacao por Teclado

### Teclas Padrao

| Tecla | Acao |
|---|---|
| Tab | Move foco para o proximo elemento focavel |
| Shift+Tab | Move foco para o elemento anterior |
| Enter/Space | Ativa o elemento focado |
| Escape | Fecha modal/dropdown/popover |
| Arrow keys | Navega entre opcoes em menus/tabs/listbox |

### Focus Management em Modal

```tsx
import { useEffect, useRef, type ReactNode } from 'react'

interface ModalProps {
  isOpen: boolean
  onClose: () => void
  children: ReactNode
  title: string
}

function Modal({ isOpen, onClose, children, title }: ModalProps) {
  const dialogRef = useRef<HTMLDialogElement>(null)
  const previousFocusRef = useRef<HTMLElement | null>(null)

  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement as HTMLElement
      dialogRef.current?.showModal()
    } else {
      dialogRef.current?.close()
      previousFocusRef.current?.focus()
    }
  }, [isOpen])

  return (
    <dialog
      ref={dialogRef}
      onClose={onClose}
      aria-labelledby="modal-title"
      className="rounded-lg p-6 backdrop:bg-black/50"
    >
      <h2 id="modal-title">{title}</h2>
      {children}
      <button onClick={onClose} aria-label="Fechar">
        <XIcon />
      </button>
    </dialog>
  )
}

export { Modal }
```

### Skip Link

```tsx
// shared/components/layout/SkipLink.tsx
function SkipLink() {
  return (
    <a
      href="#main-content"
      className="
        sr-only
        focus:not-sr-only
        focus:fixed
        focus:left-4
        focus:top-4
        focus:z-50
        focus:rounded-md
        focus:bg-white
        focus:px-4
        focus:py-2
        focus:shadow-lg
      "
    >
      Pular para o conteudo principal
    </a>
  )
}

export { SkipLink }
```

Uso no layout:
```tsx
<body>
  <SkipLink />
  <Header />
  <main id="main-content">{children}</main>
  <Footer />
</body>
```

---

## Focus Visible

```css
/* styles/globals.css */
@layer base {
  :focus-visible {
    @apply outline-2 outline-offset-2 outline-blue-600;
  }

  :focus:not(:focus-visible) {
    outline: none;
  }
}
```

NUNCA remova focus outline sem substituir por um indicador visual equivalente.

---

## Contraste de Cores

| Tipo de Texto | Ratio Minimo (AA) | Ratio Minimo (AAA) |
|---|---|---|
| Texto normal (< 18px) | 4.5:1 | 7:1 |
| Texto grande (>= 18px ou >= 14px bold) | 3:1 | 4.5:1 |
| Componentes de UI e graficos | 3:1 | N/A |

Verifique contraste com ferramentas como WebAIM Contrast Checker.

---

## Imagens e Midia

```tsx
// Imagem informativa -- alt descritivo
<Image src="/chart.png" alt="Grafico de vendas mostrando crescimento de 20% em marco" />

// Imagem decorativa -- alt vazio
<Image src="/decoration.svg" alt="" aria-hidden="true" />

// Video com legendas
<video controls>
  <source src="/demo.mp4" type="video/mp4" />
  <track kind="captions" src="/demo-pt.vtt" srcLang="pt-BR" label="Portugues" default />
</video>
```

---

## Live Regions (Conteudo Dinamico)

```tsx
// Notificacao que deve ser anunciada por screen readers
<div role="status" aria-live="polite">
  Formulario enviado com sucesso
</div>

// Erro urgente
<div role="alert" aria-live="assertive">
  Erro ao processar pagamento
</div>
```

| Atributo | Urgencia | Uso |
|---|---|---|
| `aria-live="polite"` | Baixa | Mensagens de status, atualizacoes de lista |
| `aria-live="assertive"` | Alta | Erros criticos, alertas |
| `role="status"` | Polite implicito | Status de operacao |
| `role="alert"` | Assertive implicito | Erros e alertas |

---

## Teste de Acessibilidade

### axe-core com Testing Library

```typescript
import { render } from '@testing-library/react'
import { axe, toHaveNoViolations } from 'jest-axe'
import { describe, it, expect } from 'vitest'

expect.extend(toHaveNoViolations)

describe('LoginForm a11y', () => {
  it('nao deve ter violacoes de acessibilidade', async () => {
    const { container } = render(<LoginForm onSubmit={vi.fn()} />)
    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })
})
```

### Playwright a11y

```typescript
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

test('pagina de login deve ser acessivel', async ({ page }) => {
  await page.goto('/login')

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze()

  expect(results.violations).toEqual([])
})
```

---

## Regras de Acessibilidade

1. **SEMPRE** use HTML semantico antes de ARIA
2. **SEMPRE** forne alt text para imagens informativas (alt vazio para decorativas)
3. **NUNCA** remova focus outline sem substituir
4. **SEMPRE** garanta contraste minimo 4.5:1 para texto
5. **SEMPRE** implemente navegacao por teclado em componentes interativos
6. **SEMPRE** use `aria-label` em botoes com apenas icone
7. **SEMPRE** associe labels a inputs (`htmlFor` / `id`)
8. **SEMPRE** use `role="alert"` para erros de formulario
9. **USE** skip link para pular navegacao
10. **USE** `aria-live` para conteudo que atualiza dinamicamente
11. **TESTE** com axe-core em testes de integracao

---

## Checklist de Validacao

1. HTML semantico usado antes de ARIA?
2. Todos inputs tem label associada?
3. Botoes com icone tem aria-label?
4. Focus visible em todos elementos interativos?
5. Navegacao por teclado funcional (Tab, Enter, Escape)?
6. Contraste de cores >= 4.5:1 para texto normal?
7. Skip link implementado?
8. Alt text em imagens informativas?
9. Live regions para conteudo dinamico?
10. axe-core sem violacoes nos testes?
11. Modal trap focus e restaura focus ao fechar?
12. Formularios exibem erros com role="alert"?
