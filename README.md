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
