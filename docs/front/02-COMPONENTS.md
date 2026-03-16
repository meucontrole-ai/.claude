# Skill: Components

## Quando Ativar
- Criar/modificar componentes React
- Implementar patterns de composicao (Compound Components, Render Props, HOCs)
- Criar componentes de UI reutilizaveis (Design System primitives)
- Definir props e interfaces de componentes
- Implementar formularios e inputs controlados

## Quando NAO Ativar
- Gerenciar estado global (use State Management)
- Integrar com APIs (use Data Fetching)
- Estilizar componentes (use Styling)

---

## Anatomia de um Componente

```tsx
import { type ComponentPropsWithoutRef, forwardRef } from 'react'

interface ButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'primary' | 'secondary' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  isLoading?: boolean
}

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', size = 'md', isLoading = false, disabled, children, className, ...rest }, ref) => {
    return (
      <button
        ref={ref}
        disabled={disabled || isLoading}
        className={cn(buttonVariants({ variant, size }), className)}
        {...rest}
      >
        {isLoading ? <Spinner size={size} /> : children}
      </button>
    )
  }
)

Button.displayName = 'Button'

export { Button }
export type { ButtonProps }
```

### Regras do Componente

1. **Props explicitamente tipadas**: sempre declare interface de props separada
2. **Defaults via destructuring**: atribua valores default no destructuring, nao com `defaultProps`
3. **forwardRef em componentes de UI**: todo componente que renderiza um elemento HTML deve suportar ref
4. **displayName obrigatorio com forwardRef**: para debugging em DevTools
5. **Spread de props nativas**: use `...rest` para passar props nativas ao elemento raiz
6. **Extend de props nativas**: use `ComponentPropsWithoutRef<'element'>` para herdar props HTML

---

## Componentes com Estado

```tsx
// features/auth/components/LoginForm.tsx
import { useState } from 'react'
import { useAuth } from '@/features/auth/hooks/useAuth'
import { Button, Input } from '@/shared/components/ui'

function LoginForm() {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const { login, isLoading } = useAuth()

  async function handleSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault()
    await login({ email, password })
  }

  return (
    <form onSubmit={handleSubmit}>
      <Input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      <Input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Senha"
        required
      />
      <Button type="submit" isLoading={isLoading}>
        Entrar
      </Button>
    </form>
  )
}

export { LoginForm }
```

---

## Componentes de Apresentacao

Componentes sem estado que apenas recebem props e renderizam UI:

```tsx
// features/users/components/UserProfile.tsx
interface UserProfileProps {
  name: string
  email: string
}

function UserProfile({ name, email }: UserProfileProps) {
  return (
    <section>
      <h1>{name}</h1>
      <p>{email}</p>
    </section>
  )
}

export { UserProfile }
```

---

## Compound Components Pattern

Para componentes complexos com partes inter-relacionadas:

```tsx
import { createContext, useContext, useState, type ReactNode } from 'react'

interface AccordionContextValue {
  openItem: string | null
  toggle: (id: string) => void
}

const AccordionContext = createContext<AccordionContextValue | null>(null)

function useAccordionContext() {
  const context = useContext(AccordionContext)
  if (!context) {
    throw new Error('Accordion compound components must be used within Accordion')
  }
  return context
}

interface AccordionProps {
  children: ReactNode
  defaultOpen?: string
}

function Accordion({ children, defaultOpen = null }: AccordionProps) {
  const [openItem, setOpenItem] = useState<string | null>(defaultOpen)

  function toggle(id: string) {
    setOpenItem((prev) => (prev === id ? null : id))
  }

  return (
    <AccordionContext.Provider value={{ openItem, toggle }}>
      <div role="region">{children}</div>
    </AccordionContext.Provider>
  )
}

interface AccordionItemProps {
  id: string
  children: ReactNode
}

function AccordionItem({ id, children }: AccordionItemProps) {
  return <div data-accordion-item={id}>{children}</div>
}

interface AccordionTriggerProps {
  id: string
  children: ReactNode
}

function AccordionTrigger({ id, children }: AccordionTriggerProps) {
  const { openItem, toggle } = useAccordionContext()

  return (
    <button
      type="button"
      aria-expanded={openItem === id}
      aria-controls={`accordion-panel-${id}`}
      onClick={() => toggle(id)}
    >
      {children}
    </button>
  )
}

interface AccordionContentProps {
  id: string
  children: ReactNode
}

function AccordionContent({ id, children }: AccordionContentProps) {
  const { openItem } = useAccordionContext()

  if (openItem !== id) return null

  return (
    <div id={`accordion-panel-${id}`} role="region">
      {children}
    </div>
  )
}

Accordion.Item = AccordionItem
Accordion.Trigger = AccordionTrigger
Accordion.Content = AccordionContent

export { Accordion }
```

Uso:
```tsx
<Accordion defaultOpen="faq-1">
  <Accordion.Item id="faq-1">
    <Accordion.Trigger id="faq-1">Pergunta 1</Accordion.Trigger>
    <Accordion.Content id="faq-1">Resposta 1</Accordion.Content>
  </Accordion.Item>
  <Accordion.Item id="faq-2">
    <Accordion.Trigger id="faq-2">Pergunta 2</Accordion.Trigger>
    <Accordion.Content id="faq-2">Resposta 2</Accordion.Content>
  </Accordion.Item>
</Accordion>
```

---

## Controlled vs Uncontrolled Components

### Controlled (recomendado para formularios complexos)

```tsx
interface ControlledInputProps {
  value: string
  onChange: (value: string) => void
  label: string
  error?: string
}

function ControlledInput({ value, onChange, label, error }: ControlledInputProps) {
  return (
    <div>
      <label>{label}</label>
      <input
        value={value}
        onChange={(e) => onChange(e.target.value)}
        aria-invalid={!!error}
        aria-describedby={error ? `${label}-error` : undefined}
      />
      {error && <span id={`${label}-error`} role="alert">{error}</span>}
    </div>
  )
}
```

### Uncontrolled (formularios simples, melhor performance)

```tsx
function SimpleSearchForm({ onSearch }: { onSearch: (query: string) => void }) {
  function handleSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault()
    const formData = new FormData(event.currentTarget)
    const query = formData.get('query') as string
    onSearch(query)
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="query" type="search" placeholder="Buscar..." />
      <button type="submit">Buscar</button>
    </form>
  )
}
```

---

## Form Pattern com React Hook Form + Zod

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const createUserSchema = z.object({
  name: z.string().min(2, 'Nome deve ter pelo menos 2 caracteres').max(100),
  email: z.string().email('Email invalido'),
  role: z.enum(['admin', 'user', 'viewer']),
})

type CreateUserFormData = z.infer<typeof createUserSchema>

interface CreateUserFormProps {
  onSubmit: (data: CreateUserFormData) => Promise<void>
}

function CreateUserForm({ onSubmit }: CreateUserFormProps) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<CreateUserFormData>({
    resolver: zodResolver(createUserSchema),
  })

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <div>
        <label htmlFor="name">Nome</label>
        <input id="name" {...register('name')} aria-invalid={!!errors.name} />
        {errors.name && <span role="alert">{errors.name.message}</span>}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')} aria-invalid={!!errors.email} />
        {errors.email && <span role="alert">{errors.email.message}</span>}
      </div>

      <div>
        <label htmlFor="role">Papel</label>
        <select id="role" {...register('role')}>
          <option value="user">Usuario</option>
          <option value="admin">Administrador</option>
          <option value="viewer">Visualizador</option>
        </select>
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Criando...' : 'Criar Usuario'}
      </button>
    </form>
  )
}

export { CreateUserForm }
export type { CreateUserFormData }
```

---

## Render Props Pattern (escape hatch)

Use apenas quando composicao e props simples nao resolvem:

```tsx
interface RenderListProps<T> {
  items: T[]
  renderItem: (item: T, index: number) => ReactNode
  renderEmpty?: () => ReactNode
}

function RenderList<T>({ items, renderItem, renderEmpty }: RenderListProps<T>) {
  if (items.length === 0) {
    return renderEmpty ? renderEmpty() : null
  }

  return <ul>{items.map((item, index) => <li key={index}>{renderItem(item, index)}</li>)}</ul>
}
```

---

## Polymorphic Components

Para componentes que podem renderizar como diferentes elementos HTML:

```tsx
import { type ElementType, type ComponentPropsWithoutRef } from 'react'

type PolymorphicProps<E extends ElementType> = {
  as?: E
} & ComponentPropsWithoutRef<E>

function Text<E extends ElementType = 'p'>({ as, children, ...rest }: PolymorphicProps<E>) {
  const Component = as || 'p'
  return <Component {...rest}>{children}</Component>
}
```

Uso:
```tsx
<Text>Paragrafo padrao</Text>
<Text as="h1">Titulo</Text>
<Text as="span">Inline text</Text>
<Text as="label" htmlFor="email">Label</Text>
```

---

## Principios de Composicao

### Prefira composicao a props booleanas

Ruim:
```tsx
<Card showHeader showFooter showBorder variant="outlined" />
```

Bom:
```tsx
<Card>
  <Card.Header>Titulo</Card.Header>
  <Card.Body>Conteudo</Card.Body>
  <Card.Footer>Acoes</Card.Footer>
</Card>
```

### Prefira children a render props

Ruim:
```tsx
<Layout renderSidebar={() => <Sidebar />} renderContent={() => <Content />} />
```

Bom:
```tsx
<Layout>
  <Layout.Sidebar>
    <Navigation />
  </Layout.Sidebar>
  <Layout.Content>
    <Dashboard />
  </Layout.Content>
</Layout>
```

---

## Checklist de Validacao

1. Props tipadas com interface explicita?
2. `forwardRef` em componentes de UI que renderizam elementos HTML?
3. `displayName` definido em componentes com forwardRef?
4. Props nativas herdadas via `ComponentPropsWithoutRef`?
5. Defaults via destructuring (nao `defaultProps`)?
6. Formularios com validacao via Zod + React Hook Form?
7. Composicao preferida a props booleanas?
8. Componente com menos de 250 linhas?
9. Sem logica de negocio em componentes de UI (delegado para hooks/services)?
