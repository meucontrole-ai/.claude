# Event Log & Audit Trail

## Quando Ativar

- Implementar tabela de auditoria (`event_logs`) pra rastrear mudanças de agregados
- Exibir timeline por recurso (por ID, por sessão, por usuário)
- Adicionar soft-delete com trilha de quem/quando apagou
- Debugar eventos agrupados errado ou "sumidos" na timeline

---

## Modelo mínimo

```go
// domain/eventlog/event_log.go
type EventLog struct {
    ID            string
    AggregateID   string      // ID do recurso logado
    AggregateType string      // categoria: "ORDER", "USER", "SYSTEM", ...
    EventType     string      // "<aggregate>.<verb>": "order.created", etc.
    Actor         string      // quem disparou (user id/email ou "system")
    SessionID     string      // amarra eventos a uma sessão/correlation
    Snapshot      string      // JSON do agregado no momento (estado final)
    Diff          string      // JSON com mudanças relevantes + identidade
    Metadata      string      // JSON com extras não-tipados
    CreatedAt     time.Time

    // Soft-delete (preenche só quando "limpado" via UI/admin)
    ClearedAt         time.Time
    ClearedBy         string
    ClearedClientTime time.Time  // relógio do cliente — detecta drift
}
```

### Schema SQL correspondente

```sql
CREATE TABLE event_logs (
    id                   VARCHAR(36) PRIMARY KEY,
    aggregate_id         VARCHAR(255) NOT NULL,
    aggregate_type       VARCHAR(50)  NOT NULL,
    event_type           VARCHAR(100) NOT NULL,
    actor                VARCHAR(100) NOT NULL DEFAULT 'system',
    session_id           VARCHAR(36)  DEFAULT '',
    snapshot             JSONB        NOT NULL,
    diff                 JSONB        NOT NULL DEFAULT '{}',
    metadata             JSONB        NOT NULL DEFAULT '{}',
    created_at           TIMESTAMP    NOT NULL,
    cleared_at           TIMESTAMP,
    cleared_by           VARCHAR(120) DEFAULT '',
    cleared_client_time  TIMESTAMP
);

CREATE INDEX idx_event_logs_aggregate  ON event_logs(aggregate_id, aggregate_type, created_at);
CREATE INDEX idx_event_logs_session    ON event_logs(session_id);
CREATE INDEX idx_event_logs_created_at ON event_logs(created_at);
CREATE INDEX idx_event_logs_cleared_at ON event_logs(cleared_at);
```

---

## Regra #1: SessionID top-level, sempre

Eventos em contexto de uma sessão DEVEM carregar o `SessionID` no `LogParams`. Consumers (timeline UI, queries de agrupamento) filtram pela coluna top-level. Esquecer = evento cai no bucket "sem sessão" ou cria grupo fantasma na UI.

```go
sessionID := ""
if hasSessionContext(ctx, aggregate) {
    sessionID = loadSessionID(ctx, aggregate)
}
eventLogger.Log(ctx, shared.LogParams{
    AggregateID:   agg.ID,
    AggregateType: "AGGREGATE_TYPE",
    EventType:     "aggregate.verb",
    SessionID:     sessionID,  // ← top-level, não só no diff
    Actor:         actor(ctx),
    Snapshot:      toResult(agg),
    Diff:          map[string]any{...},
})
```

---

## Regra #2: Diff replica identidade do parent pra queries de grouping funcionarem

Eventos cujo `aggregate_id` é diferente do ID pelo qual você quer agrupar precisam replicar a identidade do parent no `diff`:

```go
eventLogger.Log(ctx, shared.LogParams{
    AggregateID:   parent.ID,
    AggregateType: "PARENT",
    EventType:     "parent.state_changed",
    SessionID:     parent.SessionID,
    Diff: map[string]any{
        "parent_id":   parent.ID,
        "parent_name": parent.Name,
        "session_id":  parent.SessionID,
        "reason":      reason,
    },
})
```

Isso facilita consultas que agrupam por `(parent_id, session_id)` independente do tipo de evento.

---

## Regra #3: Snapshot = estado final; Diff = delta + identidade

- `snapshot` = agregado completo DEPOIS da mudança (JSON do Result DTO).
- `diff` = só os campos que mudaram + identidade (ids, display names, session correlation).

UI renderiza `diff` como contexto curto e usa `snapshot` pra detalhes completos.

---

## Timeline por sessão — scope-limitado

Timeline de um recurso ativo deve mostrar só a sessão ATUAL. Cada re-ativação gera `SessionID` novo — histórico das sessões antigas fica acessível via `/event-logs` geral com filtro de data.

```go
func (uc *UseCase) Execute(ctx context.Context, aggregateID string) (any, error) {
    agg, _ := aggregateRepo.FindByID(ctx, aggregateID)
    if agg.SessionID == "" {
        return emptyResult, nil
    }
    events, _ := eventLogRepo.FindBySessionID(ctx, agg.SessionID)
    sort.Slice(events, func(i, j int) bool { return events[i].CreatedAt.Before(events[j].CreatedAt) })
    return buildResult(events), nil
}
```

**NÃO faça merge** de eventos de todas as sessões históricas do mesmo agregado — vira ruído. Sessão atual isolada é o comportamento correto.

---

## Soft-delete ("Limpar logs")

Limpar via UI deve ser **soft**: UPDATE marcando `cleared_at`/`cleared_by`/`cleared_client_time`, NUNCA DELETE. Motivo: auditoria tem que mostrar quem apagou e quando (pelo relógio do cliente, pra detectar drift vs server).

```go
func (r *EventLogRepository) SoftClearOlderThan(
    ctx context.Context, before time.Time, clearedBy string, clientTime time.Time,
) (int64, error) {
    result := r.db.WithContext(ctx).
        Model(&model.EventLogModel{}).
        Where("created_at < ? AND cleared_at IS NULL", before).
        Updates(map[string]interface{}{
            "cleared_at":          time.Now().UTC(),
            "cleared_by":           clearedBy,
            "cleared_client_time": clientTime,
        })
    return result.RowsAffected, result.Error
}
```

Depois do soft-clear, emita um evento `logs.cleared` (AggregateType=SYSTEM) registrando a ação:

```go
_ = eventLogRepo.Save(ctx, eventlog.EventLog{
    ID:            shared.GenerateID(),
    AggregateID:   "system",
    AggregateType: "SYSTEM",
    EventType:     "logs.cleared",
    Actor:         clearedBy,
    Diff: map[string]any{
        "deleted_count":       deleted,
        "cleared_by":          clearedBy,
        "cleared_client_time": clientTime,
        "before":              before,
    },
    CreatedAt: time.Now().UTC(),
})
```

Esse novo evento nasce com `cleared_at = NULL` → aparece ativo no timeline administrativo.

---

## Endpoint de purge exige `clearedBy`

```go
// HTTP: DELETE /event-logs?older_than_days=N
// body: {"clearedBy":"<nome>","clientTime":"<ISO8601>"}
if cmd.ClearedBy == "" {
    return shared.NewDomainError(shared.ErrCodeInvalidArgument, "clearedBy é obrigatório")
}
```

Aceita `older_than_days=0` como escape hatch pra wipe total.

---

## API expõe campos de soft-delete top-level

```json
{
  "id": "...",
  "aggregate_id": "...",
  "event_type": "...",
  "session_id": "<uuid>",
  "snapshot": { ... },
  "diff": { ... },
  "cleared_at": "2026-04-15T22:24:01Z",
  "cleared_by": "Fulano",
  "cleared_client_time": "2026-04-15T19:23:50Z"
}
```

Senão o cliente precisa fazer lookup dentro do diff e a filtragem fica inconsistente.

---

## Regras

1. Todo evento em contexto de sessão carrega `SessionID` no `LogParams` E no `diff`.
2. Eventos cujo `aggregate_id` difere do ID de agrupamento replicam a identidade no `diff`.
3. `snapshot` = estado final pós-mudança. `diff` = só o delta + identidade.
4. Timeline por recurso mostra só a sessão ATUAL (não merged history).
5. Limpar logs = UPDATE (soft-delete), NUNCA DELETE. Exige `clearedBy`.
6. Depois de soft-clear, emitir evento `logs.cleared` (SYSTEM) com contador + autor.
7. API expõe `session_id`, `cleared_at`, `cleared_by`, `cleared_client_time` top-level.
