# Generate Project Documentation

Analyze the repository and generate complete documentation in `ai_docs/` for AI assistants, AND update the project `README.md` (and `docs/README.md` if it exists) to reflect the current state of the codebase.

## Usage

```
/generate-docs [additional-info]
```

<arguments>
$ARGUMENTS
</arguments>

## Constraints

| Rule | Details |
| --- | --- |
| Output | `ai_docs/` folder + updated `README.md` at project root |
| Accuracy | All info based on REAL code — never guess |
| Interaction | No questions — analyze and generate autonomously |
| Language | `ai_docs/` in English, `README.md` in PT-BR |
| Format | Markdown with tables, real code examples |

## Workflow

| Step | Action | Condition |
| --- | --- | --- |
| 1 | Repository discovery | Always |
| 2 | Generate AI docs (`ai_docs/`) | Always |
| 3 | Update README.md | Always |
| 4 | Validate | Always |

### Step 1: Discovery

Automatically analyze:
- Directory structure and architecture
- Dependencies (go.mod, package.json, requirements.txt, Cargo.toml)
- Tech stack, frameworks, libraries
- Design patterns (MVC, DDD, Clean Arch, Hexagonal)
- Endpoints, routes, entry points
- Business rules (validations, workflows, calculations)
- External integrations and service communication
- TODOs, FIXMEs, HACKs, workarounds in code
- CI/CD configs, Dockerfiles, K8s manifests
- Existing README.md — identify outdated sections

### Step 2: Generate AI Docs

Create `ai_docs/` with:

| File | Content | Required |
| --- | --- | --- |
| `index.md` | Index with links | Yes |
| `stack.md` | Languages, frameworks, infra, architectural decisions | Yes |
| `patterns.md` | Architectural and code patterns, naming, testing | Yes |
| `features.md` | Main, secondary, planned features | Yes |
| `business-rules.md` | Critical rules, validations, workflows, calculations, compliance | Yes |
| `gotchas.md` | Pitfalls, workarounds, non-obvious configs, tech debt | Yes |
| `integrations.md` | External services, events, contracts, resilience | Yes |
| `apis.md` | Endpoints, auth, rate limiting, examples | If exposes APIs |
| `services.md` | Domain, communication, persistence, SLAs | If microservice |

Each file:
- Title + one-liner
- Sections with tables (token-efficient)
- REAL code examples (from the project, not generic)
- Anti-patterns when relevant
- Non-applicable sections: omit entirely (don't leave empty)
- Undeterminable info: mark "TO BE COMPLETED" with instructions

### Step 3: Update README.md

Update the project `README.md` to reflect the **current** codebase. Also update `docs/README.md` if it exists.

The README MUST include (in order):
1. **Project title + one-liner description**
2. **Features table** — main capabilities
3. **Architecture** — diagram + explanation matching REAL folder structure
4. **Folder structure** — generated from actual `ls`, NOT copy-pasted from old docs
5. **Tech stack table** — with real versions from go.mod / package.json
6. **Setup instructions** — prerequisites, env vars, docker, run commands
7. **Available services** — local ports, credentials
8. **Useful commands** — make targets, test, lint, proto
9. **API examples** — grpcurl / curl examples for main endpoints
10. **Health check** — how to verify the service is running
11. **Environment variables table**
12. **Additional docs links** — only link files that actually exist
13. **License**

Rules:
- Remove references to directories/files that no longer exist
- Remove "Proximos Passos" checklist items that are already done
- Keep language in PT-BR for README.md
- Architecture diagram must match actual code structure
- Folder structure must be max 3 levels deep (don't list every file)
- If `docs/README.md` exists and is outdated, update it too or recommend removal

### Step 4: Validate

- [ ] All content based on real code
- [ ] Functional examples
- [ ] Internal links working
- [ ] No generic/placeholder sections
- [ ] Gotchas captured
- [ ] Content specific to the project
- [ ] Optional files included only when relevant
- [ ] README.md folder structure matches reality
- [ ] README.md tech versions match go.mod/package.json
- [ ] No dead links to non-existent docs
- [ ] README.md architecture diagram matches actual code
