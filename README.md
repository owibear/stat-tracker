# Baseline

A personal tracker for goals, savings, habits, hours and sleep. Runs as an installable
app on a laptop and an iPhone from the same code, keeps working with no signal, and syncs
between devices through a Supabase project you own.

---

## 1. Put it online (10 minutes, free)

Everything here is static, so any static host works. GitHub Pages:

1. Create a repository and upload every file in this folder to the root.
2. Settings → Pages → Source: `main`, folder `/ (root)`. Save.
3. Wait a minute, then open `https://<your-user>.github.io/<repo>/`.

Netlify or Cloudflare Pages work the same way — drag the folder onto their deploy page.

**HTTPS is required.** Service workers, install prompts and notifications are all
disabled over plain `http://`. GitHub Pages gives you HTTPS automatically.

### Files

| File | Why it's there |
|---|---|
| `index.html` | The whole app. No build step, no dependencies. |
| `manifest.webmanifest` | Makes it installable and sets the name and icons. |
| `sw.js` | Service worker. Caches the shell so it opens offline. Never caches API calls. |
| `icon-192.png`, `icon-512.png` | Standard app icons. |
| `icon-maskable.png` | Android's icon mask crops aggressively; this one has padding for it. |
| `apple-touch-icon.png` | iOS home screen icon. |

---

## 2. Install it

**Laptop (Chrome or Edge):** open the URL, then the install icon in the address bar,
or ⋮ → Cast, save and share → Install page as app.

**iPhone (must be Safari):** open the URL → Share → Add to Home Screen. Launch it from
the home screen icon, not from Safari — only then does it run fullscreen, get its own
storage, and become eligible for notifications.

> iOS quirk worth knowing: an installed PWA keeps its data separately from Safari.
> Anything you logged in the Safari tab before installing will not appear in the
> installed app. Export from the tab and import into the app once, and you're aligned.

---

## 3. Turn on sync (optional, free)

Sync uses one row of JSON in your own Supabase project. There is no server code to write.

1. Create a project at supabase.com (free tier is plenty).
2. Open the SQL editor and run this:

```sql
create table public.baseline_state (
  user_id    uuid primary key references auth.users on delete cascade,
  data       jsonb not null default '{}',
  updated_at timestamptz not null default now()
);

alter table public.baseline_state enable row level security;

create policy "own row: read"   on public.baseline_state
  for select using (auth.uid() = user_id);
create policy "own row: insert" on public.baseline_state
  for insert with check (auth.uid() = user_id);
create policy "own row: update" on public.baseline_state
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

3. Authentication → Providers → Email: enabled. For a single-user app, turn
   **Confirm email** off so you can sign in immediately.
4. Settings → API: copy the **Project URL** and the **anon public** key.
5. In Baseline: Settings → Sync across devices → paste both → Create account with any
   email and password → Sign in. Repeat on the other device with the same login.

The anon key is designed to be public. Row-level security is what protects the data:
each row is readable only by the account that owns it.

### How the sync behaves

- Local storage is the source of truth. The app never blocks on the network.
- It syncs on launch, four seconds after any change, when the connection returns, and
  whenever you bring the app back to the foreground.
- Merging is per-record, not last-file-wins, so logging on your phone while your laptop
  is asleep does not overwrite anything. Trackers and entries merge by id; badges union.
- Deletions are recorded as tombstones, so removing a tracker on one device removes it
  on the other instead of having it resurrect on the next sync.

---

## 4. The AI chat in Updates (optional)

The Updates tab computes everything locally — pace, projections, week-over-week change,
which goal is starving the others. That needs no key and no connection.

To also get the chat: console.anthropic.com → API keys → create one → paste it into
Settings → AI chat. It's stored only on that device, and calls go straight from the
browser to Anthropic. Usage is a few cents a month at normal chat volume.

If you'd rather the key not sit on your phone, the cleaner version routes calls through
a Supabase Edge Function so the key lives server-side. Worth doing if you ever share the
app with anyone.

---

## 5. Notifications

Settings → Daily check-in. Baseline asks the browser for permission, then fires one
local notification a day at your chosen time, naming whatever is furthest behind.

On iPhone this only works from the installed home-screen app, on iOS 16.4 or newer.
It is a local notification, so it fires when the app is open or recently backgrounded.
Guaranteed background delivery needs a push service, which needs a server — a later
project if you want it.

---

## 6. Your data

Settings → Your data → Export writes a JSON file with every tracker, entry, badge and
tombstone. Worth doing occasionally even with sync on: browser storage can be cleared by
the OS under storage pressure, and a file on disk can't be.
