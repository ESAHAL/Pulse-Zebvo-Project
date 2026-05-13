# Pulse — AI Social Media Content Pipeline

A full-stack AI-powered content generation platform built with Next.js 15, Prisma, NextAuth, and Google Gemini.

## Tech Stack

- **Framework**: Next.js 15 (App Router)
- **Styling**: Tailwind CSS + custom design system
- **Database**: Prisma ORM + SQLite (dev) / PostgreSQL (prod)
- **Auth**: NextAuth.js v4 (Credentials provider)
- **AI** *(Phase 2)*: Google Gemini 2.5 Flash + Imagen 3
- **Export** *(Phase 4)*: Markdown, PDF (jsPDF), JSON

## Architecture

```
ai-content-pipeline/
├── app/
│   ├── (auth)/            # Login, Signup pages
│   ├── (dashboard)/       # Protected dashboard routes
│   ├── api/
│   │   ├── auth/          # NextAuth + signup endpoint
│   │   └── workspaces/    # Workspace CRUD endpoints
│   └── globals.css        # Design system tokens
├── components/
│   ├── dashboard/         # Sidebar, topbar, modals
│   └── workspace-card.tsx
├── lib/
│   ├── auth.ts            # NextAuth config
│   ├── db.ts              # Prisma client singleton
│   └── utils.ts           # Helpers + platform config
├── prisma/
│   ├── schema.prisma      # DB schema
│   └── seed.ts            # Demo data
├── services/              # ← Business logic layer (NOT in route files)
│   ├── authService.ts     # Registration, credential verification
│   ├── workspaceService.ts# Workspace CRUD + ownership checks
│   ├── aiService.ts       # Gemini/Imagen integration (Phase 2)
│   └── exportService.ts   # MD/PDF/JSON export (Phase 4)
├── templates/
│   └── promptTemplates.ts # Reusable AI prompt builders
└── middleware.ts          # Route protection
```

## Setup

### 1. Install dependencies

```bash
npm install
```

### 2. Set environment variables

```bash
cp .env.example .env
# Edit .env — set NEXTAUTH_SECRET to a random 32+ char string
```

### 3. Initialize the database

```bash
npm run db:generate   # Generate Prisma client
npm run db:push       # Push schema to SQLite
npm run db:seed       # Optional: add demo data
```

### 4. Run the dev server

```bash
npm run dev
```

Visit [http://localhost:3000](http://localhost:3000)

**Demo credentials** (after seeding):
- Email: `demo@pulse.ai`
- Password: `Demo1234`

## Phase Progress

| Phase | Description | Status |
|-------|-------------|--------|
| **Phase 1** | Foundation, Auth, Dashboard | ✅ Complete |
| **Phase 2** | AI Engine (Gemini + Imagen) | ⏳ Pending |
| **Phase 3** | Content Feed, Edit, Schedule | ⏳ Pending |
| **Phase 4** | Export System (MD/PDF/JSON) | ⏳ Pending |

## Key Design Decisions

- **Services layer**: All business logic lives in `services/`. API routes are thin — they only parse the request and delegate.
- **Ownership checks**: Every service method verifies `userId` ownership before touching data.
- **Consistent error types**: Services return `{ success: true, data } | { success: false, error }` — never throw to routes.
- **Zod validation**: All user input is validated at the service boundary, not in the route.
- **JWT sessions**: NextAuth uses JWT strategy — no extra DB round-trip per request.
