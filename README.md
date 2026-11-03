# GJIMT Smart AI Faculty Attendance

This project uses Google Apps Script as the web app/backend, Google Sheets for employee and attendance records, and face-api.js in the kiosk browser to create a live face descriptor. The descriptor is sent to Apps Script for matching; the stored descriptors are not downloaded to the kiosk.

## Files
- `Code.gs` — Apps Script backend, sheet setup, faculty enrollment, matching, and attendance logging.
- `index.html` — GJIMT kiosk UI, camera, face recognition, admin enrollment and today's attendance report.
- `camera-test.html`, `face-detection-test.html`, `google-login-test.html` — original test pages retained for reference; not required for deployment.

## 1. Prepare the Google Sheet
1. Convert `GJIMT_Smart_Attendance_Database_Template.xlsx` to a native Google Sheet in Drive (open with Google Sheets, then File > Save as Google Sheets if needed).
2. Open that Google Sheet.
3. Choose **Extensions > Apps Script**.
4. Replace the default `Code.gs` with the supplied `Code.gs` contents.
5. Add an HTML file named exactly **index** (plus sign > HTML), then paste the supplied `index.html` contents.
6. Save the project.

## 2. Initialize the database
1. In Apps Script, select `setupAttendanceDatabase` and click Run.
2. Approve the requested Google permissions.
3. Return to the spreadsheet and confirm the tabs: Employees, Attendance, Face_Enrollment, Settings, Audit_Log, and Face_Descriptors.
4. Add faculty to `Employees`. Required fields: `employee_id`, `employee_name`, `department`, `employment_status` = `Active`, and `consent_status` = `Granted` only after the faculty member has given informed consent.
5. The `Face_Descriptors` tab is hidden and protected. Do not share the spreadsheet with ordinary kiosk users or broad groups; spreadsheet editors may still be able to view biometric information despite protections. Limit spreadsheet access to authorised administrators.

## 3. Change the default admin PIN
The initial PIN created by setup is `GJIMT@2026`. Change it before deployment:
1. In Apps Script, select `changeAdminPin` and click Run.
2. Enter a new passphrase of at least 10 characters in the prompt.
3. Do not share this PIN with faculty or embed it in the HTML.

## 4. Deploy the kiosk
1. In Apps Script click **Deploy > New deployment**.
2. Select type **Web app**.
3. Execute as: **Me** (the spreadsheet owner / authorised service account).
4. Who has access: choose the narrowest option available for your institute (prefer signed-in institute users). Avoid public anonymous access if possible.
5. Deploy, authorise, and copy the Web app URL ending in `/exec`.
6. Open that URL in Chrome on the kiosk computer. Camera access requires HTTPS and browser permission.
7. Keep the computer awake and allow the site to use its camera. Use the deployed Apps Script URL, not the GitHub Pages URL or a local `file://` path.

## 5. Enroll faculty
1. In `Employees`, add each faculty member and set employment_status to `Active`.
2. Obtain and record informed consent first; only then set consent_status to `Granted`.
3. Open the kiosk web app and Start Camera.
4. Expand **Administrator: faculty enrollment / reports**.
5. Enter the admin PIN and employee ID, then click **Capture & Enroll Face** while the faculty member is in front of the camera.
6. Repeat for each faculty member. Enrollment is a single sample in this pilot; capture under normal kiosk lighting and re-enroll if matching is unreliable.

## 6. Attendance
- Faculty look at the kiosk camera. A successful match writes one `Face` check-in per employee per calendar day to `Attendance`.
- Duplicate face check-ins on the same date are prevented.
- Today's attendance report is available in the administrator panel.
- This version implements **check-in only**. It does not yet calculate late status, absence, working hours, or check-out. Configure institutional rules before adding those features.

## Important security and privacy limitations
- This is a pilot/prototype, not a certified biometric attendance product. Face descriptors are biometric data. Obtain informed, documented consent; provide an accessible non-biometric alternative; define retention/deletion procedures; and consult your institution's privacy/legal team before deployment.
- The face-api.js descriptor comparison is not proof of liveness. A photo/video spoof may fool basic face recognition. Do not use this alone for payroll, disciplinary action, or other high-impact decisions. Add a tested liveness/anti-spoofing system and independent verification before production use.
- Face descriptors are stored in a hidden/protected sheet in the same spreadsheet. Hiding/protecting a sheet is not encryption or read-access control. Restrict spreadsheet access to a very small trusted admin group. For production, move biometric templates to a dedicated encrypted backend with access controls and audit logging.
- The default matching threshold (0.48) is a pilot setting, not universally valid. Test false accepts and false rejects with the actual camera, distance, lighting, and faculty population; adjust only after controlled evaluation.
- Keep Apps Script deployment access restricted to institute accounts if feasible. The app executes as its owner, so a broadly shared URL can allow outsiders to cause writes even though they cannot see the sheet directly.
- The browser loads face-api.js and model weights from public CDNs; the kiosk needs internet access. Pin and self-host reviewed library/model assets for a controlled production deployment.

## Troubleshooting
- **Camera blocked:** open the `/exec` URL in Chrome, grant camera permission, and ensure no other app is using the webcam.
- **Models fail to load:** check internet access to jsDelivr and retry.
- **Employee not found:** check exact `employee_id` in Employees (case-insensitive matching is used).
- **Consent error:** set `consent_status` to `Granted` only after actual informed consent.
- **Missing sheet:** run `setupAttendanceDatabase` again from the bound Apps Script project.
- **Authorization error:** redeploy the web app after changing code; check the deployment's execute-as and access settings.
