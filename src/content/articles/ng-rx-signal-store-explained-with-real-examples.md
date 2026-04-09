---
title: "NgRx Signal Store Explained with Real Examples"
summary: "NgRx Signal Store gives Angular developers a practical middle path — more structured than ad hoc component state, lighter than a full reducer-heavy setup, and closely aligned with Angular's signal-based direction."
publishedAt: 2026-03-15
---

NgRx introduces Signals as a standalone library. It provides a clean way to define and manage state in an Angular application.

In this article, I will build a small Signal Store and show how to derive values from it, update it, and compose more advanced behavior as a feature grows.

## Why Signal Store exists

In Angular, we have two main ways to manage state in our applications:

- Using services with RxJS
- Using NgRx Store

Each approach can work depending on the situation, but there is a significant gap between the convenience and flexibility of managing state in services and the fully structured approach enforced by NgRx. Managing state in services and components is fast and easy at first, but as application complexity grows, it often becomes messy and difficult to maintain.

More formal state management enforces structure in an application, but it usually comes with a large amount of boilerplate.

This is where Signal Store shines: it gives you structure without forcing a large architecture.

## Basics

Signal operates around these key concepts:

- **state** — the source of truth that holds raw data, such as a list of users, filters, loading flags, and error messages. This data can change over time.
- **computed values** — a simple way to read and derive values from the store.
- **methods** — the way you update the store and perform side effects, such as HTTP requests.

## Building a first store

All code used in this article is available on my GitHub:
https://github.com/janskiba/signal-store-example

Let's build a simple feature: a user list with filtering and role-based data. This feature needs a store to keep user data, fetch user information, and provide a way to filter the results.

Here is a simple example:

```typescript
export const UsersStore = signalStore(
  withState(initialState),
  withComputed((store) => ({
    filteredUsers: computed(() => {
      const query = store.query().toLowerCase().trim();
      const selectedRole = store.selectedRole();

      return store.users().filter((user) => {
        const matchesQuery = user.name.toLowerCase().includes(query);
        const matchesRole = selectedRole ? user.role === selectedRole : true;
        return matchesQuery && matchesRole;
      });
    }),
    filteredCount: computed(() => store.filteredUsers().length),
  })),
  withMethods((store) => ({
    setQuery(query: string) {
      patchState(store, { query });
    },
    setRole(role: UserRole | null) {
      patchState(store, { selectedRole: role });
    },
    resetFilters() {
      patchState(store, {
        query: "",
        selectedRole: null,
      });
    },
  })),
);
```

- `withState` defines source state
- `withComputed` retrieves values from the store
- `withMethods` is like an API to interact with our state

As we can see everything is defined in one place. No reducer files, no action types, and no selectors in separate files.

In the component we can use our state like this:

```typescript
import { Component, inject } from "@angular/core";
import { UsersStore } from "../users.store";
import { UserItemComponent } from "./user-item.component";

@Component({
  selector: "app-users-list",
  imports: [UserItemComponent],
  templateUrl: "./users-list.component.html",
})
export class UsersListComponent {
  readonly store = inject(UsersStore);
}
```

And in the template:

```html
<ul class="space-y-2">
  @for (user of store.filteredUsers(); track user.id) {
  <app-user-item [user]="user" />
  } @empty {
  <li class="text-center text-gray-400 py-12">No users match your filters.</li>
  }
</ul>
```

Component stays focused on rendering and handling user actions and store handles all the business logic. Everything is separated, clean and readable.

## Updating state and handling async data

A big advantage of Signal Store is that its methods act as the feature's public API. Components do not need to know the store's internal state or update logic. They just call clear methods.

State management is most useful when it handles async work well. In Angular apps, that usually means API calls, loading states, and error handling.

Here is an example of updating user data from the store — we are adding a new method:

```typescript
updateUser(id: number, changes: Partial<Pick<User, 'name' | 'role'>>) {
    usersService.updateUser$(id, changes).subscribe({
        next: (updated: User) =>
            patchState(store, {
                users: store.users().map((u) => (u.id === id ? updated : u)),
                editingUserId: null,
            }),
        error: (err: Error) =>
            patchState(store, {
                error: err?.message ?? 'Failed to update user',
            }),
    });
},
```

`patchState` helps keep updates centralized. This improves readability in small features and makes larger ones easier to update later.

## Wrap Up

NgRx Signal Store gives Angular developers a practical middle path. It is more structured than ad hoc component state, lighter than a full reducer-heavy setup for many feature-level cases, and closely aligned with Angular's signal-based direction.

The official docs and ecosystem material consistently present it as a composable API built from small features, and that is exactly why it feels approachable in real projects.

Thanks for reading, and don't forget to try Signal Store for yourself!
https://github.com/janskiba/signal-store-example
