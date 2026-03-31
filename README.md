# IG Scheduler Pro

A production-grade, multi-user Instagram scheduling application built as a Turborepo monorepo. This application allows users to connect their Meta/Instagram accounts, generate captions using Anthropic's Claude AI, upload media via AWS S3/Cloudflare R2, and schedule or immediately publish single-image and carousel posts directly via the Meta Graph API.

## 🏗️ Architecture Stack
- **Workspaces:** Managed via Turborepo
- **Frontend (`apps/web`):** React 19, Vite, Tailwind CSS, Zustand (State Management), React Router v7, Axios.
- **Backend (`apps/api`):** Node.js, Express, TypeScript, Prisma ORM, Zod (Validation), BullMQ (Background Jobs).
- **Security:** bcrypt (Hashing), jsonwebtoken (JWT logic), AES-256-GCM (Meta Token Encryption).
- **Databases:** PostgreSQL 16 (Relational state), Redis (BullMQ queue & caching).

---

## 🚀 Getting Started (Local Development)

### 1. Start Infrastructure
Start the local PostgreSQL database and Redis cache:
```bash
docker compose up -d
```

### 2. Database Sync
Navigate to the API layer and push the schema to the database:
```bash
cd apps/api
npx prisma db push
npx prisma generate
```

### 3. Run Development Servers
From the root folder, launch both the frontend and backend concurrently:
```bash
npm run dev
```

The frontend will run on `http://localhost:5173` and the backend will run on `http://localhost:3001`.

---

## 🔒 Authentication (Development Bypass)
*Currently, the authentication flow is bypassed to speed up local UI and API development.*

1. **Backend:** The `authMiddleware` inside `apps/api/src/middleware/auth.ts` automatically intercepts API requests and impersonates a `dev@example.com` user. It will provision this user in your Postgres database on the fly if it doesn't exist. You do not need valid JWTs to hit the protected endpoints.
2. **Frontend:** The `<ProtectedRoute />` wrapper and the Zustand `authStore` are hardcoded to allow access to the dashboard (`/`) without forcing a redirect to `/login`. The Axios client interceptors have also been silenced to prevent layout loops.

To revert to the production JWT strategy, restore the code inside `authMiddleware`, `api/client.ts`, and `App.tsx`.

---

## 🌐 API Endpoints

The backend Express application (`apps/api/src/index.ts`) handles the following domains:

- **`/api/auth`**: Standard JWT Registration, Login, Refresh, and Logout operations.
- **`/api/accounts`**: Meta Graph API integration. Features validation of long-lived access tokens, fetching valid `igUsername` profiles, and tracking token expiration. Note: the `accessToken` is heavily encrypted using AES-256-GCM (`lib/crypto.ts`) before being stored via Prisma.
- **`/api/posts`**: CRUD boundaries for Feed, Carousel, and Story scheduled objects.
- **`/api/posts/:id/publish`**: Direct Meta Graph API publishing endpoint. This handles container creation for single-images, Stories, and highly-complex deep-nested Carousel items.
- **`/api/captions/generate`**: Hooks into the Anthropic Claude (`claude-sonnet-4`) API to formulate punchy, niche-dependent story/feed captions and hashtags returned in raw JSON format.
- **`/api/uploads/presign`**: AWS S3 pre-signed URL generator for secure frontend-to-bucket multi-part photo uploads.
- **`/api/settings`**: User profile mutation (Default Tone, Niche customization).

> Note: All incoming requests undergo strict schema parsing via the Zod `validate.ts` middleware before reaching the controllers. If parsing fails, the global `errorHandler.ts` intercept returns highly descriptive 400 responses.

---

## 🖥️ Frontend Structure (`apps/web`)

The Vite/React frontend utilizes clean route separation and a modular component tree:

- `src/App.tsx`: Base router handling `<BrowserRouter>`. All pages are dynamically lazy-loaded.
- `src/components/layout/Shell.tsx`: The primary dashboard layout equipped with a Tailwind CSS Sidebar (`lucide-react` icons) and an `<Outlet />` payload region.
- `src/stores/authStore.ts`: Zustand store managing user state.
- `src/api/client.ts`: Globally configured Axios instance wrapping `http://localhost:3001/api`.

### Active Views
- `/` - Main Queue for analyzing scheduled pipelines
- `/compose` - High level multi-format post designer 
- `/calendar` - Temporal month/week grid scheduling 
- `/connect` - Meta OAuth handler page mapping active IGAccounts 
- `/settings` - Theme & default workflow manager
