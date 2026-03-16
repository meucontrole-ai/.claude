# Frontend Global Rules

Quick reference for React frontend conventions. Routing is handled by CLAUDE.md — this file contains only global rules that apply across ALL frontend skills.

## Stack

- **TypeScript** mandatory (strict mode enabled)
- **React 18+** as UI library
- **Vite** as bundler
- Adapt for other frameworks when project requires, keeping principles

## Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Components | PascalCase | `UserProfile`, `PaymentForm`, `OrderList` |
| Custom hooks | `use` prefix camelCase | `useAuth`, `useDebounce`, `useLocalStorage` |
| Utility functions | camelCase with verbs | `formatCurrency`, `validateEmail`, `parseDate` |
| Types/Interfaces | PascalCase, no `I` prefix | `UserProfile`, `PaymentResult` (NOT `IUserProfile`) |
| Constants | UPPER_SNAKE_CASE | `API_BASE_URL`, `MAX_RETRY_COUNT` |
| Component files | PascalCase | `UserProfile.tsx`, `PaymentForm.tsx` |
| Utility files | camelCase | `formatCurrency.ts`, `validators.ts` |
| Test files | `.test.ts` / `.test.tsx` suffix | `UserProfile.test.tsx` |
| Directories | kebab-case | `user-profile/`, `payment-form/` |

## TypeScript Strict

- `strict: true` in tsconfig.json (includes `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`)
- NEVER use `any` — use `unknown` and do type narrowing
- NEVER use `as` for type assertion except in tests or with verifiable guarantees
- Prefer `interface` for objects/contracts, `type` for unions/intersections/primitives

## Principles

- **SOLID**, **KISS**, **DRY**, **YAGNI**
- **Composition over Inheritance**: prefer component composition
- **Colocation**: keep related code together (styles, tests, types next to component)
- **Immutability**: treat state as immutable. Never mutate objects/arrays directly

## Modularization

- Component > 250 lines: MUST be split
- One component per file
- Group by feature/domain, NOT by technical type

## Coverage

- Minimum 80% coverage for hooks and utility functions
- Every critical component (forms, payments, auth) must have tests
- Every custom hook must have unit tests

## Code Style

- No inline comments (only JSDoc on exported functions and complex utilities)
- No emojis in files, comments, or code
