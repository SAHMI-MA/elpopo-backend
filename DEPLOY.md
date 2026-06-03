# Deploying ELPOPO Academy on Hostinger

This project is two parts that ship together as **one Node process**:

- [frontend/](frontend/) — the static marketing site (`index.html`, CSS, JS, assets).
- [backend/](backend/) — Express server that exposes the payment API **and**
  serves the frontend as static files.

The backend in [backend/server.js](backend/server.js) calls
`express.static('frontend')` itself, so the frontend and the API live on the
**same origin**. No CORS gymnastics, no separate static host needed.

---

## 1. Pick the right Hostinger plan

| Hostinger plan | Works for this app? | Why |
|---|---|---|
| Web Hosting (Single / Premium / Business) | ❌ **No** | Shared/PHP only — cannot run Node.js. |
| **Cloud Hosting** (Startup / Pro / Enterprise) | ✅ Yes | Has the Node.js Selector. |
| **VPS** (KVM 1 / 2 / 4 / 8) | ✅ Yes (recommended) | Full SSH; install Node, run `pm2`, configure Nginx yourself. |

For real Stripe + PayPal payments you need **Cloud Hosting or VPS**.
The cheapest VPS (**KVM 1**) is more than enough for this app.

---

## 2. Files to upload

Upload the **whole project root** EXCEPT the items in the "Do NOT upload"
table below. The relative structure on the server must look like:

```
/<your-app-root>/
├── backend/
│   ├── data/         (products.js — keep; orders.json is created at runtime)
│   ├── middleware/
│   ├── routes/
│   ├── package.json
│   ├── package-lock.json
│   └── server.js
└── frontend/
    ├── index.html
    ├── success.html
    ├── cancel.html
    ├── robots.txt
    ├── sitemap.xml
    ├── css/
    ├── js/
    └── assets/
```

### Do NOT upload

| Item | Reason |
|---|---|
| `backend/node_modules/` | Reinstalled on the server with `npm install --production`. |
| `backend/.env` (if present locally) | Set env vars in Hostinger's UI instead. **Never** upload `.env`. |
| `backend/server.log` | Local debug log (if present). |
| `frontend/assets/videos/README.md` | Internal doc — optional. |
| `frontend/css/polish.css` | No longer exists — was merged into `styles.css`. Skip if you see an old copy. |
| `.env` at project root | Same rule as `backend/.env` — server reads from process env in production. |
| `.DS_Store`, `Thumbs.db`, `.vscode/`, `.git/` | Standard exclusions. |

---

## 3. Environment variables — set in Hostinger, not in a file

In Hostinger's **Node.js application** panel (Cloud) or in `pm2`'s ecosystem
file (VPS), set these variables. **All keys are server-side only — never paste
secret keys into `frontend/`.**

```ini
# REQUIRED
NODE_ENV=production
PORT=5000                                   # or whatever Hostinger assigns
PUBLIC_BASE_URL=https://your-real-domain.com

# Stripe (LIVE keys for production)
STRIPE_SECRET_KEY=sk_live_xxxxxxxxxxxxxxxxx
STRIPE_PUBLISHABLE_KEY=pk_live_xxxxxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxx

# PayPal (LIVE)
PAYPAL_CLIENT_ID=AeXXXXXXXXXXXX
PAYPAL_CLIENT_SECRET=ELXXXXXXXXXXXX
PAYPAL_MODE=live

# Business identity (used in receipts, page titles, brand_name for PayPal)
BUSINESS_NAME=ELPOPO Academy
BUSINESS_EMAIL=elpopo@your-real-domain.com

# Optional: enable the /api/orders/summary admin endpoint
# Generate with: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
ADMIN_TOKEN=
```

`server.js` derives `FRONTEND_URL`, `BACKEND_URL`, `SUCCESS_URL`, `CANCEL_URL`
from `PUBLIC_BASE_URL` automatically. You only need to set the four listed
above unless you have a split-deployment edge case.

---

## 4. Hostinger Cloud Hosting — exact steps

The simplest layout: point the Node.js app at the **`backend/` folder**, and
place **`frontend/` as its sibling**. The server resolves the frontend via
`path.join(__dirname, '..', 'frontend')`, so this requires zero code changes.

1. **Upload** your project so on the server it looks like:
   ```
   /home/u123456789/domains/your-real-domain.com/
   ├── backend/      ← all files from local backend/
   └── frontend/     ← all files from local frontend/
   ```
   Use Hostinger's **File Manager** or any FTP client. **Do NOT upload**
   `backend/node_modules/`, `backend/.env`, `backend/server.log`.

2. **hPanel → Advanced → Node.js → Create Application**:
   - Node.js version: **20.x** (or 18.x minimum)
   - Application mode: **Production**
   - Application root: **`domains/your-real-domain.com/backend`**
     (point it at the `backend/` folder, NOT the project root)
   - Application URL: pick your domain
   - Application startup file: **`server.js`**

3. **Set environment variables** (the table in section 3) in the same panel.

4. Click **Run NPM Install**. Hostinger runs `npm install --production` in
   `backend/`. This succeeds because `backend/package.json` is right there.

5. Click **Start App**.

6. Verify: visit `https://your-real-domain.com` — the marketing page loads.
   Then `https://your-real-domain.com/api/products` should return JSON with
   the 5 products.

## 4'. Hostinger VPS — exact steps

```bash
# 1. SSH in
ssh root@your-vps-ip

# 2. Install Node 20 and pm2
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs nginx
npm install -g pm2

# 3. Upload your project (use scp, rsync, or git clone). Example:
#    From your laptop:
#    scp -r backend frontend root@your-vps-ip:/var/www/elpopo/

# 4. Install deps
cd /var/www/elpopo/backend
npm install --production

# 5. Create the env file (NEVER commit this)
cat > .env <<'EOF'
NODE_ENV=production
PORT=5000
PUBLIC_BASE_URL=https://your-real-domain.com
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
PAYPAL_CLIENT_ID=...
PAYPAL_CLIENT_SECRET=...
PAYPAL_MODE=live
BUSINESS_NAME=ELPOPO Academy
BUSINESS_EMAIL=elpopo@your-real-domain.com
EOF
chmod 600 .env

# 6. Start with pm2, set it to auto-restart on boot
pm2 start server.js --name elpopo
pm2 save
pm2 startup     # follow the printed command to enable auto-restart
```

Then put Nginx (or Hostinger's bundled reverse proxy) in front of port 5000
and terminate SSL with Let's Encrypt (`certbot --nginx -d your-real-domain.com`).
Hostinger's docs cover the Nginx+certbot setup step-by-step for VPS.

---

## 5. Frontend → backend connection

`frontend/js/config.js` ships with:

```js
apiBaseUrl: '',           // same-origin
stripePublishableKey: 'pk_test_...',
```

Because the backend serves the frontend on the same origin, `apiBaseUrl: ''`
is correct as-is. **You only change it** if you ever decide to host the
frontend on a different domain than the backend — then set it to the backend
URL, e.g. `apiBaseUrl: 'https://api.your-real-domain.com'`.

**You DO need to update** `frontend/js/config.js`:

- `stripePublishableKey`: replace the demo `pk_test_…` with your real
  `pk_live_…` before going live.
- `businessEmail`, `whatsappNumber`, `calendlyUrl`: replace placeholders with
  real values.

That's the only frontend file you touch for production. Every other
secret (Stripe secret key, PayPal secret, webhook secret) lives only in
the backend env vars — never in the frontend bundle.

---

## 6. Stripe webhook setup (do this AFTER the app is live)

1. In Stripe Dashboard → **Developers → Webhooks → Add endpoint**:
   - Endpoint URL: `https://your-real-domain.com/webhook/stripe`
   - Listen for events: `checkout.session.completed`, `payment_intent.succeeded`,
     `payment_intent.payment_failed`, `charge.refunded`.
2. Copy the **Signing secret** (starts with `whsec_…`).
3. Paste it as `STRIPE_WEBHOOK_SECRET` in your Hostinger env vars.
4. **Restart the app** so the new env var loads.

Without a valid webhook secret the backend rejects all webhooks (by design)
and orders stay in `pending` state forever.

---

## 7. PayPal live mode setup

1. Get **live** credentials from PayPal Developer → My Apps & Credentials → **Live** tab.
2. Set `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET`, and `PAYPAL_MODE=live` in env vars.
3. Restart the app.

The backend automatically switches between `api-m.sandbox.paypal.com` and
`api-m.paypal.com` based on `PAYPAL_MODE`. No code change needed.

The PayPal client ID is safe to expose publicly (it's a public identifier).
The client secret is **backend-only** — never paste it in any frontend file.

---

## 8. Testing the live payment flow

Once the app is reachable at `https://your-real-domain.com`:

### Smoke tests

```
GET  https://your-real-domain.com/             → marketing page loads
GET  https://your-real-domain.com/api/products → JSON list of 5 products
GET  https://your-real-domain.com/success.html → "Payment received" page
GET  https://your-real-domain.com/cancel.html  → "Checkout canceled" page
```

### Stripe end-to-end

1. Click any pricing tier's "Pay Securely" button on the site.
2. Enter name, email, real card. Submit.
3. On success the modal flips to the success state and `payment_intent.succeeded`
   fires the webhook.
4. Stripe Dashboard → Payments shows the charge.
5. (If `ADMIN_TOKEN` set) `curl -H "x-admin-token: $ADMIN_TOKEN" https://your-real-domain.com/api/orders/summary`
   should show `paid: 1`.

### PayPal end-to-end

1. Click "Pay with PayPal" in the checkout modal.
2. Browser redirects to PayPal, log in, approve.
3. PayPal redirects back to `/api/checkout/paypal/capture?token=…`.
4. Backend captures, verifies amount, marks order as paid, redirects to
   `/success.html`.

### First real charge

Run a tiny live charge ($1 product) using your own card to confirm:
- Stripe webhook delivers and is accepted (no 400 in logs).
- The success page shows.
- Email receipt arrives.

Then turn the price back up and you're live.

---

## 9. SSL, cache, and Hostinger limitations

### SSL is required
Stripe Elements **refuses to load over plain HTTP** outside `localhost`.
- Cloud Hosting: free Let's Encrypt SSL via hPanel → **SSL**. Force HTTPS = ON.
- VPS: `certbot --nginx -d your-real-domain.com` after Nginx is configured.

### Caching
- The Express server doesn't set caching headers itself. On Hostinger Cloud
  this is fine. On a VPS with Nginx, add `expires 1y;` for `/assets/` and
  `expires -1;` for `*.html` so config updates ship instantly.
- Enable gzip/brotli at the reverse-proxy layer for ~70% payload reduction.

### Limitations to be aware of

| Limitation | Impact | Fix when ready |
|---|---|---|
| Orders are stored in `backend/data/orders.json` | Survives restarts on a VPS or Cloud disk; if Hostinger's disk is wiped on plan changes, the local order log is lost. Stripe + PayPal dashboards keep the canonical record. | Move to a real DB (Postgres, MySQL, MongoDB) by replacing `backend/routes/orders.js`. |
| Rate limiting is in-process | Works on a single Node process. If you ever run multiple instances behind a load balancer they won't share counters. | Add `rate-limit-redis` or similar. |
| Single Node process | If Node crashes, the site is down until `pm2` restarts it (a few seconds). | `pm2`'s default restart policy already handles this. |

---

## 10. Pre-launch checklist

- [ ] Hostinger plan supports Node.js (Cloud or VPS).
- [ ] Real domain replaces `elpopoacademy.com` placeholders in:
  - `frontend/index.html` (10 occurrences total — 4 in `<head>` meta tags, 6 in the JSON-LD block at the end of `<body>`)
  - `frontend/sitemap.xml`
  - `frontend/robots.txt`
- [ ] `frontend/js/config.js` has `pk_live_…` (not `pk_test_…`) and a real
  Calendly URL, real WhatsApp number, real business email.
- [ ] All 6 video poster `data-yt` / `data-video-yt` attributes filled (or
  left empty — the JS no-ops empty IDs safely).
- [ ] Real Terms / Privacy / Refund pages created and footer links updated.
- [ ] Backend env vars set in Hostinger UI (section 3 above).
- [ ] Stripe webhook endpoint registered (section 6).
- [ ] PayPal in `PAYPAL_MODE=live` (section 7).
- [ ] SSL certificate active.
- [ ] `https://your-real-domain.com/api/products` returns JSON.
- [ ] One end-to-end live test purchase from your own card succeeds.

When every box is checked, the site is ready.

---

# Appendix A — Troubleshooting & common mistakes

The 12 things that go wrong most often when deploying this project. Skim
before the deploy call so you recognise the symptoms in real time.

## Mistakes that block deployment from succeeding

### 1. Picking a shared / PHP Hostinger plan
**Symptom:** Node.js app cannot be created in hPanel, or it appears to start
but the URL returns a PHP default page.
**Fix:** Buy Cloud Hosting (Startup or above) or VPS (KVM 1+).

### 2. Uploading `backend/node_modules/`
**Symptom:** FTP upload takes hours; on first start the app crashes with
"binding not found" or OS-mismatch errors.
**Fix:** Never upload `node_modules`. Click **Run NPM Install** in
Hostinger's Node.js panel — it installs the correct Linux binaries on the
server.

### 3. Uploading a `.env` file via FTP
**Symptom:** Secrets sit in the filesystem in plaintext and may be served by
the static handler. Hostinger's Node.js Selector reads env vars from its
own UI, not from a `.env` file in some configurations.
**Fix:** Set every variable in **hPanel → Node.js → Environment variables**.
Never upload `.env`.

### 4. Pointing the Node app at the project root instead of `backend/`
**Symptom:** "Run NPM Install" fails with `ENOENT package.json`. App will
not start.
**Fix:** Application root must be `domains/theirdomain.com/backend`, NOT
the project root. Startup file is `server.js` (relative to that root).

### 5. Mixing test and live keys
**Symptom:** Stripe rejects the charge with an opaque error like
"Authentication required" or "Invalid API key for this environment".
**Fix:** Frontend `pk_live_…` MUST be paired with backend `sk_live_…`. Same
for sandbox/live PayPal. Triple-check both ends use the same mode.

## Mistakes that break the live site

### 6. Forgetting to set `PUBLIC_BASE_URL`
**Symptom:** Stripe Checkout Session creation returns 500. The Stripe Pay
button does nothing.
**Fix:** Set `PUBLIC_BASE_URL=https://www.theirdomain.com` in env vars and
restart. The `productionSanityCheck` in `server.js` logs this if missed.

### 7. Skipping SSL / Let's Encrypt
**Symptom:** The Stripe card field never appears inside the checkout modal.
Console shows "Stripe.js refused to load over insecure context".
**Fix:** hPanel → SSL → **Install free SSL** (Let's Encrypt) → Force HTTPS.
Wait ~5 min for cert propagation.

### 8. Registering the Stripe webhook before SSL is active
**Symptom:** Stripe rejects the webhook endpoint because the URL is not yet
HTTPS.
**Fix:** Order matters: deploy → SSL active → THEN register the webhook.

### 9. Forgetting to restart the Node app after changing env vars
**Symptom:** New keys/URLs don't take effect.
**Fix:** Hostinger does NOT auto-reload on env changes. Click **Restart**
in the Node.js panel after every env var edit.

### 10. Stripe webhook configured but `STRIPE_WEBHOOK_SECRET` env var empty
**Symptom:** Orders never flip to "paid" — they stay in "pending" forever.
The backend rejects unsigned webhooks (by design).
**Fix:** Copy the `whsec_…` from Stripe Dashboard into
`STRIPE_WEBHOOK_SECRET`, restart the app, click "Resend" on the failed
webhook in Stripe Dashboard.

### 11. PayPal still in sandbox mode
**Symptom:** PayPal redirect goes to `sandbox.paypal.com`. Real PayPal
accounts can't log in.
**Fix:** Set `PAYPAL_MODE=live` explicitly. Default is `sandbox` if unset.

### 12. Forgetting to update the 8 domain placeholders
**Symptom:** Site works for payments, but social shares, sitemap, and
canonical link all point to `elpopoacademy.com` (the placeholder).
**Fix:** Find/replace `https://www.elpopoacademy.com` →
`https://www.theirdomain.com` in `frontend/index.html` (10 occurrences —
canonical / OG / Twitter in `<head>` plus the JSON-LD block at the bottom
of `<body>`), `frontend/sitemap.xml`, and `frontend/robots.txt`.

## Sanity check before saying "we're done"

| Check | Expected | If broken, see |
|---|---|---|
| Browser address bar shows padlock? | HTTPS active | #7 |
| `https://theirdomain.com/api/products` returns JSON? | Backend running, env loaded | #4, #6 |
| Pricing modal opens, card field visible? | Stripe.js loaded over HTTPS | #5, #7 |
| Real $1 charge from your own card succeeds? | All keys are live mode | #5 |
| Stripe Dashboard → Webhooks shows green 200? | `STRIPE_WEBHOOK_SECRET` matches | #10 |
| PayPal flow redirects to `paypal.com` (not sandbox)? | `PAYPAL_MODE=live` | #11 |

If all six pass, the deployment is real.

## Less common issues

- **"Run NPM Install" fails with engine version errors** — Hostinger's Node
  version is older than what the package needs. hPanel → Node.js → change
  to **20.x**.
- **Calendly modal shows a blank iframe** — domain in `config.js` is wrong,
  or CSP blocks it. Verify the URL works in a regular tab first. The CSP in
  `index.html` already permits Calendly.
- **Form submissions don't arrive** — the Formspree endpoint `xeojknpb` in
  `frontend/js/app.js` may be a demo. Create a real form at formspree.io,
  paste its ID into `app.js`.
- **Rate limit hits during testing** — backend limits `/api/checkout/*` to
  20 req per 10 min per IP in production. Wait, or set `NODE_ENV=development`
  while testing.
- **Order status stuck at "pending"** — webhook misconfigured (see #10).
  Stripe Dashboard → Webhooks → failed delivery → "Resend". If it now
  returns 200, future webhooks work too.

---

# Appendix B — Client input checklist

Send this to the client before the deploy call.

```
LOGIN ACCESS
[ ] Hostinger hPanel login (or sub-user invite)
[ ] Domain registrar login (only if domain is NOT with Hostinger)

DOMAIN
[ ] The exact domain (e.g. elpopoacademy.com)
[ ] Confirmed: use www. or root domain?

STRIPE LIVE (from dashboard.stripe.com/apikeys with test toggle OFF)
[ ] Publishable key  pk_live_...
[ ] Secret key       sk_live_...

PAYPAL LIVE (from developer.paypal.com → My Apps → Live tab)
[ ] Client ID
[ ] Client Secret

CONTACT
[ ] Real business email
[ ] WhatsApp number with country code (digits only)
[ ] Calendly event link

LEGAL (optional, can come after launch)
[ ] Terms of Service URL
[ ] Privacy Policy URL
[ ] Refund Policy URL
[ ] Results Disclaimer URL

OPTIONAL
[ ] Real YouTube video IDs for the 6 video sections
[ ] Replacement footer logo seal (current is 1320×2136 — too large)
```

The first **8 items** are blockers. Everything else can land after launch.
