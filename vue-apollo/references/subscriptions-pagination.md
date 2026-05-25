# Subscriptions & Pagination

## useSubscription — Real-Time Updates

### Basic Subscription

```typescript
import { watch } from 'vue'
import { useSubscription } from '@vue/apollo-composable'
import UserUpdatedDocument from '@/graphql/subscriptions/UserUpdated.gql'

const newUserData = ref<unknown>(null)

const { result, error } = useSubscription(UserUpdatedDocument)

watch(result, (data) => {
  if (data?.userUpdated) {
    newUserData.value = data.userUpdated
  }
})
```

### subscribeToMore — Merge Subscription into Query

Add real-time updates to an existing query result:

```typescript
import { useQuery } from '@vue/apollo-composable'
import UsersListDocument from '@/graphql/queries/UsersList.gql'
import UserUpdatedSubscription from '@/graphql/subscriptions/UserUpdated.gql'

const { result, subscribeToMore } = useQuery(UsersListDocument)
const users = computed(() => result.value?.users ?? [])

subscribeToMore({
  document: UserUpdatedSubscription,
  updateQuery: (previousResult, { subscriptionData }) => {
    if (!subscriptionData.data?.userUpdated) return previousResult

    const updatedUser = subscriptionData.data.userUpdated
    return {
      ...previousResult,
      users: (previousResult.users ?? []).map((u: Record<string, unknown>) =>
        u.id === updatedUser.id ? updatedUser : u,
      ),
    }
  },
})
```

## Pagination with fetchMore

### Offset-Based Pagination

```typescript
const PAGE_SIZE = 10
const page = ref(0)

const { result, loading, error, fetchMore } = useQuery(USERS_QUERY, () => ({
  limit: PAGE_SIZE,
  offset: page.value * PAGE_SIZE,
}))

const users = computed(() => result.value?.users ?? [])

async function loadNextPage() {
  await fetchMore({
    variables: {
      offset: (page.value + 1) * PAGE_SIZE,
    },
    updateQuery: (previousResult, { fetchMoreResult }) => {
      if (!fetchMoreResult?.users) return previousResult

      return {
        ...previousResult,
        users: [...previousResult.users, ...fetchMoreResult.users],
      }
    },
  })
  page.value++
}
```

### Cursor-Based Pagination (Relay Style)

```typescript
const { result, loading, fetchMore } = useQuery(USERS_QUERY)

async function loadNextPage() {
  await fetchMore({
    variables: {
      cursor: result.value?.users_pageInfo.endCursor ?? null,
    },
    updateQuery: (previousResult, { fetchMoreResult }) => {
      if (!fetchMoreResult?.users_edges) return previousResult

      const existingIds = new Set(
        previousResult.users_edges.map((e: Record<string, unknown>) => e.node.id),
      )
      const newEdges = fetchMoreResult.users_edges.filter(
        (e: Record<string, unknown>) => !existingIds.has(e.node.id),
      )

      return {
        ...previousResult,
        users_edges: [...previousResult.users_edges, ...newEdges],
      }
    },
  })
}
```

## Cache Management

### Read from Cache Directly

```typescript
import { useApolloClient } from '@vue/apollo-composable'

const client = useApolloClient()

// Query with cache-only policy
const cachedUser = await client.query({
  query: GetUserDocument,
  fetchPolicy: 'cache-only',
})
```

### Write to Cache

```typescript
client.writeQuery({
  query: GetUserDocument,
  data: { user: { __typename: 'User', id: '123', name: 'Updated Name' } },
})
```

### Modify Specific Fields (cache.modify)

```typescript
client.cache.modify({
  id: client.cache.identify({ __typename: 'User', id: '123' }),
  fields: {
    name() {
      return 'Updated Name'
    },
    email(newValue, { readField }) {
      const existingEmail = readField('email')
      return newValue !== existingEmail ? newValue : existingEmail
    },
  },
})
```

### Invalidate Cache

```typescript
// Evict specific entity
client.cache.evict({ id: client.cache.identify({ __typename: 'User', id: '123' }) })
client.cache.gc() // Garbage collection

// Or refetch all queries
await client.refetchQueries({ include: 'all' })
```
