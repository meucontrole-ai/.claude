# Multi-Tenancy (Schema-per-tenant)

## Quando Ativar

- Adicionar suporte a múltiplos merchants/tenants no mesmo banco
- Diagnosticar queries batendo no schema errado (dados "sumindo")
- Migrar dados entre tenants
- Debugar `TRUNCATE` ou `DELETE` que não teve efeito (provavelmente rodou no schema errado)

---

## Arquitetura

- **Um schema PostgreSQL por merchant**, nomeado como `tenant_<uuid_sem_hifens>`.
- Connection pool **única conexão** (`MaxOpenConns=1`) pra garantir que `SET search_path` afeta TODAS as queries subsequentes.
- Schema ativado via `cfg.MerchantID` no bootstrap.
- Migrations rodam por schema (cada tenant tem as tabelas próprias isoladas).

```go
// bootstrap/service.go
sqlDB.SetMaxOpenConns(1)
sqlDB.SetMaxIdleConns(1)

schemaName, _ := tenant.SchemaName(cfg.MerchantID)  // tenant_<hex>
db.Exec(fmt.Sprintf("CREATE SCHEMA IF NOT EXISTS %s", schemaName))
db.Exec(fmt.Sprintf("SET search_path TO %s, public", schemaName))
postgres.RunMigrations(db)
cfg.ActiveTenantSchema = schemaName
```

---

## Pitfall #1: `TRUNCATE` ou DELETE no schema errado

Quando você abre `psql` manualmente pra fazer limpeza, o search_path padrão é `public`. Se os dados vivem em `tenant_<uuid>`, o TRUNCATE roda mas **não afeta nada**:

```bash
# ❌ errado — zera public que está vazio
psql -U <user> -d <db> -c "TRUNCATE event_logs;"
# TRUNCATE TABLE  (success, mas 0 rows affected)

# ✅ certo — qualifica com o schema do tenant
psql -U <user> -d <db> -c "TRUNCATE tenant_<uuid>.event_logs;"
```

Pra limpar TODOS os tenants:

```bash
psql -U <user> -d <db> -t \
  -c "SELECT nspname FROM pg_namespace WHERE nspname LIKE 'tenant_%';" \
  | awk 'NF' | while read schema; do
    psql -U <user> -d <db> \
      -c "TRUNCATE $schema.<tabela_alvo>;"
done
```

---

## Pitfall #2: MaxOpenConns > 1 faz `SET search_path` vazar

Postgres pools múltiplas conexões. Cada conexão tem seu próprio `search_path`. Se você rodar:

```go
db.SetMaxOpenConns(10)
db.Exec("SET search_path TO tenant_x, public")
// query que cai numa conexão DIFERENTE → search_path = public ainda
db.First(&order, "id = ?", orderID)  // ❌ busca em public, não em tenant_x
```

Solução: **MaxOpenConns=1** pra esse serviço, OU use `SET LOCAL` dentro de cada transação (requer BEGIN/COMMIT explícito).

---

## Pitfall #3: `SwitchTenant` não afeta DB workers já rodando

Se você tem workers (outbox relay, telemetry sync) que pegaram handler de DB ANTES do switch, eles continuam no schema antigo. Sempre que trocar de tenant em runtime:

1. Drenar filas dos workers em background
2. Chamar `SwitchTenant(newSchema)`
3. Reiniciar workers

Pra evitar complexidade, o mais comum é NÃO trocar de tenant em runtime — use uma instância por merchant/tenant (compose/pods separados).

---

## Descoberta de schemas existentes

```sql
SELECT nspname, pg_get_userbyid(nspowner) AS owner
FROM pg_namespace
WHERE nspname LIKE 'tenant_%'
ORDER BY nspname;
```

---

## Migrations

Cada migration roda **no schema ativo**. Ao adicionar nova migration, teste contra TODOS os schemas de tenant existentes:

```go
// migration.go
if err := postgres.RunMigrations(db); err != nil {
    return fmt.Errorf("failed to run migrations in schema %s: %w", schemaName, err)
}
```

No deploy, itere:

```bash
for schema in $(psql -tAc "SELECT nspname FROM pg_namespace WHERE nspname LIKE 'tenant_%'"); do
  POSTGRES_SEARCH_PATH=$schema ./migrator up
done
```

---

## Regras

1. Schema ativo é definido por `cfg.MerchantID` no bootstrap, não na query.
2. `MaxOpenConns=1` nesse serviço pra garantir `SET search_path` global.
3. Qualquer comando manual (TRUNCATE, DELETE, ALTER) DEVE qualificar o schema: `tenant_<uuid>.table_name`.
4. Não trocar de tenant em runtime se houver workers ativos — reinicie o processo.
5. Migrations rodam por schema; deploy itera todos os tenants.
6. Debug de "dados sumidos": primeiro cheque o schema ativo (`SHOW search_path`), depois a tabela.
