# Create Pull Request

Preparar branch, commitar, push e criar PR no GitHub.

## Constraints

| Regra | Detalhes |
| --- | --- |
| Idioma | Commits e PR em **ingles** |
| Force push | `--force-with-lease` (NUNCA `--force`) |
| Commit format | `{type}: {description}` |
| PR title format | `{type}({scope}): {title}` ou `{type}: {title}` |
| Commit types | `fix` \| `feat` \| `chore` \| `refactor` \| `test` \| `docs` |
| Max title | 70 caracteres |

## Progressive Context Loading

**IF** criando changeset **THEN** read `.claude/docs/back/05-DOMAIN-EVENTS.md` para entender eventos afetados

## Workflow

| Step | Action | Condition |
| --- | --- | --- |
| 1 | Determinar base branch | Sempre |
| 2 | Fetch e verificar estado do branch | Sempre |
| 3 | Stash, rebase, apply | Sempre (stash so se houver uncommitted) |
| 4 | Rodar testes | Sempre |
| 5 | Commit changes | So se houver uncommitted |
| 6 | Push branch | Sempre |
| 7 | Criar PR | Sempre |

### Step 1: Determinar Base Branch

```bash
git remote show origin | grep 'HEAD branch'
```

### Step 2: Fetch e Verificar

```bash
git fetch origin
git log origin/main..HEAD --oneline
git diff origin/main...HEAD --stat
```

### Step 3: Rebase

```bash
# Se houver uncommitted changes
git stash
git rebase origin/main
git stash pop

# Se nao houver
git rebase origin/main
```

Se houver conflitos: resolva, `git add`, `git rebase --continue`.

### Step 4: Rodar Testes

```bash
go test ./... -race
golangci-lint run
```

Se falhar: corrija, commite a correcao, repita.

### Step 5: Commit

```bash
git add <files>
git commit -m "type: description"
```

### Step 6: Push

```bash
# Primeira vez
git push -u origin HEAD

# Subsequentes
git push --force-with-lease
```

### Step 7: Criar PR

```bash
gh pr create --title "type(scope): title" --body "$(cat <<'EOF'
## Summary
- bullet points do que mudou

## Test plan
- [ ] testes passando
- [ ] lint passando

EOF
)"
```

## Validation

- [ ] Testes passando
- [ ] Lint passando
- [ ] PR criado com URL retornada
- [ ] Branch atualizado com base
