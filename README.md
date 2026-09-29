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

## Required database migrations

Two migrations are required. Both are idempotent (safe to re-run).

```
supabase/migrations/037_admin_read_all.sql
supabase/migrations/038_admin_row_counts.sql
```

Run them once in the **Supabase dashboard → SQL Editor**.

**037** grants admins `SELECT` on `messages`, `calls`, `notifications` and
`device_tokens`, which are otherwise restricted to their own participants, and
adds the `calls.caller_name` / `calls.call_type` columns.

**038** adds `admin_row_counts()`, a `SECURITY DEFINER` function that returns
true row counts while bypassing RLS. The portal uses it to tell apart the three
reasons a page can look empty:

| In database | Visible | Meaning |
| --- | --- | --- |
| 0 | 0 | Genuinely no data yet |
| > 0 | 0 | RLS is hiding the rows — re-run 037 and sign out/in |
| > 0 | = total | Working correctly |

If 038 is missing, the Dashboard shows a notice saying so instead of guessing.

## Pages

| Page | Contents |
| --- | --- |
| Dashboard | Headline counts plus a **Data Access Check** diagnostic table |
| Users | All profiles with role badges |
| Businesses | Verification queue, logos, media counts |
| Jobs | All jobs with a **View** button opening a per-job detail modal |
| Messages | Every chat transcript, grouped per job, with search + job filter |
| Images | All job photos, quote photos, business logos, galleries and certificates |
| Quotes | All quotes with totals and photo counts |
| Invoices | All invoices with a **View** button: full invoice document, print, and PDF download |
| Calls | Call history with talk-time summary and per-call duration |
| Reviews | Ratings and written feedback |
| Notifications | Full notification log with recipient and read state |
| Categories | Service category management |

The **View** modal on a job is the fastest way to audit one job end to end:
overview, messages, quotes, invoices, calls and photos in separate tabs.

Invoices render as a real document (logo, parties, line items, serial numbers,
VAT, PAID stamp) and can be downloaded as a PDF or printed. PDFs are generated
client-side with jsPDF, so no server round-trip is involved.

## Caching

GitHub Pages can serve a stale copy after a push. The build hash is shown at the
bottom of the sidebar. If a page is missing new features, hard-reload with
**Ctrl+Shift+R** and confirm the build hash matches the latest commit.
