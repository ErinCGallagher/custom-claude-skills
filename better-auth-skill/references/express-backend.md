# Express Backend Reference (Railway)

Full annotated configuration for the Express.js backend hosted on Railway.

---

## `auth.ts` — Better Auth Configuration

```ts
import { betterAuth } from "better-auth";
import { Pool } from "pg";

export const auth = betterAuth({
  // MUST be the Express/Railway URL — NOT the Next.js/Vercel URL.
  // If wrong, better-auth generates redirect URLs pointing at the frontend,
  // causing the 307 loop.
  baseURL: process.env.BETTER_AUTH_URL!, // e.g. https://api.railway.app

  // Every origin that is allowed to make auth requests.
  // Must exactly match the Origin header browsers send (no trailing slash).
  trustedOrigins: [
    process.env.FRONTEND_URL!,     // https://app.vercel.app
    "http://localhost:3000",        // local Next.js dev
  ],

  // PostgreSQL via pg Pool — Railway provides DATABASE_URL automatically
  database: new Pool({
    connectionString: process.env.DATABASE_URL,
  }),

  emailAndPassword: {
    enabled: true,
  },

  // Required for cross-domain cookie support (Vercel <-> Railway)
  advanced: {
    defaultCookieAttributes: {
      sameSite: "none",   // Allows cross-site cookie sending
      secure: true,       // Mandatory when sameSite=none (HTTPS only)
      httpOnly: true,     // Prevents JS access to session token
      path: "/",
    },
  },
});
```

### Using Drizzle adapter instead of raw Pool

```ts
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./db/schema";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const db = drizzle(pool, { schema });

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL!,
  trustedOrigins: [process.env.FRONTEND_URL!, "http://localhost:3000"],
  database: drizzleAdapter(db, { provider: "pg" }),
  // ...same advanced config as above
});
```

---

## `server.ts` — Express Server Setup

```ts
import express from "express";
import cors from "cors";
import { toNodeHandler } from "better-auth/node";
import { auth } from "./auth";

const app = express();

// ─── STEP 1: CORS middleware ────────────────────────────────────────────────
// Must run BEFORE the better-auth handler so that OPTIONS preflight requests
// get the correct headers. If cors() runs after toNodeHandler(auth), preflight
// fails and the browser blocks the actual request.
app.use(cors({
  origin: process.env.FRONTEND_URL, // exact Vercel URL or see dynamic version below
  credentials: true,                // REQUIRED — allows cookies to be sent cross-origin
  methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
  allowedHeaders: [
    "Content-Type",
    "Authorization",
    "User-Agent",   // Required for better-auth >= 1.4.0
  ],
}));

// Handle OPTIONS preflight explicitly for all routes
app.options("*", cors());

// ─── STEP 2: Better Auth handler ───────────────────────────────────────────
// Catches all /api/auth/* routes. Use /*splat for Express v5.
app.all("/api/auth/*", toNodeHandler(auth));   // Express v4
// app.all("/api/auth/*splat", toNodeHandler(auth)); // Express v5

// ─── STEP 3: express.json() — MUST come after better-auth ─────────────────
// If placed before toNodeHandler(auth), the body gets consumed and auth
// requests will hang indefinitely on "pending".
app.use(express.json());

// ─── Your other routes ─────────────────────────────────────────────────────
app.get("/api/me", async (req, res) => {
  const { fromNodeHeaders } = await import("better-auth/node");
  const session = await auth.api.getSession({
    headers: fromNodeHeaders(req.headers),
  });
  if (!session) return res.status(401).json({ error: "Unauthorized" });
  return res.json(session);
});

app.listen(process.env.PORT || 3001, () => {
  console.log("Server running");
});
```

### Dynamic CORS for Vercel preview deployments

```ts
app.use(cors({
  origin: (origin, callback) => {
    // Allow requests with no origin (e.g. server-to-server, curl)
    if (!origin) return callback(null, true);

    const allowed = [
      process.env.FRONTEND_URL,   // production Vercel URL
      "http://localhost:3000",
      "http://localhost:3001",
    ];

    // Allow any Vercel preview deploy for your project
    if (allowed.includes(origin) || origin.endsWith(".vercel.app")) {
      return callback(null, true);
    }

    callback(new Error(`CORS: origin ${origin} not allowed`));
  },
  credentials: true,
  methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
  allowedHeaders: ["Content-Type", "Authorization", "User-Agent"],
}));
```

---

## `package.json` notes

Better Auth requires ESM. Ensure your package.json has:

```json
{
  "type": "module"
}
```

Or configure `tsconfig.json` to use `"module": "ESNext"` / `"NodeNext"`.

---

## Railway Environment Variables

Set these in Railway → your service → Variables:

| Variable | Value | Notes |
|---|---|---|
| `BETTER_AUTH_SECRET` | random 32+ char string | Use `openssl rand -base64 32` |
| `BETTER_AUTH_URL` | `https://your-app.railway.app` | Railway public URL |
| `DATABASE_URL` | auto-provided | From Railway Postgres plugin |
| `FRONTEND_URL` | `https://your-app.vercel.app` | Vercel production URL |
| `PORT` | auto-provided | Railway sets this automatically |

---

## Schema Migration

After configuring `auth.ts`, generate and apply the database schema:

```bash
# Generate migration files (review before applying)
npx @better-auth/cli generate

# Apply migrations directly
npx @better-auth/cli migrate
```

Tables created: `user`, `session`, `account`, `verification` (plus plugin tables if used).
