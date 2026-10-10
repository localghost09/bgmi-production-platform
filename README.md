# BGMI Broadcast V4.5 — Tactical Show Flow

V4.5 extends the V4.4 smooth-transition production suite with a complete operator-led broadcast rundown: full-screen pre-show slate and 20-minute stream-start countdown, auto-cue to full-screen caster, tournament format, selected group/team lineup, full-screen caster hype, a 30-second corner countdown, then live match graphics. It also adds BGMI player UID-based top-fragger aggregation, clearer group-filter feedback, compact squad status controls, safer media URL normalization and CSV alignment for older registrations.

This is an independent community tournament tool. It is not an official KRAFTON product, is not KRAFTON-affiliated, and does not include proprietary BGMI telemetry.

This is an independent community tournament tool. It is not an official KRAFTON product, is not KRAFTON-affiliated, and does not include proprietary BGMI telemetry.

## Requirements
- Node.js 20+
- OBS Studio 28+ recommended
- Windows, macOS or Linux
- MongoDB / MongoDB Atlas optional for persistence across deployments; local JSON works for rehearsal

## Upgrade from V4.4 without losing your event
1. Stop the old server with `Ctrl+C` and back up its entire project folder.
2. Extract V4.5 to a new folder. The ZIP intentionally omits the generated `data/tournament.json` so it cannot silently overwrite an organizer's event data.
3. Before the first launch, copy these files from the old project's `data` folder into V4.5's `data` folder when they exist: `tournament.json`, `admin.json`, `captains.json`, and `.session-secret`. Copy your `.env` file separately if you use MongoDB or deployment settings. Keep these private.
4. Run the start commands below. V4.5 normalizes the tournament state to schema version 8 on load.

## Start
Open PowerShell in the extracted project folder:

```powershell
Copy-Item .env.example .env   # optional; configure MongoDB if using it
npm install
npm run check
npm start
```

For first run, open `http://localhost:3000/login` and create the administrator account (username 3–32 allowed characters; password at least 12 characters). Team captains can create an account only after the admin has been configured.

By default, local development binds to `127.0.0.1` (this computer only). For a hosted deployment, configure `NODE_ENV=production`, HTTPS at the reverse proxy, and use the supplied `BIND_HOST`/`TRUST_PROXY` settings as appropriate. Do not expose the development server directly to the internet.

## GitHub and Vercel
This repository includes `vercel.json` and `api/index.js` so Vercel can build and serve the Express HTTP routes. Before deploying, add these Vercel environment variables:

- `NODE_ENV=production`
- `MONGODB_URI=<MongoDB Atlas connection string>`
- `MONGODB_DB=bgmi_broadcast`
- `SESSION_SECRET=<long random secret>`
- `COOKIE_SECURE=true`
- `TRUST_PROXY=true`

Vercel is suitable for an HTTP preview and API hosting, but it does not provide the persistent WebSocket process required by Socket.IO. The operator dashboard and OBS overlay need the live Socket.IO channel, so use a long-running Node host such as Render, Railway, Fly.io or an Azure Web App for the full production deployment. Keep MongoDB Atlas enabled there as well. Do not commit `.env`, administrator credentials, captain credentials or live tournament data.

To publish the code to GitHub, create an empty repository, then run:

```powershell
git init
git add .
git commit -m "Prepare BGMI broadcast app for deployment"
git branch -M main
git remote add origin https://github.com/<account>/<repository>.git
git push -u origin main
```

For Vercel, import that GitHub repository in the Vercel dashboard, set the environment variables above, and deploy. The project has no build step; Vercel uses `api/index.js` as the Node function entrypoint.

## URLs
- Operator control room (admin account required): `http://localhost:3000/operator`
- Captain/team registration portal: `http://localhost:3000/register`
- Transparent OBS Browser Source: `http://localhost:3000/overlay`
- Admin-only health endpoint: `http://localhost:3000/api/health`
- Admin-only standings CSV: `http://localhost:3000/api/standings.csv`
- Admin-only registration CSV: `http://localhost:3000/api/registrations.csv`

## Captain registration and anti-duplicate rules
1. The organizer configures the administrator account first.
2. A captain creates an account with name, email, mobile number and a password of at least 12 characters, or signs in to an existing account.
3. Mobile number inputs strip all non-digits in the browser; the server accepts digits only (10–15 digits).
4. A signed-in captain can submit one team application only. The server blocks a second submission for the same captain account, and rejects duplicate team names, contact emails or mobile numbers across registrations.
5. Team applications can include a team-logo URL and optional player portrait URLs. Only HTTP/HTTPS URL formats are accepted; use trusted HTTPS image hosts.
6. The organizer reviews submissions, selects a group, and approves the team. Approval copies its team name, logo, college, roster names and portraits into an available tournament slot.
7. The captain can sign in again to view the saved application status. The account does not allow a second application after rejection; contact the organizer to resolve changes or eligibility questions.

### Important identity-verification limitation
Captain login, password hashing, one-team-per-account, unique email/phone checks and rate limiting are implemented. However, **email/SMS OTP ownership verification is not configured** in this project. Without a configured verification provider, a determined person could create accounts using alternate contact details. Before public production, integrate an email/SMS verification service (and preferably CAPTCHA/abuse monitoring) and verify your event eligibility rules. Do not treat an unverified email or phone as proven identity.

## Broadcast production features
- BGMI-inspired dark gunmetal / amber broadcast graphics, angular panels, match counter and squad status HUD.
- Current match setup, 2–16 team lineups, group selection, team finish controls and player finish attribution.
- Current-match MVP calculated from recorded player finishes; overall top fragger aggregated from finalized match snapshots plus current unfinalized match data.
- Caster intro lower-third using an optional portrait URL and custom name/tagline.
- Operator-triggered 30-second countdown. It updates continuously on the overlay and automatically switches the overlay to Live when the countdown ends.
- Current MVP and overall top-fragger spotlight scenes show player portrait, team name/logo and finish count.
- Map Feed scene can display an organizer-configured map image URL and match/map label.
- Match results and cumulative standings with live Socket.IO state synchronization.


### V4.5 production rundown
- Configure event logo, caster portrait/name/tagline, theme audio URL or `/assets/` path, and stream-start duration in Broadcast Control.
- For a self-hosted licensed track, create `public/assets/` and put the media file there; use a URL such as `/assets/tournament-theme.mp3`. The HTML audio element loops only during the full-screen stream-start slate and stops when the production leaves that state. OBS browser audio autoplay may be blocked; use an OBS Media Source fallback when needed.
- Trigger **Stream starts · 20 min**: full-screen event logo/title and countdown appear; the server automatically changes the overlay state to full-screen caster when the countdown reaches zero.
- Operator cues then proceed manually: **Tournament format** → **Match lineup** (reflects the saved match roster, team names and logos) → **Full-screen caster** → **Hype + 30-second timer**. The countdown appears at the upper-left over the full-screen caster and, at zero, the app changes state to **Live match**.
- When ready, select **Match results**, **Overall standings**, **Current match MVP**, **Overall top fragger**, **Map feed** or **Intermission** as appropriate. Native OBS scene switching/source automation is not integrated; these controls change the transparent browser-overlay graphic state.
- Group filter applies to displayed squad controls and lineup choices. If a group is selected but another group's squads remain in the saved cross-group lineup, the interface shows a note; use **Load roster** to replace the active lineup intentionally. This protects semifinal/grand-final custom lineups from silent changes.
- BGMI UID fields are numeric and are used to deduplicate player identities for the overall top-fragger calculation when present. They help staff verify registrations manually; this app does not verify IDs against a BGMI API.

### V4.4 smooth transition test
- Keep one OBS Browser Source open at `http://localhost:3000/overlay` (1920×1080, 30 FPS) over the gameplay source.
- In the operator dashboard, change states in this order: Pre-show → Caster intro → 30-second countdown → Live match → Current match MVP → Map feed → Match results → Overall standings → Intermission → Live match.
- The overlay panels now fade/transform both in and out over approximately 320 ms; don't reload the Browser Source between states.
- For switching actual OBS scenes, set OBS's global Scene Transitions to Fade at 300–500 ms. Dashboard state selection changes the browser overlay graphic; it does not remotely switch OBS scenes in this version.
- The map panel is a configured image/graphic. A real moving gameplay map must be added as its own OBS capture/video source.

### Player finish attribution
In Live Match Operations, use `+ PLAYER FINISH` when the player who earned a finish is known. This increments both that player's finish count and the squad's total finish count. Use `+ UNASSIGNED FINISH` only when the squad finish is known but player attribution is not. **Do not record the same finish using both controls.** Player MVP and top-fragger graphics are only as accurate as the player-level finish entries recorded by the operator.

### Map feed limitation
The `Map Feed` graphic can show a map image URL and broadcast layout. It does **not** automatically capture BGMI's live in-game map or receive player coordinates. For a real moving observer/map feed, create an OBS scene with the relevant authorized observer capture/video source. Native OBS scene switching/source automation is not integrated in this version; the dashboard currently controls the web overlay state.

## OBS setup
1. Set the OBS canvas/output layout to 1920×1080 (16:9) unless your designed stream layout uses another aspect ratio.
2. Add your BGMI gameplay/observer capture.
3. Add a Browser Source above gameplay at `http://localhost:3000/overlay`, set width 1920, height 1080, FPS 30.
4. Leave the browser source loaded during the match. The operator dashboard updates overlay state over Socket.IO.
5. Use the Broadcast Control panel to edit the caster and map-image URLs, start the countdown, show the caster, current match MVP, overall top fragger, map graphic, results, standings or intermission.
6. For a hosted server, use its HTTPS URL and ensure the OBS machine can reach it.

To combine the overlay with a real moving map video, add the observer map feed as a separate OBS source/scene. Verify that the game's broadcast and asset usage is permitted for your event.

## Access control and security
- Admin pages and admin APIs (operator dashboard, CSV exports and health) require an administrator session.
- Tournament write actions from Socket.IO require an authenticated administrator session; an unauthenticated overlay socket receives public display state only.
- Captain account endpoints are role-specific. Captain sessions cannot access operator pages or admin socket commands.
- Registration submission requires a captain session, is rate-limited, allows one team per account, and validates all critical fields server-side.
- Sessions use HttpOnly, SameSite=Strict cookies and regenerate the session on login. Passwords are bcrypt-hashed. Local default binding is loopback only.
- The public overlay receives no registration contact records or audit log. Registration/export data is restricted to the administrator.
- Set `SESSION_SECRET` to a long random secret for deployment. Set `COOKIE_SECURE=true` behind HTTPS and `TRUST_PROXY=true` only behind a correctly configured trusted reverse proxy. Configure least-privilege MongoDB credentials and restrict network access.
- This software has not received a professional penetration test. Use HTTPS, CAPTCHA, verified contact ownership, secure backups and monitored hosting before a public competition.

## Storage
- Blank `MONGODB_URI`: `data/tournament.json` is the local tournament store.
- Configured `MONGODB_URI`: MongoDB is primary for tournament state and a local JSON recovery snapshot is maintained.
- Administrator credentials are kept in `data/admin.json` and captain account hashes in `data/captains.json`; these files are excluded from version control by `.gitignore`. Back them up securely and never commit them.
- MongoDB is used for session persistence when configured. With the local session store, users need to sign in again after server restart.

## Testing
Run `npm run check` to check JavaScript syntax and package script configuration (V4.5 is version 4.5.0/schema version 8). Also smoke-test on the target machine before a live event:

1. Anonymous `GET /operator` redirects to login; anonymous admin API/CSV requests return 401/redirect.
2. A captain cannot access `/operator`, `/api/health`, or `/api/standings.csv`.
3. Registration submission with no captain login returns 401.
4. Same account cannot submit twice; duplicate email, phone or team name is rejected.
5. Phone signup accepts digits only; letters, spaces and punctuation are rejected server-side.
6. Approve a sample roster and verify team name/logo/player names/portrait URLs appear in the selected team slot.
7. Start a test match, increment a player finish, and confirm both player finishes and squad finishes update.
8. Show current MVP and overall top fragger; verify player portrait, team logo, squad name and finish count.
9. Start the 30-second countdown and verify it transitions to Live automatically at zero.
10. Show caster, map image graphic, results, overall standings and intermission without reloading the OBS browser source.
11. Confirm contact details do not appear on the public overlay/socket state.
12. Rehearse a full match and verify the actual observer map video is configured as an OBS source when required.

## Scoring defaults
Sample default placement points: 1st=10, 2nd=6, 3rd=5, 4th=4, 5th=3, 6th=2, 7th=1 and 8th=1, with 1 point per finish. These are editable sample defaults, not a guarantee of current official rules. Confirm the applicable rulebook before competition.

## Public vs protected routes
Intentionally public pages/endpoints are `/login` (to sign in), `/register` (captain account and registration portal), `/overlay` (read-only production graphics), `/api/public/registration-info` (event name, open/closed state and capacity counters), and the authentication endpoints needed to sign up/sign in. The first administrator setup endpoint is only allowed while no administrator is configured. Operator HTML, admin health/export endpoints, all tournament write commands, and registration review commands are administrator-protected. Captain registration and status endpoints require the current captain's own authenticated session. The public overlay receives sanitized display state only, with contact records and audit history omitted.

Security is defense in depth, not an absolute guarantee. Captains have one application per account; duplicate team names, emails and phone numbers are rejected; account creation and registration attempts are rate-limited; and phone numbers are validated server-side. **Because contact ownership verification and CAPTCHA are not configured, this local version cannot prove that two different accounts are controlled by different real people.** Before publishing the portal to the internet, integrate email/SMS OTP verification and anti-abuse checks.

## V4.5 release notes and limitations
- `schemaVersion` is now 8; the app normalizes older V4.4 JSON state on startup. The release archive excludes `data/tournament.json`, administrator credentials, captain credential hashes, session secrets, `.env`, and `node_modules`.
- A BGMI UID is a numeric user-provided identifier for staff review; the software cannot query an official BGMI service to confirm ownership. Overall player finish aggregation prefers UID, with a team/name fallback for old match snapshots or slots without UID.
- The 20-minute pre-stream slate, theme audio, full-screen caster, tournament format, selected group lineup, corner countdown and live HUD are overlay states. Stream-start completion and the 30-second match countdown are timed by the backend; the other cues are selected by the operator.
- Native OBS WebSocket scene/source control and dynamic in-game map telemetry are not included. Use the OBS scene transition for actual scene cuts and add any real observer/map feed as a separate authorized OBS source. The configured map graphic is an image panel.
