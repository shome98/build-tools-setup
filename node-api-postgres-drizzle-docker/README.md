
---

# 📦 PostgreSQL Setup with Drizzle ORM

## Option 1: Using `pg` (with connection pooling)

```bash
npm i pg
```

Create `src/db/db.ts` (or `index.ts`) and paste:

```ts
// db.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import { env } from '../config/env';

/**
 * Shared Pool instance — supports both local PostgreSQL and Neon.
 */
const pool = new Pool({
  connectionString: env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  ssl:
    env.DB_TYPE === 'neon' ||
    /[?&]sslmode=(?!disable)([^&]+)/i.test(env.DATABASE_URL)
      ? { rejectUnauthorized: false }
      : undefined,
});

pool.on('error', (err) => {
  console.error('❌ Unexpected database pool error:', err.message);
});

/**
 * Drizzle ORM instance bound to the pool.
 * Import and export schema as well if needed.
 */
// export const db = drizzle(pool, { schema });
```

---

## Option 2: Using `postgres` (with built‑in pooling)

```bash
npm i postgres
```

Create `src/db/db.ts` (or `index.ts`) and paste:

```ts
// db.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';
import { env } from '../config/env';

const client = postgres(env.DATABASE_URL, {
  max: 10, // connection pool size
  idle_timeout: 20, // seconds before idle connection closes
  connect_timeout: 10, // seconds before failing to connect
});

export const db = drizzle(client, { schema });
export { client };
```

---

### 🔎 Notes

- **`pg`**: More traditional, widely used, explicit SSL handling.
- **`postgres` (postgres‑js)**: Cleaner, modern, promise‑based, pooling built‑in.
- Both integrate seamlessly with Drizzle ORM — choose based on ecosystem preference.

---

This structure keeps your **`db` folder clean** and gives you two clear options to wire Drizzle with Postgres.
