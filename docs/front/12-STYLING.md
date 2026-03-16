# Skill: Styling

## Quando Ativar
- Implementar CSS architecture (Tailwind CSS)
- Definir design tokens (cores, espacamentos, tipografia)
- Implementar responsividade (mobile-first)
- Criar tema (light/dark mode)
- Implementar animacoes e transicoes
- Configurar Tailwind CSS

## Quando NAO Ativar
- Logica de negocio (use Components/State Management)
- Performance sem relacao com CSS (use Performance)
- Testes (use Testing)

---

## Tailwind CSS como Padrao

Tailwind CSS e a ferramenta de estilizacao padrao. Motivos:
- Utility-first elimina CSS morto
- Design tokens via configuracao
- Responsividade nativa com prefixos (`sm:`, `md:`, `lg:`)
- Dark mode via seletor com `darkMode: "selector"`
- Tree-shaking automatico

---

## Funcao cn() (Classe Condicional)

```typescript
// src/common/utils/cn.ts
import clsx, { ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

Import padrao no projeto:
```typescript
import { cn } from '@/common/utils/cn'
```

Uso:
```tsx
<button
  className={cn(
    'rounded-md px-4 py-2 font-semibold transition-colors',
    outlined && 'border border-primary-500 bg-white text-primary-500 hover:bg-zinc-200',
    disabled && 'cursor-not-allowed opacity-50',
    className
  )}
>
```

---

## Class Variance Authority (CVA)

Para componentes com multiplas variantes (ex: `src/components/ui/button.tsx`):

```typescript
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/common/utils/cn'

const buttonVariants = cva(
  'inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 p-3',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
)
```

---

## Design System – Paleta de Cores

As cores base estao definidas em `docs/DESIGN_SYSTEM.md`:

| Token            | Hex       | Uso                          |
|------------------|-----------|------------------------------|
| `primary`        | `#E5770B` | Botoes principais, destaques |
| `primary-green`  | `#0EAB14` | Sucesso                      |
| `primary-red`    | `#DB1B0D` | Erro / Exclusao              |
| `primary-orange` | `#F36A15` | Alertas / Hover              |
| `money`          | `#15803D` | Indicadores financeiros      |
| `text-color`     | `#3F3F46` | Cor padrao para textos       |

### Cores Dinamicas via Tema (Multi-Tenant)

O projeto suporta temas dinamicos. A cor primaria e gerada via `useThemeColors`:

```typescript
// src/hooks/useThemeColors.ts
// Gera escalas completas a partir de uma cor base
useThemeColors(primaryColor, contrastColor)
```

As escalas sao aplicadas como CSS vars:
- `--primary-50` a `--primary-950`
- `--secondary-50` a `--secondary-950`
- `--contrast` (calculado automaticamente: `#000` ou `#fff`)

Uso no Tailwind:
```tsx
// Cor primaria do tema
<div className="bg-primary-500 text-contrast" />

// Variantes da paleta
<span className="text-primary-700 bg-primary-100" />
```

### Tokens Tailwind Principais

```javascript
// tailwind.config.js – cores mapeadas para CSS vars
primary: {
  DEFAULT: 'hsl(var(--primary))',
  foreground: 'hsl(var(--primary-foreground))',
  50: 'var(--primary-50)',
  // ...até 950
},
background: 'hsl(var(--background))',
foreground: 'hsl(var(--foreground))',
border: 'hsl(var(--border))',
muted: { DEFAULT: 'hsl(var(--muted))', foreground: 'hsl(var(--muted-foreground))' },
accent: { DEFAULT: 'hsl(var(--accent))', foreground: 'hsl(var(--accent-foreground))' },
destructive: { DEFAULT: 'hsl(var(--destructive))', foreground: 'hsl(var(--destructive-foreground))' },
contrast: 'var(--contrast)',
bg: { 50..950: 'var(--bg-*)' },
```

---

## CSS Variables para Temas

```css
/* src/common/styles/globals.css */
@layer base {
  :root {
    --primary-500: #e5770b;   /* cor primaria padrao */
    --contrast: #ffffff;

    --background: 0 0% 100%;
    --foreground: 224 71.4% 4.1%;
    --border: 220 13% 91%;
    --radius: 0.5rem;

    /* escalas primary, secondary, bg (50-950) */
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    /* ... */
  }
}
```

Dark mode e controlado via Redux e aplicado pela classe `.dark` no elemento raiz:

```typescript
// src/redux-store/features/config/theme/theme.slice.ts
type StateTheme = {
  isDark: boolean
  primary_color: string  // default: '#e5770b'
  contrast_color: string // default: '#000000'
}
```

---

## Tipografia

**Fonte principal: Montserrat** (obrigatoria conforme Design System).

Carregada via Google Fonts em `globals.css`:
```css
@import url("https://fonts.googleapis.com/css2?family=Montserrat:ital,wght@0,100..900;1,100..900...");
```

Aplicada globalmente:
```css
* {
  font-family: "Montserrat", sans-serif;
}
```

Fontes auxiliares disponiveis via Tailwind:
```tsx
<p className="font-montserrat" />  // Montserrat (padrao)
<p className="font-poppins" />     // Poppins
<p className="font-courier" />     // Courier Prime (monospace)
<p className="font-outfit" />      // Outfit
```

---

## Animacoes

### Utilitarios CSS Globais

```tsx
<div className="animate-fade-in" />    // fade-in 0.3s ease-in-out
<div className="animate-fade-out" />   // fade-out 0.3s ease-in-out
<div className="animate-item-entrance" /> // translateY + scale 0.2s ease-out
```

### Animacoes Tailwind (tailwindcss-animate)

```tsx
<div className="animate-accordion-down" />  // abrir acordeao
<div className="animate-accordion-up" />    // fechar acordeao
<div className="animate-overlayShow" />     // overlay de dialog
<div className="animate-contentShow" />     // conteudo de dialog
```

### Transicoes CSS (preferivel para hover/focus simples)

```tsx
<div className="transition-colors duration-300 hover:bg-primary-100" />
<div className="transition-all duration-200 ease-out hover:scale-105" />
```

---

## Scrollbar

```tsx
// Ocultar scrollbar
<div className="no-scrollbar overflow-auto" />

// Scrollbar com estilo custom
<div className="custom-scrollbar overflow-auto" />

// Scrollbar gutter (reserva espaco)
<div className="scrollbar-gutter overflow-auto" />
```

---

## Cards

Padrao de card no projeto:

```tsx
<div className="w-full bg-white rounded-lg shadow-sm md:p-4 p-2 border border-zinc-300">
  {/* conteudo */}
</div>
```

| Propriedade | Valor              | Descricao                        |
|-------------|--------------------|----------------------------------|
| `shadow`    | `shadow-sm`        | Sombra leve (nao usar `shadow-md`) |
| `border`    | `border border-zinc-300` | Borda padrao em todos os cards |
| `rounded`   | `rounded-lg`       | Border radius padrao             |
| `padding`   | `md:p-4 p-2`       | Responsivo: menor no mobile      |

> **Regra:** Todo card deve ter `border border-zinc-300`. Use `shadow-sm`, nunca `shadow-md`.

---

## Border Radius

Valores via CSS var `--radius` (0.5rem):

```tsx
<div className="rounded-sm" />  // calc(var(--radius) - 4px)
<div className="rounded-md" />  // calc(var(--radius) - 2px)
<div className="rounded-lg" />  // var(--radius)
```

---

## Responsividade (Mobile-First)

```tsx
<div className="
  grid
  grid-cols-1
  gap-4
  px-4
  sm:grid-cols-2
  sm:gap-6
  sm:px-6
  lg:grid-cols-3
  lg:gap-8
  lg:px-8
">
```

### Breakpoints Padrao

| Prefixo | Min-width | Uso               |
|---------|-----------|-------------------|
| (none)  | 0px       | Mobile            |
| `sm:`   | 640px     | Mobile landscape  |
| `md:`   | 768px     | Tablet            |
| `lg:`   | 1024px    | Desktop           |
| `xl:`   | 1280px    | Desktop largo     |
| `2xl:`  | 1400px    | Container max     |

### Container Responsivo

```tsx
<div className="container mx-auto px-4 sm:px-6 lg:px-8">
  {children}
</div>
```

O container tem `center: true`, `padding: "2rem"` e `max-width: 1400px` em `2xl`.

---

## Regras de Styling

1. **SEMPRE** mobile-first: comece sem prefixo, adicione `sm:`, `md:`, `lg:`
2. **SEMPRE** use `cn()` de `@/common/utils/cn` para classes condicionais
3. **SEMPRE** use CVA para componentes com multiplas variantes
4. **SEMPRE** use CSS variables para tokens de tema (suportam dark mode e multi-tenant)
5. **SEMPRE** use `font-montserrat` ou heranca global — nao troque a fonte padrao
6. **SEMPRE** use `bg-primary-500 text-contrast` para botoes e acoes primarias
7. **NUNCA** use inline styles exceto para valores dinamicos calculados
8. **NUNCA** use `!important` — revise a especificidade
9. **NUNCA** crie classes CSS custom quando Tailwind utilities resolvem
10. **EVITE** `@apply` excessivo — use apenas em `@layer base` para estilos globais
11. **USE** `transition-*` para hover/focus states
12. **USE** `.dark` class no root para dark mode (controlado via Redux `isDark`)

---

## Checklist de Validacao

1. Fonte Montserrat aplicada (global ou via `font-montserrat`)?
2. Cores usando tokens do projeto (`primary-*`, `bg-*`, `contrast`)?
3. Mobile-first em todos os layouts?
4. `cn()` de `@/common/utils/cn` para classes condicionais?
5. CVA para componentes com variantes multiplas?
6. Dark mode via classe `.dark` (Redux `isDark`)?
7. Animacoes usando classes globais ou `tailwindcss-animate`?
8. Scrollbar com utilitarios `.no-scrollbar` / `.custom-scrollbar`?
9. Container com `className="container"` ou `mx-auto max-w-[1400px]`?
10. Sem inline styles desnecessarios?
