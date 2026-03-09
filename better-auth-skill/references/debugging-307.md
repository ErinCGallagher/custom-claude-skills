# Debugging 307 Redirects & Cookie Issues

Systematic playbook for diagnosing cross-domain auth failures with Better Auth on Vercel + Railway.

---

## What a 307 from Better Auth Means

A 307 Temporary Redirect response from `/api/auth/sign-in/email` (or any auth endpoint) means **Better Auth rejected the request origin** before processing it. This is a security feature, not a network error. The fix is always a configuration issue, not a code bug.

Better Auth checks:
1. The `Origin` header against `trustedOrigins`
2. The `baseURL` is set to the correct server URL
3. CORS preflight was answered correctly (so the browser sent the real request)

---

## Step-by-Step Diagnosis

### Step 1: Verify the server is reachable

```
GET https://api.railway.app/api/auth/ok
```

Expected: `{"ok": true}`

If this 404s → the Express route is misconfigured or Railway isn't deployed correctly.
If this 307s → `baseURL` is wrong (probably set to the Vercel URL).

---

### Step 2: Inspect the failing sign-in request

Open browser DevTools → Network tab → attempt sign-in → click on the POST to `/api/auth/sign-in/email`.

Check:
- **Request headers**: What is the `Origin` value? Copy it exactly.
- **Response status**: 307? 403? CORS error?
- **Response headers**: Is `Access-Control-Allow-Origin` present?

---

### Step 3: Match Origin to trustedOrigins

Take the exact `Origin` header value from Step 2 and compare it to your `trustedOrigins` array.

Common mismatches:
| Browser sends | trustedOrigins has | Result |
|---|---|---|
| `https://app.vercel.app` | `https://app.vercel.app/` | ❌ Trailing slash mismatch |
| `https://app.vercel.app` | `http://app.vercel.app` | ❌ Protocol mismatch |
| `https://www.app.vercel.app` | `https://app.vercel.app` | ❌ www mismatch |
| `https://app-git-main-abc.vercel.app` | `https://app.vercel.app` | ❌ Preview URL not covered |

Fix: copy-paste from the Network tab directly into `trustedOrigins`.

---

### Step 4: Check CORS preflight (OPTIONS request)

In DevTools Network tab, filter for `OPTIONS` requests. If you see an OPTIONS request to `/api/auth/sign-in/email` that returns a CORS error or 404:

- CORS middleware is not running before `toNodeHandler(auth)`
- `credentials: true` is missing from the CORS config
- The route order in Express is wrong

Correct order in `server.ts`:
```
app.use(cors(...))       ← 1st
app.options("*", cors()) ← 2nd (explicit preflight)
app.all("/api/auth/*")   ← 3rd
app.use(express.json())  ← 4th (never before auth handler)
```

---

### Step 5: Check if cookies are being set

After a successful sign-in (200 response):
- DevTools → Application → Cookies → select the **Railway API domain** (e.g. `api.railway.app`)
- Look for `better_auth.session_token`

Cookie not appearing?
- `sameSite` is not `none` → browser blocks cross-domain cookies
- `secure` is not `true` → required when `sameSite=none`
- `credentials: "include"` missing from authClient `fetchOptions`
- The cookie is being set on `vercel.app` instead of `railway.app` — check `baseURL`

---

### Step 6: Check better-auth version (1.4+ breaking change)

Better Auth 1.4.0 added a default `User-Agent` header to all requests. If your CORS config does not include `User-Agent` in `allowedHeaders`, Safari and some Chrome versions will fail the preflight.

Fix:
```ts
app.use(cors({
  allowedHeaders: ["Content-Type", "Authorization", "User-Agent"],
}));
```

---

### Step 7: Check `baseURL` vs `trustedOrigins` relationship

- `baseURL`: the URL where the **Express server lives** — used to generate redirect URLs and sign cookies
- `trustedOrigins`: the list of **frontends** allowed to call the auth endpoints

They must NOT be the same value in a separated frontend/backend architecture.

```ts
// WRONG — both pointing at same place
baseURL: "https://app.vercel.app",
trustedOrigins: ["https://app.vercel.app"],

// CORRECT — backend URL as baseURL, frontend URL in trustedOrigins
baseURL: "https://api.railway.app",
trustedOrigins: ["https://app.vercel.app"],
```

---

## Common Error Messages and Fixes

| Error | Cause | Fix |
|---|---|---|
| `307` on sign-in POST | Origin not in `trustedOrigins` | Add exact origin to `trustedOrigins` |
| `Invalid Origin` | Same as above | Same fix |
| CORS error, no `Access-Control-Allow-Origin` | CORS middleware after auth handler | Move `app.use(cors())` before `app.all("/api/auth/*")` |
| Request stuck on "pending" | `express.json()` before auth handler | Move `app.use(express.json())` after the auth route |
| Cookie not set after sign-in | Wrong `sameSite`/`secure` or missing `credentials: "include"` | Set `sameSite: "none"`, `secure: true`, `credentials: "include"` |
| Session lost on page refresh | Cookie set on wrong domain | Ensure `baseURL` is the Railway API URL, not Vercel |
| Works locally, fails in prod | env vars not set on Railway/Vercel | Check `BETTER_AUTH_URL`, `FRONTEND_URL`, `NEXT_PUBLIC_API_URL` |
| Vercel preview URLs fail | Hardcoded origin in `trustedOrigins` | Use function form of `trustedOrigins` with `.endsWith(".vercel.app")` |
| CORS fails only in Safari | `User-Agent` not in `allowedHeaders` | Add `"User-Agent"` to CORS `allowedHeaders` |

---

## Minimal Reproduction Config

If nothing works, strip down to this minimal config to isolate the issue:

### Railway — `auth.ts`
```ts
import { betterAuth } from "better-auth";
import { Pool } from "pg";

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL!, // https://api.railway.app
  trustedOrigins: [process.env.FRONTEND_URL!], // https://app.vercel.app
  database: new Pool({ connectionString: process.env.DATABASE_URL }),
  emailAndPassword: { enabled: true },
  advanced: {
    defaultCookieAttributes: { sameSite: "none", secure: true, httpOnly: true },
  },
});
```

### Railway — `server.ts`
```ts
import express from "express";
import cors from "cors";
import { toNodeHandler } from "better-auth/node";
import { auth } from "./auth";

const app = express();
app.use(cors({ origin: process.env.FRONTEND_URL, credentials: true, allowedHeaders: ["Content-Type", "Authorization", "User-Agent"] }));
app.options("*", cors());
app.all("/api/auth/*", toNodeHandler(auth));
app.use(express.json());
app.listen(process.env.PORT || 3001);
```

### Vercel — `lib/auth-client.ts`
```ts
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  fetchOptions: { credentials: "include" },
});
```

If this minimal config works, add your customizations back one at a time to find the breaking change.
