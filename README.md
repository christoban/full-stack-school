# Lama Dev School Management Dashboard

A full-stack school management system built with **Next.js 14 App Router**. It provides role-based dashboards for four user types (admin, teacher, student, parent) with CRUD operations for school entities, scheduling calendars, charts, and event management.

---

## Table of Contents

1. [Project Overview & Features](#1-project-overview--features)
2. [Tech Stack](#2-tech-stack)
3. [Architecture (Next.js App Router Structure)](#3-architecture-nextjs-app-router-structure)
4. [Authentication & Authorization](#4-authentication--authorization)
5. [Database Model (Prisma Schema)](#5-database-model-prisma-schema)
6. [Key Flows & Pages](#6-key-flows--pages)
7. [Server Actions & Data Layer](#7-server-actions--data-layer)
8. [Form Validation (Zod Schemas)](#8-form-validation-zod-schemas)
9. [UI Components Overview](#9-ui-components-overview)
10. [Image / Upload Handling](#10-image--upload-handling)
11. [Environment Variables](#11-environment-variables)
12. [Local Setup with Postgres (Docker Compose)](#12-local-setup-with-postgres-docker-compose)
13. [Prisma Migrate / Seed Workflow](#13-prisma-migrate--seed-workflow)
14. [Docker Build / Run Notes](#14-docker-build--run-notes)
15. [Scripts](#15-scripts)
16. [Known Caveats & TODOs](#16-known-caveats--todos)

---

## 1. Project Overview & Features

**App name:** "Lama Dev School Management Dashboard"  
(see `src/app/layout.tsx:11`; `package.json:2` uses the slug `lama-dev-next-dashboard`)

### Key features

- Role-based dashboards with distinct views per role (admin / teacher / student / parent)
- CRUD management for teachers, students, subjects, classes, and exams
- Weekly schedule calendar powered by `react-big-calendar`
- Attendance tracking displayed as a bar chart
- Student gender breakdown via a radial/pie chart
- Finance overview line chart *(currently uses hardcoded mock data)*
- Performance gauge chart *(hardcoded)*
- Event calendar with date-based filtering via `react-calendar`
- Role-scoped announcements feed
- Profile photo upload via Cloudinary (`next-cloudinary`)
- Paginated, searchable list pages (10 items per page)
- Clerk-based authentication with automatic post-login redirect to the user's role dashboard

---

## 2. Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Framework | Next.js (App Router) | 14.2.5 |
| Language | TypeScript | ^5 |
| Auth | Clerk (`@clerk/nextjs`, `@clerk/elements`) | ^5.4.1 / ^0.14.6 |
| Database ORM | Prisma (`@prisma/client`, `prisma`) | ^5.19.1 |
| Database | PostgreSQL | 15 (via Docker) |
| Form handling | `react-hook-form` + `@hookform/resolvers` | ^7.52.2 / ^3.9.0 |
| Validation | Zod | ^3.23.8 |
| Charts | Recharts | ^2.12.7 |
| Schedule calendar | `react-big-calendar` + `moment` | ^1.13.2 / ^2.30.1 |
| Date picker | `react-calendar` | ^5.0.0 |
| Image upload | `next-cloudinary` | ^6.13.0 |
| Toast notifications | `react-toastify` | ^10.0.5 |
| CSS | Tailwind CSS | ^3.4.1 |

---

## 3. Architecture (Next.js App Router Structure)

```
src/
├── app/
│   ├── layout.tsx                   # Root layout: ClerkProvider + ToastContainer
│   ├── globals.css                  # Tailwind base + react-big-calendar / react-calendar overrides
│   ├── [[...sign-in]]/page.tsx      # Catch-all sign-in page (Clerk Elements)
│   └── (dashboard)/
│       ├── layout.tsx               # Sidebar (Menu) + Navbar wrapper
│       ├── admin/page.tsx           # Admin dashboard
│       ├── teacher/page.tsx         # Teacher dashboard
│       ├── student/page.tsx         # Student dashboard
│       ├── parent/page.tsx          # Parent dashboard
│       └── list/
│           ├── teachers/            # page.tsx + [id]/page.tsx (detail)
│           ├── students/            # page.tsx + [id]/page.tsx (detail)
│           ├── parents/page.tsx
│           ├── subjects/page.tsx
│           ├── classes/page.tsx
│           ├── lessons/page.tsx
│           ├── exams/page.tsx
│           ├── assignments/page.tsx
│           ├── results/page.tsx
│           ├── events/page.tsx
│           ├── announcements/page.tsx
│           ├── attendance/          # (folder exists in structure)
│           └── loading.tsx          # Shared loading state for list pages
├── components/                      # 23 component files + forms/ subdirectory
├── lib/
│   ├── actions.ts                   # Server actions (CRUD)
│   ├── data.ts                      # Legacy static / mock data arrays
│   ├── formValidationSchemas.ts     # Zod schemas
│   ├── prisma.ts                    # Prisma singleton
│   ├── settings.ts                  # routeAccessMap + ITEM_PER_PAGE
│   └── utils.ts                     # Schedule adjustment utility
└── middleware.ts                    # Clerk middleware + RBAC enforcement
```

The app uses **Next.js Route Groups**: `(dashboard)` places all authenticated pages under a shared layout with sidebar and navbar (`src/app/(dashboard)/layout.tsx:6-31`). The sign-in page uses a catch-all route `[[...sign-in]]` (`src/app/[[...sign-in]]/page.tsx`).

---

## 4. Authentication & Authorization

### Clerk integration

- The **root layout** (`src/app/layout.tsx:21`) wraps the entire app in `<ClerkProvider>`.
- The **sign-in page** (`src/app/[[...sign-in]]/page.tsx:10-70`) uses `@clerk/elements` for a custom username/password form. After sign-in, a `useEffect` reads `user.publicMetadata.role` and redirects to `/${role}` (lines 17-19).
- The **Navbar** (`src/components/Navbar.tsx:6`) uses `currentUser()` server-side and renders Clerk's `<UserButton />`.

### RBAC via middleware (`src/middleware.ts`)

The middleware imports `routeAccessMap` from `src/lib/settings.ts` and builds a list of `{ matcher, allowedRoles }` pairs using `createRouteMatcher` (lines 5-8). On every request it extracts `role` from `sessionClaims.metadata` (line 17). If the matched route does not include the user's role, the request is redirected to `/${role}` (lines 19-23).

### Route access map (`src/lib/settings.ts:7-23`)

| Route pattern | Allowed roles |
|---|---|
| `/admin(.*)` | admin |
| `/student(.*)` | student |
| `/teacher(.*)` | teacher |
| `/parent(.*)` | parent |
| `/list/teachers` | admin, teacher |
| `/list/students` | admin, teacher |
| `/list/parents` | admin, teacher |
| `/list/subjects` | admin |
| `/list/classes` | admin, teacher |
| `/list/exams` | admin, teacher, student, parent |
| `/list/assignments` | admin, teacher, student, parent |
| `/list/results` | admin, teacher, student, parent |
| `/list/attendance` | admin, teacher, student, parent |
| `/list/events` | admin, teacher, student, parent |
| `/list/announcements` | admin, teacher, student, parent |

### Menu visibility (`src/components/Menu.tsx:5-118`)

Each menu item carries a `visible` array of roles. The `Menu` component calls `currentUser()` to obtain the role and only renders links whose `visible` array contains that role (line 131).

---

## 5. Database Model (Prisma Schema)

**File:** `prisma/schema.prisma` | **Provider:** PostgreSQL

### Entities & relationships

| Model | PK type | Key fields | Relationships |
|---|---|---|---|
| **Admin** | `String @id` | `username` (unique) | — |
| **Student** | `String @id` | username, name, surname, email?, phone?, address, img?, bloodType, sex (enum), birthday | → Parent (many-to-one), → Class, → Grade; ← Attendance[], ← Result[] |
| **Teacher** | `String @id` | same profile fields + birthday | ↔ Subject[] (many-to-many); ← Lesson[], ← Class[] (as supervisor) |
| **Parent** | `String @id` | username, name, surname, email?, phone (required), address | ← Student[] |
| **Grade** | `Int @id @autoincrement` | `level` (unique) | ← Student[], ← Class[] |
| **Class** | `Int @id @autoincrement` | name (unique), capacity, supervisorId? → Teacher | ← Lesson[], ← Student[], ← Event[], ← Announcement[]; → Grade |
| **Subject** | `Int @id @autoincrement` | name (unique) | ↔ Teacher[]; ← Lesson[] |
| **Lesson** | `Int @id @autoincrement` | name, day (enum Day), startTime, endTime | → Subject, → Class, → Teacher; ← Exam[], ← Assignment[], ← Attendance[] |
| **Exam** | `Int @id @autoincrement` | title, startTime, endTime | → Lesson; ← Result[] |
| **Assignment** | `Int @id @autoincrement` | title, startDate, dueDate | → Lesson; ← Result[] |
| **Result** | `Int @id @autoincrement` | score, examId?, assignmentId? | → Exam? (optional), → Assignment? (optional), → Student |
| **Attendance** | `Int @id @autoincrement` | date, present (bool) | → Student, → Lesson |
| **Event** | `Int @id @autoincrement` | title, description, startTime, endTime | → Class? (null = school-wide event) |
| **Announcement** | `Int @id @autoincrement` | title, description, date | → Class? (null = school-wide) |

**Enums:**
- `UserSex`: `MALE`, `FEMALE`
- `Day`: `MONDAY` … `FRIDAY`

> **Note:** `Admin`, `Student`, `Teacher`, and `Parent` use `String @id` — these IDs come directly from Clerk. All other models use auto-increment integer PKs.

> **Typo in schema:** the `Grade` model declares `classess` (line 73 of `schema.prisma`) — should be `classes`.

---

## 6. Key Flows & Pages

### Dashboards

| Dashboard | File | Contents |
|---|---|---|
| **Admin** | `src/app/(dashboard)/admin/page.tsx` | 4× UserCard (entity counts), CountChart (gender), AttendanceChart (weekly), FinanceChart, EventCalendarContainer, Announcements |
| **Teacher** | `src/app/(dashboard)/teacher/page.tsx` | BigCalendar filtered by `teacherId`, Announcements |
| **Student** | `src/app/(dashboard)/student/page.tsx` | Fetches student's class, BigCalendar filtered by `classId`, EventCalendar, Announcements |
| **Parent** | `src/app/(dashboard)/parent/page.tsx` | Fetches all children, one BigCalendar per child (by `classId`), Announcements |

### List pages

All 11 list pages share a common pattern:

1. Extract `page` from `searchParams` (default 1).
2. Build a Prisma `where` clause from remaining URL params (e.g. `search`, `classId`, `teacherId`).
3. Run `prisma.$transaction([findMany, count])` for paginated data.
4. Render `<TableSearch>` + `<Table>` + `<Pagination>` + `<FormContainer>` (for create/update/delete buttons).
5. Show "Actions" column only when `role === "admin"`.

### Detail pages

- `/list/teachers/[id]` — Profile card with photo, stats (subjects / lessons / classes counts), weekly schedule, shortcut links, Performance widget, Announcements.
- `/list/students/[id]` — Profile card, real attendance percentage (via `<StudentAttendanceCard>` in `<Suspense>`), grade / lessons / class stats, schedule, shortcuts, Performance, Announcements.

---

## 7. Server Actions & Data Layer

### Server actions (`src/lib/actions.ts`, 469 lines)

All functions are marked `"use server"` (line 1). Each action signature is `(currentState: CurrentState, data: Schema | FormData) => Promise<CurrentState>`.

| Action | Entity | Notes |
|---|---|---|
| `createSubject` / `updateSubject` / `deleteSubject` | Subject | `connect` / `set` for teacher relations |
| `createClass` / `updateClass` / `deleteClass` | Class | Direct data pass-through |
| `createTeacher` / `updateTeacher` / `deleteTeacher` | Teacher | Creates/updates/deletes the Clerk user **first**, then mirrors to Prisma using Clerk's `user.id` |
| `createStudent` / `updateStudent` / `deleteStudent` | Student | `createStudent` checks class capacity before proceeding (lines 256-263) |
| `createExam` / `updateExam` / `deleteExam` | Exam | Teacher-level ownership check is commented out (lines 374-385) |

> `revalidatePath` calls are commented out everywhere — list pages rely on `router.refresh()` called inside form components instead.

### Static mock data (`src/lib/data.ts`)

Contains legacy hardcoded arrays (teachers, students, parents, etc.) from early development. Actual pages query Prisma directly; this file is no longer consumed by the UI. It also exports `let role = "student"` (line 3) — an unused artifact.

### Prisma singleton (`src/lib/prisma.ts`)

Standard Next.js pattern: caches `PrismaClient` on `globalThis` in development to avoid exhausting the connection pool (lines 7-15).

---

## 8. Form Validation (Zod Schemas)

**File:** `src/lib/formValidationSchemas.ts`

| Schema | Key validations |
|---|---|
| `subjectSchema` | `name` (min 1), `teachers` (string array of IDs) |
| `classSchema` | `name` (min 1), `capacity` (coerced number ≥ 1), `gradeId` (coerced number ≥ 1), `supervisorId?` |
| `teacherSchema` | `username` (3–20 chars), `password` (min 8 or empty string), `email` (valid or empty), `bloodType` (min 1), `birthday` (coerced date), `sex` (MALE/FEMALE enum), `subjects?` (string array) |
| `studentSchema` | Same as teacher + `gradeId`, `classId`, `parentId` (all required) |
| `examSchema` | `title` (min 1), `startTime` / `endTime` (coerced date), `lessonId` (coerced number) |

Each schema exports a corresponding TypeScript type via `z.infer<typeof …Schema>`.

---

## 9. UI Components Overview

### Data display

| Component | Type | Description |
|---|---|---|
| `Table` | Server | Generic table accepting `columns` and a `renderRow` function; each list page defines its own columns |
| `Pagination` | Client | Prev / page numbers / Next; updates `?page=` in the URL |
| `TableSearch` | Client | Form that sets `?search=` on submit |

### Charts

| Component | Library | Data source |
|---|---|---|
| `CountChart` + `CountChartContainer` | Recharts `RadialBarChart` | Live — `prisma.student.groupBy({ by: ["sex"] })` |
| `AttendanceChart` + `AttendanceChartContainer` | Recharts `BarChart` | Live — current week attendance from Prisma |
| `FinanceChart` | Recharts `LineChart` | **Hardcoded** monthly income/expense mock data |
| `Performance` | Recharts `PieChart` (half-donut) | **Hardcoded** 92/8 split |

### Calendars

| Component | Library | Notes |
|---|---|---|
| `BigCalendar` (`BigCalender.tsx`) | `react-big-calendar` | Work-week view; note the typo in the filename |
| `BigCalendarContainer` | — | Server component; fetches lessons by `teacherId` or `classId`, adjusts dates to current week via `adjustScheduleToCurrentWeek` (`src/lib/utils.ts:14-45`) |
| `EventCalendar` | `react-calendar` | Client component; pushes `?date=` to the URL on selection |
| `EventList` | — | Server component; fetches events for the selected date from Prisma |
| `EventCalendarContainer` | — | Combines `EventCalendar` + `EventList` |

### Cards & widgets

| Component | Notes |
|---|---|
| `UserCard` | Server component; counts records per entity type via `prisma.[model].count()` |
| `Announcements` | Server component; shows 3 latest announcements, role-scoped (admin sees all; others see class-relevant or global ones) |
| `StudentAttendanceCard` | Server component; calculates attendance percentage for the current calendar year |

### Forms & modals

| Component | Type | Notes |
|---|---|---|
| `FormContainer` | Server | Fetches related data (teachers, grades, lessons…) and passes it to `FormModal` |
| `FormModal` | Client | Modal overlay; lazily loads form components via `next/dynamic`; handles delete confirmation |
| `TeacherForm` | Client | react-hook-form + zodResolver + `CldUploadWidget` (Cloudinary, preset `"school"`) |
| `StudentForm`, `SubjectForm`, `ClassForm`, `ExamForm` | Client | Same pattern as `TeacherForm` |
| `InputField` | — | Reusable input with label + react-hook-form `register` + inline error display |

### Layout

| Component | Notes |
|---|---|
| `Menu` | Async server component; reads `currentUser()`, renders role-filtered icon links |
| `Navbar` | Decorative search bar, notification icons, user info, `<UserButton />` |

---

## 10. Image / Upload Handling

- **Cloudinary upload:** `CldUploadWidget` from `next-cloudinary` is used in `TeacherForm.tsx` (and other forms) with `uploadPreset="school"`. On success, `result.info.secure_url` is stored in local state and submitted as the `img` field.
- **Remote patterns (`next.config.mjs:3-5`):** Only `images.pexels.com` is whitelisted.

> ⚠️ **Caveat:** Cloudinary URLs (`res.cloudinary.com`) are **not** in `remotePatterns`. Next.js `<Image>` will fail to serve uploaded photos unless the Cloudinary hostname is added. The app falls back to `/noAvatar.png` for missing images, which is why this doesn't surface as an obvious error.

---

## 11. Environment Variables

Create a `.env` file (gitignored) with the following variables:

| Variable | Used in | Purpose |
|---|---|---|
| `DATABASE_URL` | `prisma/schema.prisma:7` | PostgreSQL connection string |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk SDK | Clerk frontend key |
| `CLERK_SECRET_KEY` | Clerk SDK | Clerk backend / server key |
| `NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME` | `next-cloudinary` | Cloudinary cloud name |
| `NEXT_PUBLIC_CLERK_SIGN_IN_URL` | Clerk SDK | Sign-in route (set to `/`) |

Example for local development with the Docker Compose Postgres service:

```env
DATABASE_URL="postgresql://myuser:mypassword@localhost:5432/mydb"
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="pk_test_..."
CLERK_SECRET_KEY="sk_test_..."
NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME="your_cloud_name"
NEXT_PUBLIC_CLERK_SIGN_IN_URL="/"
```

---

## 12. Local Setup with Postgres (Docker Compose)

### Start the database

```bash
docker-compose up -d postgress   # note the service name typo — see caveats below
```

### Full local workflow

```bash
# 1. Install dependencies
npm install

# 2. Copy and fill in your .env (see section 11)
cp .env.example .env   # create manually if no example file exists

# 3. Start Postgres
docker-compose up -d postgress

# 4. Run database migrations
npx prisma migrate dev

# 5. Seed the database with sample data
npx prisma db seed

# 6. Start the development server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### Docker Compose caveats

**File:** `docker-compose.yml`

1. **Service name typo** (line 4): the Postgres service is named `postgress` (double-s) but `depends_on` on line 24 references `postgres` (single-s). This mismatch causes `docker-compose up` to fail with an unresolved dependency error.
2. **`DATABASE_URL` placeholder** (line 22): the value contains `[YOUR_SERVER_IP]` and must be replaced with the actual host. For container-to-container communication use the service name (e.g. `postgress`), for host access use `localhost`.
3. **Environment variable syntax** (line 22): uses `- DATABASE_URL: value` which mixes list and mapping YAML syntax. Use `DATABASE_URL=value` instead.
4. **Missing variables:** the `app` service does not define Clerk or Cloudinary environment variables.

---

## 13. Prisma Migrate / Seed Workflow

### Migrations (`prisma/migrations/`)

| Migration folder | Change |
|---|---|
| `20240905145454_init` | Initial schema |
| `20240913083652_addbirthday` | Added `birthday` field to `Teacher` and `Student` |

### Seed script (`prisma/seed.ts`, 216 lines)

Configured in `package.json:42`:

```json
"prisma": {
  "seed": "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
}
```

The script creates deterministic sample data:

| Entity | Count |
|---|---|
| Admins | 2 (`admin1`, `admin2`) |
| Grades | 6 (levels 1–6) |
| Classes | 6 (`1A`–`6A`, capacity 15–20) |
| Subjects | 10 (Mathematics → Art) |
| Teachers | 15 (`teacher1`–`teacher15`) |
| Lessons | 30 |
| Parents | 25 |
| Students | 50 |
| Exams | 10 |
| Assignments | 10 |
| Results | 10 |
| Attendance records | 10 |
| Events | 5 |
| Announcements | 5 |

Run seeding with:

```bash
npx prisma db seed
```

---

## 14. Docker Build / Run Notes

**File:** `Dockerfile`

```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npx prisma migrate dev --name init   # ⚠ runs at BUILD time — no DB available
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

### Caveats

1. **`prisma migrate dev` at build time** (line 17): `migrate dev` is a development command that requires a live database connection. During `docker build` no database is available, so this step **will always fail**. Replace with `npx prisma generate` at build time and run `npx prisma migrate deploy` at container start (e.g. via an entrypoint script).
2. **No `.dockerignore`**: `node_modules`, `.next`, and `.env` are copied into the image unnecessarily, increasing build time and image size.
3. **No multi-stage build**: uses a single large `node:18` base image (~1 GB). A multi-stage build (build stage → slim runtime stage) would reduce the final image size significantly.
4. **Missing environment variables**: no `ENV` directives for `DATABASE_URL`, Clerk keys, or Cloudinary config — these must be provided at runtime.

---

## 15. Scripts

Defined in `package.json:5-9`:

| Script | Command | Purpose |
|---|---|---|
| `dev` | `next dev` | Start the development server with hot reload |
| `build` | `next build` | Compile and optimise for production |
| `start` | `next start` | Serve the production build |
| `lint` | `next lint` | Run ESLint via the Next.js config |

---

## 16. Known Caveats & TODOs

| # | Location | Description |
|---|---|---|
| 1 | `src/lib/actions.ts` (all actions) | `revalidatePath` is commented out everywhere. Cache invalidation relies on `router.refresh()` inside form components. |
| 2 | `src/components/FormModal.tsx:25-31` | `deleteActionMap` for `parent`, `lesson`, `assignment`, `result`, `attendance`, `event`, and `announcement` all point to `deleteSubject` as a placeholder. Deleting these entity types will silently delete a subject instead. Marked with `// TODO`. |
| 3 | `src/lib/actions.ts:374-385` | Teacher-level ownership check for exam actions is commented out — any authenticated user can create/edit/delete any exam. |
| 4 | `src/middleware.ts:10` | `console.log(matchers)` is left in and runs on every request in production. |
| 5 | `src/lib/actions.ts:254` and `src/app/(dashboard)/student/page.tsx:17` | `console.log` debug statements left in production code. |
| 6 | `src/components/FinanceChart.tsx:15-76` and `src/components/Performance.tsx:5-8` | Finance and Performance charts use hardcoded mock data. |
| 7 | `src/components/Navbar.tsx:30` | User display name is hardcoded as `"John Doe"`. |
| 8 | `src/lib/data.ts:3` | Exports `let role = "student"` — a legacy artifact, not used by any page. |
| 9 | `src/components/BigCalender.tsx` | Filename is misspelled (`BigCalender` instead of `BigCalendar`). |
| 10 | `prisma/schema.prisma:73` | `Grade.classess` is a typo — should be `classes`. |
| 11 | `next.config.mjs:3-5` | Only `images.pexels.com` is whitelisted; `res.cloudinary.com` is missing, so Cloudinary-uploaded images cannot be served through `<Image>`. |
| 12 | `docker-compose.yml` | Service name typo (`postgress`), broken `depends_on` reference, invalid env var syntax, and missing Clerk/Cloudinary vars. |
| 13 | `Dockerfile:17` | `prisma migrate dev` runs at build time where no database is available — the build will fail. |