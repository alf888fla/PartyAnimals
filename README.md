# PartyAnimals
Party Animals App Repository. Group 2. APD. DST.
## Branches
- main: stable work only
- develop: integration branch, everything merges here first
- feature: one branch per task, e.g. feature/button-lydia

## Database + login (PARS-49, PARS-48)

### Current setup

- **App**: https://party-animals-sigma.vercel.app (hosted on Vercel, deployed from `main` of this repo, single `index.html`)
- **Database**: Supabase project "Party Animals" (org "Party Animals", region eu-central-1 / Frankfurt). `schema.sql` is already applied.
- **Keys**: project URL and publishable key are already in `index.html`. Never put the secret key in the app.
- **Auth**: email + password, email confirmation ON. Site URL and redirect URL point to the Vercel site above.
- **Team access**: accept the Supabase invitation from your email to get into the project.
- **Updating the app**: follow the branch workflow above. Once a change is merged into `main`, Vercel redeploys the site automatically.

Files:
- `index.html`: the app. With empty keys it runs as a local demo; with keys it uses the database.
- `schema.sql`: the whole database. Only re-run it on a NEW project.
- `lydia-pars-46-47.sql`: additions for PARS-46 and PARS-47 (favorites, venue history opt-in, friends).

### Setup from scratch (reference, already done for this project)

1. **Create the project** on supabase.com (region: Europe, e.g. Frankfurt or Paris). Invite the team in Organization > Members.
2. **Run the schema**: SQL Editor > New query > paste all of `schema.sql` > Run.
   If the last block (pg_cron) errors, enable `pg_cron` in Database > Extensions and run that block again.
3. **Auth settings** (Authentication > URL Configuration):
   - Site URL: the URL where the app is hosted (see step 5).
   - Redirect URLs: add the same URL. Confirmation and password-reset emails send people back there.
   - Email confirmation is on by default. For quick team testing you can turn it off in Authentication > Sign In / Providers > Email.
4. **Paste the keys** at the top of `index.html` (Project Settings > API):
   ```js
   const SUPABASE_URL = 'https://xxxx.supabase.co';
   const SUPABASE_ANON_KEY = 'eyJ...';   // "anon public" or "sb_publishable_..." key
   ```
   This key is public by design. Never paste the `service_role` / secret key in the app.
5. **Host it**: import the GitHub repo on Vercel (Framework Preset "Other", no build command). Login and email links need a real https URL; a file opened from your disk can log in, but email links won't come back to it.

### What the database contains

| Table | What it holds | Who can read it |
|---|---|---|
| `profiles` | name, username, 18+ flag, visibility (public / friends / private). Created automatically at sign-up. | yourself, public profiles, friends' profiles |
| `venues` | the 19 seed venues from the app (placeholders, `verified = false`) | everyone |
| `pings` | party-mode check-ins: venue + time only, server-stamped | nobody (insert-only) |
| `going` | "going tonight", one row per user/venue/night | yourself |
| `friendships` | requests and status (pending / accepted / declined / blocked) | both people involved |
| `visits` | venue history, empty for now (needs opt-in) | yourself |

Functions:
- `get_crowd()`: the only way to read activity. Returns 0 for any venue with fewer than 5 people.
- `delete_my_account()`: "delete my data" in the app. Removes the user and all their rows.
- `are_friends(a, b)`: used by the profile visibility rule.

Clean-up jobs: pings deleted after 2 hours, old "going" rows deleted every morning.

### What the app does

- Login, sign-up (with optional name), forgot password, set new password, resend confirmation email.
- "Just look around": the map is usable without an account. Party mode and "going tonight" ask you to log in.
- Me tab: account panel (logged in as / log out). "Delete my data" deletes the account for real.
- Live crowd numbers come from `get_crowd()` and are added on top of the demo data while demo mode is on.

### Starting points for the other tickets

- **PARS-45 privacy**: `profiles.visibility` and its read policy already exist. Build the settings UI and apply the same rule to anything new that shows people.
- **PARS-46 profile / onboarding** and **PARS-47 friends + live questions**: see `lydia-pars-46-47.sql`. Live questions need a table (e.g. `venue_reports`: venue, kind = price / crowd / grade, value, created_at) plus an aggregate function like `get_crowd()` with the same 5-person rule. Fill `visits` only after the user opts in, because the app promises check-ins disappear after 2 hours.
- **PARS-30 emails**: Supabase already sends confirmation and reset emails. Customize them in Authentication > Email Templates, and set up a custom SMTP sender (e.g. Resend) before real users, since the built-in sender is rate-limited.
- **Venues**: the app still uses its built-in list for the map. Next step is loading `venues` from the database so venues can be fixed without redeploying.
