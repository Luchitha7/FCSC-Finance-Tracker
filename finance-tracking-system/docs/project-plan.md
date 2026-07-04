# Wiramaya 2026 Sponsor Outreach Tracker — Project Plan

Team: **Luchitha**, **Sahan**, **Ruwaneke**
Source design: Claude Design handoff (`Sponsor Tracker.dc.html`) — nautical/gold theme, dark-background dashboard replacing the old Google Sheet.

---

## 1. The basics (read this first)

You're going from a static HTML mockup to a real multi-user web app. Three things need to exist that don't today:

1. **A real database** — right now the mockup saves everything to the browser's `localStorage`. That means only the person who typed the data sees it, and it's gone if they clear their browser. We need one shared database on the internet that everyone's app talks to.
2. **Real logins** — right now "owner" is just a text dropdown, not an account. We need actual accounts for Luchitha, Sahan, and Ruwaneke so the app knows who's logged in.
3. **A server-connected frontend** — the Next.js app needs to fetch/save data from that database instead of `localStorage`.

**Why Supabase.** Supabase is a company that gives you a free hosted Postgres database, plus a login system (Auth), plus live update pushes (Realtime), all bundled with a dashboard and a JS library that plugs straight into Next.js. Concretely:

- You create a free project at supabase.com → you get a private Postgres database + a URL + an API key.
- You paste a block of SQL (given below) into their "SQL Editor" to create your tables. That's your schema — done once.
- You create 3 login accounts (you, Sahan, Ruwaneke) in their Auth tab.
- In the Next.js app, you install `@supabase/supabase-js` and `@supabase/ssr`, put the project URL + key into `.env.local`, and then any page/component can ask Supabase for data (`supabase.from('companies').select()`) instead of reading `localStorage`.
- Because it's Postgres, filtering ("show me Pending companies owned by Sahan") is just a normal database query — no custom backend server needed.

**Key vocabulary you'll see below:**
- **Table** — like a spreadsheet tab (e.g. `companies`).
- **Column** — like a spreadsheet column (e.g. `status`).
- **Primary key (`id`)** — the unique ID for each row.
- **Foreign key** — a column that points to a row in another table (e.g. each activity log entry points to the company it belongs to).
- **RLS (Row Level Security)** — a rule you write once telling Postgres "only logged-in committee members may read/write this table." Without it, your database would be open to the entire internet.

---

## 2. Database schema (Postgres / Supabase)

Run this once in the Supabase SQL Editor, in order. This is the *entire* schema for the app — 3 tables.

```sql
-- 1. Committee member profiles (mirrors Supabase's built-in auth.users table)
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text not null,
  created_at timestamptz not null default now()
);

-- Auto-create a profile row whenever someone signs up
create function handle_new_user() returns trigger as $$
begin
  insert into profiles (id, full_name)
  values (new.id, coalesce(new.raw_user_meta_data->>'full_name', new.email));
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure handle_new_user();

-- 2. Companies
create table companies (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  phones text[] not null default '{}',
  emails text[] not null default '{}',
  owner_id uuid references profiles(id),
  status text not null default 'Pending'
    check (status in ('Answered', 'Not Answered', 'Rejected', 'Pending')),
  proposal_channel text
    check (proposal_channel in ('Email', 'WhatsApp') or proposal_channel is null),
  next_action_text text,
  next_action_date date,
  is_new boolean not null default true,
  is_positive boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index companies_owner_idx on companies(owner_id);
create index companies_status_idx on companies(status);
create index companies_next_action_idx on companies(next_action_date);

-- keep updated_at fresh on every edit
create function set_updated_at() returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger companies_set_updated_at
  before update on companies
  for each row execute procedure set_updated_at();

-- 3. Activity log (the dated call-notes history, replacing the old one-column-per-day sheet)
create table activity_log (
  id uuid primary key default gen_random_uuid(),
  company_id uuid not null references companies(id) on delete cascade,
  author_id uuid references profiles(id),
  note text not null,
  created_at timestamptz not null default now()
);

create index activity_log_company_idx on activity_log(company_id);

-- ---- Row Level Security ----
-- Internal team tool: any logged-in committee member can read/write everything,
-- same trust level as the old shared Google Sheet. No public/anonymous access.
alter table profiles enable row level security;
alter table companies enable row level security;
alter table activity_log enable row level security;

create policy "profiles readable by team" on profiles
  for select using (auth.role() = 'authenticated');

create policy "companies readable by team" on companies
  for select using (auth.role() = 'authenticated');
create policy "companies writable by team" on companies
  for insert with check (auth.role() = 'authenticated');
create policy "companies updatable by team" on companies
  for update using (auth.role() = 'authenticated');
create policy "companies deletable by team" on companies
  for delete using (auth.role() = 'authenticated');

create policy "log readable by team" on activity_log
  for select using (auth.role() = 'authenticated');
create policy "log writable by team" on activity_log
  for insert with check (auth.role() = 'authenticated');
```

Notes:
- `phones`/`emails` are Postgres arrays — matches the design's "can hold multiple numbers/emails" requirement directly.
- `is_new` / `is_positive` are the flags that put a company into the "New Companies" / "Positive" tabs.
- The "needs follow-up today" view is just `where next_action_date <= today and status != 'Rejected'` — no extra table needed.
- No separate "members" table for the picklist — `profiles` (backed by real logins) replaces it.

---

## 3. Full task list, in build order

### Phase 0 — Accounts & setup
- [ ] Create the Supabase project (free tier).
- [ ] Run the schema SQL above in the SQL Editor.
- [ ] In Supabase Auth, invite/create 3 accounts: Luchitha, Sahan, Ruwaneke.
- [ ] Copy the project URL + anon key into `.env.local` in `finance-tracking-system/`.
- [ ] `npm install @supabase/supabase-js @supabase/ssr` in the Next.js project.

### Phase 1 — Data layer
- [ ] Supabase client helper (browser client + server client per `@supabase/ssr` docs).
- [ ] Login page (email/password or magic link) + logged-out redirect.
- [ ] Typed data-access functions: `listCompanies()`, `createCompany()`, `updateCompany()`, `deleteCompany()`, `addLogEntry()`.

### Phase 2 — Main list view (biggest chunk)
- [ ] Header (logo, search box, theme toggle, Export CSV, Members, Add Company buttons).
- [ ] Dashboard stat cards (total, per-status counts, proposals by channel, follow-ups due).
- [ ] Tabs: All Companies / New Companies / Positive-Promising / Follow-up Today.
- [ ] Filters (owner, status, proposal) + sort + search — all as real DB queries.
- [ ] Table with inline-editable owner/status/proposal, quick-log input, row select + bulk delete.

### Phase 3 — Detail & modals
- [ ] Detail slide-over: owner/status/proposal editors, next-action date+text, full activity-log timeline, composer to add a new log entry.
- [ ] "Mark promising" / "Mark as new" toggle buttons.
- [ ] Add Company modal.
- [ ] Manage Committee Members view (now just lists the 3 profiles + counts, no add/remove since membership = real accounts).
- [ ] CSV export of the current filtered view.

### Phase 4 — Theme & polish
- [ ] Port the nautical/gold CSS theme tokens (light + dark) from the mockup into the real components.
- [ ] Mobile layout pass (table → responsive cards or horizontal scroll).

### Phase 5 — Data migration & deploy
- [ ] Migrate the existing Google Sheet rows into `companies` (one-time import).
- [ ] Deploy to Vercel, set env vars there.
- [ ] Real-device test: each person logs in on their own phone and laptop.

---

## 4. Who does what

**Sahan + Luchitha — everything technical:**
- Luchitha: Phase 0 (Supabase setup, schema, auth accounts), Phase 1 (data layer), Phase 3 (detail panel + modals + CSV).
- Sahan: Phase 2 (main table/dashboard/filters — the largest surface), Phase 4 (theme port), Phase 5 (Vercel deploy).
- Adjust the split between yourselves as you go — these two phases are independent enough to build in parallel once Phase 0/1 land.

**Ruwaneke — simple, no-setup-required tasks:**
1. **Clean up the sponsor contact list.** Open the old Google Sheet and put every company's name, phone number(s), email(s), and current owner into a simple spreadsheet (one row per company, plain columns — no formulas needed). This becomes the data we import in Phase 5.
2. **Manual testing once there's something to click.** Open the deployed app link on a phone and on a laptop, click through every button, and note anything broken, confusing, or misspelled in a shared doc/chat.
3. *(Optional, if Ruwaneke wants a small coding task)*: build the static header bar (logo + "WIRAMAYA 2026 / Sponsor Outreach Tracker" text) as a single React component, copying the exact markup/styling from `Sponsor Tracker.dc.html` lines 67–104 — no logic, just matching the look.

No database, auth, or state-management work goes to Ruwaneke — those all sit with Sahan/Luchitha.
