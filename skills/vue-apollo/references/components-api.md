# Components API — Declarative GraphQL in Templates

Vue Apollo provides declarative components (`ApolloQuery`, `ApolloMutation`) that let you write GraphQL directly in templates without `<script setup>`. Useful for simple queries where a composable would be overkill.

## ApolloQuery Component

The simplest way to run a query is with the `ApolloQuery` component:

```vue
<template>
  <ApolloQuery :query="gql => gql`
    query GetUser($id: ID!) {
      user(id: $id) {
        id
        name
        email
      }
    }
  `" :variables="{ id: userId }">
    <template v-slot="{ result: { loading, error, data } }">
      <!-- Loading -->
      <div v-if="loading" class="loading">Loading...</div>

      <!-- Error -->
      <div v-else-if="error" class="error">{{ error.message }}</div>

      <!-- Result -->
      <div v-else-if="data?.user">
        {{ data.user.name }} — {{ data.user.email }}
      </div>

      <!-- No result -->
      <div v-else>No user found</div>
    </template>
  </ApolloQuery>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps<{ userId: string }>()
const userId = computed(() => props.userId)
</script>
```

### Using .gql Files with ApolloQuery

For better organization, use separate `.gql` files:

```vue
<template>
  <ApolloQuery :query="require('@/graphql/queries/GetUser.gql')" :variables="{ id: userId }">
    <template v-slot="{ result: { loading, error, data } }">
      <!-- ... same as above ... -->
    </template>
  </ApolloQuery>
</template>

<script setup lang="ts">
import { ref } from 'vue'
const props = defineProps<{ userId: string }>()
const userId = computed(() => props.userId)
</script>
```

### Accessing the Query Object (for fetchMore, refetch)

The `query` slot prop gives you access to smart query methods:

```vue
<template>
  <ApolloQuery :query="gql => gql`
    query GetFeed($offset: Int, $limit: Int) {
      feed(offset: $offset, limit: $limit) {
        id
        title
      }
    }
  `" :variables="{ offset: page * pageSize, limit: pageSize }" v-slot="{ result: { loading, error, data }, query }">

    <ul>
      <li v-for="item of data?.feed ?? []" :key="item.id">{{ item.title }}</li>
    </ul>

    <button @click="loadMore(query)">Load More</button>
  </ApolloQuery>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const page = ref(0)
const pageSize = 10

function loadMore(query: any) {
  query.fetchMore({
    variables: { offset: (page.value + 1) * pageSize },
    updateQuery: (prev, { fetchMoreResult }) => {
      if (!fetchMoreResult?.feed.length) return prev
      return { ...prev, feed: [...prev.feed, ...fetchMoreResult.feed] }
    },
  }).then(() => page.value++)
}
</script>
```

### Using Fragments with ApolloQuery

Fragments help share GraphQL field selections across queries and mutations:

```vue
<template>
  <ApolloQuery :query="gql => gql`
    query GetMessages {
      messages {
        ...messageFields
      }
    }
    ${$options.fragments.message}
  `">
    <!-- ... -->
  </ApolloQuery>
</template>

<script setup lang="ts">
import gql from 'graphql-tag'

defineOptions({
  fragments: {
    message: gql`
      fragment messageFields on Message {
        id
        text
        created
        user {
          id
          name
        }
      }
    `,
  },
})
</script>
```

### Reusing Fragments Across Components

Import fragments from another component:

```vue
<template>
  <ApolloQuery :query="gql => gql`
    mutation SendMessage($input: SendMessageInput!) {
      sendMessage(input: $input) {
        newMessage {
          ...messageFields
        }
      }
    }
    ${MessageList.fragments.message}
  `">
    <!-- ... -->
  </ApolloQuery>
</template>

<script setup lang="ts">
import gql from 'graphql-tag'
import MessageList from './MessageList.vue'
</script>
```

## ApolloMutation Component

Execute mutations declaratively in templates:

```vue
<template>
  <ApolloMutation :mutation="gql => gql`
    mutation UpdateUser($id: ID!, $input: UserInput!) {
      updateUser(id: $id, input: $input) {
        id
        name
        email
      }
    }
  `" v-slot="{ mutate, loading, error }">

    <form @submit.prevent="updateUser">
      <input v-model="name" placeholder="Name" />
      <button :disabled="loading">{{ loading ? 'Saving...' : 'Save' }}</button>
    </form>

    <div v-if="error" class="error">{{ error.message }}</div>
  </ApolloMutation>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps<{ userId: string }>()
const name = ref('')

function updateUser() {
  mutate({
    variables: { id: props.userId, input: { name: name.value } },
  })
}
</script>
```

## When to Use Components vs Composables

| Approach | Best For | Pros | Cons |
|----------|----------|------|------|
| **ApolloQuery/ApolloMutation** | Simple queries in templates | Declarative, no script needed | No TypeScript types, harder to test |
| **Composables (useQuery/useMutation)** | Complex logic, reusable patterns | Full TS support, testable, composable | More code to write |

> 💡 **Recommendation:** Use composables for production apps. Components are great for prototyping or simple read-only views.
