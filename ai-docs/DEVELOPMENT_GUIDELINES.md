# Development Guidelines

This document consolidates the Briskhaven development standards into a single reference for Idea Board developers. It covers approved technologies, coding rules, API conventions, frontend patterns, database design, and everything else you need to build within this ecosystem.

If something in this document conflicts with a project-level `CLAUDE.md` or `README.md`, the project-level file takes precedence.

---

## Table of Contents

- [Development Philosophy](#development-philosophy)
- [Approved Technology Stack](#approved-technology-stack)
- [Code Style and Naming](#code-style-and-naming)
- [Environment Variables and Configuration](#environment-variables-and-configuration)
- [Logging](#logging)
- [API Standards](#api-standards)
- [React and Frontend Standards](#react-and-frontend-standards)
- [Database Conventions](#database-conventions)
- [Data Access Architecture](#data-access-architecture)
- [Testing](#testing)
- [Source Control and CI/CD](#source-control-and-cicd)
- [Monorepo Structure](#monorepo-structure)
- [Documentation Standards](#documentation-standards)
- [Deployment](#deployment)
- [Security](#security)
- [UX and UI Design](#ux-and-ui-design)

---

## Development Philosophy

### MVP Mentality

We are in the POC (proof of concept) stage. The goal is to make things work, not to make them perfect.

**During POC, prioritize:**
- Speed to a working prototype
- Following Briskhaven patterns (this document)
- Keeping code extensible and maintainable

**During POC, don't worry about:**
- Performance optimization
- Polish and visual perfection
- Industry best practices that slow you down

The rule is simple: when choosing between perfect architecture and shipping, choose shipping. The POC stage earns the right to move fast. Production earns the right to be thorough.

### Accessibility of Code

All code and documentation must be understandable by a junior developer. If something requires senior-level expertise just to read, it needs to be simplified or better explained.

### AI-First Development

We build with AI coding assistants as a core part of the workflow. This means:
- Structure code into smaller, focused modules (not monoliths) so AI can work with them effectively
- Write documentation that both humans and AI can parse
- Build reusable skills and scripts rather than making AI figure out the same thing repeatedly

---

## Approved Technology Stack

### Language

**JavaScript only. No TypeScript.** This is non-negotiable across all repos. Use JSDoc comments when you need type documentation.

### Module System

- **ES modules** (`import`/`export`) for Bun and Hono applications
- **CommonJS** (`require`/`module.exports`) for standalone Node.js scripts

### Runtime and Package Manager

- **Bun** is the runtime and package manager for all POC applications
- **Node.js 22+** for standalone scripts, background tasks, and utilities

### Cloud Provider

- **AWS** is the only approved cloud provider

### Databases

| Database | Use Case |
|---|---|
| **MongoDB** | Primary database for this POC — flexible schemas, fast iteration |
| **Redis** | Caching, session management, pub/sub |

### ODM

| Database | Approved Tool |
|---|---|
| MongoDB | **Mongoose** |

### Frontend

| Platform | Technology |
|---|---|
| Web and Desktop | **React 19 + Vite** |
| SSR / Streaming | **Next.js** with React 19, Server Components, Server Actions |
| Mobile | **Expo** with React Native |

### Service Layer (APIs)

| Framework | Notes |
|---|---|
| **Hono on Bun** | Preferred for new services |
| **Express.js** | Established default (uses npm) |

### Infrastructure as Code

**Terraform with HCL** is the only approved IAC toolchain. Ansible and AWS CDK are not approved.

### Dependency Choices

When choosing third-party libraries, prefer:
- Free and open source
- Well-maintained with active development
- Widely adopted with good community support

---

## Code Style and Naming

### Naming Conventions

| What | Convention | Example |
|---|---|---|
| Files | kebab-case | `get-user-by-id.js` |
| Variables and functions | camelCase | `getUserById` |
| React components | PascalCase | `UserProfile` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |

### Import Ordering

1. Third-party packages (from npm)
2. Internal project modules
3. Blank line between the two groups

### Linting and Formatting

- **ESLint** is required for all projects
- **Prettier** is encouraged for code formatting
- Each project configures its own rules, but linting must be set up and enforced

### Indentation

2 spaces. See `.editorconfig` in each repo.

---

## Environment Variables and Configuration

### No Default Values

If your application depends on an environment variable, you must **never** set a default value in code. No fallbacks, no hardcoded substitutes.

```js
// WRONG — never do this
const port = process.env.NODE_PORT || 3000;

// RIGHT — validate and fail if missing
const port = process.env.NODE_PORT;
if (!port) {
  console.error('Missing required environment variable: NODE_PORT');
  process.exit(1);
}
```

### Validation at Startup

Every application must check all required environment variables when it starts, before doing any real work. Check both:
- **Presence** — is the variable defined?
- **Data type** — if you expect a number, verify it's actually a number

If validation fails, display a clear error message saying which variables are missing or invalid, and refuse to start. Do not crash with an unhandled exception. Do not silently proceed with `undefined` values.

### Naming

Always prefix environment variables with their consumer. Never use generic names.

| Wrong | Right |
|---|---|
| `PORT` | `NODE_PORT` |
| `DATABASE_URL` | `POSTGRES_URL` |
| `API_KEY` | `VITE_API_URL` |

### Secrets During POC

During POC, formal secrets management is not required. Use safe, non-sensitive values only — no production secrets, real API keys, or real user data.

---

## Logging

Use **`console`** methods only (`console.log`, `console.warn`, `console.error`). No logging libraries.

- **Local development and POC:** Verbose logging is fine. Debug output, request details, and diagnostics are encouraged.
- **Production:** Reduce to meaningful events only — errors, warnings, and significant operational info.

Write to stdout/stderr and let infrastructure capture and route logs.

---

## API Standards

### Route Prefixes

| Prefix | Purpose | Authentication |
|---|---|---|
| `/ux/v1/` | UI-to-API communication (frontends call these) | JWT bearer token or session cookie |
| `/api/v1/` | API-to-API communication (services call these) | `x-api-key` header |

All endpoints must include the major version number (`v1`) in the path. This gives us a clean migration path for future breaking changes.

### Health Check

Every API must respond to `GET /` (no authentication required) with:

```json
{
  "name": "ideaboard-api",
  "description": "Idea Board API Service",
  "version": "1.0.0",
  "serverDateTime": "2026-02-18T12:00:00.000Z"
}
```

The values for `name`, `description`, and `version` come from `package.json`.

### Error Response Format

All errors must return this shape:

```json
{
  "code": "IDB-1001",
  "message": "User not found"
}
```

- **`code`** (required) — a product-wide unique error code. This is not an HTTP status code. Once used, a code can never be reused, even if the original error is removed.
- **`message`** (optional) — a human-readable description.

The error code registry lives in the `common` package (e.g., `ideaboard-common`). Before adding a new code, check the registry to make sure it's not already taken.

### Exception Handling

Exceptions must **never** bubble up to the end user. No raw stack traces, no unhandled exceptions, no internal error details in API responses.

Whoever calls a function that might throw is responsible for catching and translating to a clean error response. A `500` status code should never be returned under normal circumstances — it means something went catastrophically wrong.

### Pagination

APIs returning lists must support offset-based pagination, accepting a page number and page size. Each API sets its own sensible defaults.

### Request Validation

Every route handler must have a separate `validate(req)` function:
- Returns `null` if the request is valid (proceed with the handler)
- Returns an error message string if invalid (return 400 Bad Request)

Validation libraries like Zod or Joi are acceptable but not required. Plain JavaScript is fine for straightforward cases.

### Project Structure

APIs must not be monolithic single-file applications. Organize code like this:

```
src/
  app.js                  Entry point
  routes/                 All route handlers
    users/
      get-users.js
      get-user-by-id.js
      create-user.js
      index.js            Exposes handlers as public interface
    products/
      get-products.js
      index.js
  middleware/              Express/Hono middleware
  data/                   Data access, queries, schemas
  utils/                  Common utilities
```

Each major entity gets its own folder in `routes/`, with each route handler in its own file.

### Response Shaping

API responses must be pre-structured for direct UI rendering. The frontend should never have to reshape payloads it receives from the API.

---

## React and Frontend Standards

### No Direct Third-Party Imports

Never import a third-party library directly in a React component. Always wrap it in a hook or library file first. This way, if you need to swap out the library later, you only change one file.

```js
// WRONG — component imports library directly
import DatePicker from 'react-datepicker';

// RIGHT — component imports your wrapper
import { DatePicker } from '../components/DatePicker';
```

### HTTP Communication

`fetch` is the HTTP client. No Axios or other third-party HTTP libraries.

`fetch` must **never** be called directly from components or business logic. Instead, use a layered hook pattern:

1. **`useFetch`** — wraps `fetch` itself. Handles response parsing and error sanitization.
2. **`useApi`** — wraps `useFetch`. Adds base URLs, auth headers, endpoint paths.

The consuming component calls `useApi`, passes data, and gets back what it needs. It has zero knowledge of URLs, headers, HTTP methods, or authentication details.

### State Management

| Scope | Tool |
|---|---|
| Complex or global state | **Zustand** |
| Simple, localized state | **React Context** |
| Mobile (Expo) | **React Context** |

### Next.js Applications

When using Next.js:
- **Server Components** are the default — only use client components when you need interactivity
- **Server Actions** for mutations and form handling (not `/api` routes)
- Bias toward doing work on the server

### Application Structure

```
src/
  components/       Reusable UI components
  hooks/            Custom React hooks (useApi, etc.)
  lib/              Library wrappers (fetch wrapper, etc.)
  pages/            Page-level components / route targets
  layouts/          Shared layout components
```

---

## Database Conventions

This POC uses MongoDB with Mongoose. A future rebuild may move to PostgreSQL, so conventions are designed to be portable where practical.

### Core Principles

- Soft deletes only — never hard delete documents; use a `deletedAt` timestamp for logical deletion
- Never expose MongoDB `_id` values publicly — use UIDs
- Store dates as epoch values (milliseconds since January 1, 1970 UTC) for portability

### Collection Types

| Type | Purpose | Notes |
|---|---|---|
| **Type collections** | Lookups, enums, classifications | Small, relatively static documents |
| **Entity collections** | Core business data (users, boards, ideas) | Main application documents |

### Type Collections

Type collections store lookup values (like dropdown options). Required fields:

| Field | Description |
|---|---|
| `code` | Short string identifier (1-3 chars), unique |
| `name` | Human-readable display name |
| `enumName` | UPPERCASE_SNAKE_CASE code identifier |

Type collections can be hierarchical (up to 3 levels deep). Child codes build on parent codes:

| Level | Code Length | Example |
|---|---|---|
| Root | 1 char | `P` |
| Child | 2 chars | `P1` |
| Grandchild | 3 chars | `P1A` |

If you need more than 3 levels, reconsider your design.

### UID Fields

UIDs are the public-facing identifiers for entities. They appear in URLs, API responses, and frontends. Never send a MongoDB `_id` to the outside world.

| Property | Value |
|---|---|
| Format | v4 UUID with hyphens removed, uppercased |
| Length | 32 characters |
| Constraint | Unique index |
| Example | `A1B2C3D4E5F64A7B8C9D0E1F2A3B4C5D` |

Use UIDs whenever an entity appears in a public UI, API endpoint, URL, or is transmitted over the internet.

### Date and Time

Store all dates as **epoch milliseconds** (Number type). This keeps date handling consistent, avoids timezone ambiguity, and makes the data portable to any future database.

Use `0` (zero) to mean "not set" instead of `null`. This avoids issues with unique indexes that treat multiple `null` values as distinct.

### Audit Fields

Not every collection needs all audit fields. Use what's appropriate:

**Timestamp fields:**

| Field | Type | Purpose |
|---|---|---|
| `createdAt` | Number (epoch ms) | When the document was created |
| `updatedAt` | Number (epoch ms) | When last updated (0 initially) |
| `deletedAt` | Number (epoch ms) | When soft deleted (0 initially) |

**User tracking fields:**

| Field | Purpose |
|---|---|
| `createdBy` | UID of user who created the document |
| `updatedBy` | UID of user who last updated it |
| `deletedBy` | UID of user who deleted it |

**Typical usage by collection type:**

| Collection Type | Typical Audit Fields |
|---|---|
| Immutable log entries | `createdAt` only |
| Configuration documents | `createdAt`, `updatedAt` |
| User-managed entities | All timestamp and user fields |

### Data Population Responsibility

MongoDB auto-generates `_id` values. **Your application code** (the data logic layer) is responsible for populating everything else: UIDs, audit timestamps, user tracking fields, and all business fields. Do not rely on Mongoose defaults for these — set them explicitly in the repository layer.

This keeps logic visible, testable, and portable.

### Index Naming

| Type | Pattern | Example |
|---|---|---|
| Unique | `uk_{collection}_{field(s)}` | `uk_users_email` |
| Regular | `ix_{collection}_{field(s)}` | `ix_ideas_boardId` |

---

## Data Access Architecture

All database access follows a three-layer pattern. This is not optional.

```
Application / API Layer (business logic)
        |
        v
Repository Layer (validation, preparation)
        |
        v
Data Access Layer (Mongoose models, database connection)
```

### Application / API Layer

Contains business logic and request handling. **Never accesses the database directly.** Always goes through the repository.

### Repository Layer

One repository per collection. Responsibilities:
- Validate input
- Cleanse and sanitize data
- Prepare objects for the data layer
- Call data layer functions
- Handle collection-specific business rules
- **Never constructs queries directly**

### Data Access Layer

The only code that touches the database. Responsibilities:
- Define Mongoose schemas and models
- Manage the database connection (never export it)
- Automatically handle soft delete filtering (exclude deleted documents by default)
- Automatically manage audit fields

The data layer exposes semantic functions like:

| Function | Purpose |
|---|---|
| `findOne(model, criteria)` | Find a single document |
| `findMany(model, criteria)` | Find multiple documents |
| `findById(model, id)` | Find by internal ID |
| `findByUid(model, uid)` | Find by UID |
| `createOne(model, item, auditUser)` | Insert a document |
| `updateOne(model, id, updates, auditUser)` | Update a document |
| `deleteOne(model, id, auditUser)` | Soft delete a document |
| `transaction(callback)` | Run operations in a session/transaction |

All find operations automatically exclude soft-deleted documents unless you pass `{ includeDeleted: true }`.

---

## Testing

### Frameworks

| Project Type | Framework |
|---|---|
| Express.js / npm projects | **Jest** |
| Vite-based projects | **Vitest** |

No minimum test coverage threshold at this time.

### Test Location

Tests live in a `test/` folder at the root of each package, alongside `src/`.

### POC Testing Scope

During POC, focus on **happy path** testing only — verify that core functionality works as intended. Edge cases and failure scenarios will be addressed when moving to production.

### API Testing

Structure your API code so it can be tested at both:
- **Unit level** — individual route handlers, middleware, and business logic in isolation
- **Integration level** — full request/response lifecycle

Also create `.http` or `.rest` files compatible with the VS Code REST Client extension. These serve as both a testing tool and a living reference for how to call each endpoint.

### End-to-End Testing

No established E2E process yet. When implemented, the direction is **Playwright** with AI-assisted test generation.

---

## Source Control and CI/CD

### Hosting and CI

- **GitHub** for all source code
- **GitHub Actions** for all CI/CD pipelines

### Commit Messages

Follow the **Conventional Commits** specification:

| Prefix | Use |
|---|---|
| `feat:` | New feature |
| `fix:` | Bug fix |
| `chore:` | Maintenance, dependency updates |
| `docs:` | Documentation changes |
| `refactor:` | Code restructuring (no behavior change) |
| `test:` | Adding or updating tests |

### Signed Commits

Every commit to every repository must be **signed** (GPG or SSH). No exceptions at any stage.

### Branching Strategy

**GitFlow** — long-lived `main` and `develop` branches, with feature, release, and hotfix branches as needed.

### Pull Requests

During POC with a solo developer and AI assistant, PRs are not required. In production, PRs are mandatory with appropriate review.

### Protected Branches

`main` is not protected during POC. This will change as projects mature.

---

## Monorepo Structure

During POC, all work lives in a single monorepo prefixed with `poc-` (e.g., `poc-ideaboard`).

```
poc-ideaboard/
  docs/               Human-authored documentation, design decisions
  ai-docs/            AI-researched documentation, conventions, patterns
  apps/               Standalone applications (APIs, UIs, processors)
  packages/           Reusable npm packages installed into apps
  assets/             Static resources (schemas, logos, seed data)
    data/
      schemas/
    logos/
    images/
  iac/                Infrastructure as Code (Terraform)
  scripts/            Automation and utility scripts
  research/           Third-party research and reference material
  spikes/             Experimental prototypes and throwaway code
```

### apps/ vs packages/

- **`apps/`** — Standalone applications that run independently (APIs, UIs, processors, CLIs)
- **`packages/`** — Reusable npm packages that get installed as dependencies into apps

### Application Naming

Follow the pattern: `{prefix}-{component}-{technology}`

Examples: `idb-entrance-api`, `idb-entrance-ui`, `idb-web-react`

### Port Assignments

- Applications: 4100+
- MongoDB: 27017
- Redis: 6379

### Docker

All apps are containerized. Each app folder has its own Dockerfile. Use `docker compose` (not the deprecated `docker-compose`). Never include a `version:` tag in compose YAML files.

### Workspaces

Use **Bun workspaces** at the repo root for managing dependencies across apps and packages.

### Production Transition

When moving to production, the monorepo gets broken apart. Each app and package becomes its own repository named `{product}-{component}-{technology}` (e.g., `ideaboard-web-react`, `ideaboard-ux-api-js`).

---

## Documentation Standards

### READMEs

Every application and package must have a README explaining:
- What it does
- How to set it up and run it locally

### JSDoc

Any function that is exported or consumed by another file must have a JSDoc comment block documenting its purpose, parameters, and return value.

### API Documentation (OpenAPI)

Every API must be documented using the **OpenAPI specification** in YAML format. This includes all paths, request/response objects, and reusable schema definitions. OpenAPI docs must be created when the API is created and updated whenever endpoints change.

### Database Schema Documentation

Mongoose schemas serve as the primary schema documentation. If a collection's schema is not captured in a Mongoose model, it must be documented using **JSON Schema in YAML format** covering every field, type, and index. There should never be a situation where a schema exists only in the database with no corresponding definition in the repo.

---

## Deployment

### Local Development and Small Applications

**Docker Compose** on a single host. All services run together.

### Scaling Individual Services

Break Docker Compose apart and deploy each service as a separate **AWS ECS task** with its own Docker image published to **AWS ECR**.

### Serverless SPA

For React SPA applications: **AWS Lambda** for the API layer behind **Amazon CloudFront**. CloudFront behaviors route requests to the appropriate backend. Never use AWS API Gateway — all APIs use CloudFront + Lambda.

### Docker Image Tags

Image tags must never be overwritten once published, except for the `latest` tag which always points to the most recent build.

---

## Security

### CORS

CORS middleware must be installed in every API, except those deployed to AWS Lambda behind CloudFront (where CloudFront handles CORS).

- **Development:** CORS is wide open (all origins allowed)
- **Production:** Restrict to specific allowed origins

### Rate Limiting

- **Public-facing APIs:** Must implement rate limiting
- **Internal APIs** (service-to-service): No rate limiting needed

Use an existing, well-maintained open source library rather than building your own.

---

## UX and UI Design

### During POC

The goal is speed and functionality, not polish. Use lightweight components that get the job done. Pre-built templates and admin dashboard themes are encouraged — they provide layouts, navigation, forms, and tables so you can focus on application logic.

Acceptable UI frameworks and styling approaches include Material-UI, shadcn/ui, Tailwind CSS, CSS Modules, or any other well-maintained library. Choose whatever gets the app working fastest.

### During Production

Refine for accessibility (WCAG compliance, keyboard navigation, screen reader support), performance (bundle size, lazy loading, code splitting), branding, and user experience based on real feedback.
