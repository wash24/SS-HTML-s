# Turf Legends: live multiplayer on your own site (Supabase)

Inside the claude.ai artifact, multiplayer works on its own: anyone you share the artifact with shows up live. These steps are only for the copy you host yourself, such as GitHub Pages. With them, anyone with the link can play together.

The free Supabase plan is enough.

## 1. Create a project
1. Go to https://supabase.com, sign in, and click **New project**.
2. Pick any name (e.g. `turf-legends`) and a database password, then wait for it to finish setting up.

## 2. Turn on anonymous sign-ins
Players don't make accounts. Each browser gets an anonymous ID automatically.

**Authentication → Sign In / Providers** (called **Providers** in some versions) → turn on **Allow anonymous sign-ins** → Save.

## 3. Create the players table
Open **SQL Editor → New query**, paste this, and click **Run**:

```sql
create table if not exists public.players (
  id uuid primary key references auth.users(id) on delete cascade,
  data jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table public.players enable row level security;

-- anyone signed in can read profiles (player cards, Home Turf visits)
create policy "players readable" on public.players
  for select to authenticated using (true);

-- each player can only create/update their own row
create policy "players insert own" on public.players
  for insert to authenticated with check (auth.uid() = id);

create policy "players update own" on public.players
  for update to authenticated using (auth.uid() = id) with check (auth.uid() = id);
```

This table stores each player's name, look, level, stats and Home Turf layout. Live positions and chat don't touch the database: they go through Supabase Realtime (presence), which is on by default.

## 4. Paste your keys into the game
1. In Supabase, open **Project Settings → API** (or **API Keys**). Copy the **Project URL** and the **anon / publishable** key. Never use the `service_role` / secret key.
2. In `index.html`, find this line near the multiplayer section:

```js
const NET_CFG={supabaseUrl:'',supabaseAnonKey:''};
```

and fill it in:

```js
const NET_CFG={supabaseUrl:'https://YOUR-PROJECT.supabase.co',supabaseAnonKey:'YOUR-ANON-OR-PUBLISHABLE-KEY'};
```

The anon/publishable key is meant to be public. The row-level security policies above are what protect the data.

## 5. Deploy and test
Push `index.html` to your GitHub Pages repo. Open the page in two different browsers, or one normal window and one private window.

The pill at the top-right should say **🟢 1 player online** in each window, and you should see each other walk around and chat.

## Troubleshooting
- **Stuck on "🟡 Connecting…" or "🤖 Solo mode":** open the browser console (F12) and look for `[Turf Legends]` messages. The usual causes are anonymous sign-ins being off (step 2) or a typo in the URL or key.
- **Players see each other, but "Visit Turf" says it couldn't load:** the SQL in step 3 wasn't run, or a policy is missing.
- **Chat moderation:** chat goes through a basic word filter only. If you open the game to strangers, you'll want stronger moderation (and possibly a Safe-Chat-only mode).
