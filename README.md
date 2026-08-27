# JobsPilot Singapore

A production-ready recruitment website for Singapore, built with React + Vite,
styled with vanilla CSS, and deployed on **Cloudflare Pages** with a Cloudflare
Pages Function backend (`/api/apply`) that writes applications to Supabase.

## Stack

- React 18 + Vite 5 + React Router
- Vanilla CSS (no Tailwind/UI kit)
- Cloudflare Pages (static hosting)
- Cloudflare Pages Functions (`/api/apply`)
- Supabase (Postgres) for storing applications

---

## ⚠️ Deployment platform: Pages, not Workers

This project is a **static frontend (Vite build) + a couple of small serverless
API routes** (`functions/api/apply.js`). That is exactly what **Cloudflare
Pages** is designed for: it builds your `dist/` folder, serves it as static
assets, and automatically turns anything under `functions/` into Pages
Functions (Cloudflare Workers running at the edge, wired up by file-based
routing) with zero extra config.

**Cloudflare Workers** (a raw Worker with an `index.js`/`fetch` handler and an
`[assets]` binding) is the right call when you're building a single
Worker-native app from scratch or need advanced Worker-only bindings (Durable
Objects, Queues, etc.). Migrating this project to that model would mean
rewriting `functions/api/apply.js` into a manual routed `fetch` handler and
configuring an assets binding for the whole `dist/` folder — a real
restructure, for no benefit here. **Pages is the correct, simpler choice for
this project as-is**, and it already supports everything the project needs.

### Why the dashboard upload failed

Cloudflare Pages actually offers **two different upload paths**, and the
error you hit is specific to one of them:

| Path | What it does | Works for this project? |
|---|---|---|
| **"Upload assets" (drag-and-drop)** | Takes a folder of *already-built* static files, as-is. It never runs `npm install` or `npm run build`, and it rejects any folder containing a `wrangler.jsonc`/`wrangler.toml` because that usually signals the project needs a build step it can't perform. | ❌ This is the flow that gave you the error. |
| **"Connect to Git"** | Clones your GitHub repo on every push and runs your real build command (`npm run build`) in Cloudflare's own build environment, then deploys the output plus your `functions/` folder. | ✅ This is what you want. |

Nothing in the project needed to be removed to "fix" the uploader — the fix
is simply to use **Connect to Git** (recommended) or the **Wrangler CLI**
(`wrangler pages deploy dist`) instead of the drag-and-drop uploader. Both
paths are documented below.

---

## Option A (recommended): Deploy via GitHub — "Connect to Git"

This is the easiest path and needs no local build step or CLI on your machine.

1. Push this project to a GitHub repository (the whole folder, including
   `functions/`, `public/`, `wrangler.jsonc`, etc. — no restructuring needed).
2. In the Cloudflare dashboard, go to **Workers & Pages → Create application →
   Pages → Connect to Git**.
3. Select your repository.
4. In **Build settings**, set:
   - **Framework preset:** `Vite`
   - **Build command:** `npm run build`
   - **Build output directory:** `dist`
5. Under **Settings → Environment variables**, add for both **Production**
   and **Preview**:
   - `SUPABASE_URL`
   - `SUPABASE_ANON_KEY`
6. Click **Save and Deploy**.

Cloudflare will install dependencies, run the build, publish `dist/` as your
site, and automatically detect `functions/api/apply.js` and deploy it as a
Pages Function available at `https://<your-project>.pages.dev/api/apply`.
Every future push to your connected branch redeploys automatically.

## Option B: Deploy via Wrangler CLI (no Git required)

Useful for a one-off deploy or testing before you set up Git integration.

```bash
npm install
npm run build
npx wrangler login          # one-time browser auth
npx wrangler pages deploy dist --project-name=jobspilot-singapore
```

Or, once dependencies are installed, just run:

```bash
npm run deploy
```

(This runs `npm run build && wrangler pages deploy dist` in one step — see
the `deploy` script in `package.json`.)

Set the environment variables for this project afterwards under
**Workers & Pages → jobspilot-singapore → Settings → Environment variables**
(same two variables as above), or pass them ahead of time with:

```bash
npx wrangler pages secret put SUPABASE_URL --project-name=jobspilot-singapore
npx wrangler pages secret put SUPABASE_ANON_KEY --project-name=jobspilot-singapore
```

### What NOT to do

Do not use the Cloudflare dashboard's **"Upload assets"** drag-and-drop
uploader for this project — it cannot run a build step, which is exactly the
error you saw. Use Option A or Option B above instead.

---

## Local development

```bash
npm install
npm run dev
```

The app runs at `http://localhost:5173`. The `/api/apply` endpoint only runs
in the Cloudflare (Pages Functions) runtime, so to test it locally, build
first and use Wrangler's local dev server:

```bash
npm run build
npx wrangler pages dev dist
```

## Build

```bash
npm run build
```

Vite outputs the static production build to `dist/` (configured in
`vite.config.js`). This is the exact folder Cloudflare Pages serves and the
exact folder passed to `wrangler pages deploy`.

## Supabase setup

1. Create a Supabase project.
2. Create a `job_applications` table:

```sql
create table job_applications (
  id uuid primary key default gen_random_uuid(),
  full_name text not null,
  phone text not null,
  whatsapp text not null,
  age int4 not null,
  gender text not null,
  occupation text not null,
  created_at timestamptz not null default now()
);

alter table job_applications enable row level security;

create policy "Allow inserts from anon key"
  on job_applications for insert
  to anon
  with check (true);
```

3. Copy your **Project URL** and **anon public API key** from
   Supabase → Settings → API.
4. Set them as `SUPABASE_URL` and `SUPABASE_ANON_KEY` environment variables in
   Cloudflare Pages (never commit real values — see `.env.example`).
   Credentials are never hardcoded anywhere in this project;
   `functions/api/apply.js` reads them only from `env.SUPABASE_URL` /
   `env.SUPABASE_ANON_KEY` at request time.

---

## Project structure

```
jobspilot/
├── functions/
│   └── api/
│       └── apply.js        # POST /api/apply → Supabase insert (Pages Function)
├── public/
│   ├── _headers            # security + caching headers
│   ├── _redirects          # SPA routing fallback
│   ├── favicon.svg
│   ├── robots.txt
│   └── sitemap.xml
├── src/
│   ├── components/          # Header, Hero, Stats, FeaturedJobs, WhyChoose,
│   │                         # ApplySection, ApplicationForm, FAQ, Footer, ScrollToTop
│   ├── pages/                # Home, Privacy, Terms
│   ├── styles/                # tokens, base, buttons, header, hero, stats,
│   │                         # jobs, why, apply, faq, footer, legal
│   ├── App.jsx
│   └── main.jsx
├── index.html
├── package.json
├── vite.config.js
├── wrangler.jsonc            # Pages build/output config (used by CLI + Git builds)
├── .env.example
├── .gitignore
└── README.md
```

## Notes

- The application form validates Singapore-format phone/WhatsApp numbers,
  age (16–75), gender and occupation client-side before submitting to
  `/api/apply`, which re-validates server-side.
- `_redirects` is configured so client-side routing (`/privacy-policy`,
  `/terms-of-service`) works correctly on Cloudflare Pages.
- `wrangler.jsonc` is read by the Wrangler CLI and Git-connected Pages
  builds — not by the dashboard's drag-and-drop asset uploader (see
  the deployment section above).
