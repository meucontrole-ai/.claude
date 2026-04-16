# Modal & Collapsible Patterns

## Quando Ativar

- Criar/modificar modais que mudam de tamanho dinamicamente
- Painéis com seções colapsáveis (Accordion)
- Flows complexos onde modal-pai e modal-filho precisam coordenar fechamento
- UI "pula" ao mostrar/esconder conteúdo

---

## Transição suave no Modal base

Modais que crescem/encolhem com o conteúdo dão sensação de "salto". Use transição CSS no contêiner + animate-in do entry:

```tsx
// src/components/ui/modal.tsx
<div
  className={cn(
    "relative z-10 bg-card border border-border shadow-xl flex flex-col",
    "transition-[max-height,max-width,transform] duration-200 ease-out",
    "motion-safe:animate-in motion-safe:fade-in motion-safe:zoom-in-95",
    isFull ? "absolute inset-0 rounded-none" : cn("w-full mx-4 max-h-[90vh] rounded-2xl", sizeClasses[size]),
  )}
>
  {children}
</div>
```

---

## Seções colapsáveis com grid-rows trick

Animar `height: auto` não funciona em CSS. Use o truque do `grid-template-rows: 1fr ↔ 0fr`:

```tsx
// src/components/ui/collapsible-section.tsx
export function CollapsibleSection({ title, summary, defaultOpen = false, children }: Props) {
  const [open, setOpen] = useState(defaultOpen);

  return (
    <div className="rounded-xl border border-border">
      <button
        type="button"
        onClick={() => setOpen((v) => !v)}
        aria-expanded={open}
        className="flex w-full items-center justify-between gap-3 px-3 py-2 text-left hover:bg-muted/40"
      >
        <h4 className="text-xs font-semibold uppercase">{title}</h4>
        <div className="flex items-center gap-2">
          {summary && <span className="text-xs text-muted-foreground">{summary}</span>}
          <ChevronDown className={cn("size-4 transition-transform duration-200", open && "rotate-180")} />
        </div>
      </button>
      <div
        className={cn(
          "grid transition-[grid-template-rows] duration-200 ease-out",
          open ? "grid-rows-[1fr]" : "grid-rows-[0fr]",
        )}
      >
        <div className="overflow-hidden">
          <div className="px-3 pb-3 pt-1">{children}</div>
        </div>
      </div>
    </div>
  );
}
```

Por que funciona: `grid-template-rows` É animável, e o filho `overflow-hidden` com altura automática se ajusta ao `1fr`.

---

## Modal-pai + Modal-filho: propagação de fechamento

Quando um modal abre outro, e o filho tem uma ação que deve **fechar ambos** (ex: concluir um fluxo fecha o filho E o pai), o filho NÃO pode apenas chamar seu próprio `onClose`. Precisa propagar via callback dedicado.

### Padrão: callback `onFinalized` opcional

```tsx
interface ChildModalProps {
  open: boolean;
  onClose: () => void;        // fecha só esse modal
  onFinalized?: () => void;   // sinaliza ao pai: "feche você também"
}

async function handleFinalize() {
  await doAction();
  setShowConfirm(false);
  doClose();           // fecha este modal
  onFinalized?.();     // propaga ao pai
}
```

```tsx
// Componente pai
<ChildModal
  open={showChild}
  onClose={() => setShowChild(false)}   // user cancelou, mantém parent aberto
  onFinalized={() => {
    setShowChild(false);
    onClose();                          // ← fecha o pai também
  }}
/>
```

### Armadilha: `open={outer && !inner}` re-abre sozinho

Se o pai usa `<Modal open={open && !showChild}>` pra esconder enquanto o filho está aberto, quando o filho fecha (só `showChild` vira false), o pai **reaparece sozinho** porque `open` ainda é true.

**Sintoma típico**: usuário completa uma ação no filho, ele fecha, e o pai aparece de novo inesperadamente.

**Fix**: propagar `onFinalized` pro pai e ele também fecha.

---

## Gatilho automático via `useEffect` + flag de dismiss

Modal que deve aparecer em resposta a estado derivado (ex: alguma condição fica `true`), mas o usuário pode optar por ignorar:

```tsx
const [showAutoModal, setShowAutoModal] = useState(false);
const [dismissed, setDismissed] = useState(false);

const conditionMet = computeCondition(state);

useEffect(() => {
  if (conditionMet && !dismissed && !showAutoModal) {
    setShowAutoModal(true);
  }
  if (!conditionMet && dismissed) {
    setDismissed(false);  // reset se condição voltar a ser false
  }
}, [conditionMet, dismissed, showAutoModal]);

function handleDecline() {
  setShowAutoModal(false);
  setDismissed(true);     // não reabrir até estado mudar
}
```

Sem a flag `dismissed`, o modal abre de novo toda vez que re-renderiza.

---

## Checkbox com indicador visual

Tick real (✓ em caixa), não só texto "Marcar":

```tsx
<button
  type="button"
  role="checkbox"
  aria-checked={checked}
  disabled={readOnly}
  onClick={onToggle}
  className="flex w-full items-center gap-3 rounded-lg border px-3 py-2"
>
  <span className={cn(
    "flex size-5 shrink-0 items-center justify-center rounded border-2",
    checked ? "border-emerald-500 bg-emerald-500 text-white" : "border-border bg-background",
  )}>
    {checked && <Check className="size-3.5" strokeWidth={3} />}
  </span>
  <span className="flex-1">{label}</span>
</button>
```

---

## Regras

1. Use `transition-[max-height]` + `animate-in fade-in zoom-in-95` no Modal base pra entry suave.
2. Para seções colapsáveis, use o truque `grid-template-rows: 1fr ↔ 0fr` com filho `overflow-hidden`.
3. Modais aninhados: filho expõe `onFinalized?` e o pai decide se fecha junto.
4. `useEffect` + flag "dismissed" pra triggers automáticos que o usuário pode cancelar.
5. Checkbox visual: `role="checkbox"` + ícone `Check` real dentro de caixa colorida.
6. `motion-safe:` em animações — respeita `prefers-reduced-motion`.
