---
name: react-router
description: Canonical React Router data mode implementations — loader, action, Form, useFetcher, async states, client boundaries, and TypeScript patterns. Use as reference when writing React Router routes, loaders, actions, or data fetching.
license: MIT
---

# React Router: Canonical Patterns

Target: React Router v7 (data mode).

## Route with loader, action, and all four async states

```typescript
// loader — data arrives before the route renders
import { json, type LoaderFunctionArgs } from "react-router";

type User = { id: string; name: string; email: string };

export async function loader({ params }: LoaderFunctionArgs) {
  const response = await fetch(`/api/users/${params.id}`);
  if (!response.ok) {
    throw json({ message: "User not found" }, { status: 404 });
  }
  const user: User = await response.json();
  return json({ user });
}

// action — mutations flow through here, not through hand-rolled fetch
import { redirect, type ActionFunctionArgs } from "react-router";

export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const name = formData.get("name") as string;
  await updateUser(name);
  return redirect("/users");
}

// component — consumes loader data, handles loading via useNavigation
import { useLoaderData, useNavigation } from "react-router";

export default function UserPage() {
  const { user } = useLoaderData<typeof loader>();
  const navigation = useNavigation();

  if (navigation.state === "loading") {
    return <UserSkeleton />;
  }

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <EditUserForm user={user} />
    </div>
  );
}
```

## Route config with layout-level error boundary

```typescript
// router.tsx — errorElement at every level that has async data
import { createBrowserRouter } from "react-router";

export const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    errorElement: <RootError />,       // catches errors from all child routes
    children: [
      {
        path: "users",
        element: <UserList />,
        loader: userListLoader,
        errorElement: <UserListError />, // scoped to /users
        children: [
          {
            path: ":id",
            element: <UserPage />,
            loader: userLoader,
            action: userAction,
            errorElement: <UserPageError />,
          },
        ],
      },
    ],
  },
]);
```

## Empty state (inside the component)

```typescript
export default function UserList() {
  const { users } = useLoaderData<typeof loader>();

  if (users.length === 0) {
    return (
      <div className="flex flex-col items-center gap-4 py-16">
        <p className="text-muted-foreground">No users yet.</p>
        <a href="/users/new" className="text-primary underline">
          Create your first user
        </a>
      </div>
    );
  }

  return (
    <ul className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
      {users.map((u) => <UserCard key={u.id} user={u} />)}
    </ul>
  );
}
```

## Client state pushed to the leaf

```typescript
// UserCard.tsx — only this leaf carries useState, parent stays stateless
import { useState } from "react";

type Props = { user: User };

export function UserCard({ user }: Props) {
  const [expanded, setExpanded] = useState(false);

  return (
    <div onClick={() => setExpanded((v) => !v)}>
      <p>{user.name}</p>
      {expanded && <UserDetails user={user} />}
    </div>
  );
}
```

## useFetcher for non-navigation data

```typescript
import { useFetcher } from "react-router";

type SearchResults = { items: Array<{ id: string; name: string }> };

export function SearchPanel() {
  const fetcher = useFetcher<SearchResults>();

  return (
    <div>
      <fetcher.Form action="/search">
        <input name="q" />
      </fetcher.Form>
      {fetcher.state === "loading" && <Spinner />}
      {fetcher.data?.items.length === 0 && <p>No results found.</p>}
      {fetcher.data && <ResultList items={fetcher.data.items} />}
    </div>
  );
}
```

## Action with validation feedback

```typescript
// action — returns validation errors without redirecting
export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const name = formData.get("name") as string;

  if (name.length < 2) {
    return json({ errors: { name: "Name must be at least 2 characters." } }, { status: 422 });
  }

  await createUser(name);
  return redirect("/users");
}

// component — reads validation errors via useActionData
import { useActionData, Form, useNavigation } from "react-router";

export function CreateUserForm() {
  const actionData = useActionData<typeof action>();
  const navigation = useNavigation();
  const submitting = navigation.state === "submitting";

  return (
    <Form method="post">
      <input name="name" aria-invalid={!!actionData?.errors?.name} />
      {actionData?.errors?.name && <p className="text-destructive">{actionData.errors.name}</p>}
      <button type="submit" disabled={submitting}>
        {submitting ? "Creating..." : "Create"}
      </button>
    </Form>
  );
}
```

## TypeScript patterns

```typescript
// Discriminated union for component-local async state
type UserViewState =
  | { status: "loading" }
  | { status: "error"; error: string }
  | { status: "empty" }
  | { status: "success"; users: User[] };

// Loader inference — use typeof loader, not as-casts
const { user } = useLoaderData<typeof loader>();

// Action inference — same pattern
const data = useActionData<typeof action>();
```
