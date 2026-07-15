# GraphQL Codegen

**Status:** Understood (React); port to Angular pending
**Tool:** `@graphql-codegen/cli`
**Scope:** object-search feature (external + internal apps)

---

## What It Does

Generates TypeScript types from a GraphQL schema + operation files, so GraphQL network calls are type-safe at compile time in **both directions** — request variables and response shape. It produces **types only**: no runtime code, no framework-specific code. Types are erased at compile, so codegen has **zero runtime footprint**. It is a developer maintenance tool, not a runtime dependency.

## Two Inputs → One Output

| Input | Gives you |
|---|---|
| `schema.graphql` (org introspection) | schema-derived types — `Account_Filter`, `Account_OrderBy`, `Scalars`, enums |
| `.graphql` operation files | operation types — `<Name>Query` + `<Name>QueryVariables`, one pair per file |

**Output:** a single committed file `src/api/graphql-operations-types.ts` (~11,260 lines). Two zones:
- **Top** (~lines 2–11130) = schema-derived types (from `schema.graphql`)
- **Bottom** (~11131–11260) = operation types (one block appended per `.graphql` file)

## Name Derivation

Type names come mechanically from the operation name in the `.graphql` file:

```graphql
query SearchAccounts($first: Int, $after: String, $where: Account_Filter, $orderBy: Account_OrderBy) { ... }
```

→ `SearchAccountsQuery` (response) + `SearchAccountsQueryVariables` (variables). Rename the query → the generated type names change with it. Delete the `.graphql` file + regenerate → its type block **vanishes** from the output; schema types at the top stay.

## The Command

`package.json`:
```json
"graphql:codegen": "graphql-codegen",
"graphql:schema": "node scripts/get-graphql-schema.mjs"
```

- **`graphql:schema`** — introspects a live org (`SF_TARGET_ORG` env or default `sf` org) → writes `schema.graphql`.
- **`graphql:codegen`** — runs the CLI against `codegen.yml`: loads schema → scans `documents: src/**/*.{graphql,ts,tsx}` → runs plugins (`typescript-operation-types` + `typescript-operations`) → writes output with `overwrite: true`.

`codegen.yml` config knobs: `onlyOperationTypes`, `skipTypename`, `preResolveTypes`, and the key value-add — a **scalar map** teaching TS how Salesforce scalars serialize (`Currency`/`BigDecimal`/`Double` → `number`; `Date`/`Picklist`/`Email` → `string`).

## Key Gotchas

- **`schema.graphql` is NOT committed** — fetched on demand. If absent, codegen **silently skips** (build log: "schema.graphql not found — skipping codegen"). Regeneration requires an authenticated org first.
- **The generated output IS committed** — so using the feature needs no codegen. Codegen is only for the maintenance loop (change a query, add a query, org schema changes).
- **`dist/` is gitignored** — a composed build artifact. It has full codegen wiring and is runnable, but edits there don't stick (regenerated from `src/` on next build). Real source lives in `feature-react-object-search/src/` (authoritative) and the app `src/`.
- **Generated types are framework-agnostic** — plain TS via `typescript-operations` (NOT `typescript-react-apollo`). Reusable by Angular verbatim.

## Angular Design (decided) — `.ts` const queries in `feature-angular-object-search`

The Angular side does NOT copy React's `.graphql` files. Instead all operations live as **`.ts` const exports in `feature-angular-object-search`** (framework-neutral GraphQL document constants), e.g.:

```ts
export const SEARCH_ACCOUNTS = /* GraphQL */ `query SearchAccounts(...) { ... }`;
```

The 4 operations: `searchAccounts.ts`, `getAccountDetail.ts`, `distinctAccountIndustries.ts`, `distinctAccountTypes.ts` (in `feature-angular-object-search/.../src/account/`), re-exported via `index.ts`. Consumed by `feature-angular-object-search` itself and the other Angular features (`feature-angular-authentication`, `feature-angular-agentforce-conversation-client`).

### This `.ts` form IS codegen-compatible (verified empirically)

The `/* GraphQL */` magic comment before the template literal is a **recognized codegen marker**. graphql-codegen extracts GraphQL from `.ts`/`.tsx` via `@graphql-tools/graphql-tag-pluck`, which reads three sources: `.graphql` files, `gql`/`graphql` tagged literals, and **plain template literals prefixed with `/* GraphQL */`**.

**Proven:** ran `gqlPluckFromCodeStringSync` (the installed pluck lib, v8.3.27) against `searchAccounts.ts` with zero config → it extracted the full `SearchAccounts` query. So the existing `@graphql-codegen/cli@6.2.1` toolchain WILL generate types from the const queries — no code changes to the operations needed.

This `.ts` approach is arguably **better than React's**: one file gives both the runtime query string (importable const) AND the codegen document. React needs two mechanisms (`.graphql` for codegen + `?raw` import for runtime).

### The gap in the current design

`feature-angular-object-search` has the `.ts` operations but **NO codegen wiring** — no `codegen.yml`, no `graphql:codegen` script, no generated `graphql-operations-types.ts`. So consumers currently **hand-write** types: `account-search.service.ts` declares its own `interface SearchAccountsData` / `AccountSearchArgs` and imports only the query string. That is exactly the drift risk codegen exists to prevent (edit query → hand-types silently stale).

### Where the wiring belongs

Rule: **codegen.yml lives in the package that owns the operations + emits the types.** Angular consolidated all operations into `feature-angular-object-search`, so:

| Piece | Home | Why |
|---|---|---|
| `codegen.yml` + `graphql:codegen` script + devDeps | `feature-angular-object-search` | co-located with operations it reads + types it writes |
| generated `graphql-operations-types.ts` | `feature-angular-object-search/.../src/api/` | consumed by all Angular features via barrel |
| `get-graphql-schema.mjs` + `.graphqlrc.yml` | `base-angular-app` | shared schema-fetch infra (mirrors React: base-react-app owns these) |

NOT `base-angular-app` for codegen.yml — base owns no operations. React parallel: `feature-react-object-search` AND `base-react-app` each have their own codegen.yml because each owns operations; base-*-app owns the shared schema-fetch script.

**Next step (not yet done — no code written):** add `codegen.yml` (adapt React's, repoint output to object-search `src/api/`) + `graphql:codegen` script + 3 devDeps (`@graphql-codegen/cli`, `graphql-codegen-typescript-operation-types`, `typescript-operations`) to `feature-angular-object-search`; fetch schema; regenerate; then delete the hand-written interfaces in consumers and import generated types instead.

## File Reference (React, authoritative source)

- Generated types: `packages/template/feature/feature-react-object-search/src/force-app/main/default/uiBundles/feature-react-object-search/src/api/graphql-operations-types.ts`
- Codegen config: `packages/template/feature/feature-react-object-search/codegen.yml`
- Operation files: `.../feature-react-object-search/src/features/object-search/__examples__/api/query/` — `searchAccounts.graphql`, `getAccountDetail.graphql`, `distinctAccountIndustries.graphql`, `distinctAccountTypes.graphql`
- Schema script: `.../scripts/get-graphql-schema.mjs` (in composed bundles)

## Related

- [[template-generator]] — `sf template generate` wiring
- [[angular-cli-plugin]] — Angular build pipeline the ported feature runs in
