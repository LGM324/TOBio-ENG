# TOBio login and records setup (Supabase, from scratch)

Login/sign-up and "내 기록" (My records) in this React app (microbe-ai-landing) are handled by **Supabase**. The backend (Express/Render) and its existing 6 endpoints are **not touched**.

> ⚠️ This repository **does not include** a Supabase project or database. You must create **a new project from scratch** with the steps below.

---

## 0. Environment variables (frontend, `VITE_` only)

Create and fill in `microbe-ai-landing/.env` (never commit `.env`; it is already in .gitignore).

```
VITE_SUPABASE_URL=https://<project>.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=<publishable (anon) key>
```

- If the values are empty, only login/records are disabled; recommendation and spraying keep working (login is optional).
- ⚠️ **Never put secret keys** such as `service_role` in the frontend or in commits. This app uses only the publishable (anon) key.

---

## 1. Create a Supabase project
1. https://supabase.com → New project (pick a nearby region, e.g. Northeast Asia).
2. After it is created, copy the **Project URL** and the **anon/publishable key** from **Project Settings → API** into the `.env` above.

## 2. Table + RLS (run as is in the SQL Editor)

```sql
-- farming records (saved recommendation/spray results)
create table if not exists public.records (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null default auth.uid() references auth.users(id) on delete cascade,
  kind       text not null check (kind in ('recommend','spray')),
  title      text,
  crop       text,
  summary    text,
  payload    jsonb,
  created_at timestamptz not null default now()
);

-- RLS so users see only their own rows
alter table public.records enable row level security;

create policy "records_select_own" on public.records
  for select using (user_id = auth.uid());
create policy "records_insert_own" on public.records
  for insert with check (user_id = auth.uid());
create policy "records_update_own" on public.records
  for update using (user_id = auth.uid()) with check (user_id = auth.uid());
create policy "records_delete_own" on public.records
  for delete using (user_id = auth.uid());

create index if not exists records_user_created_idx
  on public.records (user_id, created_at desc);
```

- `user_id` defaults to `auth.uid()`, so the frontend **does not send** `user_id`. Even if someone else's id is sent by mistake, `with check` rejects it in the database.

## 3. Authentication options
- Turn on **Authentication → Providers → Email** (on by default).
- **Email confirmation (Confirm email)**:
  - **On (recommended, secure)**: after sign-up, the user must click the link in the confirmation email before logging in. The app shows a "확인 메일을 보냈어요" (We sent a confirmation email) notice.
  - **Off (convenient for demos)**: users are logged in immediately after sign-up. Toggle it at Authentication → Providers → Email → "Confirm email".
- **Authentication → URL Configuration**:
  - Add the GitHub Pages address to **Site URL** and **Redirect URLs** (e.g. `https://minseok09011.github.io/Microbe_recommend_website/`). Also add `http://localhost:5173` for local development.

---

## 4. How it works (per-user separation)
1. On login, the Supabase session (JWT) is stored in the browser (localStorage) and persists across reloads.
2. When `records` are read or saved, RLS uses `auth.uid()` from the JWT to allow **only the user's own rows**.
3. So even `select * from records` returns only the user's own rows from the database (no reliance on frontend filtering).

## 5. Secret key cautions
- The frontend gets only the publishable (anon) key. Never put the `service_role` key in the frontend or in git.
- In this scope the service_role key is **not needed** (sign-up is also handled client-side with `signUp`).
