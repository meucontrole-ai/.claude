# Realtime (SSE) & Cache Invalidation

## Quando Ativar

- Implementar push de eventos do backend pro frontend (SSE, WebSocket)
- Invalidar cache RTK Query quando evento externo atualiza dados
- Criar dashboards que precisam refletir mudanças em outras instâncias
- Debugar dados desatualizados apesar de o backend ter mudado

---

## SSE (Server-Sent Events) vs WebSocket

| | SSE | WebSocket |
|---|---|---|
| Uplink | ❌ só server→client | ✅ bi-direcional |
| Reconnect automático | ✅ built-in no browser | ❌ manual |
| Proxy/firewall-friendly | ✅ HTTP/2 simples | ⚠️ requer upgrade |
| Escopo típico | notificações, live feeds | chat, colaboração |

Para dashboards, painéis admin e live feeds, **SSE é suficiente e mais simples**.

---

## Padrão: hook `useSSE` + invalidação de cache

```ts
// src/core/hooks/use-sse.ts
import { useEffect } from "react";
import { useAppDispatch, getApiUrl, tablesApi, ordersApi } from "@/core/store";

type SSEEvent = {
  type: string;
  resource: string;
  id?: string;
};

export function useSSE() {
  const dispatch = useAppDispatch();

  useEffect(() => {
    const es = new EventSource(`${getApiUrl()}/orders/events/stream`);

    es.onmessage = (msg) => {
      try {
        const evt: SSEEvent = JSON.parse(msg.data);
        invalidateByEvent(evt);
      } catch { /* ignore malformed */ }
    };

    es.onerror = () => {
      // browser will auto-reconnect; log only for debugging
      console.debug("[SSE] connection dropped, auto-reconnecting");
    };

    return () => es.close();
  }, [dispatch]);

  function invalidateByEvent(evt: SSEEvent) {
    if (evt.resource === "table") {
      dispatch(tablesApi.util.invalidateTags(
        evt.id ? [{ type: "Table", id: evt.id }, "Table"] : ["Table"],
      ));
    }
    if (evt.resource === "order") {
      dispatch(ordersApi.util.invalidateTags(
        evt.id ? [{ type: "Order", id: evt.id }, "Order"] : ["Order"],
      ));
    }
  }
}
```

Use no `App.tsx` top-level:

```tsx
function App() {
  useSSE();
  return <Routes />;
}
```

---

## Invalidação granular vs genérica

Mesma regra de `04-DATA-FETCHING.md` vale pra SSE: **invalide AMBAS** a tag específica `{type, id}` E a genérica `type`. Senão queries de lista não refetcham.

```ts
// ❌ errado — só invalida item, não lista
dispatch(ordersApi.util.invalidateTags([{ type: "Order", id: orderID }]));

// ✅ correto — força refetch da lista também
dispatch(ordersApi.util.invalidateTags([{ type: "Order", id: orderID }, "Order"]));
```

---

## Broadcast helper no backend (Fiber)

```go
// adapters/http/handler/sse_hub.go
type SSEHub struct {
    clients map[chan string]struct{}
    mu      sync.Mutex
}

func (h *SSEHub) Broadcast(eventType, resource, id string) {
    payload := fmt.Sprintf(`{"type":"%s","resource":"%s","id":"%s"}`, eventType, resource, id)
    h.mu.Lock()
    for ch := range h.clients {
        select {
        case ch <- payload:
        default: // drop if slow consumer
        }
    }
    h.mu.Unlock()
}

// Uso em handlers
BroadcastTableEvent(sseHub, "table.occupied", tableID)
```

---

## Polling como fallback

Para endpoints críticos (ex: KDS), combine SSE + polling periódico:

```ts
const { data } = useListOpenOrdersQuery(undefined, {
  pollingInterval: 15_000,  // 15s fallback
});
```

SSE entrega push instantâneo; polling cobre reconexões curtas e ambientes com proxy que dropam SSE.

---

## Cuidados

### 1. Authentication em SSE

`EventSource` não aceita headers custom. Se sua API exige `Authorization: Bearer`, passe o token via query string OU cookie:

```ts
new EventSource(`${apiUrl}/stream?token=${authToken}`);
```

Backend deve validar o token antes de iniciar o stream.

### 2. Backpressure

Cliente lento = buffer enchendo. Broadcast usa `select { case ch <- payload: default: }` pra descartar em vez de bloquear o hub.

### 3. Reconexão

Browsers fazem reconnect automático no `EventSource`. Use o campo `Last-Event-ID` pra enviar só eventos novos após reconect:

```go
lastID := c.Get("Last-Event-ID")
events := queryEventsSince(lastID)
for _, e := range events {
    c.Write([]byte(fmt.Sprintf("id: %s\ndata: %s\n\n", e.ID, e.Payload)))
}
```

---

## Regras

1. SSE pra push simples server→client; WebSocket só se precisar uplink ou binary.
2. Um hook `useSSE` centralizado no `App.tsx`, não espalhado por componentes.
3. Ao receber evento, invalidar tag **específica + genérica** no RTK Query.
4. Polling como fallback em endpoints críticos (15-30s).
5. Backend broadcast não bloqueante (`select default` pra drop de cliente lento).
6. Token no query string ou cookie — `EventSource` não aceita header.
7. Reconexão automática via `Last-Event-ID` para recuperar eventos perdidos.
