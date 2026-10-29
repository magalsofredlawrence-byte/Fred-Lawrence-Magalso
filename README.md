# Safe File Middleman

A beginner-friendly local web app for sending a file to one named recipient. Uploads are stored privately in quarantine and checked before any download is allowed. This is a development MVP, not a guarantee that a file is safe.

## Project structure

```text
safe-file-middleman/
├── app/
│   ├── main.py                 # FastAPI routes, SQLite, uploads, scanning, access checks
│   ├── static/
│   │   ├── app.js
│   │   └── style.css
│   └── templates/              # Home, auth, dashboard, transfer, admin and error pages
├── tests/
│   └── test_security.py        # Authorization, download status and filename checks
├── .env.example
├── .dockerignore
├── .gitignore
├── Dockerfile
├── docker-compose.yml
└── requirements.txt
```

## Run locally with Docker (recommended)

Requirements: Docker Desktop with Compose enabled.

1. Copy `.env.example` to `.env`.
2. Change `APP_SESSION_SECRET` in `.env` to a random value. For example, generate one with `python -c "import secrets; print(secrets.token_urlsafe(48))"`.
3. Optional: configure email delivery as described in [Email notifications](#email-notifications).
4. Start the app and ClamAV:

   ```sh
   docker compose up --build
   ```

   The first start may take a few minutes while ClamAV downloads its signature database. Compose waits for ClamAV to become healthy before starting the web app.
5. Visit [http://localhost:8000](http://localhost:8000).
6. Stop with `Ctrl+C`, or run `docker compose down`. Uploaded data and the ClamAV database remain in Docker volumes. To remove those development volumes, run `docker compose down -v` (this permanently deletes app data).

Sample development accounts (enabled only when `APP_ENV=development` and `SEED_DEMO_USERS=true`):

| Role | Email | Password |
|---|---|---|
| Sender | `alice@example.com` | `password123` |
| Recipient | `bob@example.com` | `password123` |
| Admin | `admin@example.com` | `admin123!` |

Change these before sharing a development instance. In local development, sign up for additional recipient accounts as needed.

## Run locally with Python

Requirements: Python 3.12+ and Docker for ClamAV.

1. Create a virtual environment and install dependencies:

   ```sh
   python -m venv .venv
   # Windows PowerShell:
   .venv\Scripts\Activate.ps1
   # macOS/Linux:
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

2. Copy `.env.example` to `.env`, set a random session secret, and set `CLAMAV_HOST=127.0.0.1`.
3. Start ClamAV:

   ```sh
   docker compose up -d clamav
   ```

   The ClamAV port is bound to the local machine only (`127.0.0.1`).

4. In a second terminal, activate the virtual environment and start FastAPI:

   ```sh
   uvicorn app.main:app --reload
   ```

5. Open [http://localhost:8000](http://localhost:8000).

Wait for ClamAV to finish starting before uploading. The app expects the scanner at `127.0.0.1:3310`. If ClamAV is offline, the upload is still saved in private quarantine and its status stays **queued**; it is not released or approved. The background queue worker retries queued scans every 30 seconds. An admin can also start the scanner and select **Run scan** on the Admin page. A retry never skips scanning or directly approves a file.

## Email notifications

Email is optional and disabled when `SMTP_HOST` is empty. Without SMTP configuration, no email is sent; the transfer page and admin page show this explicitly, and the event is recorded in the audit log. Recipients can still see approved transfers by signing in or registering with the exact addressed email and opening their dashboard. Set the SMTP values in `.env` before starting the app, then restart it. For Gmail, use `SMTP_HOST=smtp.gmail.com`, `SMTP_PORT=587`, `SMTP_STARTTLS=true`, your Gmail address as `SMTP_USERNAME` and `SMTP_FROM_EMAIL`, and a Google App Password as `SMTP_PASSWORD` (Google requires 2-Step Verification to create an App Password). Do not use your normal Gmail password. Keep `.env` private; it is excluded from Git. The app cannot send real email until you configure a real mail provider and valid credentials.

The recipient receives a plain-text notice only after a scan passes. Email contains no attachment or public download link. Recipients sign in with the addressed email and still pass the normal authorization check. Gmail or GitHub-associated email addresses can receive notifications; this does not add Google or GitHub account sign-in. Set `APP_BASE_URL` to the app's reachable HTTPS address for a deployed service; `localhost` links work only on the computer running the app and cannot be used from another person's device. Recipients can also see transfers in their dashboard after registering with the same email address.

## Profile

Signed-in users can open **Profile** in the top navigation and change their display name. Their account email remains the transfer identity so incoming files and recipient permissions keep matching the correct account.

## Run tests

With the project environment active, run:

```sh
pytest -q
```

The test suite uses a temporary SQLite database and private storage directory. It does not need a live ClamAV service.

## File workflow and security notes

- Uploads are generated under private `data/private_uploads/quarantine` names. The original, sanitized filename is shown to users; it is never used as the storage path.
- Transfer rows start as `uploading`, then become `queued`. A background task changes them to `scanning` and then to `approved`, `suspicious`, `blocked`, or `scan_failed`. If the ClamAV service is unreachable, the file returns to `queued` with a clear message instead of being marked as a completed scan. Flagged files remain quarantined. No admin control can approve them. Admins can retry queued scans or delete flagged files.
- The default allowlist is `.pdf`, `.txt`, `.csv`, `.jpg`, `.jpeg`, `.png`, `.docx`, and `.xlsx`, with an upload limit of 25 MiB. Other extensions are blocked. File signatures and text encoding are checked with a pure-Python MIME detector; Office formats are checked for expected ZIP package entries. ClamAV scans streamed file content using its INSTREAM protocol. If ClamAV is unavailable, the file remains in quarantine and queued; it cannot be downloaded.
- The approved notice is: “Automated checks found no known threat, but no scan can guarantee a file is safe.” Failed files show: “This file was not approved by automated security checks and is unavailable for download.”
- Only the sender and intended recipient can view a transfer. The intended recipient must have an account for the exact recipient email. Only that recipient can download an approved file. Authorization and status are checked again on every download request.
- Passwords use bcrypt hashes. Sessions are signed, `HttpOnly`, `SameSite=Lax` cookies; set `SESSION_HTTPS_ONLY=true` when serving through HTTPS. Forms use CSRF tokens.
- Login and uploads have per-process, in-memory rate limits. They are suitable for a local MVP, not a multi-worker/production deployment.
- Upload, checking, scan result, status, download and deletion events are recorded in SQLite audit events. The admin page is role-restricted and only lists suspicious, blocked, or failed scans.
- The SQLite database and private files are stored under `data/`, ignored by Git. Never commit `.env`, session secrets, user files, or a development database.

## Limitations

This is a small local-development MVP. Background work runs in the web process; rate limits are process-local; malware-signature freshness monitoring, a retention scheduler, external identity providers, and hardened production operations are not included. SMTP notifications require your own provider configuration and are not an authentication mechanism. There is no public sharing link, payment processing, escrow, or automatic release of suspicious files. Do not expose it directly to the internet or use it for sensitive/regulated data without a security review and production-grade storage, queueing, monitoring, backups, TLS, and operational controls.
