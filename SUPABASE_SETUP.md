# Supabase login and farming-record setup guide

Static frontend (HTML/JS) + Supabase (Auth + DB + RLS). This is completely separate from the recommendation/sequence features (Render backend); all authentication and records are handled by Supabase.

---

## 0. Fill in the connection keys

Fill in the two constants in `js/supabaseConfig.js`.

```js
const SUPABASE_URL = "https://<project>.supabase.co";
const SUPABASE_PUBLISHABLE_KEY = "sb_publishable_xxxxxxxx";   // publishable key (not anon)
```

> ⚠️ **Never put the secret key (`sb_secret_...`) here.** The frontend uses only the publishable key. The secret key is used only inside the Edge Function (server) below.

---

## 1. RLS policy SQL (run in Supabase → SQL Editor)

Run the whole script below as is. It is the core of keeping each farmer's records separate.

```sql
-- ============================================================
-- 0) admin check function (SECURITY DEFINER to avoid infinite RLS recursion)
--    Selecting from farmers inside a farmers policy recurses,
--    so a separate function bypasses RLS and checks only whether the user is admin.
-- ============================================================
create or replace function public.is_admin()
returns boolean
language sql
stable
security definer
set search_path = public
as $$
  select exists (
    select 1 from public.farmers
    where id = auth.uid() and role = 'admin'
  );
$$;

-- ============================================================
-- 1) farm_records : CRUD on own rows only
-- ============================================================
alter table public.farm_records enable row level security;

-- default so that farmer_id becomes the current user (auth.uid()) even if not sent on insert
alter table public.farm_records
  alter column farmer_id set default auth.uid();

drop policy if exists "records_select_own" on public.farm_records;
create policy "records_select_own" on public.farm_records
  for select using (farmer_id = auth.uid());

drop policy if exists "records_insert_own" on public.farm_records;
create policy "records_insert_own" on public.farm_records
  for insert with check (farmer_id = auth.uid());

drop policy if exists "records_update_own" on public.farm_records;
create policy "records_update_own" on public.farm_records
  for update using (farmer_id = auth.uid()) with check (farmer_id = auth.uid());

drop policy if exists "records_delete_own" on public.farm_records;
create policy "records_delete_own" on public.farm_records
  for delete using (farmer_id = auth.uid());

-- ============================================================
-- 2) farmers : read own row, admin reads all. Create/update by admin.
-- ============================================================
alter table public.farmers enable row level security;

drop policy if exists "farmers_select_self_or_admin" on public.farmers;
create policy "farmers_select_self_or_admin" on public.farmers
  for select using (id = auth.uid() or public.is_admin());

drop policy if exists "farmers_insert_admin" on public.farmers;
create policy "farmers_insert_admin" on public.farmers
  for insert with check (public.is_admin());

drop policy if exists "farmers_update_self_or_admin" on public.farmers;
create policy "farmers_update_self_or_admin" on public.farmers
  for update using (id = auth.uid() or public.is_admin())
  with check (id = auth.uid() or public.is_admin());

drop policy if exists "farmers_delete_admin" on public.farmers;
create policy "farmers_delete_admin" on public.farmers
  for delete using (public.is_admin());
```

> Even if the frontend mistakenly puts someone else's id in `farmer_id`, `with check (farmer_id = auth.uid())` makes the database **reject** the INSERT/UPDATE. So the frontend does not send farmer_id at all (it defaults to auth.uid()).

---

## 2. Create the first admin account

Because of RLS there is a chicken-and-egg problem: an admin is needed to create an admin row, but there is no first admin. Create the first admin directly with the **dashboard + SQL Editor** (server privileges bypass RLS).

1. Supabase → **Authentication → Users → Add user**
   - Email: `admin@farm.local` (if the username is `admin`)
   - Password: any password you choose; check **Auto Confirm User**
   - Copy the created user's **User UID** (uuid)
2. In the SQL Editor, insert the farmers profile row (using the copied UID):

```sql
insert into public.farmers (id, username, name, role)
values ('pasted-UID', 'admin', '관리자', 'admin');
```

Now log in at `login.html` with `admin` and the password. The dashboard shows a **🛠️ 관리자 (Admin)** button, and farmer accounts can be created from the screen from then on.

---

## 3. Deploy the Edge Function for creating farmer accounts

"계정 만들기" (Create account) in `admin.html` needs the Auth admin API, which requires the secret key. To keep the secret key out of the frontend, an **Edge Function (`create-farmer`)** handles it instead.

```bash
# after installing the Supabase CLI
supabase login
supabase link --project-ref <project-ref>

# deploy the function (code: supabase/functions/create-farmer/index.ts)
supabase functions deploy create-farmer
```

- `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY` are **injected automatically** into the Edge Functions runtime (no setup needed). If you need to manage secrets, set them with `supabase secrets set`.
- The function **first checks that the caller is an admin** using the caller's JWT, and only then creates the account.

> **Without the CLI (alternative b)**: if you do not use the Edge Function, an admin can add users directly in the Supabase dashboard (step 2) and insert farmers rows with SQL. In that case "계정 만들기" (Create account) in `admin.html` does not work, but the **farmer list** still does. (Deploying the function is recommended.)

---

## 4. Example test accounts

| Type | Username | Login email (internal) | Password | role |
|---|---|---|---|---|
| Admin | `admin` | `admin@farm.local` | (your choice) | admin |
| Farmer | `hong123` | `hong123@farm.local` | (your choice) | farmer |

- Farmer `hong123` can be created by the admin in `admin.html`.
- On the login screen, enter **only the username** (the code appends @farm.local automatically).

---

## 5. How records are kept separate per farmer

1. On login, a Supabase session is created and the user's JWT is attached to every DB request automatically.
2. The DB extracts `auth.uid()` (= that farmer's uuid) from the JWT.
3. `farmers.id` is set to **the same value as the Auth user id**, so the RLS rule `farm_records.farmer_id = auth.uid()` means "the farmer's own rows".
4. So even `select * from farm_records` returns **only the user's own rows from the DB**, and reading, updating, or deleting anyone else's rows is blocked at the source. It stays safe even if frontend code forgets a filter.

---

## 6. Secret key cautions (summary)

- The frontend (`js/*.js`, HTML) uses **only the publishable key**. Never `sb_secret_...`.
- The secret/service-role key is used **only inside the Edge Function runtime** (injected automatically).
- The service-role key bypasses RLS, so if it leaks, all data is exposed. Never commit it to git either.
