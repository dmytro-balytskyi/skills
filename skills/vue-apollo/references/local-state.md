# Local State Management with @client Directive

Apollo Client can store local application state in its cache alongside server data. This eliminates the need for separate state management (Vuex/Pinia) when your local state is closely tied to GraphQL queries.

## When to Use Local State

- UI preferences (theme, sidebar collapsed state)
- Form state that feeds into GraphQL mutations
- Feature flags controlled by client-side logic
- Temporary data not worth a server round-trip

> ⚠️ **For complex global state**, use Pinia or Vuex. Apollo local state is best for small, query-driven pieces of state.

## Basic Setup: Local Schema + Resolvers

```typescript
// src/plugins/apollo.ts
import { ApolloClient, InMemoryCache } from '@apollo/client/core'
import gql from 'graphql-tag'

const typeDefs = gql`
  extend type Query {
    isLoggedIn: Boolean!
    currentUser: User
  }

  extend type Mutation {
    login(email: String!, password: String!): Token!
    logout: Boolean!
  }
`

const resolvers = {
  Query: {
    isLoggedIn: () => !!localStorage.getItem('auth_token'),
    currentUser: () => {
      const token = localStorage.getItem('auth_token')
      return token ? JSON.parse(localStorage.getItem('user') || '{}') : null
    },
  },
  Mutation: {
    login: (_, { email, password }) => {
      // Call your auth API
      const token = 'jwt-token-here'
      localStorage.setItem('auth_token', token)
      return { __typename: 'Token', token }
    },
    logout: () => {
      localStorage.removeItem('auth_token')
      localStorage.removeItem('user')
      return true
    },
  },
}

export const apolloClient = new ApolloClient({
  link: httpLink, // your HTTP link from setup.md
  cache: new InMemoryCache(),
  typeDefs,       // ← Required for local state
  resolvers,      // ← Required for local mutations
})
```

## Querying Local State with @client

Use the `@client` directive to mark fields that should be resolved locally (not sent to server):

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useQuery } from '@vue/apollo-composable'
import gql from 'graphql-tag'

const AUTH_STATE = gql`
  query AuthState {
    isLoggedIn @client
    currentUser @client {
      id
      name
      email
    }
  }
`

const { result, loading } = useQuery(AUTH_STATE)

const isLoggedIn = computed(() => result.value?.isLoggedIn ?? false)
const user = computed(() => result.value?.currentUser ?? null)
</script>

<template>
  <div v-if="loading">Loading...</div>
  <nav v-else>
    <template v-if="isLoggedIn && user">
      Welcome, {{ user.name }}!
      <button @click="logout">Logout</button>
    </template>
    <template v-else>
      <button @click="showLogin = true">Login</button>
    </template>
  </nav>
</template>
```

## Local Mutations with Resolvers

Local mutations update the cache directly without server calls:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useMutation, useQuery } from '@vue/apollo-composable'
import gql from 'graphql-tag'

// Local schema type (must match typeDefs in apollo.ts)
interface TodoItem {
  __typename: string
  id: string
  text: string
  done: boolean
}

const todoItemsQuery = gql`
  query GetTodoItems @client {
    todoItems @client {
      id
      text
      done
    }
  }
`

// Initialize cache with default data
import { apolloClient } from '@/plugins/apollo'
apolloClient.cache.writeData({
  data: {
    todoItems: [
      { __typename: 'TodoItem', id: '1', text: 'Learn Apollo local state', done: true },
      { __typename: 'TodoItem', id: '2', text: 'Build something awesome', done: false },
    ],
  },
})

const { result } = useQuery(todoItemsQuery)
const todos = computed(() => result.value?.todoItems ?? [])

const newTodoText = ref('')

const toggleTodo = useMutation(gql`
  mutation ToggleTodo($id: ID!) @client {
    toggleTodo(id: $id) @client
  }
`, {
  update: (cache, { data }) => {
    const items = cache.readQuery({ query: todoItemsQuery })?.todoItems ?? []
    const updated = items.map((item: TodoItem) =>
      item.id === data?.toggleTodo.id ? { ...item, done: !item.done } : item
    )
    cache.writeQuery({ query: todoItemsQuery, data: { todoItems: updated } })
  },
})

const addTodo = useMutation(gql`
  mutation AddTodo($text: String!) @client {
    addTodo(text: $text) @client
  }
`, {
  update: (cache, { data }) => {
    const items = cache.readQuery({ query: todoItemsQuery })?.todoItems ?? []
    cache.writeQuery({
      query: todoItemsQuery,
      data: { todoItems: [...items, data!.addTodo] },
    })
  },
})

function handleAdd() {
  if (!newTodoText.value.trim()) return
  addTodo.mutate({ text: newTodoText.value })
  newTodoText.value = ''
}
</script>

<template>
  <div class="todo-app">
    <ul>
      <li v-for="todo in todos" :key="todo.id">
        <input type="checkbox" :checked="todo.done" @change="toggleTodo.mutate({ id: todo.id })" />
        <span :class="{ done: todo.done }">{{ todo.text }}</span>
      </li>
    </ul>

    <form @submit.prevent="handleAdd">
      <input v-model="newTodoText" placeholder="New todo..." />
      <button type="submit">Add</button>
    </form>
  </div>
</template>

<style scoped>
.done { text-decoration: line-through; opacity: 0.6; }
</style>
```

## Extending Remote Types with Local Fields

Add client-only fields to server types using `extend type`:

```typescript
// src/plugins/apollo.ts — extend existing remote schema
const typeDefs = gql`
  # Add local field to remote User type
  extend type User {
    isOnline: Boolean!
    lastSeenAt: String
  }
`

const resolvers = {
  User: {
    isOnline: (parent: Record<string, unknown>) => {
      // Check against your online users list
      return false
    },
    lastSeenAt: (parent: Record<string, unknown>) => {
      return null
    },
  },
}
```

Then query it alongside server fields:

```typescript
const USER_QUERY = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name           # from server
      email          # from server
      isOnline       # @client — local resolver
      lastSeenAt     # @client — local resolver
    }
  }
`

const { result } = useQuery(USER_QUERY, () => ({ id: userId.value }))
```

## Local State vs Pinia/Vuex Decision Matrix

| Scenario | Recommended | Reason |
|----------|-------------|--------|
| UI state (theme, sidebar) | **Pinia** | Simple, reactive, no GraphQL coupling |
| Auth tokens + user profile | **Apollo local** | Tied to GraphQL queries/mutations |
| Form draft state | **Pinia** | Complex validation, not query-driven |
| Feature flags from config | **Apollo local** | Can be queried alongside data |
| Real-time counters | **Apollo local** | Updated via subscriptions + cache |
| Cart / checkout flow | **Pinia** | Complex business logic, not GraphQL-centric |

## Key Points

- `@client` directive marks fields as local-only (not sent to server)
- `typeDefs` and `resolvers` are required in Apollo Client config for local state
- Local mutations use `update` function to modify cache directly
- Use `cache.writeData()` for initial data, `cache.readQuery()`/`writeQuery()` for updates
- For complex global state, prefer Pinia over Apollo local state
