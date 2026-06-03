# Client-Ready Store Template

Static HTML/CSS/JS landing page + Node.js/Express backend with **Stripe** and
**PayPal** checkout. Designed so the client only edits configuration files —
the core code stays untouched.

The template currently ships with the ELPOPO Academy design, but the same
structure works for any store: edit the config files (described below), drop
in new images/videos/products, and re-deploy.

---

## Project structure

```
project/
├── frontend/
│   ├── index.html              # main landing page
│   ├── success.html            # post-payment success page
│   ├── cancel.html             # post-payment cancel page
│   ├── css/
│   │   └── styles.css
│   ├── js/
│   │   ├── config.js           # ◀ CLIENT EDITS THIS
│   │   ├── app.js              # page interactions
│   │   └── payments.js         # redirect-style payment helpers
│   └── assets/
│       ├── images/             # logo + photos
│       └── videos/             # videos (see videos/README.md)
└── backend/
    ├── server.js
    ├── routes/
    │   ├── stripe.js           # Stripe Checkout + webhook + PaymentIntent
    │   ├── paypal.js           # PayPal Orders v2 (create + capture)
    │   └── orders.js           # JSON file order store
    ├── middleware/
    │   └── security.js         # helmet + CORS + rate limiting
    ├── data/
    │   ├── products.js         # ◀ CLIENT EDITS PRICES HERE
    │   └── orders.json         # auto-created
    ├── .env.example            # ◀ CLIENT COPIES TO .env AND EDITS
    └── package.json
```

---

## Test Stripe locally in 5 minutes

```
cd backend
npm install
cp .env.example .env          # Windows: copy .env.example .env
# edit backend/.env, paste Stripe test keys, leave PUBLIC_BASE_URL=http://localhost:5000
npm start                     # http://localhost:5000
```

The only URL you have to set is **`PUBLIC_BASE_URL`** in `backend/.env`.
Everything else (`SUCCESS_URL`, `CANCEL_URL`, `FRONTEND_URL`, `BACKEND_URL`)
is derived from it automatically. To deploy on a real domain later, change
just that one line.

In a **second terminal**, forward webhooks so order status updates land:

```
stripe login                  # one-time
stripe listen --forward-to localhost:5000/webhook/stripe
```

Copy the `whsec_...` value the CLI prints into `STRIPE_WEBHOOK_SECRET`
in `backend/.env`, then restart `npm start`.

Open http://localhost:5000, click any pricing tier's "Buy" / "Enroll"
button, fill in name + email, and enter a Stripe test card:

| Card | Result |
| --- | --- |
| `4242 4242 4242 4242` | Succeeds |
| `4000 0000 0000 9995` | Declined (insufficient funds) |
| `4000 0025 0000 3155` | Requires 3DS authentication (test the OTP) |

Any future expiry, any 3-digit CVC, any postal code.

You should see:

1. The modal show its success state ("Payment confirmed").
2. `[stripe] payment_intent.succeeded` in the `stripe listen` terminal.
3. A new row in `backend/data/orders.json` with `"status": "paid"`.

**To test the Stripe Checkout redirect flow + success.html page,** run this
in the browser console while on the home page:

```js
Payments.startStripeCheckout({ productKey: 'playbook', name: 'Test', email: 'test@example.com' })
```

You'll be redirected to a Stripe-hosted page; after paying with `4242
4242 4242 4242` you'll land on `/success.html`. Cancelling sends you to
`/cancel.html`.

---

## 1. Install

```
cd backend
npm install
```

Requires **Node.js 18+** (uses the built-in `fetch`).

## 2. Configure backend secrets

```
cd backend
cp .env.example .env       # macOS / Linux
copy .env.example .env     # Windows
```

Edit `backend/.env` and fill in:

| Variable | Where to get it | Notes |
| --- | --- | --- |
| `STRIPE_SECRET_KEY` | https://dashboard.stripe.com/apikeys | `sk_test_...` for testing, `sk_live_...` for production |
| `STRIPE_PUBLISHABLE_KEY` | same page | `pk_test_...` / `pk_live_...` — must also be set in `frontend/js/config.js` |
| `STRIPE_WEBHOOK_SECRET` | `stripe listen` CLI (dev) or Dashboard → Webhooks (prod) | `whsec_...` |
| `PAYPAL_CLIENT_ID` | https://developer.paypal.com/dashboard/applications/ | |
| `PAYPAL_CLIENT_SECRET` | same page | keep secret |
| `PAYPAL_MODE` | — | `sandbox` for testing, `live` for production |
| `BUSINESS_NAME` / `BUSINESS_EMAIL` | — | Shown on receipts + PayPal page |
| `SUCCESS_URL` / `CANCEL_URL` | — | Full URLs Stripe + PayPal redirect to |
| `FRONTEND_URL` | — | Used for CORS whitelist |
| `BACKEND_URL` | — | Optional; used as PayPal `return_url` base. Auto-detected if empty. |
| `PORT` | — | Default 5000 |

## 3. Run

Single-server mode (backend also serves the frontend on the same port):

```
cd backend
npm start
```

Open http://localhost:5000

Split mode (frontend on a separate static server, backend on :5000):

```
# terminal 1
cd backend && npm start
# terminal 2
cd frontend && npx http-server -p 3000   # or any static server
```

Then in `frontend/js/config.js` set `apiBaseUrl: 'http://localhost:5000'`.

## 4. Forward Stripe webhooks (local dev)

```
stripe login
stripe listen --forward-to localhost:5000/webhook/stripe
```

Copy the `whsec_...` value into `STRIPE_WEBHOOK_SECRET` in `.env` and
restart the server.

---

## How the client customises the site

Everything below is editable **without touching the core code**.

### Business identity, logo, contacts → `frontend/js/config.js`

```js
window.SITE_CONFIG = {
  businessName: 'Your Store Name',
  businessEmail: 'hello@yourstore.com',
  phoneNumber: '+1 555 123 4567',
  whatsappNumber: '15551234567',
  logo: 'assets/images/logo.png',
  social: { instagram: 'https://...', facebook: 'https://...' },
  // ...
}
```

`app.js` reads this file on load and injects the values into every element
marked with `data-cfg-business-name`, `data-cfg-logo`, every `mailto:`,
`tel:`, and `wa.me/` anchor, and any `<video data-cfg-video="hero">` tag.

### Booking calendar → `frontend/js/config.js` (`calendlyUrl`)

The "Book Strategy Call" modal embeds a Calendly event. The template ships
with a **temporary demo URL** (`https://calendly.com/aji/30min`) so the
modal renders out of the box — **the client must replace it** with their
own Calendly link before going live, or the booking modal will load the
wrong account.

```js
// CLIENT: Replace this with your real Calendly event link.
// Example: https://calendly.com/yourname/30min
calendlyUrl: 'https://calendly.com/aji/30min',
```

**Where to get the real link:** log into [calendly.com](https://calendly.com)
→ Event Types → click the gear (⚙) on the meeting you want → **Copy link**.
Paste it into the value above. No other code change is needed — the modal
will pick it up on next page load.

### Products and prices → `backend/data/products.js`

The **authoritative source of truth for prices**. The frontend never tells the
backend how much to charge — it sends a `productKey` and the backend looks the
price up in this file before creating a Stripe / PayPal order.

```js
{
  id: 'playbook',
  name: 'Digital Playbook',
  description: '...',
  amountCents: 19900,
  currency: 'usd',
  image: '',           // public URL — shown on Stripe Checkout
  video: '',
  category: 'digital',
}
```

If you also display these in the page, edit the matching `TIERS[]` array in
`frontend/js/app.js` (display copy only) — keep `productKey` in sync with
`id` here.

### Logo / images → `frontend/assets/images/`

Drop in `logo.png`, `logo-seal.jpg`, plus any product/hero photos. Reference
new files from `config.js` (`logo`, `logoSeal`, `images.*`).

> Compress any new logo or product images through TinyPNG or Squoosh
> before going live — typical savings 80%+.

### Videos → `frontend/assets/videos/`

See [`frontend/assets/videos/README.md`](frontend/assets/videos/README.md).
Two options:

1. Replace the file in the folder, keep the same filename — no code change.
2. Edit `SITE_CONFIG.videos.<key>.src` in `config.js`.

Any `<video data-cfg-video="hero">` tag gets its `src` + `poster` injected
at boot.

### Stripe keys → `frontend/js/config.js` + `backend/.env`

| Where | Which key |
| --- | --- |
| `frontend/js/config.js` → `stripePublishableKey` | publishable (`pk_test_...` / `pk_live_...`) — safe in the browser |
| `backend/.env` → `STRIPE_SECRET_KEY` | secret (`sk_test_...` / `sk_live_...`) — server only |
| `backend/.env` → `STRIPE_WEBHOOK_SECRET` | webhook signing secret (`whsec_...`) |

### PayPal keys → `backend/.env`

`PAYPAL_CLIENT_ID` + `PAYPAL_CLIENT_SECRET` + `PAYPAL_MODE` (`sandbox` /
`live`).

### Success / cancel URLs

Two places — keep consistent:

1. `backend/.env` — `SUCCESS_URL` + `CANCEL_URL` (Stripe + PayPal redirect targets)
2. `frontend/js/config.js` — `successUrl` + `cancelUrl` (relative paths the frontend uses)

---

## Payment flows available

### Stripe — two options

| Flow | Endpoint | When to use |
| --- | --- | --- |
| **In-page Stripe Elements** (default) | `POST /api/create-payment-intent` | Card form lives inside your modal. Buyer never leaves the site. |
| **Stripe Checkout (redirect)** | `POST /api/checkout/stripe` | Buyer is redirected to a Stripe-hosted page. Simplest, most secure. |

To switch to redirect mode, set `stripeFlow: 'redirect'` in `config.js` and
wire your "Pay" button to `window.Payments.startStripeCheckout({...})`.

### PayPal — one flow

`POST /api/checkout/paypal` returns an `approveUrl`. The browser redirects
there, the buyer approves on PayPal, PayPal sends them to
`GET /api/checkout/paypal/capture` which captures the order **server-side**,
verifies the amount matches `products.js`, and then redirects to
`SUCCESS_URL` or `CANCEL_URL`.

---

## Production readiness checklist

A single audit list — everything you need to change between local dev and a
public domain. The server runs `productionSanityCheck()` at startup when
`NODE_ENV=production` and will warn (in stdout) about any of these items if
they look unsafe.

### 1. Domain
- [ ] `PUBLIC_BASE_URL` in `backend/.env` is your real `https://` URL. No
      `localhost`, no `http://`. Every other URL (CORS, redirect, PayPal
      return) derives from it.

### 2. Payment keys
- [ ] `STRIPE_SECRET_KEY` is a `sk_live_...` key (not `sk_test_...`).
- [ ] `STRIPE_PUBLISHABLE_KEY` is the matching `pk_live_...` AND the same
      value is mirrored in `frontend/js/config.js` → `stripePublishableKey`.
- [ ] `STRIPE_WEBHOOK_SECRET` is the `whsec_...` from your **production**
      webhook endpoint (Stripe Dashboard → Developers → Webhooks → your
      endpoint at `${PUBLIC_BASE_URL}/webhook/stripe`).
- [ ] `PAYPAL_CLIENT_ID` + `PAYPAL_CLIENT_SECRET` are **live** PayPal
      credentials and `PAYPAL_MODE=live`.

### 3. Server config
- [ ] `NODE_ENV=production` so the sanity check runs.
- [ ] `ADMIN_TOKEN` is a long random string (use the `node -e` command in
      `.env.example` to generate one). Without it the
      `/api/orders/summary` endpoint stays disabled, which is safe but
      means you can't curl it from your laptop.
- [ ] HTTPS is terminated upstream (Render, Heroku, Cloudflare, nginx —
      whichever you use). Stripe.js refuses to mount card fields over
      plain HTTP outside `localhost`.

### 4. Frontend
- [ ] `frontend/js/config.js` → `calendlyUrl` is replaced with the client's
      real Calendly event link (the shipped value is a demo).
- [ ] `frontend/js/config.js` → business name, email, phone, WhatsApp,
      social links match the client.
- [ ] Logo files in `frontend/assets/images/` are the client's, not the
      ELPOPO defaults.

### 5. SEO / domain rewrites
- [ ] Find & replace `https://www.elpopoacademy.com` → your real domain
      across `frontend/index.html` (canonical + OG + JSON-LD),
      `frontend/robots.txt`, `frontend/sitemap.xml`.

### 6. Verification
- [ ] Run an end-to-end Stripe test charge on the live site (`4242 ...`).
- [ ] Confirm the webhook hits and `orders.json` flips to `status: 'paid'`.
- [ ] Run a PayPal sandbox-to-live capture once.
- [ ] Run Lighthouse on the public URL — should hit 90+ on Performance,
      Accessibility, Best Practices, SEO.

---

## Going live checklist

- [ ] **Set `PUBLIC_BASE_URL` in `backend/.env`** to your real production
      URL (e.g. `https://www.your-store.com`). Everything else
      (`SUCCESS_URL`, `CANCEL_URL`, `FRONTEND_URL`, `BACKEND_URL`) inherits
      from it. Leave them blank in `.env` unless you have split-mode needs.
- [ ] Swap Stripe keys to `sk_live_...` / `pk_live_...` (`.env` + `config.js`)
- [ ] Set `PAYPAL_MODE=live` and use live PayPal credentials
- [ ] In the Stripe Dashboard create a webhook endpoint at
      `${PUBLIC_BASE_URL}/webhook/stripe` and put its signing secret in
      `STRIPE_WEBHOOK_SECRET`
- [ ] Serve over HTTPS — Stripe.js refuses to mount card fields over plain HTTP
- [ ] Compress `logo.png` and any large images
- [ ] Verify the test flow end-to-end before announcing the launch:
      `4242 4242 4242 4242` for Stripe, sandbox PayPal account for PayPal

## Verify payments are working

1. `npm start` in `backend/`
2. `stripe listen --forward-to localhost:5000/webhook/stripe`
3. Open the site → click any "Buy" button → pay with `4242 4242 4242 4242`
4. Check the `stripe listen` terminal for `payment_intent.succeeded`
5. Check `backend/data/orders.json` — your order should have `"status": "paid"`
6. For PayPal: use a sandbox buyer account from the PayPal developer dashboard
   → buyer approves → redirected to success page → `orders.json` shows `paid`

---

## Security guarantees built in

- **No secrets in the browser.** `sk_*`, `whsec_*`, PayPal client secret all
  live in `backend/.env` only.
- **Prices verified on the server.** The browser sends a `productKey`; the
  backend looks the amount up in `products.js`. A tampered client can't
  charge a different amount.
- **Webhooks are signature-verified** (`stripe.webhooks.constructEvent`).
- **PayPal captures are verified server-side** — the capture step re-checks
  the amount and currency against `products.js` before marking `paid`.
- **`helmet`** sets security headers (X-Frame-Options, HSTS in prod, NoSniff, …)
- **`cors`** limits which origins can call the API (`FRONTEND_URL`).
- **`express-rate-limit`** — 100 req / 15 min on `/api`, 20 / 10 min on checkout creation endpoints.
- **Input validation** on every payment endpoint (productKey, name, email,
  applyOnly guard, length caps).
- **Safe error handling** — internal error messages never leak to the client.

---

## Deploy

The backend is a plain Node app — deploy it anywhere that runs Node 18+
(Render, Railway, Fly, AWS, a $5 VPS, etc.).

1. `npm install --omit=dev` in `backend/`
2. Set the same environment variables in your host's dashboard
3. Run `node server.js` (or `npm start`)
4. Point your domain at the host; ensure HTTPS is on
5. Update Stripe Dashboard webhook URL to `https://YOUR-DOMAIN/webhook/stripe`
6. Test the flow end-to-end on production with a $0.50 charge before sharing
   the site publicly

---

## What the client must provide before going live

- A Stripe account (and KYC approved for live payouts)
- A PayPal Business account
- A domain name with HTTPS
- Logo + product images (compressed)
- Final product names + descriptions + prices
- Real business email + WhatsApp number
- (Optional) Calendly link if using the booking modal
