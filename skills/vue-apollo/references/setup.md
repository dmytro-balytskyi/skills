# Apollo Client Setup & Configuration

## Project Structure

```
src/
├── graphql/                          # GraphQL documents (.gql)
│   ├── queries/                      # Queries
│   │   └── GetUser.gql
│   ├── mutations/                    # Mutations
│   │   └── UpdateUser.gql
│   ├── subscriptions/                # Subscriptions
│   │   └── UserUpdated.gql
│   └── fragments/                    # Shared fragments
│       └── UserInfo.gql
├── composables/                      # Vue composables with Apollo
│   ├── useUser.ts
│   └── usePosts.ts
├── plugins/
│   └── apollo.ts                     # Apollo Client configuration
├── utils/
│   └── apollo-links.ts               # Additional Apollo links
├── types/
│   └── graphql.ts                    # .gql file type declarations
└── App.vue                           # Root with provide()
```

## Apollo Client Configuration (`src/plugins/apollo.ts`)

```typescript
import { ApolloClient, InMemoryCache, createHttpLink } from '@apollo/client/core'
import { onError } from '@apollo/client/link/error'
import { setContext } from '@apollo/client/link/context'
import type { ApolloLink } from '@apollo/client/core'

// HTTP link to GraphQL endpoint
const httpLink = createHttpLink({
  uri: import.meta.env.VITE_GRAPHQL_URL || 'http://localhost:4000/graphql',
})

// Auth link — attach token to requests
const authLink: ApolloLink = setContext((_, { headers }) => {
  const token = localStorage.getItem('auth_token')
  return {
    headers: {
      ...headers,
      ...(token ? { authorization: `Bearer ${token}` } : {}),
    },
  }
})

// Error link — global error logging (dev only)
const errorLink = onError(({ graphQLErrors, networkError }) => {
  if (import.meta.env.DEV && graphQLErrors) {
    graphQLErrors.forEach(({ message, locations, path }) =>
      console.error(`[GraphQL error]: Message: ${message}, Path: ${path.join('.')}`, { locations }),
    )
  }
  if (networkError) {
    console.error(`[Network error]: ${networkError}`)
  }
})

// Create Apollo Client instance
export const apolloClient = new ApolloClient({
  link: from([errorLink, authLink.concat(httpLink)]),
  cache: new InMemoryCache({
    typePolicies: {
      User: {
        keyFields: ['id'],
      },
      Query: {
        fields: {
          users: {
            keyArgs: ['filter', 'limit'],
            merge(existing = [], incoming) {
              return [...existing, ...incoming]
            },
          },
        },
      },
    },
  }),
  defaultOptions: {
    query: { fetchPolicy: 'cache-first', errorPolicy: 'all' },
    mutate: { errorPolicy: 'all' },
    watchQuery: { fetchPolicy: 'cache-and-network', errorPolicy: 'all' },
  },
})
```

## Register in App (`src/App.vue`)

```vue
<script setup lang="ts">
import { DefaultApolloClient } from '@vue/apollo-composable'
import { provide } from 'vue'
import { apolloClient } from '@/plugins/apollo'

provide(DefaultApolloClient, apolloClient)
</script>

<template>
  <router-view />
</template>
```

## Type Declarations for .gql Files (`src/types/graphql.ts`)

```typescript
import type { DocumentNode } from 'graphql'

declare module '*.gql' {
  import type { TypedDocumentNode } from '@apollo/client/core'
  const value: TypedDocumentNode
  export default value
}
```

## Multiple Apollo Clients

For projects needing multiple GraphQL endpoints (e.g., main API + upload service):

```typescript
import { ApolloClients } from '@vue/apollo-composable'
import { provide } from 'vue'

provide(ApolloClients, {
  default: apolloClient,       // Main GraphQL API
  uploads: uploadApolloClient, // File upload endpoint
})
```

Then specify `clientId` in composables:

```typescript
useQuery(QUERY, null, { clientId: 'uploads' })
```

## Usage Outside Vue Context

When using Apollo composables outside of setup (e.g., in utility functions):

```typescript
import { provideApolloClient } from '@vue/apollo-composable'

const query = provideApolloClient(apolloClient)(() => useQuery(gql`query hello { hello }`))
const hello = computed(() => query.result.value?.hello ?? '')
```

For multiple clients:

```typescript
import { provideApolloClients } from '@vue/apollo-composable'

provideApolloClients({
  default: apolloClient,
  clientA: apolloClientA,
})
```

## WebSocket Setup for Subscriptions

Subscriptions require a WebSocket link in addition to HTTP. Apollo Client supports two protocols:
- **graphql-ws** (recommended, actively maintained)
- **subscriptions-transport-ws** (legacy, deprecated)

### graphql-ws (Recommended)

```bash
npm install graphql-ws
```

```typescript
import { ApolloClient, InMemoryCache, HttpLink, split } from '@apollo/client/core'
import { GraphQLWsLink } from '@apollo/client/link/subscriptions'
import { createClient } from 'graphql-ws'
import { getMainDefinition } from '@apollo/client/utilities'

// HTTP link for queries and mutations
const httpLink = new HttpLink({
  uri: import.meta.env.VITE_GRAPHQL_URL || 'http://localhost:4000/graphql',
})

// WebSocket link for subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: import.meta.env.VITE_WS_URL || 'ws://localhost:4000/graphql',
    // Auth over WebSocket (see below)
    connectionParams: () => {
      const token = localStorage.getItem('auth_token')
      return token ? { authorization: `Bearer ${token}` } : {}
    },
  })
)

// Split links by operation type
const link = split(
  ({ query }) => {
    const definition = getMainDefinition(query)
    return (
      definition.kind === 'OperationDefinition'
      && definition.operation === 'subscription'
    )
  },
  wsLink,
  httpLink
)

export const apolloClient = new ApolloClient({
  link,
  cache: new InMemoryCache(),
})
```

### subscriptions-transport-ws (Legacy)

```bash
npm install subscriptions-transport-ws
```

```typescript
import { WebSocketLink } from '@apollo/client/link/ws'

const wsLink = new WebSocketLink({
  uri: 'ws://localhost:4000/graphql',
  options: {
    reconnect: true,
    connectionParams: {
      authToken: localStorage.getItem('auth_token'),
    },
  },
})
```

> ⚠️ **Note:** `subscriptions-transport-ws` is no longer actively maintained. Use `graphql-ws` for new projects.
