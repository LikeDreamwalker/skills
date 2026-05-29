---
name: nextjs
description: Canonical Next.js App Router implementations — Server Components, Server Actions, loading.tsx, error.tsx, client boundaries, and streaming patterns. Use as reference when writing Next.js pages, server components, or data mutations.
license: MIT
---

# Next.js App Router: Canonical Patterns

Target: Next.js 15+ (App Router, Promise-based params, React 19).

## Page: Server Component with typed async params

```typescript
// app/users/[id]/page.tsx
import { notFound } from "next/navigation";

type Props = {
  params: Promise<{ id: string }>;
};

type User = { id: string; name: string; email: string };

export default async function UserPage({ params }: Props) {
  const { id } = await params;
  const user: User | null = await db.user.findUnique({ where: { id } });

  if (!user) notFound();

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <EditUserForm user={user} />
    </div>
  );
}
```

## generateStaticParams for static generation

```typescript
// app/users/[id]/page.tsx — generates static pages at build time
export async function generateStaticParams() {
  const users = await db.user.findMany({ select: { id: true } });
  return users.map((u) => ({ id: u.id }));
}

// Combined with the page above, these IDs are pre-rendered.
// Other IDs still work at request time (ISR/SSR fallback).
```

## Server Action with useActionState

```typescript
// app/users/actions.ts
"use server";

import { revalidatePath } from "next/cache";

export async function createUser(
  _prevState: { error?: string } | null,
  formData: FormData,
): Promise<{ error?: string } | null> {
  const name = formData.get("name") as string;

  if (name.length < 2) {
    return { error: "Name must be at least 2 characters." };
  }

  await db.user.create({ data: { name } });
  revalidatePath("/users");
  return null;
}

// app/users/create-form.tsx
"use client";

import { useActionState } from "react"; // React 19
import { createUser } from "./actions";

export function CreateUserForm() {
  const [state, action, pending] = useActionState(createUser, null);

  return (
    <form action={action}>
      <input name="name" />
      {state?.error && <p className="text-destructive">{state.error}</p>}
      <button type="submit" disabled={pending}>
        {pending ? "Creating..." : "Create"}
      </button>
    </form>
  );
}
```

## loading.tsx (sibling to page.tsx)

```typescript
// app/users/loading.tsx
export default function UsersLoading() {
  return (
    <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
      {Array.from({ length: 6 }).map((_, i) => (
        <div key={i} className="h-32 animate-pulse rounded-lg bg-muted" />
      ))}
    </div>
  );
}
```

## error.tsx (sibling to page.tsx)

```typescript
// app/users/error.tsx — must be a Client Component
"use client";

type Props = {
  error: Error;
  reset: () => void;
};

export default function UsersError({ error, reset }: Props) {
  return (
    <div className="flex flex-col items-center gap-4 py-16">
      <h2 className="text-lg font-semibold">Something went wrong</h2>
      <p className="text-muted-foreground">{error.message}</p>
      <button onClick={reset} className="rounded-md bg-primary px-4 py-2 text-primary-foreground">
        Try again
      </button>
    </div>
  );
}
```

## not-found.tsx

```typescript
// app/users/[id]/not-found.tsx
export default function UserNotFound() {
  return (
    <div className="flex flex-col items-center gap-4 py-16">
      <h2 className="text-lg font-semibold">User not found</h2>
      <a href="/users" className="text-primary underline">
        Back to all users
      </a>
    </div>
  );
}
```

## Granular Suspense for streaming

```typescript
// app/users/page.tsx
import { Suspense } from "react";

export default function UsersPage() {
  return (
    <div>
      <h1>Users</h1>
      <section>
        <h2>Active</h2>
        <Suspense fallback={<UserListSkeleton />}>
          <ActiveUserList />
        </Suspense>
      </section>
      <section>
        <h2>All</h2>
        <Suspense fallback={<UserListSkeleton />}>
          <AllUserList />
        </Suspense>
      </section>
    </div>
  );
}
```

## Client boundary: "use client" only at the leaf

```typescript
// app/users/search-input.tsx — only this leaf is a Client Component
"use client";

import { useRouter, useSearchParams } from "next/navigation";

export function SearchInput() {
  const router = useRouter();
  const searchParams = useSearchParams();

  return (
    <input
      type="search"
      defaultValue={searchParams.get("q") ?? ""}
      onChange={(e) => {
        const params = new URLSearchParams(searchParams);
        if (e.target.value) {
          params.set("q", e.target.value);
        } else {
          params.delete("q");
        }
        router.replace(`?${params.toString()}`);
      }}
    />
  );
}
```

The page that renders `<SearchInput />` remains a Server Component.

## Empty state

```typescript
// app/users/page.tsx
export default async function UsersPage() {
  const users = await db.user.findMany();

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

  return <UserGrid users={users} />;
}
```

## TypeScript patterns

```typescript
// Discriminated union for client-component async state
type AsyncState<T> =
  | { status: "loading" }
  | { status: "error"; error: string }
  | { status: "empty" }
  | { status: "success"; data: T };

// Page props with Promise params (Next.js 15+)
type PageProps = {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ q?: string; page?: string }>;
};

// Server Action return type
type ActionResult = { error?: string; fields?: Record<string, string> } | null;
```
