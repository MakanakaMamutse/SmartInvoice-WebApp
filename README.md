# SmartInvoice — Full-Stack Invoicing & Job Management Platform 🏗️💼

> A production SaaS platform built for a private client in the construction and contracting industry — now being expanded into a multi-tenant product serving multiple businesses. Built and delivered end-to-end by **Makanaka Mamutse**.

---

<img width="1904" height="1079" alt="image" src="https://github.com/user-attachments/assets/7e969dbd-1ca4-4bd5-89d9-13a281617b40" />

<img width="1905" height="1079" alt="image" src="https://github.com/user-attachments/assets/aa7cdef2-29fc-40e6-b665-7877add7b84e" />


---

## The Brief 📝

The client had been running their contracting business on a combination of **Microsoft Word, Excel, and paper-based job sheets**. Invoicing was a manual process — copy-paste a Word template, fill in numbers by hand, save it with an ad-hoc filename, email it as an attachment. Job cost tracking lived in spreadsheets that were always a version behind. Payments were chased manually. Nothing talked to anything else.

On top of that, the business was paying a **part-time bookkeeper** to manage two overlapping responsibilities: keeping the numbers reconciled and chasing outstanding invoices. Both jobs existed largely because the tools weren't doing enough. The bookkeeper was essentially compensating for the absence of a system.

The ask was clear: **replace all of it.**

Build something modern, fast, and purpose-built for how a contracting business actually operates — where a foreman needs to log workers on-site from a phone, where the owner needs to know the profit margin on a job before it's finished, and where an invoice needs to go out professionally within minutes of completing work.

The result is SmartInvoice — a production-grade, multi-tenant SaaS platform that now runs this business and is actively being expanded to serve additional clients.

---

## The Transformation 📈

| Before | After |
|---|---|
| Invoice creation via Word template copy-paste | Structured invoice builder — sent in under 5 minutes |
| Job costs tracked in disconnected spreadsheets | Real-time profit and margin visibility per job |
| No quotation system — verbal agreements | Full quotation pipeline with client-facing documents and acceptance flow |
| Manual payment follow-ups | Automated status tracking, overdue detection, and email notifications |
| Paper-based labour records on site | Digital worker assignment, day tracking, and wage calculation per job |
| Bookkeeper managing two jobs: numbers + invoice chasing | Bookkeeper's role reduced to high-level review — the platform handles the rest |
| No business analytics | Live revenue dashboard with monthly, weekly, and daily breakdowns |

**Invoice creation time dropped from ~30 minutes to under 5 minutes.** Labour cost tracking, previously done at the end of a job from memory and paper notes, is now logged continuously — reducing end-of-job reconciliation errors by an estimated **85%**. The bookkeeper's time on invoice management dropped significantly — the system chases, tracks, and flags. They review.

---

## What It Does 🔧

SmartInvoice is not a generic invoicing tool. It is built around the specific operational model of a contracting business, with three tightly integrated domains:

### 💳 Invoicing & Quotations

The platform handles the full document lifecycle. Quotations are created, sent to clients, and tracked through an acceptance or decline flow. When a quotation is accepted, it automatically seeds a new job with the correct contract value — no re-entry. Invoices support multiple statuses (`DRAFT`, `PENDING`, `PAID`, `OVERDUE`) with automatic overdue detection based on due date logic. Invoices carry discounts, deposit splits, and itemised line items.

Every document is generated as a **custom-branded PDF** — logo embedded, correctly formatted, accurate to the line item — and delivered directly to the client via email from within the platform. No manual file management. No forwarding attachments.

### 🏗️ Job Management

Each job is a living workspace. Workers are assigned from a central labour pool, days worked and days paid are tracked independently per worker, and all site expenses are logged by category. The system calculates wages owed, wages outstanding, total cost, contract value, estimated profit, and margin — all in real time, all from the actual numbers on the job.

### 📊 Business Intelligence

The dashboard gives a real-time view of the business's financial health: monthly and weekly revenue trends, top clients by revenue, job profit margins across all projects. Server-rendered for instant load. No client-side data fetching for primary content.

---

## Engineering Depth ⚙️

### 📄 PDF Generation Pipeline

Invoices and quotations are rendered as **production-quality PDFs entirely server-side**, using a React-based document rendering library (`@react-pdf/renderer`). The pipeline works as follows:

1. A typed data object (invoice or quotation) is passed to a JSX document template
2. The template renders the full layout — logo, header, line items, totals, payment terms — as a React component tree
3. The renderer compiles that tree into a PDF binary on the server
4. The binary is streamed directly to the browser as a `application/pdf` response

No third-party PDF API. No client-side rendering. No font inconsistencies across operating systems. The client's logo is embedded as a base64-encoded asset at render time. Each PDF is named with a consistent, readable convention — e.g. `INV-042_ClientName_April2026.pdf` — so files are immediately identifiable without opening them. Custom fonts are embedded in the output for guaranteed visual consistency across all devices and email clients.

This pipeline runs as a Next.js API route, meaning it's serverless, scales on demand, and costs nothing when idle.

### 🔐 Magic Link Authentication

SmartInvoice uses **passwordless authentication via magic links**, implemented with Auth.js and a Prisma database adapter.

Here's why this matters: there is no password. The user enters their email address and receives a **time-limited, single-use cryptographic token** delivered to their inbox. They click it, they're in. The token is invalidated on use and expires automatically if unused.

From a security standpoint, this is strictly superior to password-based auth:
- No password database to breach
- No credential stuffing attacks — there are no credentials
- No "forgot password" flow — the login *is* the email
- Authentication is gated on access to the user's email inbox, which is already a high-trust signal

From a user experience standpoint, it is frictionless. Contractors using this on-site from a phone don't want to remember a password. They type their email, tap a link, they're working.

Sessions are stored in the database via the Prisma adapter, scoped per user, and validated server-side on every protected request before any data is touched.

### 📧 Email Delivery

Transactional emails — invoice delivery, quotation dispatch, magic link authentication — are sent via the **Nodemailer SDK** configured against a live SMTP relay (Mailtrap). Every email is:

- Sent from an authenticated sender domain (not a generic `noreply@someservice.com`)
- Formatted with a structured HTML template
- Delivered with correct `Reply-To` headers so clients can respond directly to the business
- For invoice and quotation emails: the PDF is generated server-side, attached inline, and included in the same send — no separate steps, no manual attachment

The email send and PDF generation are decoupled enough that either can fail independently without crashing the other. If PDF generation fails, the error is caught and surfaced — the email does not go out with a broken attachment silently.

### 🏛️ Render Architecture

Built on **Next.js 15 App Router** with a deliberate split between React Server Components and Client Components. Data-heavy views are fully server-rendered — no `useEffect`, no loading skeletons for primary content. The dashboard loads with 12 months of revenue data, all active jobs, and all client summaries in a single server pass before the user sees anything.

Client Components are scoped only to interactive islands: forms, charts, inline edit controls, tab panels. Everything else is static HTML delivered from the server.

### 🗄️ Database Design & Query Strategy

PostgreSQL on **Neon's serverless infrastructure** — edge-pooled connections, auto-suspend when idle, instant warm-up on request. The schema is multi-tenant from day one: every business record is scoped to a `userId` foreign key. New business accounts require zero schema changes.

The schema models real business relationships precisely:

- `Job` maintains foreign keys to both `Invoice` and `Quotation` — the full document lifecycle is navigable in both directions
- `WorkerAssignment` tracks `daysWorked` and `daysPaid` as independent integer fields — the system knows who is owed money at any moment without a separate calculation table
- `Expense` is categorised, dated, and job-scoped — cost breakdowns are accurate to the line item and queryable by category
- `Invoice` and `Quotation` carry client contact data denormalised at creation time — historical documents remain accurate even if client details change later

Queries are written with tight **Prisma `select` projection** — only the fields each view actually needs are fetched. Related entities are resolved in single round-trips using nested `include`. No N+1 patterns. The jobs list page, for example, resolves job metadata, all worker assignments with their daily rates, and all expense amounts for every job in a single query, computing totals post-fetch without a second database call.

### ✅ Type-Safe Forms & Server Validation

Every form in the application — invoice creation, quotation builder, job setup, worker assignment, expense logging — is validated using **Conform** (a type-safe form library for React) paired with **Zod** schemas shared between client and server.

This means:
- Form field types are derived from the same Zod schema that validates the server action
- Validation errors from the server are returned as structured `SubmissionResult` objects and mapped back to specific fields client-side — not generic toast messages
- A malformed form submission cannot reach the database. The server action parses and validates before any Prisma call runs
- The schema is the single source of truth for what a valid document looks like — no duplicated validation logic

### ⚡ Non-Blocking Mutations

User actions that trigger database writes use **React Server Actions** wrapped in `useTransition`. The UI never freezes waiting for a server round-trip. The transition stays interactive while the mutation processes server-side, and React re-renders with fresh server data on completion — no manual cache invalidation, no `setState` after an API call, no stale UI.

---

## Standout Product Decisions 💡

**The Quotation → Job → Invoice pipeline** — Documents are not isolated objects. A quotation that gets accepted seeds a job automatically. That job eventually produces an invoice. The whole lifecycle is connected and navigable — every document links to its origin and its outcome.

**Live job profitability** — Profit and margin are calculated continuously as labour and expenses are logged. The owner doesn't discover a job was unprofitable after the invoice is sent. They see it during the job.

**Labour pool architecture** — Workers are stored as persistent entities with their trade and daily rate. Assigning someone to a job is one action. Wage costs calculate automatically. Rate changes update once and flow forward.

**Overdue intelligence** — The system reasons about time. Invoices past their due date are surfaced as overdue across the dashboard, the invoice list, and the client view automatically. The business doesn't need to remember to follow up.

**Cost calculator** — Each job has a built-in scenario tool to model different cost structures before committing. Useful for quoting accuracy and protecting margin.

---

## Where This Goes Next 🚀

SmartInvoice began as a bespoke client build. It is now being evolved into a **multi-tenant SaaS product** targeting the South African construction and trades market — a segment massively underserved by tools built around US tax logic, dollar billing, and enterprise complexity. The architecture was built multi-tenant from day one. Adding new business accounts requires no schema changes.

**Roadmap:**

- 🧾 **Recurring invoices** — Automated generation for retainer clients and monthly billing cycles
- 🌐 **Client portal** — Read-only view for end clients to access invoice history and outstanding balances
- 💳 **Payment integration** — Direct payment links via Stripe and PayFast (South African card and EFT gateway)
- 📄 **Job report exports** — Formatted PDF cost summaries per job for client-facing reporting
- 📱 **PWA / offline support** — Site-based usage often has poor connectivity; offline labour logging is a priority
- 🔔 **Push notifications** — Invoice viewed, payment received, quotation accepted — real-time alerts
- 📦 **Open sourcing the job tracking core** — The labour and expense tracking module has standalone value. Considering extraction and public release

---

## Stack 🛠️

| Layer | Technology | Notes |
|---|---|---|
| Framework | Next.js 15 — App Router | Server Components, Server Actions, streaming |
| Language | TypeScript | Strict throughout — no `any` escapes |
| Database | PostgreSQL via Neon Serverless | Edge-pooled, auto-suspend, multi-tenant schema |
| ORM | Prisma | Type-safe, nested includes, tight projection |
| Auth | Auth.js + Prisma Adapter | Magic link — passwordless, session-based |
| Forms & Validation | Conform + Zod | Shared schema, structured server-side errors |
| UI Components | shadcn/ui + Radix UI | Accessible primitives, fully custom-styled |
| Styling | Tailwind CSS | Mobile-first, utility-driven |
| Charts | Recharts | Server-data-driven, interactive |
| PDF Generation | @react-pdf/renderer | Server-side, branded, custom-named, embedded fonts |
| Email | Nodemailer + Mailtrap SMTP | Authenticated sender, HTML templates, PDF attachment |
| Deployment | Vercel | Edge network, serverless functions, preview deployments |

---

## Built By ✍️

**Makanaka Mamutse** — Full-stack developer.

This project was designed, architected, and delivered solo — from initial brief and schema design through to production deployment and ongoing iteration with a live user base. Every layer of the stack was chosen deliberately and built from scratch.

📫 [makanakamamutse@gmail.com](mailto:makanakamamutse@gmail.com)  
🐙 [github.com/MakanakaMamutse](https://github.com/MakanakaMamutse)  
💼 Available for freelance and contract work

---

> **Note on privacy:** This is a private client project. Source code is not public per client agreement. The codebase is available for confidential review in the context of a serious hiring or freelance conversation.
