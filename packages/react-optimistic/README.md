# @TanStack/react-optimistic

React hooks and utilities for creating fast optimistic updates with flexible backend support that pairs seamlessly with sync engines (like [ElectricSQL](https://electric-sql.com/)).

## Installation

```bash
pnpm add @TanStack/react-optimistic
```

## Overview

`@TanStack/react-optimistic` provides React-specific hooks and utilities for managing data synchronization between your frontend application and backend services. It offers:

- **Optimistic Updates**: Apply changes instantly in the UI while syncing in the background
- **Flexible Backend Support**: Works with any backend or sync engine
- **Immutable Snapshots**: Create immutable snapshots of updates that can be persisted and rolled back
- **React Integration**: Seamless integration with React components and state management

## React Hooks

### `useCollection`

The primary hook for interacting with collections in React components.

```typescript
const { data, insert, update, delete: deleteFn } = useCollection({
  id: 'todos',
  sync: { /* sync configuration */ },
  mutationFn: { /* mutation functions */ },
  schema: /* optional schema */
});
```

Returns:

- `data`: An array of all items in the collection
- `state`: A Map containing all items in the collection with their internal keys
- `insert`: Function to add new items to the collection
- `update`: Function to modify existing items
- `delete`: Function to remove items from the collection

### `preloadCollection`

Preloads data for a collection before rendering components.

```typescript
await preloadCollection({
  id: 'todos',
  sync: { /* sync configuration */ },
  mutationFn: { /* mutation functions */ },
  schema: /* optional schema */
});
```

Features:

1. Returns a promise that resolves when the first sync commit is complete
2. Shares the same collection instance with `useCollection`
3. Handles already-loaded collections by returning immediately
4. Avoids duplicate initialization when called multiple times with the same ID

## Data Operations

### Insert

```typescript
// Insert a single item
insert({ text: "Buy groceries", completed: false })

// Insert multiple items
insert([
  { text: "Buy groceries", completed: false },
  { text: "Walk dog", completed: false },
])

// Insert with custom key
insert({ text: "Buy groceries" }, { key: "grocery-task" })
```

### Update

We use a proxy to capture updates as immutable draft optimistic updates.

```typescript
// Update a single item
update(todo, (draft) => {
  draft.completed = true
})

// Update multiple items
update([todo1, todo2], (drafts) => {
  drafts.forEach((draft) => {
    draft.completed = true
  })
})

// Update with metadata
update(todo, { metadata: { reason: "user update" } }, (draft) => {
  draft.text = "Updated text"
})
```

### Delete

```typescript
// Delete a single item
delete todo

// Delete multiple items
delete [todo1, todo2]

// Delete with metadata
delete (todo, { metadata: { reason: "completed" } })
```

## Implementing Backend Integration with ElectricSQL

The `mutationFn` property is where you define how your application interacts with your backend. Here's a comprehensive example of integrating with ElectricSQL:

```typescript
import { useCollection } from "@TanStack/react-optimistic"
import { createElectricSync } from "@TanStack/optimistic/electric"

// Create a collection configuration for todos
const todosConfig = {
  id: "todos",
  // Create an ElectricSQL sync configuration
  sync: createElectricSync(
    {
      // ShapeStream options
      url: `http://localhost:3000/v1/shape`,
      params: {
        table: "todos",
      },
    },
    {
      // Electric client instance
      client: electric,
    }
  ),
  // Define mutation functions for backend persistence
  mutationFn: {
    insert: async (items) => {
      // Insert items into Electric database
      const db = electric.db
      await db.todos.createMany({
        data: items,
      })
    },
    update: async (items) => {
      // Update items in Electric database
      const db = electric.db
      for (const item of items) {
        await db.todos.update({
          where: { id: item.id },
          data: item,
        })
      }
    },
    delete: async (items) => {
      // Delete items from Electric database
      const db = electric.db
      await db.todos.deleteMany({
        where: {
          id: {
            in: items.map((item) => item.id),
          },
        },
      })
    },
  },
}

// Use the collection in a component
function TodoList() {
  const { data, insert, update, delete: deleteFn } = useCollection(todosConfig)

  // Now you can use these functions to interact with your data
  // with automatic optimistic updates and backend persistence
}
```
