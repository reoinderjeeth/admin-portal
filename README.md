# Admin Portal — Maintenance Platform

Single-file HTML admin portal that connects directly to Supabase.

## Deploy to GitHub Pages (free)

1. Create a new GitHub repository (public or private)
2. Push the `admin_portal/` folder:
   ```
   git init
   git add admin_portal/
   git commit -m "Add admin portal"
   git remote add origin https://github.com/YOUR_USER/YOUR_REPO.git
   git push -u origin main
   ```
3. Go to repo **Settings → Pages**
4. Under "Branch", select `main` and folder `/docs` or `/ (root)`
5. Save — your portal will be live at `https://YOUR_USER.github.io/YOUR_REPO/admin_portal/`

### Minimal option (single file)
Upload just `index.html` to any static host — Netlify, Vercel, or even Supabase Storage static hosting. No build step required.

## Login

Sign in with an admin account (email + password). The portal checks the `role` field in `profiles` table — only users with `role = 'admin'` can access.

## Required database migration

The **Messages**, **Calls** and **Notifications** pages read tables that are
restricted to their own participants by row-level security. Admins have no
read access to them until this migration is applied:

```
supabase/migrations/037_admin_read_all.sql
```

Run it once in the **Supabase dashboard → SQL Editor**. It is idempotent
(safe to re-run) and adds:

- `SELECT` policies for admins on `messages`, `calls`, `notifications`, `device_tokens`
- the missing `calls.caller_name` and `calls.call_type` columns
- supporting indexes for the admin listings

If the migration has not been applied, those pages show a "run migration
037" notice instead of silently appearing empty.

## Pages

| Page | Contents |
| --- | --- |
| Dashboard | Headline counts incl. messages, calls, missed calls, average rating, revenue |
| Users | All profiles with role badges |
| Businesses | Verification queue, logos, media counts |
| Jobs | All jobs with a **View** button opening a per-job detail modal |
| Messages | Every chat transcript, grouped per job, with search + job filter |
| Images | All job photos, quote photos, business logos, galleries and certificates |
| Quotes | All quotes with totals and photo counts |
| Invoices | All invoices with tax, total and paid state |
| Calls | Call history with talk-time summary and per-call duration |
| Reviews | Ratings and written feedback |
| Notifications | Full notification log with recipient and read state |
| Categories | Service category management |

The **View** modal on a job is the fastest way to audit one job end to end:
overview, messages, quotes, invoices, calls and photos in separate tabs.
