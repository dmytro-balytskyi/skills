# Mutations — useMutation & Cache Updates

## Basic Mutation Pattern

```typescript
// src/composables/useUpdateUser.ts
import { useMutation } from '@vue/apollo-composable'
import UpdateUserDocument from '@/graphql/mutations/UpdateUser.gql'
import type { UpdateUserMutation, UpdateUserMutationVariables } from '@/gql/graphql'

export function useUpdateUser() {
  const { mutate, loading, error } = useMutation<
    UpdateUserMutation,
    UpdateUserMutationVariables
  >(UpdateUserDocument)

  return {
    updateUser: (variables: UpdateUserMutationVariables) => mutate(variables),
    loading,
    error,
  }
}
```

## Component Usage with Error Handling

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useUpdateUser } from '@/composables/useUpdateUser'

const name = ref('')
const { updateUser, loading, error } = useUpdateUser()

async function handleSubmit() {
  try {
    await updateUser({ id: '123', input: { name: name.value } })
    // Success handled via onDone hook or toast
  } catch (err) {
    console.error('Failed to update user:', err)
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <input v-model="name" placeholder="Name" />
    <button :disabled="loading">{{ loading ? 'Saving...' : 'Save' }}</button>
    <div v-if="error" class="error">{{ error.message }}</div>
  </form>
</template>
```

## Dynamic Variables with Function Syntax

Variables set statically in options are evaluated only once. For dynamic values, use function syntax:

```typescript
const text = ref('')

// ❌ Static — text.value is captured at definition time (empty string)
useMutation(MUTATION, { variables: { text: text.value } })

// ✅ Function — re-evaluated on each mutate() call
useMutation(MUTATION, () => ({ variables: { text: text.value } }))
```

## Cache Update After Mutation

### Single Entity Auto-Update
If the mutation returns an entity with `id` and `__typename`, Apollo updates it automatically:

```typescript
const { mutate: editMessage } = useMutation(EDIT_MESSAGE, () => ({
  variables: { id: 'abc', text: 'Updated message' },
}))
// If messages list is cached, the item with id='abc' updates automatically
```

### Manual Cache Update (multiple entities or new/deleted items)

```typescript
import { useMutation, useQuery } from '@vue/apollo-composable'
import SendMessageDocument from '@/graphql/mutations/SendMessage.gql'
import MessagesListDocument from '@/graphql/queries/MessagesList.gql'

const { result: messagesResult } = useQuery(MessagesListDocument)
const messages = computed(() => messagesResult.value?.messages ?? [])

const { mutate: sendMessage } = useMutation(SendMessageDocument, () => ({
  variables: { text: newMessage.value },
  update: (cache, { data }) => {
    const newMsg = data?.sendMessage
    if (!newMsg) return

    let cachedData = cache.readQuery({ query: MessagesListDocument })
    if (!cachedData) return

    cache.writeQuery({
      query: MessagesListDocument,
      data: {
        ...cachedData,
        messages: [...cachedData.messages, newMsg],
      },
    })
  },
}))
```

### Using `cache.modify()` (preferred for single field updates)

```typescript
const { mutate: updateUser } = useMutation(UPDATE_USER, () => ({
  update: (cache, { data }) => {
    const updatedUser = data?.updateUser
    if (!updatedUser) return

    cache.modify({
      id: cache.identify(updatedUser), // Uses __typename + id
      fields: {
        name() {
          return updatedUser.name
        },
      },
    })
  },
}))
```

## Optimistic UI Updates

Show results immediately before server confirms:

```typescript
const { mutate: addUser } = useMutation(CREATE_USER, () => ({
  optimisticResponse: {
    __typename: 'Mutation',
    createUser: {
      __typename: 'User',
      id: `temp-${Date.now()}`,
      name: variables.input.name,
      email: variables.input.email,
    },
  },
  update: (cache, { data }) => {
    const newUser = data?.createUser
    if (!newUser) return

    cache.modify({
      id: cache.identify(newUser),
      fields: {
        name() {
          return newUser.name
        },
      },
    })
  },
}))
```

## Mutation Event Hooks

```typescript
const { mutate, onDone, onError } = useMutation(MUTATION)

onDone(({ data, context }) => {
  console.log('Mutation succeeded:', data)
  // Show success toast
})

onError((error, context) => {
  console.error('Mutation failed:', error.message)
  // Show error toast
})
```

## refetchQueries vs update

| Approach | When to Use | Pros | Cons |
|----------|-------------|------|------|
| `refetchQueries` | Simple list updates after mutation | Easy, declarative | Extra network request |
| `update` (cache.modify) | Precise cache manipulation | No extra requests, instant UI update | More code to write |

## Complete Example: Create + List Update

```typescript
import { useMutation, useQuery } from '@vue/apollo-composable'
import CreateUserDocument from '@/graphql/mutations/CreateUser.gql'
import UsersListDocument from '@/graphql/queries/UsersList.gql'

const { result: usersResult } = useQuery(UsersListDocument)
const users = computed(() => usersResult.value?.users ?? [])

const { mutate: createUser, onDone, loading } = useMutation(CreateUserDocument, () => ({
  update: (cache, { data }) => {
    const newUser = data?.createUser
    if (!newUser) return

    let cachedData = cache.readQuery({ query: UsersListDocument })
    if (!cachedData) return

    cache.writeQuery({
      query: UsersListDocument,
      data: { users: [...cachedData.users, newUser] },
    })
  },
}))

onDone(() => {
  // Reset form after successful creation
})
```
