# JL Rental Management: Google Drive Backend Plan

## Why a backend is needed
A browser-only page cannot enforce who may read or write renter data, cannot safely hold the admin's Google refresh token, and cannot arbitrate two users editing at once. The minimum safe addition is one small server that holds secrets, checks identity on every request, and talks to Drive.

## Architecture
- **Frontend:** the existing single-page app, served over HTTPS (Firebase Hosting, Cloudflare Pages or Netlify).
- **API:** one small service (Cloud Run, Cloudflare Worker or Firebase Function), Node/TypeScript. It is the only component that touches Drive.
- **Identity:** Google Sign-In (OpenID Connect). The frontend sends the Google ID token; the API verifies signature, audience and expiry on every call.
- **Authorization:** an allow-list file `users.json` inside the Drive folder (email, role, assigned units). The API checks it on every request, so roles are enforced server-side. Only ADMIN can edit it.
- **Storage:** a private folder "JL Rental Management" owned by the admin's Google account, containing `data/`, `invoices/`, `backups/`, `reports/`, `audit/`.

## Drive access model
- Scope: `https://www.googleapis.com/auth/drive.file` only (files the app created). No full-Drive scope.
- The admin authorizes once via OAuth 2.0 authorization-code flow with PKCE. The refresh token is stored encrypted in the API's secret manager, never in frontend code or Drive.
- Authorized users never get direct Drive access; they go through the API. Optionally share the folder to named accounts only (`type=user`, never `anyone`). The API refuses to create any permission of type `anyone` or `domain`.
- No public links, ever. Downloads stream through the API after a role check.

## Data and concurrency
- Records stored as per-collection JSON (`renters.json`, `invoices-2026-11.json`, `payments.json`), each with a `version` number.
- Writes send `If-Match: <version>`. A stale version returns 409 and the client shows a merge/reload prompt, so nobody silently overwrites someone else's edit.
- Finalized invoices are immutable on the server; corrections create a revised invoice or credit/debit entry.
- Append-only audit log (`audit/YYYY-MM.jsonl`): who, when, action, record id, old/new hash.
- Nightly backup to `backups/` plus automatic backup before any import or restore. Drive file revisions give version history. Restore is ADMIN-only.

## API endpoints
| Method | Path | Role |
|---|---|---|
| GET | /me | any |
| GET/PUT | /data/:collection | read: viewer+, write: manager+ |
| POST | /invoices/:id/finalize | manager+ |
| POST | /invoices/:id/pdf | manager+ (stores PDF in `invoices/`) |
| POST | /payments | manager+ |
| GET/PUT | /users | admin |
| POST | /backup, /restore | admin |
| GET | /audit | admin |

PDF generation moves server-side (Puppeteer or a PDF library) so finalized invoices are saved straight to Drive and the stored copy is the official one.

## Google Cloud setup
1. Create a Google Cloud project; enable the Google Drive API.
2. Configure the OAuth consent screen: User type External (or Internal if on Workspace), add test users while in Testing, scope `drive.file`.
3. Create an OAuth client ID (Web) for sign-in with authorized origins and redirect URIs for your HTTPS domain. Keep the client secret in the API's secret manager only.
4. Deploy the API with environment variables: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `TOKEN_ENCRYPTION_KEY`, `DRIVE_FOLDER_ID`, `ALLOWED_ORIGIN`.
5. Admin signs in once, approves Drive access, the API creates the folder and an initial `users.json` containing only the admin.
6. Admin adds users and roles in the User Management screen.

## Security checklist
- HTTPS only; strict CORS to your origin; CSP and security headers.
- Verify ID token on every request; session timeout 30 minutes idle.
- Validate and sanitize all input server-side; money handled as integer centavos.
- Rate limiting on the API; no secrets in the repo or frontend bundle.
- Renter data never sent to third parties other than Google Drive.
- Email sending only on explicit request, using the Gmail API with a separate `gmail.send` consent if wanted.

## Frontend changes
Replace localStorage with API calls, remove local PINs in favor of Google Sign-In, add 409 conflict dialog, add "Save to Drive" actions, and show role from `/me`.

## Test plan before real use
Import workbook; rent/electricity/previous-balance math; partial, full and advance payments; duplicate invoice block; PDF layout on A4; Drive save and retrieval; unauthorized and wrong-role requests rejected; two users editing the same record (409); backup then restore; audit entries present.

## Estimate
Roughly 2 to 4 days for API, auth and Drive storage, plus 1 to 2 days for frontend integration and testing.
