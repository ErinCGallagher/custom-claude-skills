# Next.js Frontend Reference (Vercel)

Full annotated configuration for the Next.js frontend deployed on Vercel.

---

## `lib/auth-client.ts` — Auth Client Setup

```ts
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  // Must point to the Express/Railway backend — not the Next.js app itself
  baseURL: process.env.NEXT_PUBLIC_API_URL!, // https://api.railway.app

  fetchOptions: {
    // REQUIRED for cross-origin cookie-based auth.
    // Without this, the browser will not send or accept cookies across domains.
    credentials: "include",
  },
});
```

> `NEXT_PUBLIC_` prefix is required for the variable to be available in the browser bundle.

---

## `app/api/auth/[...all]/route.ts` — API Route Handler

Only needed if you want to proxy auth through Next.js (uncommon with a separate Express backend). In most cases, the client calls Railway directly and this file is not needed.

If you do need it (e.g. to avoid CORS entirely by proxying):

```ts
import { auth } from "@/lib/auth"; // Your auth instance if server-side auth is also set up
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth);
```

For Pages Router:
```ts
// pages/api/auth/[...all].ts
import { toNodeHandler } from "better-auth/node";
import { auth } from "@/lib/auth";

export const config = { api: { bodyParser: false } };
export default toNodeHandler(auth.handler);
```

---

## Using the Auth Client in Components

```tsx
"use client";
import { authClient } from "@/lib/auth-client";

export function SignInForm() {
  const handleSignIn = async () => {
    const { data, error } = await authClient.signIn.email({
      email: "user@example.com",
      password: "password123",
      callbackURL: "/dashboard", // where to redirect after sign-in
    });

    if (error) console.error(error);
  };

  return <button onClick={handleSignIn}>Sign In</button>;
}
```

```tsx
"use client";
import { authClient } from "@/lib/auth-client";

export function UserInfo() {
  const { data: session, isPending } = authClient.useSession();

  if (isPending) return <div>Loading...</div>;
  if (!session) return <div>Not signed in</div>;
  return <div>Hello, {session.user.name}</div>;
}
```

---

## Middleware — Protecting Routes (Next.js 13–15.1.x, Edge Runtime)

The Edge Runtime cannot make database calls. Use cookie-only check for optimistic redirects:

```ts
// middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { getSessionCookie } from "better-auth/cookies";

export function middleware(request: NextRequest) {
  const sessionCookie = getSessionCookie(request);

  if (!sessionCookie) {
    return NextResponse.redirect(new URL("/sign-in", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/profile/:path*"],
};
```

**Security note**: `getSessionCookie` only checks cookie existence, not validity. Always validate the session on the server for sensitive actions.

---

## Middleware — Full Validation (Next.js 15.2+, Node.js Runtime)

```ts
// middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { headers } from "next/headers";
import { auth } from "@/lib/auth"; // Requires auth instance accessible from middleware

export async function middleware(request: NextRequest) {
  const session = await auth.api.getSession({
    headers: await headers(),
  });

  if (!session) {
    return NextResponse.redirect(new URL("/sign-in", request.url));
  }

  return NextResponse.next();
}

export const config = {
  runtime: "nodejs", // Required
  matcher: ["/dashboard/:path*"],
};
```

---

## Middleware — Fetching Session from Railway API (recommended for separate backend)

When the auth server is on Railway (not Next.js), use HTTP to validate the session from middleware:

```ts
// middleware.ts
import { betterFetch } from "@better-fetch/fetch";
import { NextRequest, NextResponse } from "next/server";

export async function middleware(request: NextRequest) {
  const { data: session } = await betterFetch(
    `${process.env.NEXT_PUBLIC_API_URL}/api/auth/get-session`,
    {
      headers: {
        cookie: request.headers.get("cookie") || "",
      },
    }
  );

  if (!session) {
    return NextResponse.redirect(new URL("/sign-in", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*"],
};
```

---

## Server Components & Server Actions

Getting session in an RSC:

```tsx
// app/dashboard/page.tsx
import { auth } from "@/lib/auth"; // Only if auth is also configured server-side in Next.js
import { headers } from "next/headers";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const session = await auth.api.getSession({
    headers: await headers(),
  });

  if (!session) redirect("/sign-in");

  return <h1>Welcome {session.user.name}</h1>;
}
```

If auth lives exclusively on Railway (not in Next.js), fetch the session over HTTP:

```tsx
import { cookies } from "next/headers";

async function getSession() {
  const cookieStore = await cookies();
  const res = await fetch(`${process.env.API_URL}/api/auth/get-session`, {
    headers: { cookie: cookieStore.toString() },
    cache: "no-store",
  });
  if (!res.ok) return null;
  return res.json();
}
```

---

## Vercel Environment Variables

Set in Vercel → Project → Settings → Environment Variables:

| Variable | Value |
|---|---|
| `NEXT_PUBLIC_API_URL` | `https://your-api.railway.app` |

For server-only variables (not exposed to browser), omit the `NEXT_PUBLIC_` prefix.
