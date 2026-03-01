# FreeJobBoard.ai — MVP Architecture

## Philosophy
- Zero hosting cost/liability for us
- Board founders get a real product, not a toy
- Fast to build, fast to iterate
- Every feature decision asks: "does this help a board founder make money?"

---

## Tech Stack

### API Layer — Cloudflare Workers
- Zero cold starts, global edge, free tier is generous
- KV for fast reads (job listings, board config)
- D1 (SQLite) for relational data (jobs, employers, candidates, applications)
- R2 for file storage (resumes, logos)
- No server to manage. Ever.

### Frontend — Next.js (hosted on Vercel)
- `/app` — the board founder's public job board (dynamically themed)
- `/admin` — board founder's dashboard
- `/embed` — the embeddable widget script endpoint

### Auth — Clerk (or Supabase Auth)
- Board founder accounts
- Employer accounts (per board)
- Candidate accounts (per board)

### Payments — Stripe Connect
- Platform takes cut of app subscriptions
- Board founders get paid for featured listings via Stripe Connect

### Email — Resend
- Job alerts to candidates
- New applicant notifications to employers
- Onboarding sequences for board founders

---

## Data Model (D1 / SQLite)

```sql
-- Board founders
boards (
  id, slug, name, custom_domain,
  owner_id, logo_url, primary_color,
  theme, created_at
)

-- Jobs
jobs (
  id, board_id, employer_id, title, slug,
  company, location, job_type, salary_min, salary_max,
  description, apply_url, apply_email,
  status, featured, expires_at, created_at
)

-- Employers (per board)
employers (
  id, board_id, user_id, company_name,
  website, logo_url, verified, created_at
)

-- Candidates
candidates (
  id, board_id, user_id, name, email,
  resume_url, created_at
)

-- Applications
applications (
  id, job_id, candidate_id,
  status, created_at
)

-- Alert subscriptions
job_alerts (
  id, board_id, email, keywords,
  categories, created_at
)

-- Installed apps (per board)
board_apps (
  id, board_id, app_slug,
  stripe_subscription_id, active, created_at
)
```

---

## Embeddable Widget

The B2B killer feature. One script tag → job listings appear anywhere.

```html
<script
  src="https://freejobboard.ai/embed.js"
  data-board="yourboard"
  data-theme="light"
  data-limit="10">
</script>
```

**How it works:**
1. Script loads from our CDN (Cloudflare)
2. Creates a shadow DOM container (no CSS conflicts with host site)
3. Fetches jobs from our Workers API
4. Renders with host site's theme or custom colors
5. Apply links stay on our hosted board OR open in modal

**Widget config options:**
- `data-board` — board slug (required)
- `data-theme` — light / dark / auto
- `data-limit` — number of listings to show
- `data-category` — filter by category
- `data-color` — primary accent color hex

---

## MVP Scope (Beta Launch)

### Phase 1 — Board Creation (Week 1)
- [ ] Sign up → create board (name, slug, category)
- [ ] Choose subdomain: `{slug}.freejobboard.ai`
- [ ] Basic theme customization (logo, primary color)
- [ ] Public job board page (list + detail)
- [ ] Post a job (employer self-service)
- [ ] Apply flow (redirect to URL or native form)
- [ ] Admin dashboard (manage jobs, view applicants)

### Phase 2 — Growth Features (Week 2)
- [ ] Custom domain support
- [ ] Candidate profiles + resume upload
- [ ] Job seeker email alerts
- [ ] Embeddable widget
- [ ] Basic analytics (views, applies)
- [ ] SEO: sitemap, JSON-LD, clean URLs

### Phase 3 — Monetization (Week 3)
- [ ] App store UI
- [ ] Featured Listings app (Stripe)
- [ ] Resume Database app (Stripe)
- [ ] Advanced Analytics app
- [ ] Remove branding app

---

## Public Board URL Structure

```
# Hosted board
{slug}.freejobboard.ai/
{slug}.freejobboard.ai/jobs/[job-slug]
{slug}.freejobboard.ai/companies
{slug}.freejobboard.ai/post-a-job

# Custom domain (CNAME to our edge)
yournichejobs.com/
yournichejobs.com/jobs/[job-slug]

# Embed
freejobboard.ai/embed.js
```

---

## Admin Dashboard Routes

```
freejobboard.ai/dashboard
freejobboard.ai/dashboard/jobs
freejobboard.ai/dashboard/jobs/new
freejobboard.ai/dashboard/employers
freejobboard.ai/dashboard/candidates
freejobboard.ai/dashboard/analytics
freejobboard.ai/dashboard/alerts
freejobboard.ai/dashboard/apps
freejobboard.ai/dashboard/settings
freejobboard.ai/dashboard/settings/domain
freejobboard.ai/dashboard/settings/theme
```

---

## App Store — Day 1 Apps

| App | Slug | Price | Description |
|-----|------|-------|-------------|
| Featured Listings | featured-listings | $29/mo | Employers pay to pin to top |
| Resume Database | resume-database | $49/mo | Unlock candidate resumes |
| Advanced Analytics | advanced-analytics | $19/mo | Deep traffic + conversion data |
| Alert Campaigns | alert-campaigns | $29/mo | Blast alerts to all subscribers |
| Remove Badge | remove-badge | $9/mo | White-label footer link |

---

## Beta Onboarding Flow

1. Sign up (email + password)
2. "Name your board" → auto-generate slug
3. Pick your niche category (or custom)
4. Upload logo (optional, skip for now)
5. Pick accent color
6. → Dashboard (board is live immediately at {slug}.freejobboard.ai)
7. Prompt: "Post your first job" or "Invite an employer"

Goal: board live in under 2 minutes.

---

## Build Order (Most Value First)

1. **Auth + board creation** — nothing works without this
2. **Public job board** — the product people see
3. **Post a job** — the core action
4. **Admin dashboard** — manage everything
5. **Embeddable widget** — B2B differentiation
6. **Custom domains** — conversion unlock
7. **Email alerts** — retention
8. **App store** — revenue

---

## Repo Structure

```
freejobboard-app/
├── apps/
│   ├── web/          # Next.js (public boards + admin)
│   └── worker/       # Cloudflare Worker (API + embed)
├── packages/
│   ├── db/           # D1 schema + migrations
│   ├── email/        # Resend templates
│   └── types/        # Shared TypeScript types
└── README.md
```

Monorepo with Turborepo. Ship fast, share types between worker and web.

---

## Key Decisions

- **Subdomain routing**: `{slug}.freejobboard.ai` via Cloudflare wildcard — no extra config per board
- **Custom domains**: CNAME `jobs.yoursite.com → proxy.freejobboard.ai` → Worker reads host header → serves correct board
- **Shadow DOM for widget**: prevents CSS bleed into host site — mandatory for B2B embed
- **D1 over Supabase**: Cloudflare-native, zero latency from Workers, no cold start penalty
- **Free tier forever**: No feature gates on core — convert on apps only
