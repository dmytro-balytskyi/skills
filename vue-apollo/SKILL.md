---
name: vue-apollo
description: Guidelines for developing Vue 3 applications with GraphQL using @vue/apollo-composable (v4.x) and Apollo Client v3. Use when working with GraphQL queries, mutations, subscriptions, caching, pagination, or type-safe data fetching in Vue 3 + TypeScript projects. Covers useQuery, useMutation, useSubscription, useLazyQuery, cache management, error handling, and integration with GraphQL Codegen for full type safety.
metadata:
  type: utility
  mode: generative
---

# Vue Apollo — Best Practices

You are an expert in integrating GraphQL into Vue 3 applications using `@vue/apollo-composable` (v4.x) and `@apollo/client` (v3.x). This skill covers all aspects of working with GraphQL: queries, mutations, subscriptions, caching, pagination, error handling, and type safety via GraphQL Codegen.

## Core Principles

### 1. Use Composition API + TypeScript
Always use `<script setup lang="ts">`. Never use Options API for Apollo integration in new code.

### 2. Typed Documents with Codegen
Use `.gql` files with `typed-document-node` preset — never inline `gql` template literals without generated types. This gives full type safety for queries, variables, and results at compile time.

```bash
# Install dependencies
npm install graphql @apollo/client @vue/apollo-composable
npm i -D @graphql-codegen/cli @graphql-codegen/client-preset typed-document-node
```

### 3. Composables over Components
Wrap Apollo operations in composables (`useUser.ts`, `usePosts.ts`) — never call `useQuery` directly in templates. Expose only the data and state your component needs.

### 4. Use `computed()` Instead of `useResult()`
The `useResult()` utility is **deprecated**. Always use Vue's native `computed()`:

```typescript
// ❌ Deprecated
const users = useResult(result, [], (data) => data.users)

// ✅ Correct
const users = computed(() => result.value?.users ?? [])
```

### 5. Variables as Functions for Reactivity
When variables depend on refs or props, always pass a function:

```typescript
// ✅ Reactive — re-fetches when userId changes
useQuery(QUERY, () => ({ id: userId.value }))

// ❌ Static — never updates
useQuery(QUERY, { id: userId.value })
```

## Quick Reference

### useQuery Return Values

| Property | Type | Description |
|----------|------|-------------|
| `result` | `Ref<QueryResult>` | Data (undefined during loading) |
| `loading` | `Ref<boolean>` | Request in-flight state |
| `error` | `Ref<ApolloError \| null>` | Error object or null |
| `variables` | `Ref<Variables>` | Current variables ref |
| `refetch()` | `Function` | Re-execute query, optionally with new vars |
| `fetchMore()` | `Function` | Pagination support |
| `subscribeToMore()` | `Function` | Real-time subscription integration |
| `onResult(cb)` | `Function` | Hook on every result |
| `onError(cb)` | `Function` | Hook on error |

### useMutation Return Values

| Property | Type | Description |
|----------|------|-------------|
| `mutate()` | `Function` | Execute mutation with variables |
| `loading` | `Ref<boolean>` | Mutation in-flight state |
| `error` | `Ref<ApolloError \| null>` | Error object or null |
| `called` | `Ref<boolean>` | Whether mutate was called |
| `onDone(cb)` | `Function` | Hook on successful completion |
| `onError(cb)` | `Function` | Hook on mutation error |

### useSubscription Return Values

| Property | Type | Description |
|----------|------|-------------|
| `result` | `Ref<SubscriptionResult>` | New data from subscription |
| `loading` | `Ref<boolean>` | Connection state |
| `error` | `Ref<ApolloError \| null>` | Error object or null |

### Fetch Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `cache-first` (default) | Return cache, fetch if miss | Most queries |
| `cache-and-network` | Return cache + update from network | Dashboard widgets |
| `network-only` | Always fetch, save to cache | Fresh data required |
| `no-cache` | Fetch only, never cache | Sensitive/one-time data |
| `cache-only` | Cache only, fail if miss | Check if data exists |

## Setup Guide

For complete Apollo Client setup (HTTP link, auth, error handling, type policies), see [references/setup.md](references/setup.md).

## GraphQL Codegen Configuration

```typescript
// codegen.ts
import type { CodegenConfig } from '@graphql-codegen/cli'

const config: CodegenConfig = {
  schema: import.meta.env.VITE_GRAPHQL_URL,
  documents: ['src/**/*.gql', 'src/**/*.vue'],
  generates: {
    './src/gql/': {
      preset: 'client',
      config: {
        useTypeImports: true,
        documentMode: 'documentNode',
      },
    },
  },
}

export default config
```

Run with `npx graphql-codegen --watch`. Generated types go to `./src/gql/`.

## Type-Safe Query Usage

```typescript
import { computed } from 'vue'
import { useQuery } from '@vue/apollo-composable'
import GetUserDocument from '@/graphql/queries/GetUser.gql'
import type { GetUserQuery, GetUserQueryVariables } from '@/gql/graphql'

const { result, loading, error } = useQuery<GetUserQuery, GetUserQueryVariables>(
  GetUserDocument,
  () => ({ id: userId.value }),
)

const user = computed(() => result.value?.user ?? null)
```

## Error Handling Strategy

- **Component level**: Use `errorPolicy: 'all'` to receive both data and errors
- **Global level**: Configure `onError` link in Apollo Client setup (see [references/setup.md](references/setup.md))
- **Mutation errors**: Handle via `onError` hook or try/catch around `mutate()`

## Anti-Patterns

### The useResult Trap
**Problem:** Using deprecated `useResult()` utility instead of native `computed()`.
**Fix:** Replace with `computed(() => result.value?.field ?? defaultValue)`.

### The Static Variables Bug
**Problem:** Passing static object for variables that depend on reactive state.
```typescript
// ❌ Never updates when userId changes
useQuery(QUERY, { id: userId.value })
```
**Fix:** Use function syntax: `useQuery(QUERY, () => ({ id: userId.value }))`.

### The Template Query
**Problem:** Calling `useQuery` directly in component template or without composable abstraction.
**Fix:** Extract to a composable (`useUser.ts`) and expose only needed data/state.

### The Missing Loading State
**Problem:** Accessing `result.value?.field` without checking `loading`.
**Fix:** Always show loading skeleton/spinner before rendering data.

### The Inline gql Literal
**Problem:** Using inline ``gql`...`` template literals without Codegen types.
```typescript
// ❌ No type safety
const { result } = useQuery(gql`query GetUser { user { id name } }`)
```
**Fix:** Use `.gql` files with `typed-document-node` for full compile-time type checking.

## Integration Graph

### Inbound (From Other Skills)
| Source Skill | Context | Leads to State |
|--------------|---------|----------------|
| vue | Vue 3 project setup | Apollo Client initialization |
| vite | Vite configuration | Codegen integration with Vite |

### Outbound (To Other Skills)
| This Skill | Leads to Skill | Purpose |
|------------|----------------|---------|
| GraphQL queries | apollo-graphql | Server-side schema design |
| TypeScript types | create-adaptable-composable | Building composable wrappers |

## When to Read References

- **Setup & configuration** → [references/setup.md](references/setup.md) — Apollo Client, links, type policies
- **Queries & variables** → [references/queries.md](references/queries.md) — useQuery, useLazyQuery, polling, refetch
- **Mutations & cache updates** → [references/mutations.md](references/mutations.md) — useMutation, optimistic UI, cache.modify
- **Subscriptions & pagination** → [references/subscriptions-pagination.md](references/subscriptions-pagination.md) — useSubscription, fetchMore
