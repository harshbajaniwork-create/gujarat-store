# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Project overview

- Stack: Next.js (App Router) + TypeScript, MongoDB via Mongoose (with GridFS), Redis (Upstash) for caching, NextAuth for user/vendor auth, custom JWT cookie for admin auth, TailwindCSS, ESLint, Inngest for background workflows, Razorpay and Shiprocket integrations.
- Monorepo: Single app with multiple surfaces: public store, admin, and vendor portals, plus a large REST-ish API surface under app/api.

Common commands

- Install dependencies
  - npm install
- Start dev server (http://localhost:3000)
  - npm run dev
- Build production artifacts
  - npm run build
- Start production server (after build)
  - npm run start
- Lint
  - npm run lint
  - Auto-fix: npm run lint -- --fix
- Type-check only
  - npx tsc --noEmit
- Tests
  - No test runner is configured in package.json at present.

Environment setup

Create a .env.local in the repo root. The following variables are referenced in the codebase and/or docker-compose.yml:

- MONGODB_URL
- NEXTAUTH_URL
- NEXTAUTH_SECRET
- NEXT_PUBLIC_APP_BASE_URL
- GOOGLE_CLIENT_ID
- NEXT_PUBLIC_GOOGLE_CLIENT_ID
- GOOGLE_CLIENT_SECRET
- UPSTASH_REDIS_REST_URL
- UPSTASH_REDIS_REST_TOKEN
- RAZORPAY_KEY_ID
- RAZORPAY_KEY_SECRET
- RAZORPAY_WEBHOOK_SECRET
- SHIPROCKET_EMAIL
- SHIPROCKET_PASSWORD
- SHIPROCKET_API_BASE_URL
- SHIPROCKET_DEFAULT_PICKUP_LOCATION
- CRON_SECRET
- SMTP_HOST
- SMTP_PORT
- SMTP_USER
- SMTP_PASSWORD
- SMTP_FROM
- ADMIN_USERNAME
- ADMIN_PASSWORD
- JWT_SECRET

You can verify the app is up via GET /api/health (used by the container healthcheck).

Docker (optional)

- Build and run: docker compose up --build -d --env-file .env
- Exposes port 3000 and hits /api/health for a container healthcheck.
- The compose file passes the same environment variables as both build args and runtime env; ensure they are provided via your environment or an env file.

Architecture and code structure

- App Router surfaces (app/)
  - Public store under app/(root): product/category pages, cart/checkout, profile, blog, etc. Each route folder contains page.tsx (and sometimes client.tsx) with layout.tsx at section roots.
  - Admin portal under app/admin: feature folders for products, categories, brands, orders, customers, blogs, referrals, sizes, etc. Uses its own layout and nested routes (e.g., add/edit/[id]).
  - Vendor portal under app/vendor: analogous feature set for vendors with gated access and its own layout.
  - Auth routes under app/(auth) and app/api/auth/[...nextauth] for NextAuth.

- API surface (app/api)
  - Organized by domain and audience (public, admin, vendor). Examples: products, categories, brands, orders, users, vendor, discounts, referrals, shiprocket, razorpay, files, etc.
  - Typical pattern: connectToDB(), domain service/action, JSON NextResponse. Public endpoints (e.g., products) support pagination and sorting via query params.
  - Health endpoint at app/api/health/route.ts returns { status: "ok" }.

- Data layer (MongoDB + Mongoose)
  - Connection and GridFS setup in lib/mongodb.ts (connectToDB). GridFS bucket "uploads" is initialized once per process.
  - Schemas/models live in lib/models/* (e.g., user.model.ts, product.model.ts, order.model.ts, vendor.model.ts, etc.).

- Authentication and authorization
  - Users and vendors authenticate via NextAuth credentials provider (lib/nextAuthConfig.ts), JWT sessions, role attached to token/session.
  - Admin routes use a separate JWT cookie (admin_auth_token) validated via lib/middleware/auth.ts helpers. Admin checks are also enforced in middleware.ts for /admin routes.
  - Vendor middleware (middleware.ts) gates /vendor routes using next-auth JWT and an additional verification check via /api/vendor/verification; redirects accordingly.

- Caching (Redis via Upstash)
  - lib/redis.ts configures the Redis client from UPSTASH_REDIS_REST_URL and UPSTASH_REDIS_REST_TOKEN.
  - services/cache.service.ts provides get/set/del/keys helpers.
  - Example: services/product.service.ts caches product queries (TTL ~300s) with keys like products:paginated:...

- Background workflows
  - Inngest client is set up in lib/inngest/client.ts and an API endpoint exists at app/api/inngest/route.ts. Event-driven tasks (e.g., notifications/emails) are organized under lib/workflows/*.

- UI and forms
  - Reusable UI primitives in components/ui/* (shadcn-style wrappers: button, dialog, table, form, etc.).
  - Complex forms are organized by domain in lib/forms/*.

- Domain actions/services
  - Server actions under lib/actions/* group business logic per domain for both admin and vendor flows.
  - Additional domain logic under services/* (e.g., product.service.ts) to encapsulate querying, transformation, and caching.

- Path aliases and TS config
  - tsconfig.json sets paths: "@/*" -> project root. Most modules import using @/...

- Linting
  - Lint is run via next lint. The repo contains eslint.config.mjs (flat config) and a legacy .eslintrc.ts; Next/Eslint will prefer the flat config.

Notes for future automation in Warp

- Use npm scripts for build/lint/start; there is no test script configured.
- When running commands that rely on environment variables (MongoDB, Redis, NextAuth, payments/shipping), ensure .env.local is present or variables are exported in your session.
