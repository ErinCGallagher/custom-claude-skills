---
name: better-auth-nextjs-express
description: >
  Use this skill whenever working with Better Auth in any capacity — setup, debugging, configuration,
  or deployment. Covers: Next.js (App Router + Pages Router) frontend, Express.js backend, PostgreSQL
  database, and cross-domain deployments (Vercel frontend + Railway backend). Always trigger this skill
  for issues involving 307 redirects after sign-in, cookies not being set, CORS errors, "Invalid Origin"
  errors, session not persisting across domains, or any Better Auth configuration question. Also trigger
  for fresh Better Auth project setup even if the user doesn't mention a bug.
---

# Better Auth — Next.js + Express + PostgreSQL Skill

This skill covers setting up, configuring, and debugging **Better Auth** with:
- **Frontend**: Next.js (App Router or Pages Router) deployed on Vercel
- **Backend**: Express.js deployed on Railway
- **Database**: PostgreSQL (direct `pg` Pool or Drizzle adapter)

For detailed reference configs, see:
- `references/express-backend.md` — Full Express server setup
- `references/nextjs-frontend.md` — Full Next.js client/server setup
- `references/debugging-307.md` — 307 redirect & cookie debugging playbook

---

## Architecture Overview

There are **two different architectures** for Next.js + Express + BetterAuth. The configuration is **completely different** between them.

### Architecture A: Cross-Domain (Direct Backend Requests)

```
[Next.js on Vercel]  <-->  [Express on Railway]  <-->  [PostgreSQL]
  app.vercel.app              api.railway.app
  (direct requests)           (better-auth handler)
```

Frontend makes **direct requests** to the backend URL. This requires:
- `sameSite: "none"` cookies
- CORS with `credentials: true`
- `baseURL` = backend domain
- `trustedOrigins` = frontend domain(s)

### Architecture B: Proxy-Based (Next.js Rewrites) ⭐ **RECOMMENDED**

```
[Next.js on Vercel] --rewrites--> [Express on Railway] <--> [PostgreSQL]
  www.example.com                    api.railway.app
  (proxy via /api/*)                 (better-auth handler)

Browser sees: www.example.com/api/auth/* (same domain!)
Actually hits: api.railway.app/api/auth/* (transparent proxy)
```

Frontend uses **Next.js rewrites** to proxy requests. From the browser's perspective, all requests are same-domain. This requires:
- `sameSite: "lax"` cookies ✅
- `baseURL` = **frontend domain** (where cookies are set)
- Client components use **relative paths** (`/api/*`)
- More secure than cross-domain setup

**This project uses Architecture B (proxy-based).** The rest of this guide covers both architectures where they differ.

---

## 307 Redirect Root Causes — Check These First

A 307 after sign-in almost always means Better Auth is **rejecting the request origin** and redirecting it. Work through these causes in order:

### 1. `baseURL` configuration — DEPENDS ON ARCHITECTURE ⚠️

**Architecture A (Cross-Domain):**
```ts
// auth.ts on Express - baseURL = BACKEND domain
export const auth = betterAuth({
  baseURL: "https://api.railway.app", // ✅ The Express/Railway URL
})
```
Set `BETTER_AUTH_URL=https://api.railway.app` in Railway environment.

**Architecture B (Proxy-Based):** ⭐ **THIS PROJECT**
```ts
// auth.ts on Express - baseURL = FRONTEND domain
export const auth = betterAuth({
  baseURL: "https://www.example.com", // ✅ The Next.js/Vercel URL
  // NOT the Railway backend URL!
})
```
Set `BETTER_AUTH_URL=https://www.example.com` in Railway environment.

**Why the difference?**
- BetterAuth uses `baseURL` to set session cookies
- In proxy setup, cookies must be set on the **frontend domain** where the browser sees requests
- If set to backend domain, cookies won't be sent with subsequent requests

### 2. `trustedOrigins` must include the exact Vercel frontend URL

```ts
export const auth = betterAuth({
  baseURL: process.env.API_URL,
  trustedOrigins: [
    process.env.FRONTEND_URL!,   // e.g. https://app.vercel.app  (no trailing slash)
    "http://localhost:3000",
  ],
})
```

**Common pitfall**: trailing slash mismatch. `https://app.vercel.app` is not equal to `https://app.vercel.app/` — check the exact string the browser sends in the `Origin` header.

### 3. CORS must be configured BEFORE the Better Auth handler

```ts
// server.ts

// ✅ CORRECT order
app.use(cors({ ... }));
app.options("*", cors());             // Handle preflight explicitly
app.all("/api/auth/*", toNodeHandler(auth));
app.use(express.json());              // express.json() MUST come AFTER better-auth

// ❌ WRONG — cors after better-auth won't cover preflight
app.all("/api/auth/*", toNodeHandler(auth));
app.use(cors({ ... }));
```

CORS config must have `credentials: true` and include `User-Agent` in `allowedHeaders` (required for better-auth 1.4+):

```ts
app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true,
  methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
  allowedHeaders: ["Content-Type", "Authorization", "User-Agent"],
}));
```

### 4. Cookie attributes — DEPENDS ON ARCHITECTURE ⚠️

**Architecture A (Cross-Domain):**
```ts
export const auth = betterAuth({
  advanced: {
    defaultCookieAttributes: {
      sameSite: "none",  // Required for cross-domain cookies
      secure: true,      // Required when sameSite=none
      httpOnly: true,
      path: "/",
    },
  },
})
```
Without `sameSite: "none"` and `secure: true`, browsers will silently block cookies from being sent or set across different domains.

**Architecture B (Proxy-Based):** ⭐ **THIS PROJECT**
```ts
export const auth = betterAuth({
  advanced: {
    useSecureCookies: process.env.NODE_ENV === "production",
    defaultCookieAttributes: {
      sameSite: "lax",  // ✅ More secure - works with proxy since same-site from browser perspective
      secure: process.env.NODE_ENV === "production",
      httpOnly: true,
      path: "/",
    },
  },
})
```
Use `sameSite: "lax"` NOT `"none"` - the proxy makes requests appear same-site, so this is more secure.

### 5. Auth client configuration — DEPENDS ON ARCHITECTURE ⚠️

**Architecture A (Cross-Domain):**
```ts
// lib/auth-client.ts (Next.js)
export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_API_URL, // https://api.railway.app
  fetchOptions: {
    credentials: "include", // REQUIRED — tells browser to send/accept cookies cross-origin
  },
});
```

**Architecture B (Proxy-Based):** ⭐ **THIS PROJECT**
```ts
// lib/auth-client.ts (Next.js)
export const authClient = createAuthClient({
  // Don't set baseURL - use same domain to leverage Next.js rewrites
  // This ensures cookies are set on the frontend domain
  plugins: [adminClient()],
});
```

**CRITICAL for proxy setup:** Don't set `baseURL` on the client. The client will use relative paths which get proxied through Next.js rewrites.

### 6. `express.json()` must NOT appear before the Better Auth handler

If `express.json()` processes the body before `toNodeHandler(auth)` runs, requests to `/api/auth/*` will hang on "pending". It must be mounted after.

---

## Proxy Setup (Architecture B) — Additional Requirements ⭐

If using the **proxy-based architecture** (Next.js rewrites), follow these additional rules:

### Next.js Rewrites Configuration

```js
// next.config.mjs
export default {
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: `${process.env.NEXT_PUBLIC_API_URL}/api/:path*`,
      },
    ];
  },
};
```

This transparently proxies all `/api/*` requests from the frontend to the backend.

### Client Component Fetch Rules — CRITICAL ⚠️

**✅ CORRECT - Use relative paths:**
```typescript
// Client component making authenticated request
const response = await fetch("/api/admin/comments", {
  credentials: "include",
});
```

**❌ INCORRECT - Direct backend URLs:**
```typescript
// This bypasses the proxy and breaks cookies!
const backendUrl = process.env.NEXT_PUBLIC_API_URL || "";
const response = await fetch(`${backendUrl}/api/admin/comments`, {
  credentials: "include",
});
```

**Why this breaks:**
- Direct backend URLs create cross-domain requests (`www.example.com` → `api.railway.app`)
- Cookies set on `www.example.com` won't be sent to `api.railway.app`
- Even with `credentials: "include"`, `sameSite: "lax"` cookies won't cross domains
- The whole point of the proxy is to keep requests on the same domain

### Login Flow with `onSuccess` Callback

**❌ INCORRECT - Using `callbackURL`:**
```typescript
await authClient.signIn.email({
  email,
  password,
  callbackURL: "/admin/comments", // Server-side redirect to backend URL - doesn't exist!
});
```

**✅ CORRECT - Use `onSuccess` callback:**
```typescript
import { useRouter } from "next/navigation";

const router = useRouter();

await authClient.signIn.email(
  { email, password },
  {
    onSuccess: () => {
      router.push("/admin/comments"); // Client-side navigation
    },
    onError: (ctx) => {
      setError(ctx.error.message);
    },
  }
);
```

**Why `callbackURL` fails in proxy setup:**
- BetterAuth tries to redirect using the backend's `baseURL` (`api.railway.app`)
- Frontend routes don't exist on the backend domain
- Client-side navigation with `router.push()` works correctly

---

## Vercel Preview Deployment CORS

Vercel generates unique URLs per deploy (e.g. `app-git-branch-abc123.vercel.app`). To handle these without hardcoding each one:

```ts
// Option A: trustedOrigins as a function (better-auth supports this)
export const auth = betterAuth({
  trustedOrigins: (origin) => {
    return (
      origin === process.env.FRONTEND_URL ||
      origin.endsWith(".vercel.app") ||
      origin === "http://localhost:3000"
    );
  },
});

// Option B: CORS middleware with dynamic origin check
app.use(cors({
  origin: (origin, callback) => {
    const allowed = [process.env.FRONTEND_URL, "http://localhost:3000"];
    if (!origin || allowed.includes(origin) || origin.endsWith(".vercel.app")) {
      callback(null, true);
    } else {
      callback(new Error("Not allowed by CORS"));
    }
  },
  credentials: true,
}));
```

---

## PostgreSQL Setup (Railway)

```ts
// auth.ts
import { betterAuth } from "better-auth";
import { Pool } from "pg";

export const auth = betterAuth({
  database: new Pool({
    connectionString: process.env.DATABASE_URL,
  }),
  emailAndPassword: { enabled: true },
  // ...
});
```

Run schema migration after configuring:
```bash
npx @better-auth/cli migrate
# or generate first then review:
npx @better-auth/cli generate
```

Railway auto-provides `DATABASE_URL`. Use the **internal Railway URL** (`*.railway.internal`) for backend-to-DB connections to avoid egress costs. Use the public URL only from external tools (e.g. your local machine).

---

## Environment Variables

### Architecture A (Cross-Domain)

**Railway (Express backend):**
```env
BETTER_AUTH_SECRET=your-random-secret-min-32-chars
BETTER_AUTH_URL=https://api.railway.app       # Backend domain
DATABASE_URL=postgresql://...                  # Provided by Railway
FRONTEND_URL=https://app.vercel.app
```

**Vercel (Next.js frontend):**
```env
NEXT_PUBLIC_API_URL=https://api.railway.app   # For direct requests
```

### Architecture B (Proxy-Based) ⭐ **THIS PROJECT**

**Railway (Express backend):**
```env
BETTER_AUTH_SECRET=your-random-secret-min-32-chars
BETTER_AUTH_URL=https://www.example.com        # ⚠️ FRONTEND domain!
DATABASE_URL=postgresql://...                  # Provided by Railway
CORS_ORIGIN=https://www.example.com,https://example.com  # Supports comma-separated
```

**Vercel (Next.js frontend):**
```env
NEXT_PUBLIC_API_URL=https://api.railway.app   # For Next.js rewrites only
```

**Key difference:** In proxy setup, `BETTER_AUTH_URL` is the **frontend domain** where cookies need to be set.

The `NEXT_PUBLIC_` prefix is mandatory for the variable to be exposed to the browser bundle.

---

## Quick Debug Checklist

1. **Health check**: `GET https://api.railway.app/api/auth/ok` → should return `{"ok":true}`
2. **Identify architecture**: Are you using cross-domain (A) or proxy-based (B)? This determines ALL configuration.
3. **Network tab on sign-in**: Look at the POST to `/api/auth/sign-in/email`
   - 307 response before component loads = Next.js middleware rejecting wrong cookie name → check `proxy.ts`/`middleware.ts`
   - 307 response from API = origin rejected by better-auth → check `trustedOrigins` and `baseURL`
   - CORS error before request = preflight failed → check CORS middleware order and `credentials: true`
4. **Check the exact `Origin` header** the browser sends — compare it character-for-character with your `trustedOrigins` array (trailing slash, https vs http, www vs no-www)
5. **Cookies tab** (DevTools → Application → Cookies):
   - **Cross-domain (A)**: `better-auth.session_token` under Railway API domain
   - **Proxy (B)**: `better-auth.session_token` (dev) or `__Secure-better-auth.session_token` (prod) under frontend domain
6. **Proxy-specific**: In Network tab, verify protected requests use **relative paths** (`/api/admin/comments`), not full backend URLs
7. **401 after login in proxy setup**: Cookie exists but not sent with requests
   - Check: Are client components using relative paths or direct backend URLs?
   - Check: Is `BETTER_AUTH_URL` set to frontend domain (not backend)?
8. **better-auth 1.4+ note**: The `User-Agent` header is now added to requests by default. Add it to `allowedHeaders` in your CORS config or Safari/preflight requests will fail

### Cookie Names by Environment

- **Development**: `better-auth.session_token`
- **Production**: `__Secure-better-auth.session_token` (automatic when `useSecureCookies: true`)

See `references/debugging-307.md` for the full step-by-step playbook.

---

## Architecture Comparison Table

| Configuration | Cross-Domain (A) | Proxy-Based (B) ⭐ THIS PROJECT |
|--------------|------------------|--------------------------------|
| **Request flow** | Frontend → Backend directly | Frontend → Next.js rewrites → Backend |
| **Browser sees** | Different domains | Same domain |
| **`BETTER_AUTH_URL`** | Backend URL (`api.railway.app`) | Frontend URL (`www.example.com`) |
| **`baseURL` in auth.ts** | Backend URL | Frontend URL |
| **Auth client `baseURL`** | Backend URL | Omit (use relative paths) |
| **`sameSite` cookie** | `"none"` | `"lax"` ✅ More secure |
| **`secure` cookie** | `true` always | Production only |
| **CORS required** | Yes, with `credentials: true` | Yes, but simpler |
| **Client fetch paths** | Full URLs (`${backendUrl}/api/...`) | Relative (`/api/...`) |
| **Login navigation** | Works with `callbackURL` or `onSuccess` | Use `onSuccess` only |
| **Cookie domain** | Backend domain | Frontend domain |
| **Production cookie name** | `__Secure-better-auth.session_token` | `__Secure-better-auth.session_token` |
| **Complexity** | Higher (cross-domain cookies) | Lower (same-site cookies) |
| **Security** | Lower (`sameSite: none`) | Higher (`sameSite: lax`) |

**Recommendation:** Use Architecture B (proxy-based) whenever possible. It's more secure, simpler to configure, and avoids cross-domain cookie issues.

---

## Complete Working Config Summary

Full annotated code lives in the reference files:
- `references/express-backend.md` — `server.ts` + `auth.ts` with all options explained
- `references/nextjs-frontend.md` — `auth-client.ts`, API route, middleware, RSC usage
- `references/debugging-307.md` — Systematic debugging flowchart for 307 + cookie issues

For this project's specific proxy-based setup, see:
- `../../CLAUDE.md` — Project decisions and learnings (dated 2026-03-09)
- `../../.claude/claude-plan/better-auth-setup.md` — Complete deployment guide for proxy architecture
