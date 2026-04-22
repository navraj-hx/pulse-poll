# AI Pulse Check

A single-file, mobile-friendly live poll for running an AI-sentiment pulse check with a room. Attendees vote on a 3×3 grid (belief in AI capability × desire to embrace AI); the presenter reveals the aggregated heatmap live.

Built as one self-contained `index.html` — no build step, no framework. Persistence and real-time updates go through a Supabase `votes` table hit directly via the REST API.

## Running locally

Any static server works. From the repo root:

```bash
python3 -m http.server 8000
# or
npx serve .
```

Then open `http://localhost:8000/`.

## The two experiences

The same file serves two different UIs depending on the URL:

### Voter (default URL)

`http://<host>/` — what attendees scan into.

- Header + 3×3 grid, nothing else
- Tap a cell to vote. The vote is written to Supabase and cached in `localStorage`
- After voting, the grid is replaced by a "Thanks! Your vote has been recorded" card with a live-updating total count
- Voters never see the heatmap, per-cell counts, or the mode toggle — the distribution is the presenter's reveal
- Refreshing keeps you in the voted state (the `localStorage` token persists)

### Presenter (`?mode=present`)

`http://<host>/?mode=present` — for the person driving the session.

- Shows a password gate first (see below)
- After unlocking, exposes the full Vote / Results / Present toggle
- **Vote** — same grid as voters see, presenter can cast their own vote
- **Results** — heatmap with per-cell counts and percentages (teal gradient: light teal → solid `#59BBB6`)
- **Present** — the heatmap plus a QR code and a "Reset all votes" control. The QR points to the base URL (without `?mode=present`), so scans land attendees on the voter experience
- The results view polls Supabase every 3 seconds and updates in place

## Password

The presenter URL is gated by a hardcoded password: **`hxpulse2026`**.

This is intentionally low-security — just enough friction to stop a curious attendee who types `?mode=present` into their address bar. The password check is entirely client-side and is not a substitute for access control. If you need real protection, put the page behind SSO or an authenticating proxy.

Details:

- On the first presenter-URL visit in a browser session, a password card appears before any presenter UI renders (the gate runs as an inline `<head>` script, so there's no flash of the protected content)
- Correct password → the session is unlocked and stored in `sessionStorage` under `hx-pulse-auth`. Refreshing or switching tabs within the same browser session stays unlocked
- Wrong password → an inline error appears under the input; no `alert()`, no reveal
- Closing the browser (or opening the URL in a different browser/incognito) re-prompts
- To change the password, edit the `PRESENTER_PASSWORD` constant near the top of the `<script>` block in `index.html`
- To force a re-prompt on your own machine, clear `sessionStorage['hx-pulse-auth']` in DevTools

## Supabase

The Supabase project URL and publishable anon key are embedded in the file. The `sb_publishable_…` key is designed for client-side use — it can only do what row-level security policies on the `votes` table allow. The schema assumed:

```sql
create table votes (
  id uuid primary key default gen_random_uuid(),
  cell text not null,
  created_at timestamptz default now()
);
```

`cell` is stored as `b{0-2}_d{0-2}` — `b` = belief index (0 = gone down, 1 = stayed the same, 2 = gone up), `d` = desire index with the same convention.

## Vote deduplication

Deduplication is client-side only, via `localStorage['ai-pulse-voted']`. A user can vote again from a different device, a different browser, or after clearing their storage. That's acceptable for a room pulse check but is not anti-abuse.

## Branding

Uses the hx theme tokens (defined as CSS variables at the top of the `<style>` block):

- Primary text: `#1C2733`
- Teal accent: `#59BBB6`
- Blue accent: `#4D6FF8`
- Purple accent: `#BCA0E7`
- Coral (danger/reset): `#EE6C5A`
- Font: JetBrains Mono via Google Fonts

The header gradient runs teal → blue → purple; the heatmap ramps from near-white through teal tints up to solid `#59BBB6`.
