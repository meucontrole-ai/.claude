# Optimistic Updates (RTK Query)

## Quando Ativar

- Mutation precisa refletir na UI ANTES da resposta do servidor (feel instantâneo)
- Toggle, check/uncheck, rename, delete — operações onde rollback é aceitável
- Remover sensação de "laggy" em listas grandes

---

## Quando NÃO usar

- Operações com validação server-side obrigatória (pagamento, approval)
- Mutations com efeitos colaterais visíveis ao usuário (criar pedido, conclude)
- Erros comuns (validation, permission) — preferir loading state e resposta real

---

## Padrão básico — `onQueryStarted`

```ts
// src/core/store/api/orders-api.ts
deliverItem: builder.mutation<void, { orderId: string; itemId: string }>({
  query: ({ orderId, itemId }) => ({
    url: `/orders/${orderId}/items/${itemId}/deliver`,
    method: "POST",
  }),
  async onQueryStarted({ orderId, itemId }, { dispatch, queryFulfilled }) {
    // Snapshot do cache atual pra rollback
    const patch = dispatch(
      ordersApi.util.updateQueryData("getOrder", orderId, (draft) => {
        const item = draft.items.find((i) => i.id === itemId);
        if (item) item.delivered = 1;
      }),
    );

    try {
      await queryFulfilled;  // espera resposta real
    } catch {
      patch.undo();  // rollback se servidor rejeitar
    }
  },
  // invalidatesTags só é útil pra refrescar do servidor DEPOIS (e rebater drift)
  invalidatesTags: (_r, _e, { orderId }) => [{ type: "Order", id: orderId }, "Order"],
}),
```

### Pontos-chave

1. **`updateQueryData`** mexe direto no cache — passa uma função que muta `draft` (Immer por baixo).
2. **`patch.undo()`** reverte se `queryFulfilled` rejeita.
3. Ainda use `invalidatesTags` pra reconciliar com server-side changes (outros usuários, triggers, etc.).

---

## Atualizar múltiplas queries

Se a mesma mutação afeta `useGetOrderQuery(id)` e `useListOpenOrdersQuery()`:

```ts
async onQueryStarted({ orderId, itemId }, { dispatch, queryFulfilled }) {
  const patches = [
    dispatch(ordersApi.util.updateQueryData("getOrder", orderId, (draft) => {
      const item = draft.items.find((i) => i.id === itemId);
      if (item) item.delivered = 1;
    })),
    dispatch(ordersApi.util.updateQueryData("listOpenOrders", undefined, (draft) => {
      const order = draft.find((o) => o.id === orderId);
      const item = order?.items.find((i) => i.id === itemId);
      if (item) item.delivered = 1;
    })),
  ];

  try {
    await queryFulfilled;
  } catch {
    patches.forEach((p) => p.undo());
  }
},
```

---

## Create com ID temporário

Para inserts, crie um ID temporário local até o server responder:

```ts
addComment: builder.mutation<Comment, NewComment>({
  query: (body) => ({ url: "/comments", method: "POST", body }),
  async onQueryStarted(newComment, { dispatch, queryFulfilled }) {
    const tempId = `temp-${Date.now()}`;
    const patch = dispatch(
      commentsApi.util.updateQueryData("listComments", newComment.postId, (draft) => {
        draft.push({ ...newComment, id: tempId, pending: true } as Comment);
      }),
    );

    try {
      const { data: saved } = await queryFulfilled;
      // troca o tempId pelo ID real
      dispatch(
        commentsApi.util.updateQueryData("listComments", newComment.postId, (draft) => {
          const idx = draft.findIndex((c) => c.id === tempId);
          if (idx >= 0) draft[idx] = saved;
        }),
      );
    } catch {
      patch.undo();
    }
  },
}),
```

UI pode destacar `pending: true` com opacidade reduzida.

---

## Delete otimista

```ts
deleteItem: builder.mutation<void, string>({
  query: (id) => ({ url: `/items/${id}`, method: "DELETE" }),
  async onQueryStarted(id, { dispatch, queryFulfilled }) {
    const patch = dispatch(
      itemsApi.util.updateQueryData("listItems", undefined, (draft) => {
        const idx = draft.findIndex((i) => i.id === id);
        if (idx >= 0) draft.splice(idx, 1);
      }),
    );

    try {
      await queryFulfilled;
    } catch {
      patch.undo();
    }
  },
}),
```

---

## Toast em erro para feedback

Roll-back silencioso frustra usuário ("cliquei, apareceu, sumiu, por quê?"). Mostre toast:

```ts
async onQueryStarted(args, { dispatch, queryFulfilled }) {
  const patch = dispatch(...);
  try {
    await queryFulfilled;
  } catch (err) {
    patch.undo();
    dispatch(showToast({
      variant: "error",
      message: "Não foi possível salvar. Tente novamente.",
    }));
  }
},
```

---

## Regras

1. Optimistic update só em ações **baixo-risco**: toggle, rename, delete leve.
2. Nunca optimistic em pagamento, approval, conclude — respeite validação do servidor.
3. Sempre tenha `patch.undo()` em catch.
4. Mostre **toast de erro** quando rollback ocorre — não deixe o usuário confuso.
5. Mantenha `invalidatesTags` pra reconciliação com mudanças externas.
6. Para inserts com ID temporário, troque pelo ID real após `queryFulfilled.data`.
