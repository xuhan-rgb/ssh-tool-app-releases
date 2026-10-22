# Membership API backend

This is a small Python standard-library HTTP server backed by SQLite. It uses the already installed `cryptography` package only for Ed25519 signing. It is a development/self-hosted membership foundation; registration does **not** verify email ownership, and there is no OTP flow, checkout, payment processor, order validation, or automated paid upgrade. Account email is simply the normalized account identity until verification is added.

## Start the server

Use the external Ed25519 issuer PEM already used for app licensing. Keep that private key outside the repository and readable only by the service account. The default bind address is loopback (`127.0.0.1:8088`); expose it to clients only behind a TLS reverse proxy and apply normal host/network protections. The SQLite database defaults to `membership.sqlite3` in the current directory.

Install the tested Python dependency on the server with `python3 -m pip install -r backend/requirements.txt`. Create the database parent directory and give the service user read/write access before starting.

```bash
export MEMBERSHIP_SIGNING_KEY="$HOME/.local/share/ssh-tool-app/licensing/p2p-issuer.pem"
export MEMBERSHIP_DB="/var/lib/ssh-tool-app/membership.sqlite3"
python3 backend/server.py
```

Optional settings are `MEMBERSHIP_BIND`, `MEMBERSHIP_PORT`, and `MEMBERSHIP_DB`. The server binds to `127.0.0.1` and port `8088` when unset. Auth endpoints have a small in-memory limit of 10 requests per source IP per 10 minutes; it resets on restart and is intended as a basic guard, not a replacement for edge rate limiting.

## API

All request/response bodies are JSON. Request bodies are capped at 8 KiB. Passwords must be 8–128 characters. Email is trimmed and lowercased and checked with a basic syntax rule. Passwords use scrypt; sessions are random opaque bearer tokens whose SHA-256 hashes are stored with a 30-day TTL. Error responses use `{ "error": "中文错误信息" }`.

### Register

`POST /v1/register` with `{ "email": "person@example.com", "password": "at-least-8-chars" }` returns HTTP 201 and `{ "token": "…", "email": "person@example.com" }`. Duplicate normalized email returns 409.

```bash
curl -sS http://127.0.0.1:8088/v1/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"person@example.com","password":"correct horse"}'
```

### Login

`POST /v1/login` takes the same fields and returns HTTP 200 with a new `{token,email}` pair. Invalid credentials return 401 with the same generic error for unknown email and wrong password.

```bash
curl -sS http://127.0.0.1:8088/v1/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"person@example.com","password":"correct horse"}'
```

### Start or reuse trial

`POST /v1/trial` requires `Authorization: Bearer <token>` and JSON containing UUID `installation_id` and 64-character lowercase hexadecimal `device_id`. The Android client derives the latter as `SHA256("ssh_tool_app:" + ANDROID_ID)`. It returns `{ "license": "base64url(payload).base64url(signature)" }`. The first eligible call starts a seven-day trial. A unique account trial and unique device record prevent reinstalling the app or creating another account on that device from starting a fresh seven-day period. A later device on the same account receives the original account expiry. Installation UUIDs are normalized to lowercase. Once the original trial expires, `/v1/trial` returns HTTP 409 with `试用已结束，请付费解锁`; it never signs an already expired license.

```bash
curl -sS http://127.0.0.1:8088/v1/trial \
  -H 'Authorization: Bearer YOUR_TOKEN' -H 'Content-Type: application/json' \
  -d '{"installation_id":"00000000-0000-4000-8000-000000000001","device_id":"8c64f57e1e219efbe04c68a755242f88b349cb1e45cdcbb6d0025a01b1197f1e"}'
```

### Retrieve an existing entitlement

`GET /v1/license?installation_id=<uuid>&device_id=<64-lowercase-hex>` requires the same bearer token and returns a newly signed license when the account has an active trial or paid entitlement. It returns 404 when none is active and does not start a trial.

### License signature format

The signed bytes are compact UTF-8 JSON for exactly these fields: `version` (2), `product` (`ssh_tool_app.p2p`), `installation`, `account` (normalized email), `kind` (`trial`, `paid`, or `lifetime`), `issued_at`, and `expires_at`. Times are Unix epoch milliseconds; lifetime licenses use `expires_at: null`. In SQLite, `paid_until = -1` explicitly represents lifetime membership, and subsequent day grants preserve it. The token is unpadded base64url of the JSON, a dot, then unpadded base64url of the Ed25519 signature over those exact JSON bytes.

## Manual paid grants

There is no fake payment endpoint. After independently confirming payment, an operator can grant days to an existing account from a host with access to the same database:

```bash
MEMBERSHIP_DB=/var/lib/ssh-tool-app/membership.sqlite3 \
  python3 backend/server.py grant person@example.com --days 365
```

This extends from the later of the current expiry or the current time. The CLI prints the resulting epoch expiry; the user then calls `GET /v1/license` to obtain a signed `kind: "paid"` license. The grant command does not process payment or send messages.

## Gaps before production account/payment claims

Email verification and OTP recovery are pending. Payment checkout, provider webhook validation, refunds, chargebacks, and an auditable operator grant workflow are also pending. Until those exist, describe accounts as unverified and paid access as manually granted after off-system payment confirmation. Use HTTPS at the deployment edge, backups and access controls for the SQLite file, and an external secret store or tightly permissioned key file for the issuer key.

Run the focused integration tests with:

```bash
python3 -m unittest backend.test_server -v
```

## Android app configuration

Build with `flutter build apk --dart-define=MEMBERSHIP_API_URL=https://your-domain.example`. The API root must use HTTPS, except loopback `http://127.0.0.1` for development. The issuer key must correspond to `lib/services/p2p_license_public_key.dart`; private keys never belong in the APK. Without this compile-time setting, the app deliberately retains the older local trial and manual-code compatibility mode. Do not distribute that mode as a server-enforced trial.

On Android, account management is the small person icon in the home toolbar. About is at the bottom of the account screen. Enabling or saving P2P configuration requires cached account login; ordinary SSH does not. Per-connection P2P settings are in the connection card’s three-dot menu, and the activation-code action is inside those settings. Register/login, then explicitly claim the trial; registration alone does not start it. After a manual grant, sign in again to synchronize rights automatically. On a replacement phone, login automatically obtains a new installation-bound license with the original account entitlement. There is no manual restore/refresh button. Cached licenses are checked locally and do not depend on a live bearer session or reachable API. Offline revocations cannot take effect instantly.

The device hash is an abuse-control identifier, not anonymous data or proof of hardware ownership. Explain its purpose and retention in your privacy notice. Normal same-signature Android reinstall usually retains ANDROID_ID; factory resets, different system users, signing-key changes, and modified clients can bypass this control. This implementation does not collect a MAC address or claim to resist a modified APK. SQLite trial records must survive server restarts and backups/restores.

## Deployed service (81.70.201.249)

The account API runs as `ssh-tool-membership.service`, with code in `/opt/ssh-tool-membership`, a dedicated system user, and its SQLite database in `/var/lib/ssh-tool-membership/membership.sqlite3`. The external signing key is `/etc/ssh-tool-membership/issuer.pem`, readable only by the service user. The API binds only `127.0.0.1:18200` and is exposed by the existing Caddy domain at `https://hitxh.xyz:8443/membership-api`. P2P traffic does not pass through this account server.

Deployment files are maintained under `backend/deploy/`:

- `install.sh`: initial installation and subsequent updates, retaining the database and signing key.
- `ssh-tool-membership.service`: service user, restart policy, and filesystem restrictions.
- `backend.env.example`: loopback port and paths; no secrets.
- `caddy-route.caddy`: path-only reverse proxy; insert inside the existing HTTPS domain block.
- `membership-ca.crt`: **public** CA certificate for this server’s existing Caddy internal CA.
- `app-defines.json`: public API URL and public CA certificate for the app build.

The current server uses an internal CA rather than a publicly trusted browser certificate. The app adds the supplied CA only to its account API HTTP client and still verifies the certificate chain and hostname. It does not disable TLS verification or install a system-wide CA. Opening this domain in an ordinary browser may produce a certificate trust warning. If the server CA is rotated, update the public certificate in the app configuration and rebuild.

Build the configured Android client:

```bash
flutter build apk --debug --dart-define-from-file=backend/deploy/app-defines.json
```

For HTTPS health checks from the project root:

```bash
curl --cacert backend/deploy/membership-ca.crt https://hitxh.xyz:8443/membership-api/health
```

Service management on the server:

```bash
sudo systemctl status ssh-tool-membership
sudo journalctl -u ssh-tool-membership -n 50 --no-pager
sudo -u ssh-tool-membership env MEMBERSHIP_DB=/var/lib/ssh-tool-membership/membership.sqlite3 \
  /opt/ssh-tool-membership/venv/bin/python /opt/ssh-tool-membership/server.py grant person@example.com --days 30
```

The Caddy route overwrites `X-Membership-Client-IP`; the backend trusts it only for loopback connections when `MEMBERSHIP_TRUST_PROXY=1`. Do not expose port 18200 directly or forward this header unchanged from untrusted clients. Account passwords, session tokens, the issuer key, and production SQLite data must never be committed.

To update, upload `server.py`, `requirements.txt` and `deploy/` to a private staging directory and run `sudo deploy/install.sh`. The first installation additionally needs the external issuer key path; later runs keep the installed key. Caddy route changes must be validated before reload. A backup of the pre-deployment Caddyfile is retained on the server at `/etc/caddy/Caddyfile.before-ssh-tool-membership-20261010-1510`.


### Mobile connection verification (2026-10-10)

On the connected Android phone, requests to port 443 failed with `Connection reset by peer`, both through the app's Dart HttpClient and through Android curl. The same CA and API on port 8443 returned HTTP 200 for `/health`; the app's login request using deliberately invalid credentials returned the expected `邮箱或密码错误` response. This isolates the observed failure to the 443 HTTPS request path; it does not identify which network component generated the reset.

The configured app now uses port 8443. `backend/deploy/membership-8443.caddy` contains the additional host block. Port 8443 was already listening; existing domain routes and the 443 account API remain available. The Caddyfile backup before adding this route is `/etc/caddy/Caddyfile.before-membership-8443-20261010-1634`. TLS chain and hostname validation remain enabled.


## Administrator web login

The independent administrator portal is at `https://hitxh.xyz:8443/membership-api/admin/`. It provides administrator login, a paginated list of app users with trial and paid expiry/status, and logout. Dates are displayed in Beijing time. For each existing account, the administrator can select a one-month, one-year, or lifetime gift membership and confirm the grant. Gift memberships are stored separately from purchased rights in `gift_memberships`; the “取消赠送会员” action removes only this gift record. Both operations require administrator authentication, session-bound CSRF validation and explicit confirmation. Existing rights in `accounts.paid_until` remain classified as purchased/manual-paid grants; their origin cannot be inferred retroactively.

Month and year gifts use calendar arithmetic in Beijing time, preserving the time of day and clamping to the last day of the destination month when necessary. One month adds one calendar month; one year adds twelve. Renewal extends from the later of the existing gift expiry or the current time. Lifetime gifts are idempotent and are not shortened by a subsequent month/year gift. The three planned purchase prices are ¥9.90, ¥99 and ¥199; the legacy `grant --days` CLI continues to grant an explicitly specified number of days.

Signed gift licenses include `source: "gift"`. On the next successful login synchronization, a removed gift is replaced by any remaining paid/trial entitlement, or cleared after the API explicitly returns `404` with `code: "no_entitlement"`. HTTP outages, generic 404 responses and errors do not clear cached rights. Offline gift licenses remain valid until synchronization (or their original expiry); cancellation cannot revoke an offline lifetime license immediately.

Administrators are stored separately from app accounts. They cannot be publicly registered or created through the app API. Passwords use scrypt and random salts. Administrator session tokens are stored only as SHA-256 hashes, expire after eight hours, and are issued in Secure/HttpOnly/SameSite=Strict host-only cookies. Login, logout and membership-grant forms validate CSRF tokens. User tables exclude password hashes, device identifiers and session tokens.

Create an additional administrator over SSH using a private password file (12–128 characters):

```bash
sudo -u ssh-tool-membership env MEMBERSHIP_DB=/var/lib/ssh-tool-membership/membership.sqlite3 \
  /opt/ssh-tool-membership/venv/bin/python /opt/ssh-tool-membership/server.py \
  admin-create YOUR_ADMIN_NAME --password-file /path/readable/by/service/private-password-file
```

The CLI refuses to overwrite an existing administrator. The password file must be readable only by its owner and removed after successful account creation. Do not put passwords on the command line or in the repository. The initial administrator credentials are saved outside the repository in the owner's local private directory.

The admin template is `backend/admin.html` and is installed alongside `server.py` by `backend/deploy/install.sh`. The deployment database backup is `/var/backups/ssh-tool-membership-20261010-1654/membership.sqlite3`, in a root-only directory. Existing Caddy routes, app accounts, signed licenses and issuer key are retained. The portal uses the same internal CA as the API, so ordinary browsers still require trust in that CA; a public browser-trusted certificate has not been provisioned.

Verification: `python3 -m unittest backend.test_admin backend.test_server -v`. The deployed HTTPS login, invalid-password rejection, authenticated dashboard, logout and session invalidation were also exercised through HTTP requests. No browser visual verification was performed for this task.


## Alipay QR purchases

The app’s “开通会员” button requires account login and opens a native purchase screen. `GET /v1/purchases` returns server-priced plans, recent orders, and a `qr_images` map containing the generated PNG for each plan. `POST /v1/purchases` returns an account-bound signed payment preview without writing an unpaid order to SQLite; `POST /v1/purchases/paid` verifies the preview and creates a saved pending-review order only after a valid payment reference is submitted. Repeating the same submission returns the existing review order. User history and the admin list exclude legacy unpaid orders. These endpoints use the existing app bearer session. Selecting a plan shows its payment code without launching Alipay. “保存付款码” saves the selected plan’s code to the Android gallery; on the same phone, users can recognize it from the album in Alipay’s scanner. Submitting a plausible, unused Alipay transaction number grants temporary P2P membership while the order is pending review. The app synchronizes the signed license immediately after submission. Returning from the purchase screen synchronizes the signed entitlement.

The fallback purchase website is `https://hitxh.xyz:8443/membership-api/membership/`. The optional `POST /v1/web-session` flow opens it with a one-use 60-second ticket. The ticket is exchanged for a two-hour Secure/HttpOnly/SameSite=Strict browser session and removed from the URL by redirect. The app bearer token and password never enter the browser URL. Public access displays the plans and an email/password login form. Browser login and registration use the same account database as the Android app; administrators remain separate. Login/registration validate SameSite host-only CSRF cookies and share the existing authentication rate limit. An app-issued ticket still signs in automatically without another password prompt.

Plans are one calendar month (¥9.90), one calendar year (¥99), and lifetime (¥199). Calendar arithmetic uses Beijing time and clamps month-end dates. The amount is stored server-side in integer cents. The current purchased plan displays `membership_label: "已购买"` and a Beijing-time expiry (or “永久有效”); the latest expired purchased plan displays “曾购买，已到期”. Trial and gift rights are not labeled as purchases. All three plans and their ordinary prices remain visible to active members, with `purchasable: false` and no checkout action. Active purchased month/year memberships cannot upgrade or buy another plan until expiry; active purchased lifetime memberships cannot buy again. Expired subscriptions use the ordinary full prices. Trial and gift rights are not treated as an active purchased subscription. Both plan discovery and submission enforce these rules on the server.

The three payment URLs were decoded from the owner-provided amount-specific codes and are stored in `ALIPAY_PAYMENT_URLS` in `backend/server.py`: month → `fkx14511zojrg8okjbkl732`, year → `fkx18067ik03xkdzqhu2v1f`, lifetime → `fkx15851ytz10ychtyjol6d`, each under `https://qr.alipay.com/`. `qrcode[pil]` generates and caches the PNG bytes in memory; no original image asset is deployed. `GET /membership/alipay-qr.png?plan=month|year|lifetime` serves the matching image for the fallback website. The source amounts were ¥9.90, ¥99.00, and ¥199.00. Users must check the recipient and amount in Alipay. These personal collection links have no merchant payment callback or automatic order reconciliation. After paying, users submit a transaction/reference string. Selecting a plan does not create an unpaid order. Submitting the signed preview and valid reference creates a `pending_review` record without changing `paid_until`; validation failure rolls back the transaction without leaving an unpaid record. Pending review grants a signed `kind: "paid", source: "pending_review"` temporary license with a renewable 24-hour offline lease. There is no total review deadline: the lease renews while any account order remains pending. Approval grants the purchased calendar-based membership; rejection removes this temporary entitlement on the next successful sync, or when its cached lease expires. Existing paid, gifted and trial rights are preserved. Transaction numbers must be 16–64 ASCII digits with a valid YYYYMMDD prefix between the payment preview selection date and today in Beijing time. This is an application plausibility rule, not proof of payment or a guarantee about every Alipay number format. A transaction number cannot be reused across orders/accounts; processed orders cannot be submitted again. Exact payment time, amount, recipient and payment status still require administrator verification in Alipay.

At `https://hitxh.xyz:8443/membership-api/admin/orders`, administrators verify actual Alipay receipts and approve or reject the order. Approval and account entitlement updates share one SQLite transaction; repeated approval of a completed order cannot grant twice. The reviewer and review time are recorded. Approved rights are stored separately from free gifts and cannot be removed by “取消赠送会员”. Approval extends the existing paid expiry by the selected calendar period, or grants lifetime rights. No Alipay merchant API or automatic payment verification is claimed.

The phone account screen displays the signed expiry timestamp in Beijing time; permanent licenses display “永久有效”. Opening this screen or returning from the native purchase screen initiates best-effort account synchronization. Offline failures preserve valid cached rights. The server still uses its internal CA, so external browsers may require trusting that CA; the app's API client already trusts the configured public CA certificate.

Legacy unpaid orders were removed from the deployed database after a consistent SQLite backup at `/var/backups/ssh-tool-membership-no-unpaid-20261010-210555/membership.sqlite3`; pending-review and completed records are retained. Selecting a plan now creates no persistent unpaid records.

Focused verification: `python3 -m unittest backend.test_purchase backend.test_admin backend.test_server -q` and the membership Dart tests.


### Browser login correction

The original public purchase page had no browser login controls and only offered app ticket handoff. It now includes “登录账号” and “注册并登录”, so visiting the purchase URL directly can authenticate the same email account as the app. Wrong credentials show “邮箱或密码错误”; registration does not start the trial. Passwords are not retained in browser URLs, returned markup, logs, or account sessions.
