# Queries — useQuery, useLazyQuery

## Basic Query Pattern

```typescript
// src/composables/useUser.ts
import { computed, type MaybeRefOrGetter } from 'vue'
import { useQuery } from '@vue/apollo-composable'
import GetUserDocument from '@/graphql/queries/GetUser.gql'
import type { GetUserQuery, GetUserQueryVariables } from '@/gql/graphql'

export function useUser(userId: MaybeRefOrGetter<string>) {
  const { result, loading, error } = useQuery<GetUserQuery, GetUserQueryVariables>(
    GetUserDocument,
    () => ({ id: toValue(userId) }),
  )

  const user = computed(() => result.value?.user ?? null)

  return { user, loading, error }
}
```

## Variables Patterns

### Function (recommended for reactivity)
```typescript
const { result } = useQuery(QUERY, () => ({ id: userId.value }))
// Re-fetches automatically when userId changes
```

### Ref
```typescript
const variables = ref({ id: 'abc' })
const { result } = useQuery(QUERY, variables)
variables.value = { id: 'new-id' } // Triggers re-fetch
```

### Reactive object
```typescript
const variables = reactive({ id: 'abc' })
const { result } = useQuery(QUERY, variables)
variables.id = 'new-id' // Triggers re-fetch
```

### Props (only if props match GraphQL variables exactly)
```typescript
const props = defineProps<{ id: string }>()
const { result } = useQuery(QUERY, props)
// ⚠️ Warning: extra props not in GraphQL schema cause validation errors
```

## Query Options

| Option | Type | Description | Example |
|--------|------|-------------|---------|
| `enabled` | `Ref<boolean>` | Enable/disable query execution | `{ enabled: !!userId }` |
| `fetchPolicy` | string | Cache/network behavior | `'cache-and-network'` |
| `errorPolicy` | string | Error handling strategy | `'all'`, `'none'`, `'ignore'` |
| `pollInterval` | number | Auto-refetch interval (ms) | `{ pollInterval: 5000 }` |
| `notifyOnNetworkStatusChange` | boolean | Trigger result on network changes | `{ notifyOnNetworkStatusChange: true }` |
| `keepPreviousResult` | boolean | Keep old data while refetching | `{ keepPreviousResult: true }` |
| `debounce` / `throttle` | number | Delay/limit re-executions | `{ debounce: 300 }` |

## Disable a Query

```typescript
const shouldFetch = ref(false)

const { result } = useQuery(QUERY, null, () => ({
  enabled: shouldFetch.value,
}))

function enable() {
  shouldFetch.value = true // Triggers query execution
}
```

## Polling

```typescript
const { result } = useQuery(QUERY, null, {
  pollInterval: 5000, // Refetch every 5 seconds
})
```

## Refetch

```typescript
const { result, refetch } = useQuery(QUERY)

// Without new variables (uses original)
await refetch()

// With new variables
await refetch({ search: 'new query' })
```

## Event Hooks

```typescript
const { onResult, onError } = useQuery(QUERY)

onResult((queryResult) => {
  console.log('Data:', queryResult.data)
  console.log('Loading:', queryResult.loading)
  console.log('Network status:', queryResult.networkStatus)
})

onError((error) => {
  console.error('GraphQL errors:', error.graphQLErrors)
  console.error('Network error:', error.networkError)
})
```

## useLazyQuery — Deferred Queries

For queries that should only execute on user action:

```typescript
import { computed } from 'vue'
import { useLazyQuery } from '@vue/apollo-composable'
import SearchDocument from '@/graphql/queries/Search.gql'

const { result, load, loading, error } = useLazyQuery(SearchDocument)

const searchResults = computed(() => result.value?.search ?? [])

async function handleSearch(query: string) {
  const success = await load({ query })
  if (!success) {
    // First call returned false — already loaded before
    // Use refetch() to force re-fetch (if available in your version)
  }
}
```

## Component Usage Example

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useUser } from '@/composables/useUser'

const props = defineProps<{ userId: string }>()
const { user, loading, error } = useUser(props.userId)
const errorMessage = computed(() => error.value?.message ?? null)
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="errorMessage" class="error">{{ errorMessage }}</div>
  <div v-else-if="user">{{ user.name }}</div>
</template>
```

## Reading Query Result from Template Directly

```vue
<template>
  <ul v-if="result && result.users">
    <li v-for="user of result.users" :key="user.id">
      {{ user.firstname }} {{ user.lastname }}
    </li>
  </ul>
</template>
```

> ⚠️ `result` is initially `undefined`. Always guard access with `v-if` or optional chaining.
