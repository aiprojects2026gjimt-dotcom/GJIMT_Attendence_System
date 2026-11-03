# GJIMT Smart Faculty Attendance — GitHub Pages + Google authentication

## What changed

The earlier `Bridge.html` approach cannot reliably work with Google Apps Script's nested sandboxed iframe. **Do not deploy the old Bridge.html.** The updated architecture is:

GitHub Pages (camera and face-api.js) → Google Identity Services sign-in → Cloudflare Worker (verifies Google ID token and authorized email) → Apps Script `/exec` (checks private shared secret) → existing Google Sheet.

The Apps Script spreadsheet tab names and column headers are unchanged. `Face_Descriptors` remains a hidden/protected sheet. **Back up the Google Sheet and Apps Script before updating.** This is an implementation package, **not a live-tested deployment**.

## Files and destinations

- `UPLOAD_TO_GITHUB/`: upload its **contents** to the root of your GitHub repository (not the folder itself).
- `APPS_SCRIPT/Code.gs`: replace your Apps Script `Code.gs` with this file. Keep your existing Google Sheet. Remove/ignore `Bridge.html`.
- `CLOUDFLARE_WORKER/worker.js`: deploy as a Cloudflare Worker. Never upload secrets to GitHub.

## Step 1 — Back up

In Google Sheets: File → Make a copy. In Apps Script: save a backup of your current `Code.gs`. Do not delete or recreate your existing Sheets. The `setupAttendanceDatabase()` function checks exact headers and does not overwrite existing records; only run if your database is not already initialized.

## Step 2 — Create a Google OAuth client ID

In Google Cloud Console, create or select a project, configure the OAuth consent screen/Google Auth Platform (Internal for a supported Google Workspace organization, or External with appropriate test users), then create a **Web application** OAuth client ID. Add this exact Authorized JavaScript origin:

`https://aiprojects2026gjimt-dotcom.github.io`

Do **not** add `/GJIMT_Attendence_System` or a trailing slash to the origin. Copy the client ID ending in `.apps.googleusercontent.com`. Google Workspace Internal is only available for qualifying Workspace projects; use a controlled allowlist regardless. Google Identity Services may require consent screen setup and test-user enrollment while in testing.

## Step 3 — Create the Cloudflare Worker

1. Sign into Cloudflare → Workers & Pages → Create Worker. Paste `CLOUDFLARE_WORKER/worker.js` as the Worker source and deploy.
2. Under Worker **Settings → Variables and Secrets**, configure:

   | Name | Type | Value |
   |---|---|---|
   | `GOOGLE_CLIENT_ID` | Text | OAuth client ID from Step 2 |
   | `ALLOWED_ORIGIN` | Text | `https://aiprojects2026gjimt-dotcom.github.io` |
   | `ALLOWED_EMAILS` | Text | Comma-separated exact approved faculty Google emails |
   | `ADMIN_EMAILS` | Text | Comma-separated exact administrator Google emails |
   | `APPS_SCRIPT_URL` | **Secret** | Your Apps Script deployed `/exec` URL |
   | `WORKER_SHARED_SECRET` | **Secret** | Random 32+ character secret, identical to Apps Script property |

3. Do not put the secret or admin PIN in `config.js`, source code, or public GitHub. **Admin emails** may also appear in `ALLOWED_EMAILS` but need not.
4. After changing settings, save/deploy the Worker. Copy its URL, e.g. `https://your-worker.your-subdomain.workers.dev`.

The Worker rejects requests from other browser origins, verifies Google ID tokens with Google's tokeninfo endpoint, checks token audience and verified email, and requires an admin email for enrollment/report actions. **CORS is not authentication**; token validation and allowlisting are the authentication controls.

## Step 4 — Update Google Apps Script

1. From your *existing* Attendance Sheet open **Extensions → Apps Script**.
2. Replace the contents of `Code.gs` with `APPS_SCRIPT/Code.gs` and save.
3. Open **Project Settings → Script Properties**, add `WORKER_SHARED_SECRET` with the same random value used in Cloudflare. Alternatively, run `configureWorkerSharedSecret()` in the Apps Script editor to enter it in a prompt.
4. If the administrator PIN is still the default, run `changeAdminPin()` to replace it. Keep it private. The admin email check and PIN check are both required for admin operations.
5. If the database was already initialized, **do not run setup again**. Your existing tab names and headers remain unchanged.
6. **Deploy → Manage deployments → Edit → New version → Deploy**. Set **Execute as: Me**. The Worker needs to reach the endpoint, so the web app deployment must permit unauthenticated HTTP requests (often **Anyone**). This is safe only insofar as the server validates the private shared secret; keep that secret confidential and rotate it if exposed. Google Workspace policy may prevent an Anyone deployment; if so this architecture will require a different backend or institution-controlled proxy.
7. Update `APPS_SCRIPT_URL` secret in Cloudflare if the `/exec` URL changed.
8. Open the `/exec` URL in a browser. Expected: `{"success":true,"message":"GJIMT attendance API is running.","version":"2.0"}`. This checks the Apps Script health endpoint only, **not** the full authenticated path.

## Step 5 — Update GitHub Pages

1. In `UPLOAD_TO_GITHUB/config.js`, replace `workerUrl` with the deployed Cloudflare Worker URL and `googleClientId` with the OAuth client ID.
2. Upload **all files inside** `UPLOAD_TO_GITHUB/` to the GitHub repository root. Replace the old `index.html` and `config.js`. Do not upload `Code.gs`, `Bridge.html`, or secrets.
3. GitHub → Settings → Pages: deploy `main` branch from `/ (root)`.
4. Open `https://aiprojects2026gjimt-dotcom.github.io/GJIMT_Attendence_System/` (use your actual Pages URL if the repository name differs), hard-refresh, sign in with an allowlisted Google account. The site should show **Institutional access verified** after the Worker successfully calls `testBackend`.

## Step 6 — Test safely

1. On mobile, use HTTPS and allow **Camera** permission. No microphone permission is needed.
2. First test the separate `camera-test.html` page.
3. In `Employees`, create/use a test employee with `employment_status=Active`. Set `consent_status=Granted` **only after obtaining informed consent**.
4. Sign in with an email listed in `ADMIN_EMAILS`, enter the admin PIN, start camera and enroll the test face. This writes `Face_Enrollment` and `Face_Descriptors`.
5. Check in once with the test face; verify a new `Attendance` row. Check in again; it should return already marked for that date.
6. Test a non-admin allowlisted email: camera check-in should work, but admin operations must be rejected.
7. Test an unlisted email: sign-in must fail at the Worker.

## Troubleshooting

- **Not signed in:** OAuth client ID or JavaScript origin incorrect; verify Google OAuth consent-screen settings and test users.
- **Unapproved website origin:** Cloudflare `ALLOWED_ORIGIN` must match the *exact* GitHub Pages origin (no repository path).
- **Google identity verification failed:** wrong client ID, expired token, or OAuth test-user restrictions.
- **Apps Script returned non-JSON:** wrong `/exec` URL, incorrect deployment permissions, Google login page, or a backend redirect issue.
- **Unauthorized backend request:** Worker and Apps Script shared secrets differ; verify the latest Apps Script deployment.
- **Camera works but attendance fails:** verify face-api.js model files, employee consent, and enrollment; inspect browser console and Cloudflare Worker logs.
- **Third-party cookies:** no iframe bridge is used in this version.

## Security and production limitations

This is a **pilot**, not a certified secure biometric time-clock. Face matching without liveness detection is vulnerable to photos, videos and replay/spoofing; biometric descriptors are sensitive even though they are not raw photos. A hidden Sheet is not encryption. Restrict Google Sheet sharing, obtain explicit informed consent, provide an alternative attendance method, set retention/deletion policies, and have the institution review applicable privacy requirements. The Worker validates the **caller account**, but face matching alone does not prove the person in front of the camera is live or is the signed-in account holder. Anyone on the allowed faculty list can submit a descriptor to check in a matching enrolled face. For a high-assurance kiosk, add server-side device authentication, anti-spoofing, rate limiting and independent verification.

Cloudflare Workers and Google Cloud OAuth configuration may incur charges or have quotas depending on usage and plan; verify current pricing before production deployment.
