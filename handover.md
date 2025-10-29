# Schabort Follow-ups — Technical Handover & Spec

## Table of Contents

- [1. Overview](#1-overview)
- [2. Environment Setup](#2-environment-setup)
- [3. Database Schema & RLS Policies](#3-database-schema--rls-policies)
- [4. Import Specifications](#4-import-specifications)
  - [4.1 Clients Import (CSV)](#41-clients-import-csv)
  - [4.2 Follow-ups Import (CSV)](#42-follow-ups-import-csv)
  - [4.3 Age Analysis Import (CSV)](#43-age-analysis-import-csv)
- [5. UI Overview](#5-ui-overview)
- [6. Troubleshooting](#6-troubleshooting)
- [7. Next Steps / Improvements](#7-next-steps--improvements)

## 1. Overview

**Goal:** A multi-tenant follow‑up tracking system using Next.js and Supabase. It supports adding clients and follow‑ups, importing clients/follow‑ups from CSV (including age‑analysis CSVs), and enforces strict per‑organisation data separation with row‑level security (RLS).

**Stack:**

- **Frontend:** Next.js 15 (App Router), React 18, TypeScript, Tailwind CSS  
- **Backend:** Supabase (Postgres + Auth + RLS)  
- **CSV parsing:** Papaparse (client‑side)  
- **Storage:** Supabase Storage for optional file uploads (not covered here)  

**Key Features:**

1. **Clients:** Each client belongs to an organisation and stores fields like prefix, name, email, cell, discipline, note, company, and legacy notes.  
2. **Follow‑ups:** Each follow‑up belongs to a client. Fields include summary, content, status, due date, and timestamps.  
3. **CSV Importer:** Users can import clients (two schemas), follow‑ups (two schemas), and age analysis CSVs.  
4. **Multi‑tenancy:** Data is scoped by `org_id` via RLS policies.  
5. **Authentication:** Supabase email/password authentication. `profiles` table links Supabase users to organisations and roles (admin or ops).  

## 2. Environment Setup

1. **Clone the repository** and navigate into it.  
2. **Create `.env.local`** with these variables (replace placeholders with your own values):

   ```env
   NEXT_PUBLIC_SITE_URL=http://localhost:3003
   NEXT_PUBLIC_SUPABASE_URL=https://<your-project>.supabase.co
   NEXT_PUBLIC_SUPABASE_ANON_KEY=<your-anon-key>
   SUPABASE_SERVICE_ROLE_KEY=<your-service-role-key> # optional for admin/storage routes
   ```

3. **Install dependencies** with PNPM (or npm):

   ```bash
   pnpm install
   ```

4. **Run the app** (port can be changed if needed):

   ```bash
   set PORT=3003
   pnpm dev
   ```

5. Open `http://localhost:3003` in your browser and log in via Supabase Auth.

## 3. Database Schema & RLS Policies

Run the following SQL in Supabase’s SQL editor. It creates the necessary tables, indexes, and RLS policies idempotently:

```sql
create extension if not exists pg_trgm;

create table if not exists organizations(
  id bigserial primary key,
  name text not null unique,
  created_at timestamptz not null default now()
);

create table if not exists profiles(
  id uuid primary key references auth.users(id) on delete cascade,
  org_id bigint not null references organizations(id) on delete cascade,
  full_name text,
  email text,
  role text check (role in ('admin','ops')) default 'ops',
  created_at timestamptz not null default now()
);

create table if not exists clients(
  id bigserial primary key,
  org_id bigint not null references organizations(id) on delete cascade,
  prefix text,
  name text not null,
  email text,
  phone text,
  cell text,
  company text,
  discipline text,
  note text,
  notes text,
  created_at timestamptz not null default now()
);

create table if not exists followups(
  id bigserial primary key,
  client_id bigint not null references clients(id) on delete cascade,
  owner_id uuid not null references profiles(id) on delete set null,
  summary text not null,
  content text,
  status text not null default 'open' check (status in ('open','done')),
  due_date date,
  created_at timestamptz not null default now()
);

create index if not exists clients_name_trgm_idx on clients using gin (name gin_trgm_ops);
create unique index if not exists clients_org_prefix_uniq on clients(org_id, prefix) where prefix is not null;
create unique index if not exists clients_org_name_uniq   on clients(org_id, name);

-- Enable row-level security (RLS)
alter table profiles  enable row level security;
alter table clients   enable row level security;
alter table followups enable row level security;

-- Profiles RLS policies
drop policy if exists profile_self_read    on profiles;
drop policy if exists profile_org_read     on profiles;
drop policy if exists profile_org_write    on profiles;

create policy profile_self_read on profiles
for select using (auth.uid() = id);

create policy profile_org_read on profiles
for select using (
  exists (select 1 from profiles me where me.id = auth.uid() and me.org_id = profiles.org_id)
);

create policy profile_org_write on profiles
for update using (
  exists (
    select 1 from profiles me where me.id = auth.uid() and me.org_id = profiles.org_id and me.role = 'admin'
  )
);

-- Clients RLS policies
drop policy if exists clients_org_read  on clients;
drop policy if exists clients_org_write on clients;

create policy clients_org_read on clients
for select using (
  exists (select 1 from profiles me where me.id = auth.uid() and me.org_id = clients.org_id)
);

create policy clients_org_write on clients
for insert with check (
  exists (select 1 from profiles me where me.id = auth.uid() and me.org_id = clients.org_id)
)
for update using (
  exists (select 1 from profiles me where me.id = auth.uid() and me.org_id = clients.org_id)
);

-- Follow-ups RLS policies
drop policy if exists fus_org_read  on followups;
drop policy if exists fus_org_write on followups;

create policy fus_org_read on followups
for select using (
  exists (
    select 1 from profiles me join clients c on c.id = followups.client_id
    where me.id = auth.uid() and me.org_id = c.org_id
  )
);

create policy fus_org_write on followups
for insert with check (
  exists (
    select 1 from profiles me join clients c on c.id = followups.client_id
    where me.id = auth.uid() and me.org_id = c.org_id
  )
)
for update using (
  exists (
    select 1 from profiles me join clients c on c.id = followups.client_id
    where me.id = auth.uid() and me.org_id = c.org_id
  )
);
```

### Seeding Example

Here’s how to insert an organisation and map Supabase Auth users to the `profiles` table (replace placeholders with real values):

```sql
insert into organizations (name) values ('Venmed')
on conflict (name) do update set name=excluded.name;

-- Map an auth user (replace IDs and email)
insert into profiles (id, org_id, full_name, email, role)
values ('43d5f4a1-03c1-4994-8f3d-58f1d21a8857',
        (select id from organizations where name='Venmed'),
        'Rinus Burke','rinus@venmed.co.za','ops')
on conflict (id) do update set org_id=excluded.org_id, full_name=excluded.full_name, email=excluded.email, role=excluded.role;
```

## 4. Import Specifications

### 4.1 Clients Import (CSV)

The importer accepts **two schemas**. A CSV is valid if it matches either schema.

1. **Schema A (legacy)**: `name, email, phone, company, notes`  
2. **Schema B (preferred)**: `prefix, name, email, cell, discipline, note`

Mapping details:

- `phone` and `notes` exist in schema A; `cell` and `note` in schema B will be mapped to `phone` and `notes` for compatibility.  
- `prefix` identifies clients via `clients.prefix` and is used to deduplicate imports.  
- Upsert rule:  
  - If `prefix` is present → upsert on `(org_id, prefix)`  
  - Else → upsert on `(org_id, name)`

API endpoint: `POST /app/import/ingest` with body `{ kind: 'clients', rows: [...] }`.

### 4.2 Follow-ups Import (CSV)

The follow‑ups importer accepts **two schemas**:

1. **Schema A (IDs)**: `client_id, owner_id, summary, content, due_date`  
2. **Schema B (Friendly)**: `client_name, owner_email, summary, content, due_date`

Resolution logic:

- For Schema B, `client_name` is looked up (case‑insensitive) in `clients.name` (same org).  
- `owner_email` is looked up in `profiles.email`.  
- New follow‑ups are inserted with `status='open'`.  
- Validate that `client_id`, `owner_id`, and `summary` exist; skip rows otherwise.

API endpoint: `POST /app/import/ingest` with body `{ kind: 'followups', rows: [...] }`.

### 4.3 Age Analysis Import (CSV)

This importer creates follow‑ups from an **Age Analysis CSV** (from medical billing). The importer detects key fields using header names:

- **Client name**: any column labelled `Account holder`, `Patient`, `Account name`, `Account holder name`, `Name`, or `name`.  
- **Amount** (optional): any column labelled `Total`, `Balance`, `Amount`, or `Outstanding`.  
- **Prefix** (optional): any column labelled `PREFIX`, `Prefix`, or `Code`.

Matching order for each row:

1. Try `clients.name ilike <name>` in the same org.  
2. If not matched, try `clients.prefix ilike <prefix>`.  
3. If still not matched and name exists, derive initials from the name (last two letters) and match `prefix` by those initials.

If a client is matched, a follow‑up is created with:

- `summary = 'Age analysis balance <amount>'` (amount may be blank)  
- `content = 'Imported from Age Analysis CSV for "<name or prefix>" on <date>'`  
- `due_date = today + 7 days`  
- `owner_id = current user`  
- `status = 'open'`

API endpoint: `POST /app/import/age-csv` with body `{ rows: [...] }`.

## 5. UI Overview

### `/app/clients`

Displays a list of clients showing:

- `prefix` (e.g., FS),  
- `name`,  
- `discipline`,  
- `email`,  
- `cell` or `phone`,  
- `note` or `notes`.  

There are buttons to add new clients and import CSVs.

### `/app/followups`

Shows follow‑ups with fields:

- `summary`,  
- `client_name`,  
- `owner_name`,  
- `due_date`,  
- `status`.  

A button allows adding new follow‑ups; there’s also an import button.

### `/app/import`

The importer page includes three options:

- **Clients (CSV)**: Accepts Schema A or Schema B. It shows a ✓ once the header matches either schema.  
- **Follow‑ups (CSV)**: Accepts Schema A (IDs) or Schema B (friendly names/emails).  
- **Age Analysis (CSV)**: Detects name/amount/prefix columns automatically.  

After selecting a file, the page shows detected columns and clarifies which schema is matched. Pressing **Import** sends the data to the appropriate API route.

## 6. Troubleshooting

- **Login fails with `fetch failed`**: Check that `.env.local` contains correct `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY`. After editing `.env.local`, restart the dev server.  
- **“Cookies can only be modified…” errors**: Ensure that calls which mutate cookies (like login/logout) run inside Server Actions or Route Handlers. Our code already handles this by splitting read‑only and mutating Supabase clients.  
- **RLS/policy errors**: Re‑run the SQL above. Make sure `profiles.org_id` matches the organisation of the logged‑in user.  
- **Import blocked by header mismatch**: Your CSV must match one of the accepted schemas. Ensure headers are exact (case‑insensitive).  
- **Age CSV import matches zero clients**: Ensure clients have been imported first and have correct `prefix` or names. Names are matched case‑insensitively; prefixes match exactly.

## 7. Next Steps / Improvements

1. **CRUD UI**: Add editing capabilities for clients and follow‑ups.  
2. **Filtering & Search**: Add search and filter by status, due date, owner, or client.  
3. **Email reminders**: Implement scheduled reminders for due follow‑ups. This requires SMTP configuration.  
4. **PDF uploader**: (Optional) Add an uploader and parser for PDF age analysis files via Supabase Storage.  
5. **Deployment**: Deploy to Vercel. In Vercel’s dashboard, set environment variables to match `.env.local`.  
6. **Security Audit**: Review the RLS policies thoroughly and test cross‑organisation isolation.

---

### Brief for the Coding Specialist

You’re inheriting a Next.js 15 + Supabase project designed to manage client follow‑ups. The core features (clients, follow‑ups, CSV importers, and RLS policies) are in place, but there are a few finishing touches and validations to implement.

Your tasks:

1. **Verify and finalise the two API routes** for CSV import: `ingest/route.ts` (Clients A/B, Follow‑ups A/B) and `age-csv/route.ts` (Age CSV → follow‑ups). Ensure proper error handling and comprehensive header detection.  
2. **Ensure the importer page** correctly validates header schemas (A or B) and sends data to the right endpoint.  
3. **Confirm all RLS policies** (see Section 3) are active and test that users cannot see or modify data outside their organisation.  
4. **Polish the UI**: make lists sortable or searchable and add edit/delete actions.  
5. **Write integration tests** for the importers using Supabase’s testing harness or end‑to‑end tests with Playwright.  

With this guide, you should be able to finish the project swiftly.