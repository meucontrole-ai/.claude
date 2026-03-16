# Generate Project Documentation

Analyze the repository and generate complete documentation in `ai_docs/` for humans and AI.

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
| Output | `ai_docs/` folder at project root |
| Accuracy | All info based on REAL code |
| Interaction | Question phase MANDATORY before generating |
| Language | English |
| Format | Markdown with tables, real code examples |

## Workflow

| Step | Action | Condition |
| --- | --- | --- |
| 1 | Repository discovery | Always |
| 2 | Ask user (8-12 questions) | Always |
| 3 | Generate documentation | After approval |
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

### Step 2: Questions (8-12)

Cover:
- **Purpose**: repo's role in ecosystem, business problem solved
- **Stack**: tech decisions and motivations
- **Features**: critical, planned, deprecated
- **Business Rules**: complex, compliance, essential validations
- **APIs** (if applicable): main endpoints, auth, rate limiting
- **Microservices** (if applicable): domain, communication, resilience
- **Integrations**: related repos, critical dependencies
- **Gotchas**: pitfalls, counter-intuitive behaviors, workarounds
- **What do you wish you knew when you started?**

Multiple rounds if needed. Present summary and ask for approval.

### Step 3: Generate Docs

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

### Step 4: Validate

- [ ] All content based on real code
- [ ] Functional examples
- [ ] Internal links working
- [ ] No generic/placeholder sections
- [ ] Gotchas captured
- [ ] Content specific to the project
- [ ] Optional files included only when relevant
