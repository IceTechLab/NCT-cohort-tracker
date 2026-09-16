# NCT Progress Tracker — Client

The web frontend for the NCT Progress Tracker, a system that records how much of a
department's curriculum an instructor has actually delivered to each cohort, and shows
students where their own cohort stands.

This is a single-page React application. It talks to the `server/` Express + Prisma API
over JSON, authenticates with a bearer token, and renders a different workspace for each
of the four account roles: **Administrator**, **Head of Department (HOD)**,
**Instructor**, and **Student**.

---

## Table of contents

- [Tech stack](#tech-stack)
- [Feature overview](#feature-overview)
- [Roles and permissions](#roles-and-permissions)
- [Getting started](#getting-started)
- [Environment variables](#environment-variables)
- [NPM scripts](#npm-scripts)
- [Project structure](#project-structure)
- [Architecture](#architecture)
  - [Routing and access control](#routing-and-access-control)
  - [Authentication and session](#authentication-and-session)
  - [API layer](#api-layer)
  - [Data fetching](#data-fetching)
  - [List controls](#list-controls)
  - [UI component library](#ui-component-library)
  - [Design system](#design-system)
- [Domain concepts](#domain-concepts)
- [API endpoints used](#api-endpoints-used)
- [Conventions](#conventions)
- [Known gaps](#known-gaps)

---

## Tech stack

| Concern            | Choice                                             |
| ------------------ | -------------------------------------------------- |
| Framework          | React 19                                           |
| Build tool         | Vite 8 (`@vitejs/plugin-react`)                    |
| Routing            | React Router DOM 7                                 |
| HTTP               | Axios 1 (`axiosInstance` with interceptors)        |
| Styling            | Tailwind CSS 4 via `@tailwindcss/vite`             |
| Icons              | `lucide-react`                                      |
| Font               | Inter Variable (`@fontsource-variable/inter`)      |
| Linting            | ESLint 9 (flat config, `react-hooks`, `react-refresh`) |
| E2E tooling        | Playwright (dev dependency, no test files yet)     |

The project is plain JavaScript (`.jsx`), not TypeScript. There is no test runner
configured.

---

## Feature overview

- **Authentication page** with a student self-registration flow and a product-preview
  panel. Instructor, HOD and administrator accounts are created by an admin, not
  self-service.
- **Administrator / HOD dashboard** listing departments with aggregate coverage,
  instructor and cohort counts, and an "attention" badge for the most blocking gap
  (no curriculum, no instructors, no cohorts). Admins can create departments.
- **Department detail** with two tabs:
  - *Overview*: cohorts (in progress / completed), instructor assignment and
    reassignment, clickable rosters, and a per-instructor delivery table.
  - *Curriculum*: a one-topic-per-line editor that publishes a new curriculum
    version, with a confirmation step when cohorts are already mid-delivery.
- **Staff management** (admins and HODs): create, edit, deactivate, reactivate and
  reset passwords for instructor and HOD accounts, filtered by role, department and
  status.
- **Student management**: searchable, filterable, sortable list with CSV export of
  exactly what is on screen, plus a per-student detail page showing every course and
  topic, with password reset, deactivate and reactivate actions.
- **Dispute queue**: students can flag a topic their instructor marked as covered but
  which they say was not delivered; admins and HODs review, expand and resolve them.
- **Instructor workspace**: create cohorts, enrol students by username, tick topics as
  covered (optimistically), mark a cohort completed, or reopen it.
- **Student progress page**: every enrolled course as a collapsible card with a
  numbered topic timeline, overall coverage, and a "Report issue" action per topic.
- **Shared profile page** for staff: upload/remove a profile picture (resized in the
  browser), change password, and see current and completed cohorts.

---

## Roles and permissions

The role comes from the JWT issued at sign-in. Each role sees a different subset of
routes, navigation and actions.

| Capability                         | ADMIN | HOD | INSTRUCTOR | STUDENT |
| ---------------------------------- | :---: | :-: | :--------: | :-----: |
| Departments dashboard              |  ✅   | ✅  |     —      |    —    |
| Create department                  |  ✅   |  —  |     —      |    —    |
| Publish curriculum                 |  ✅   | ✅  |     —      |    —    |
| Assign / reassign cohorts          |  ✅   | ✅  |     —      |    —    |
| Staff list (create / edit / reset) |  all  | own depts only | — |    —    |
| Students list                      |  all  | own depts only | — |    —    |
| Disputes queue                     |  ✅   | ✅  |     —      |    —    |
| Own cohorts workspace              |  —    | ✅  |     ✅     |    —    |
| Own profile page                   |  ✅   | ✅  |     ✅     |    —    |
| Own progress page                  |  —    |  —  |     —      |   ✅    |
| Student self-registration          |  —    |  —  |     —      |   ✅    |

Data scoping is enforced **server-side**: a HOD is only answered the departments,
staff, students and cohorts they head; a resource outside that scope returns **404**
rather than 403 so the client never learns what exists elsewhere. The client-side route
guard is a convenience, not the security boundary.

### Landing page per role

Defined in `src/routes/homePath.js`:

- `ADMIN` and `HOD` → `/admin`
- `INSTRUCTOR` → `/instructor`
- `STUDENT` → `/student`
- anything else → `/login`

---

## Getting started

### Prerequisites

- Node.js 20+ and npm
- A running instance of the backend (`server/`) — by default expected on
  `http://localhost:5001/api`

### Install and run

```bash
cd client
npm install
npm run dev
```

Vite starts a dev server (usually `http://localhost:5173`) with hot module replacement.
Open the printed URL in a browser.

To produce and preview a production build:

```bash
npm run build
npm run preview
```

---

## Environment variables

Vite exposes variables prefixed with `VITE_` to the browser. The client reads a single
one:

| Variable            | Default                        | Purpose                             |
| ------------------- | ------------------------------ | ----------------------------------- |
| `VITE_BACKEND_URL`  | `http://localhost:5000/api`    | Base URL for every API request.     |

It lives in `client/.env`:

```env
VITE_BACKEND_URL=http://localhost:5001/api
```

To point the app at a different backend, change this value and restart the dev server.
Note the default baked into `src/api/axiosInstance.js` is `http://localhost:5000/api`,
so if `.env` is missing the app will target port 5000.

---

## NPM scripts

| Script            | What it does                                            |
| ----------------- | ------------------------------------------------------- |
| `npm run dev`     | Start the Vite dev server with HMR.                     |
| `npm run build`   | Build the production bundle into `dist/`.               |
| `npm run preview` | Serve the built bundle locally.                         |
| `npm run lint`    | Run ESLint over the project (flat config in `eslint.config.js`). |

---

## Project structure

```
client/
├── index.html               # Vite entry; loads NCT-logo.png as favicon
├── vite.config.js           # react() + tailwindcss() plugins
├── tailwind.config.js       # legacy v3-style config (theme is actually in index.css)
├── eslint.config.js         # ESLint flat config
├── .env                     # VITE_BACKEND_URL
├── public/
│   ├── NCT-logo.png         # dark-surface brand mark / favicon
│   ├── NCT-logo2.png        # full wordmark for light surfaces
│   └── favicon.svg
└── src/
    ├── main.jsx             # React root; imports Inter + index.css
    ├── App.jsx              # BrowserRouter > AuthProvider > AppRoutes
    ├── index.css            # Tailwind v4 @theme tokens + .field component class
    ├── api/
    │   ├── axiosInstance.js # Axios instance, auth + 401 interceptors
    │   └── services/
    │       ├── trackerService.js  # the main endpoint map used by every page
    │       ├── profileService.js  # /me self-service endpoints
    │       └── adminService.js    # legacy helpers (unused)
    ├── context/
    │   └── authContext.jsx  # user state, login/logout, updateUser
    ├── routes/
    │   ├── appRoutes.jsx    # route table + role-guarded <Protected> layout
    │   └── homePath.js      # role -> landing path
    ├── hooks/
    │   ├── useFetch.js          # load/ready/error state machine + quiet reload
    │   └── useListControls.js   # headless search / filter / sort for lists
    ├── utils/
    │   ├── avatarFile.js    # client-side image resize to a data URL
    │   ├── csv.js           # CSV builder + download (formula-safe, BOM)
    │   ├── dateFormatter.js # en-GB date / date-time / relative formatting
    │   ├── initials.js      # "Ada Nwosu Obi" -> "AN"
    │   └── studentStatus.js # labels, tones and sort order for student status
    ├── data/
    │   └── mockCurriculum.js  # leftover mock data (unused)
    ├── components/
    │   ├── layout/          # AppShell, Sidebar, TopBar, UserMenu, Brand
    │   ├── ui/              # the design-system primitives (see below)
    │   ├── account/         # PasswordResetModal, StudentStatusBadge
    │   ├── curriculum/      # TopicTimeline
    │   ├── staff/           # StaffFormFields
    │   └── profile/         # StaffProfileView
    └── pages/
        ├── auth/Login.jsx
        ├── admin/           # dashboard, department detail, disputes, staff, students, curriculum upload
        ├── instructor/InstructorDashboard.jsx
        ├── Student/         # Studentprogress.jsx, Disputemodal.jsx
        └── profile/MyProfile.jsx
```

---

## Architecture

### Routing and access control

`src/App.jsx` wraps everything in `BrowserRouter` and `AuthProvider`, then renders
`AppRoutes`. `src/routes/appRoutes.jsx` defines the whole route table using a
`<Protected roles={[...]} />` layout route:

- `Protected` reads the current user from `useAuth()`. If there is no user, or the
  user's role is not in the allowed list, it redirects to `/login`.
- Otherwise it renders `<AppShell>` with an `<Outlet />` inside, so every protected
  page shares the same chrome (sidebar and/or top bar).

Route groups:

| Path prefix                | Roles                  | Notes                                              |
| -------------------------- | ---------------------- | -------------------------------------------------- |
| `/login`                   | guest                  | Redirects signed-in users to their home path.      |
| `/admin`, `/admin/*`       | `ADMIN`, `HOD`         | Shared management pages; server scopes HOD data.   |
| `/profile`                 | `ADMIN`, `HOD`, `INSTRUCTOR` | Self-service profile. Students have no profile page. |
| `/instructor`              | `INSTRUCTOR`, `HOD`    | A HOD can deliver cohorts too.                     |
| `/student`                 | `STUDENT`              | Read-only progress view.                           |
| `*`                        | any                    | Redirect to `/login`.                              |

`homePath.js` is a separate module because `appRoutes.jsx` may only export components
for React Fast Refresh to work.

`AppShell` chooses its chrome by role: `ADMIN` and `HOD` get the fixed sidebar (four or
five destinations); `INSTRUCTOR` and `STUDENT` get a top bar only, since they have a
single destination each.

### Authentication and session

`src/context/authContext.jsx` provides `{ user, login, logout, updateUser }`.

- On mount, `user` is hydrated from `localStorage.nct_user`.
- `login(credentials)` POSTs to `/auth/login`, stores the returned `token` in
  `localStorage.nct_token` and the user object in `localStorage.nct_user`, then sets
  state.
- `logout()` clears both keys.
- `updateUser(patch)` merges a change to the signed-in user (used after an avatar
  upload or removal so the top bar and sidebar update without a re-login).

`src/api/axiosInstance.js` maintains the token lifecycle:

- **Request interceptor** attaches `Authorization: Bearer <token>` to every request.
- **Response interceptor** on a `401` clears `nct_token` and hard-redirects to
  `/login`.

Because of that hard redirect, endpoints that can legitimately fail with `401` while the
user is still signed in are avoided. The password-change endpoint returns `400` for a
wrong current password for exactly this reason, and the client surfaces
`response.data.message` directly.

### API layer

Three service modules wrap the endpoints; pages never call `axiosInstance` directly
(except `Login.jsx`, which POSTs to `/auth/register`):

- **`trackerService.js`** — the main map: public departments, departments, curriculum
  publishing, staff and student administration, instructor cohorts and progress,
  student progress, and disputes. Many methods take an `{ acknowledge }` option used for
  the two-phase destructive confirmations (publishing over cohorts in progress; moving a
  staff member out of departments whose cohorts they hold).
- **`profileService.js`** — the `/me` endpoints: read profile, change password, set and
  remove avatar. These key off the bearer token, take no user id, and cannot reach
  another account.
- **`adminService.js`** — an older pair of helpers superseded by `trackerService`; not
  imported anywhere.

The public departments endpoint (`/public/departments`) is deliberately unauthenticated
so the registration form can list course names before an account exists.

### Data fetching

`src/hooks/useFetch.js` is a small loading/ready/error state machine used by nearly every
page:

```js
const { data, status, error, reload, setData } = useFetch(() => tracker.cohorts(), []);
```

- `status` is `'loading' | 'ready' | 'error'` and drives skeleton/error/empty rendering.
- `reload({ quiet: true })` refreshes after a mutation **without** dropping back to the
  skeleton, so the list underneath does not flash.
- `setData` is exposed for optimistic updates — the instructor dashboard toggles a topic
  immediately and rolls back on failure.
- The fetcher is recreated each render, so the `deps` array is what actually controls
  re-fetching.

Loading states are rendered as `Skeleton` placeholders rather than spinners.

### List controls

`src/hooks/useListControls.js` provides headless search, filtering and sorting for the
table screens:

```js
const { rows, query, setQuery, filterValues, setFilter, sort, toggleSort, isFiltered, reset } =
  useListControls(staff, {
    searchKeys: ['name', 'username', 'email', departmentNames],
    filters: { status: (row, value) => (value === 'active' ? row.isActive : !row.isActive) },
    initialFilters: { status: 'active' },
    sorters: { name: (row) => row.name },
    initialSort: { key: 'name', direction: 'asc' },
  });
```

- `searchKeys` accept dotted paths or accessor functions.
- A filter value of `'all'` disables that filter.
- `reset()` restores the screen's own defaults rather than widening to `'all'`.
- `isFiltered` is measured against those defaults.

The component side is `Table`, whose `TH` renders a sort button and `aria-sort` when
`sortKey` and `onSort` are supplied.

### UI component library

Reusable primitives live in `src/components/ui/` and are shared across every page:

| Component            | Purpose                                                              |
| -------------------- | -------------------------------------------------------------------- |
| `Alert`              | Inline status message; tones error/warning/success/info.             |
| `Avatar`             | Image or initials fallback; size and tone via props.                 |
| `Badge`              | Small pill label with a tone and optional icon.                      |
| `Button`             | Variants (primary, secondary, ghost, danger, danger-quiet), sizes, loading state; renders a `Link` when `to` is given. |
| `Card`               | Bordered surface for repeated items; whole card links when `to` set. |
| `ConfirmDialog`      | `Modal` + cancel/confirm pair for destructive actions.               |
| `EmptyState`         | Icon, title, description and optional action for empty lists.        |
| `Field`              | Label + input/textarea/select + hint + error with a11y wiring.       |
| `Modal`              | Portal-rendered dialog with focus trap, body scroll lock, Escape to close, focus restore. |
| `PageHeader`         | Breadcrumb, title, subtitle and actions.                             |
| `Panel`              | Titled section with header actions and a footer slot.                |
| `ProgressBar`        | Accessible percentage bar; tone follows value unless overridden.     |
| `SearchInput`        | Search box with icon and clear button.                               |
| `SegmentedControl`   | Compact segmented control for mutually exclusive view/filter choices. |
| `Skeleton`           | Pulsing placeholder block (respects `prefers-reduced-motion`).       |
| `Table`              | `Table`/`THead`/`TBody`/`TR`/`TH`/`TD` with alignment and sorting.   |

Feature components built on top of these:

- `layout/AppShell`, `layout/Sidebar`, `layout/TopBar`, `layout/UserMenu`, `layout/Brand`
- `account/PasswordResetModal` — one modal for resetting a staff or student password;
  the parent owns the reload and success notice.
- `account/StudentStatusBadge` — shared by the list and detail page so a person is never
  labelled differently in two places.
- `curriculum/TopicTimeline` — the numbered topic list shared by the student's own page
  and the manager's view of that student. A `renderAction(item)` prop is the only
  variation (the student gets a "Report issue" button).
- `staff/StaffFormFields` — the fields of a staff account, shared by create and edit so
  the two cannot drift.
- `profile/StaffProfileView` — presentational, role-agnostic profile used by both
  `/profile` and `/admin/staff/:id`; only the avatar controls differ.

### Design system

Tailwind v4 is configured in `src/index.css` with an `@theme` block rather than in
`tailwind.config.js` (that file is a leftover from the v3-style setup). Key tokens:

- **Brand scale** `--color-brand-50 … --color-brand-950`, anchored on the NeoCloud
  periwinkle (`brand-400`).
- **Semantic surfaces** `--color-surface`, `--color-surface-sunken`,
  `--color-surface-raised`, `--color-surface-inverse`, `--color-line`,
  `--color-line-strong`.
- **Text** `--color-ink`, `--color-ink-muted`, `--color-ink-subtle`, `--color-ink-faint`.
- **Type** Inter Variable as `--font-sans`.
- **Shadows** `--shadow-card`, `--shadow-overlay`.

Because the semantic names are indirection over raw values, swapping them is what would
enable a dark theme later.

The base layer also sets a single focus treatment
(`outline-2 outline-offset-2 outline-brand-600`) for every interactive element, and
`index.css` defines one shared `.field` class for native controls. Anything with variants
or state is a React component instead.

---

## Domain concepts

A few terms are used with a specific meaning throughout the UI:

- **Department** — an area of the curriculum (for example "Cloud Engineering"). It owns
  a versioned curriculum and a set of staff.
- **Curriculum** — an ordered list of topics. Publishing **appends a version** rather
  than replacing topics. Cohorts already in progress keep the topic list they started
  with; only cohorts created afterwards get the new version. This is why the client has a
  two-phase publish: the API first replies `409` with the affected cohorts, the admin
  acknowledges, and the publish is re-submitted with `acknowledge: true`.
- **Cohort** — a delivery of a department's curriculum to a group of students, run by one
  instructor (or by a head of that department). It pins a curriculum version, has a
  progress percentage, and is marked **completed** by its instructor once every topic is
  covered. Completion locks progress editing and enrolment until reopened.
- **Progress** belongs to the **cohort**, not the student. Deactivating a student leaves
  them on the roster and changes no number.
- **Student status** — derived server-side into `NOT_ENROLLED`, `IN_PROGRESS` or
  `COMPLETED`. "Completed" means every cohort they are in has been signed off by its
  instructor, which is stricter than every topic being ticked. The three states are named
  and ordered in `src/utils/studentStatus.js`.
- **Dispute** — a student reporting that a topic marked covered was not actually
  delivered. It moves `PENDING` → `RESOLVED`; resolving is permanent.
- **Soft delete** — staff and student accounts are deactivated, never destroyed, so they
  can always be reactivated. Because tokens are stateless and last about a day, deactivation
  does not sign out existing sessions; a password reset does not either.

---

## API endpoints used

All paths are relative to `VITE_BACKEND_URL`. Authentication is a bearer token unless
noted.

| Method | Path                                             | Used by                        |
| ------ | ------------------------------------------------ | ------------------------------ |
| POST   | `/auth/login`                                    | Login                          |
| POST   | `/auth/register`                                 | Login (student self-register)  |
| GET    | `/public/departments`                            | Login (unauthenticated)        |
| GET    | `/departments`                                   | Dashboard, staff/student lists |
| POST   | `/departments`                                   | Add department (admin)         |
| GET    | `/departments/:id`                               | Department detail              |
| PUT    | `/departments/:id/curriculum`                    | Publish curriculum             |
| GET    | `/staff`                                         | Staff list                     |
| POST   | `/staff`                                         | Create staff account           |
| GET    | `/staff/:id`                                     | Staff profile                  |
| PATCH  | `/staff/:id`                                     | Edit staff account             |
| PATCH  | `/staff/:id/password`                            | Reset staff password           |
| DELETE | `/staff/:id`                                     | Deactivate staff               |
| PATCH  | `/staff/:id/reactivate`                          | Reactivate staff               |
| GET    | `/students`                                      | Student list                   |
| GET    | `/students/:id`                                  | Student profile                |
| PATCH  | `/students/:id/password`                         | Reset student password         |
| DELETE | `/students/:id`                                  | Deactivate student             |
| PATCH  | `/students/:id/reactivate`                       | Reactivate student             |
| PATCH  | `/cohorts/:id/instructor`                        | Assign / reassign instructor   |
| GET    | `/cohorts/:id/students`                          | Cohort roster                  |
| GET    | `/disputes`                                      | Disputes list                  |
| PATCH  | `/disputes/:id/resolve`                          | Resolve dispute                |
| GET    | `/instructor/cohorts`                            | Instructor dashboard           |
| POST   | `/instructor/cohorts`                            | Create cohort                  |
| POST   | `/instructor/cohorts/:id/students`               | Enrol student                  |
| PUT    | `/instructor/cohorts/:id/progress/:itemId`       | Toggle topic coverage          |
| PATCH  | `/instructor/cohorts/:id/complete`               | Mark cohort completed          |
| PATCH  | `/instructor/cohorts/:id/reopen`                 | Reopen cohort                  |
| GET    | `/student/progress`                              | Student progress page          |
| POST   | `/student/disputes`                              | Raise a dispute                |
| GET    | `/me`                                            | Profile page                   |
| PATCH  | `/me/password`                                   | Change own password            |
| PUT    | `/me/avatar`                                     | Set avatar                     |
| DELETE | `/me/avatar`                                     | Remove avatar                  |

---

## Conventions

- **No comments unless they explain intent.** The codebase uses comments sparingly and
  only where a decision is non-obvious (why a `409` is treated as a confirmation, why a
  component is shared, why a HOD sees a page). Comments restating the code are avoided.
- **Server owns derived values.** Status, progress percentages and scoping are computed
  by the API, not recomputed in the client, so a badge, a filter, a sort and an export
  cannot disagree.
- **Optimistic updates only where they are cheap to roll back** — the instructor topic
  checkbox is the example.
- **One workflow component, many surfaces.** `TopicTimeline`, `StaffProfileView`,
  `PasswordResetModal` and `StudentStatusBadge` are each rendered from two or more pages
  to guarantee the same thing looks the same everywhere.
- **Accessibility is wired at the primitive.** Focus trapping, `aria-sort`, `aria-live`,
  `aria-describedby`, `sr-only` labels and reduced-motion handling live in the `ui/`
  components and the shared `:focus-visible` rule.
- **CSV export is built client-side** from the rows currently shown, so the export
  matches the search, filters and sort on screen. Cells are escaped and formula-prefixed
  for spreadsheet safety, and the file starts with a BOM so Excel reads UTF-8 names.

---

## Known gaps

These are in the tree but not part of the running app:

- `src/pages/admin/Curriculumupload.jsx` — a standalone mock of an "AI curriculum
  processor". It is not routed and uses no API.
- `src/api/services/adminService.js` — superseded by `trackerService`; not imported.
- `src/data/mockCurriculum.js` — leftover mock data; not imported.
- `tailwind.config.js` — the theme lives in `src/index.css` under Tailwind v4; this file
  is not the source of truth.
- Playwright is installed but there are currently no test files or test script.
