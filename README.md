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

These migrations are required. All are idempotent (safe to re-run).

```
  supabase/migrations/037_admin_read_all.sql
  supabase/migrations/038_admin_row_counts.sql
  supabase/migrations/039_admin_user_management.sql
  supabase/migrations/040_admin_technicians_read.sql
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

**039** adds `profiles.is_active` plus a trigger that blocks new sign-ins for
deactivated accounts. It also revokes existing sessions on deactivation. It is
required for the *Deactivate* button on the Users page.

**040** grants admins `SELECT` on `technicians` so the Technicians tab can show
a technician's business and status, and adds immediate session revocation when
`profiles.is_active` flips to `false`.

To run it, open **Supabase dashboard → SQL Editor** and paste the *contents* of
the file. Pasting the file path itself fails with a syntax error, since the
editor only accepts SQL. The file is wrapped in `BEGIN`/`COMMIT`, so either all
of it applies or none of it does.

## Required edge functions

```
supabase/functions/add-business/index.ts
supabase/functions/admin-manage-user/index.ts
```

Deploy once with the Supabase CLI:

```
supabase functions deploy add-business
supabase functions deploy admin-manage-user
```

**`add-business`.** `business_profiles.id` is a primary key that references
`profiles.id`, which in turn references `auth.users.id`. A business therefore
cannot exist without a real login behind it, and creating that login requires
the service role key. That key must never be shipped to the browser, so the
portal's *Add Business* button calls this edge function instead of writing to
the database directly.

**`admin-manage-user`.** Handles the *Remove User* action. Deleting a
`profiles` row from the browser would leave the auth user alive but
profileless — they could still sign in and land with no account. This function
deletes the auth user instead, which cascades correctly.

It also refuses to act on the calling admin's own account, and requires the
user's name typed back before deleting.

### Removing a user destroys their history

Every table below references `profiles(id) ON DELETE CASCADE`, so removing a
user permanently deletes their jobs, quotes, invoices, messages, calls,
reviews, addresses and notifications. Before deleting, the portal calls the
function in `preview` mode, which returns the row counts, and shows them in
the confirmation dialog.

**Deactivate instead** unless you are certain. Deactivation keeps the rows, so
past jobs and invoices still show the person's name, and they can be
reactivated later.

## Users

Customers and Technicians are separate tabs, because they are governed
differently: a technician belongs to a business and is switched off through
`technicians.is_active`, while a customer has no such row.

| Action | Effect |
| --- | --- |
| Edit | Change full name. For technicians, reassign their business |
| Deactivate | Block sign-in, keep all history, reversible |
| Remove | Delete the login and cascade to their data. Not reversible |

Email and phone are stored on the auth login and are shown read-only in the
edit dialog. Changing them means going through Supabase Auth.

Deactivation requires migration `039`, which adds `profiles.is_active` and a
trigger that blocks new sign-ins for deactivated accounts. Until it is
applied, the *Deactivate* button will fail.

## Setup Check

Open **Setup Check** in the sidebar. It probes the two migrations and two edge
functions live and shows exactly which are missing, with the command to fix
each. Nothing is inferred from memory, so it reflects the real state of the
project rather than what was intended.

Nothing else in the portal depends on these four items, so an unrun migration
only disables the feature that needs it. Each affected button names the file
to run instead of surfacing a database error.

| Missing | What stops working |
| --- | --- |
| Migration 039 | Deactivate / Reactivate on Users |
| Migration 038 | Dashboard → Data Access Check |
| `add-business` | Add Business button |
| `admin-manage-user` | Remove User (Edit and Deactivate still work) |

## Pages

| Page | Contents |
| --- | --- |
| Setup Check | Probes migrations and edge functions, lists what is missing |
| Dashboard | Headline counts plus a **Data Access Check** diagnostic table |
| Users | Customers and Technicians in separate tabs, with edit, deactivate and remove |
| Businesses | Verification queue, logos, media counts, plus **Add Business** to provision a new business and its owner login |
| Jobs | All jobs with a **View** button opening a per-job detail modal |
| Messages | Every chat transcript, grouped per job, with search + job filter |
| Images | All job photos, quote photos, business logos, galleries and certificates |
| Quotes | All quotes with a **View** button: sub-tabs for the quote document, communication, and images; plus print and PDF download |
| Invoices | All invoices with a **View** button: full invoice document, print, and PDF download |
| Calls | Call history with talk-time summary and per-call duration |
| Reviews | Ratings and written feedback |
| Notifications | Full notification log with recipient and read state |
| Categories | Service category management |

The **View** modal on a job is the fastest way to audit one job end to end:
overview, messages, quotes, invoices, calls and photos in separate tabs.

Invoices and quotes both render as a real document (logo, parties, line items,
VAT, status stamp) and can be downloaded as a PDF or printed. PDFs are
generated client-side with jsPDF, so no server round-trip is involved.

A quote's `View` modal is split into three sub-tabs, so the document stays
readable rather than being buried under a photo gallery:

- **Quote Document** — the document itself: scope of work, line items, VAT
  breakdown, notes, validity window (flagged once expired), scheduled start,
  acceptance date, last-updated timestamp, linked job and status, business,
  technician, and customer.
- **Communication** — the full client ↔ installer history for the linked job.
- **Images** — attached photos as a clickable gallery, opening a lightbox.
  Where a quote has no photos, it points you at the linked job's Images tab.

Photos are intentionally excluded from the printed and downloaded PDF. That
document is what a client would be sent, and it should carry the quote, not the
installer's job gallery. The print stylesheet prints the document tab only.

## Caching

GitHub Pages can serve a stale copy after a push. The build hash is shown at the
bottom of the sidebar. If a page is missing new features, hard-reload with
**Ctrl+Shift+R** and confirm the build hash matches the latest commit.
