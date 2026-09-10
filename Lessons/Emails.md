# Sending Emails (Transactional) — Day 5

⭐️ **GOAL:** Leave able to pick a 2026 provider, keep secrets server-side, send HTML+text via Nodemailer (or Resend SDK), and talk deliverability without hand-waving.

| **Elapsed** | **Time** | **Activity** |
| ----------- | -------- | ------------------------- |
| 0:00 | 0:05 | Why / Objectives |
| 0:05 | 0:40 | Overview / TT (+ Send Emails Async) |
| 0:45 | 0:15 | Lab 1 — MVP send |
| 1:00 | 0:10 | BREAK |
| 1:10 | 0:30 | Lab 2 — modular `mailer` + route |
| 1:40 | 0:15 | Build time / stretch (Pete’s purchase hook) |
| 1:55 | 0:05 | Wrap Up |
| **TOTAL** | **2:00** | |

---

## Why You Should Know This (2 min)

People have been trying to kill email for years. It still wins for **receipts, password resets, magic links, “your order shipped,” and “something weird happened on your account.”** Those are **transactional** emails — triggered by an app event, expected by one user, time-sensitive.

Say:

> “Chat apps come and go. Email is the address book every product already has. If your Node API can’t send a trustworthy transactional message, the product feels unfinished.”

### When NOT to roll your own SMTP

| Roll a provider | Don’t DIY raw SMTP |
| --- | --- |
| Deliverability (IP reputation, bounce handling) | You’re not an ISP; residential/VPS IPs get spam-foldered |
| Auth headers + domain verification workflows | SPF/DKIM/DMARC are easy to get wrong |
| Bounce/complaint webhooks | Silently failing sends = silent money loss |
| Rate limits + retries | Your `for` loop will get you blocked |

**Rookie trap:** shipping `nodemailer` pointed at `smtp.gmail.com` with a personal password is a demo, not a product.

**GOAL:** leave able to pick a 2026 provider, keep secrets server-side, send HTML+text via Nodemailer (or Resend SDK), and talk deliverability without hand-waving.

---

## Learning Objectives (3 min)

1. **Explain** why transactional email still matters and when *not* to run your own SMTP relay.
2. **Compare** Resend / SES / Mailgun / SendGrid at a beginner on-the-job level; use **one** primary path in lab (**Resend**, Ethereal fallback).
3. **Keep secrets safe:** API keys in env vars only; never commit; never ship keys to the browser.
4. **Send** a message with both `html` and `text` bodies; sketch Handlebars vs React Email; name **SPF / DKIM / DMARC** in one sentence each.
5. **Write** a small async `sendMail` helper that surfaces failures (JS competence) and wire it from an Express route.

**How you’ll know:** Lab 1 produces a `messageId` (or Ethereal preview URL) in the terminal; Lab 2 exports `utils/mailer.js` and a POST (or purchase) path that awaits send + returns a clear status.

---

## Overview / TT (40 min)

**Next action:** Destination → providers → Nodemailer abstraction → secrets → templates → auth headers → JS async/error shapes.  
**Done when:** You can sketch “browser → Express → mailer → provider” and say where the API key lives.  
**≤2m next after TT:** Open Lab 1; pick Resend *or* Ethereal.

### 1. Transactional vs marketing (~4m)

| | Transactional | Marketing / bulk |
| --- | --- | --- |
| Trigger | User/app event | Campaign / list |
| Expectation | “I did a thing; email confirms it” | Promo, newsletter |
| Examples | Reset password, receipt, 2FA | Sale blast |
| Risk if broken | Locked out / charge unclear | Unsub / spam |

Say:

> “Today is transactional. If you mix promo into the same stream without consent plumbing, deliverability tanks and lawyers get interested.”

**Pulse check 1 (≤60s):** Receipt for a purchase vs weekly newsletter — which is transactional? Expected: **purchase receipt**.

### 2. Providers 2026 — pick one primary (~8m)

Whiteboard the field; **lab primary = Resend**.

| Provider | 2026 vibe (hire-level) | Free-tier reality (verify day-of) | Good fit when… |
| --- | --- | --- | --- |
| **Resend** | DX-first API; React Email sibling; Nodemailer SMTP supported | 3,000/mo · 100/day · **3 domains** (confirm [Resend pricing](https://resend.com/pricing) day-of) | Labs, modern Node apps |
| **Amazon SES** | Cheap at scale; AWS IAM surface area | Often generous if already in AWS (esp. EC2-era myths — check current) | You already live in AWS |
| **Mailgun** | Battle-tested API; Pete’s Pets historical path | Trial/plan names change — don’t trust old Flex Trial notes | Existing Mailgun accounts / legacy labs |
| **SendGrid** | Twilio ecosystem; marketing + transactional | Free tiers shrink/shift — verify | Marketing tools already in Twilio |

**Nodemailer** = transport **abstraction**. You configure *how* to talk (SMTP / provider transport); your app code stays `transporter.sendMail({...})`.

```js
// Shape only — secrets come from env
import nodemailer from 'nodemailer';

const transporter = nodemailer.createTransport({
  host: 'smtp.resend.com',
  port: 465,
  secure: true,
  auth: {
    user: 'resend',
    pass: process.env.RESEND_API_KEY, // never hardcode
  },
});
```

**Alternate (same provider, SDK):** `import { Resend } from 'resend'` → `await resend.emails.send({...})` returns `{ data, error }` (doesn’t throw on API errors). Cover both shapes; pick **one** for the MVP so nobody bikesheds.

**Pete’s Pets note:** challenge `P06` stays **stretch / after-hours** on Mailgun tonight. Live primary = Resend/Ethereal. `nodemailer-mailgun-transport` last publish 2022 — Wave-2 rewrite to `mailgun.js`/HTTP; optional later `MAIL_PROVIDER=` — **out of scope** for Day 5 live.

### 3. Secrets — server-side only (~6m)

| Belongs | Does **not** belong |
| --- | --- |
| `.env` / host secrets manager | Git history, screenshots, Slack |
| Server process env | Vite/React `VITE_*` / any public bundle |
| CI secrets store | “Temporary” commit “just for today” |

Checklist out loud:

1. Key in `.env` as `RESEND_API_KEY=re_...`
2. `.env` in `.gitignore` (verify, don’t assume)
3. Load with `dotenv` / platform env — **never** `const KEY = 're_live_...'` in source
4. Email send only from **Express / server** routes or jobs — not from the browser

Say:

> “If the key is in the client, it’s not a secret anymore — it’s a public payment method for someone else’s spam.”

**JS competence beat:** treat config as a small object you validate once at boot.

```js
function getMailConfig() {
  const apiKey = process.env.RESEND_API_KEY;
  if (!apiKey) {
    throw new Error('RESEND_API_KEY is missing — set it in .env (server only)');
  }
  return {
    apiKey,
    from: process.env.MAIL_FROM || 'Acme <onboarding@resend.dev>',
  };
}
```

**Pulse check 2 (≤60s):** Where does `RESEND_API_KEY` live — browser bundle or server env? Expected: **server env only**.

### 4. Templates: HTML + text; Handlebars vs React Email (~8m)

Clients still matter: some are HTML, some are text-only, some strip styles. **Always send both** when you can.

```js
function escapeHtml(value) {
  return String(value)
    .replaceAll('&', '&amp;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;')
    .replaceAll('"', '&quot;');
}

await transporter.sendMail({
  from: config.from,
  to: user.email,
  subject: 'Pet purchased!',
  text: `Hi ${user.name}, your purchase went through.`,
  html: `<p>Hi <strong>${escapeHtml(user.name)}</strong>, your purchase went through.</p>`,
});
```

| Approach | One-breath take |
| --- | --- |
| **String templates** | Fine for MVP; watch XSS if you interpolate user input into HTML |
| **Handlebars** | Pete’s path — `{{name}}` in `.handlebars`; good for Express-centric stacks |
| **React Email** | Component → HTML; pairs naturally with Resend; optional stretch, not required for ACS-3210 MVP |
| **Handlebars (Resend path)** | `handlebars.compile` → pass `html`/`text`, **or** `nodemailer-express-handlebars` on a Nodemailer transporter. Do **not** use Mailgun `template:` on the Resend path. |

### Spoofing / SPF / DKIM / DMARC — one-slide hire tips

| Acronym | One-liner |
| --- | --- |
| **SPF** | DNS allowlist: which servers may send as your domain |
| **DKIM** | Cryptographic signature on the message so receivers can verify it wasn’t tampered |
| **DMARC** | Policy that ties SPF+DKIM together and tells receivers what to do on fail (`none` / `quarantine` / `reject`) |

Say:

> “Anyone can put `from: ceo@yourbank.com` in SMTP. Auth headers + a verified domain are why providers exist. In lab: `from: onboarding@resend.dev` may only go **to the email on your Resend account** (other `to:` → 403). Event sinks need a verified-domain `from`, or use Ethereal. Production wants *your* verified domain.”

**Pulse check 3 (≤60s):** Can you trust `from:` alone? Why can anyone set `from: ceo@yourbank.com` in raw SMTP? Expected: **SMTP doesn’t prove domain ownership** — SPF/DKIM/DMARC + verified domain do.

### 5. Send Emails Async (~14m)

Goal: practice **real Node**, not just paste provider snippets.

### Pattern A — Nodemailer (throws / Promise rejection)

> **Scaffold note:** Pete’s is **CJS** (`require`) unless `"type": "module"`. Use `const nodemailer = require('nodemailer')` / `module.exports = { sendTransactionalEmail }` to match Pete’s. ESM `import` is fine only if the course entry is already ESM.

```js
const nodemailer = require('nodemailer');

function getMailConfig() {
  const apiKey = process.env.RESEND_API_KEY;
  if (!apiKey) {
    throw new Error('RESEND_API_KEY is missing — set it in .env (server only)');
  }
  return {
    apiKey,
    from: process.env.MAIL_FROM || 'Acme <onboarding@resend.dev>',
  };
}

async function sendTransactionalEmail({ to, subject, text, html }) {
  const config = getMailConfig();
  const transporter = nodemailer.createTransport({
    host: 'smtp.resend.com',
    port: 465,
    secure: true,
    auth: { user: 'resend', pass: config.apiKey },
  });

  try {
    const info = await transporter.sendMail({
      from: config.from,
      to,
      subject,
      text,
      html,
    });
    return { ok: true, messageId: info.messageId };
  } catch (err) {
    // Log server-side; return a safe shape to the route
    console.error('sendMail failed', err);
    return { ok: false, error: 'email_send_failed' };
  }
}

module.exports = { sendTransactionalEmail };
```

### Pattern B — Resend SDK (`{ data, error }` — often does *not* throw on API errors)

```js
import { Resend } from 'resend';

export async function sendTransactionalEmail({ to, subject, html, text }) {
  const resend = new Resend(process.env.RESEND_API_KEY);
  const { data, error } = await resend.emails.send({
    from: process.env.MAIL_FROM || 'Acme <onboarding@resend.dev>',
    to: [to],
    subject,
    html,
    text,
  });
  if (error) {
    console.error(error);
    return { ok: false, error: error.message };
  }
  return { ok: true, messageId: data.id };
}
```

**Say the contrast out loud:**

- Nodemailer → `try/catch` around `await sendMail`
- Resend SDK → check `error` on the returned object; `try/catch` only for network/unexpected
- Routes should **await** and decide status codes (`202` / `502`), not fire-and-forget without logging
- Prefer a **module** (`utils/mailer.js`) over dumping transport setup into `server.js`

**Anti-patterns to call out (rookie traps):**

- `.then()` chain with empty `.catch(() => {})`
- Sending email inside the request *before* the DB commit without a clear failure story (at least log + don’t pretend success)
- Using user-controlled HTML without escaping

**Pulse check 4 (≤60s):** Nodemailer `sendMail` fails — do you `try/catch`, or check a returned `{ error }`? Expected: **`try/catch`** (throws). Resend SDK → check **`{ data, error }`** (often no throw on API errors).

---

## Lab 1 — MVP send (15 min)

**Work shape:** solo · **MVP** · **visible checkpoint** · **artifact**.

| | |
| --- | --- |
| **Next action** | Choose path A (Ethereal — zero signup) *or* path B (Resend — closer to production). |
| **Done when** | Terminal shows `messageId` **or** Ethereal preview URL. |
| **Artifact** | Screenshot or pasted log line in your notes / shared channel when asked. |
| **Checkpoint (visible)** | “I see a messageId / preview URL” in chat or unmuted shout-out. |

### Path A — Ethereal (fastest; fake SMTP inbox)

1. `npm install nodemailer`
2. Create `scripts/send-test.js` (or a throwaway route) that:
   - `const testAccount = await nodemailer.createTestAccount()`
   - builds a transporter with `testAccount.smtp`
   - `await transporter.sendMail({ from, to, subject, text, html })`
   - logs `nodemailer.getTestMessageUrl(info)`
3. Run it: `node scripts/send-test.js`
4. Open the preview URL — that’s your artifact.

### Path B — Resend (preferred primary)

1. Create a Resend account → API key → set `RESEND_API_KEY` in `.env`
2. Sending rules for lab (avoid 403):
   - **Sandbox:** `from: 'Acme <onboarding@resend.dev>'` → `to:` **only** the email on your Resend account
   - **Event sinks** (`delivered@` / `bounced@` / `complained@` / `suppressed@resend.dev`): use after you have a **verified domain** `from`, *or* stick to Ethereal if signup/DNS is too slow
   - Do **not** promise `onboarding@resend.dev` → `delivered@resend.dev` works for every Resend account
3. Nodemailer SMTP **or** `resend` SDK — one path only
4. Log `messageId` / `data.id`

**MVP definition of done (≤15m):** one successful send logged. Templates polish waits for Lab 2.

---

## BREAK (10 min)

---

## Lab 2 — modular mailer + Express (30 min)

| | |
| --- | --- |
| **Next action** | Extract transport into `utils/mailer.js`; call it from a route. |
| **Done when** | `POST /api/send-test` (or purchase hook) awaits send and returns JSON `{ ok, messageId }` or `{ ok: false, error }`. |
| **Artifact** | `curl` / Thunder Client response body + server log line. |
| **Checkpoint** | Paste `{ "ok": true, "messageId": "..." }` (redact keys) when you hit it. |

### Steps

1. Add `utils/mailer.js` exporting `sendTransactionalEmail` (Pattern A or B from TT).
2. Ensure env load happens **once** at boot (`dotenv.config()` in `server.js` / entry).
3. Wire a minimal route:

```js
// sketch — adapt to course router style
app.post('/api/send-test', async (req, res) => {
  const to = req.body?.to || process.env.MAIL_TO_TEST;
  if (!to) {
    return res.status(400).json({ ok: false, error: 'missing_to' });
  }

  const result = await sendTransactionalEmail({
    to,
    subject: 'ACS-3210 test',
    text: 'It works (text).',
    html: '<strong>It works (html).</strong>',
  });

  if (!result.ok) {
    return res.status(502).json(result);
  }
  return res.status(202).json(result);
});
```

1. Hit the route; confirm artifact.
2. **JS stretch inside the same block:** add `text` + `html`; escape any user-provided name if you interpolate into HTML; return stable error codes (`email_send_failed`) instead of raw provider dumps to the client.

### Stretch (if ahead)

- Handlebars template file for the body (Pete’s-shaped)
- Idempotency key (Resend) for retry-safe sends
- Don’t block the HTTP response forever — log failures; optional queue mention (name only)

---

## Build time (15 min)

Optional Pete’s Pets alignment (do **not** require Mailgun signup mid-block if Resend/Ethereal already worked):

1. On purchase success, call `sendTransactionalEmail` with buyer email + pet name.
2. Keep redirects/UX working even if email fails — **log** the failure (beginner tip: silent catch that still redirects is OK for UX, bad if you never log).
3. If continuing the legacy P06 path: Mailgun + `nodemailer-mailgun-transport` — verify **current** Mailgun dashboard steps; ignore outdated Flex Trial blog dates.

---

## Wrap Up (5 min)

Takeaways to say out loud:

1. Transactional email is product-critical; **don’t DIY SMTP** for real deliverability.
2. **Resend** (lab) / SES / Mailgun / SendGrid — pick for DX vs AWS vs legacy; **Nodemailer** abstracts transport.
3. **Secrets stay on the server** — env vars, never the client bundle.
4. Send **html + text**; know SPF/DKIM/DMARC as interview vocabulary.
5. **Await + surface failures** — JS that lies about send success is worse than no email.

**≤2m wrap sticky (optional):**

```text
Shipped today:
Stuck on:
Tomorrow's first 15m:
```

---

## Additional Resources

1. **[Nodemailer](https://nodemailer.com/)** — transport abstraction, Ethereal test accounts, message options.
2. **[Resend — Node.js SDK](https://resend.com/docs/send-with-nodejs)** — primary lab API docs.
3. **[Resend — Nodemailer SMTP](https://resend.com/docs/send-with-nodemailer-smtp)** — SMTP credentials (`smtp.resend.com:465`).
4. **[Resend pricing](https://resend.com/pricing)** — verify free-tier numbers day-of (limits change).
5. **[Amazon SES](https://docs.aws.amazon.com/ses/)** — scale / AWS path.
6. **[Mailgun docs](https://documentation.mailgun.com/)** — prefer over Medium; Pete’s P06 legacy.
7. **[SendGrid docs](https://docs.sendgrid.com/)** — Twilio SendGrid.
8. **[MDN — SPF / DKIM overview articles via provider blogs]** — keep explanations to the one-liners above in session; deep-dive optional.
9. **[React Email](https://react.email)** — stretch only.

**Do not use as primary:** Medium “Nodemailer + Mailgun” (likely stale; linked from old stub).

---

<details>
<summary>For curriculum authors</summary>

## For curriculum authors

<details>
<summary>For curriculum authors</summary>


### In Class (whole block)

| | |
| --- | --- |
| **Next action** | Open this file → skim the agenda table → start Why / Objectives at 4:00. |
| **Done when** | You can send a transactional message from the server, explain why the API key never touches the browser, and name SPF/DKIM/DMARC in one on-the-job sentence each. |
| **≤2m next** | Optional standup sticky in your notes: Feeling / Behind\|On track\|Ahead / Today’s MVP. |

```text
Feeling (1 word):
Behind | On track | Ahead:
Today's MVP (1 sentence): send a test email; log messageId (or Ethereal preview URL)
```

### Facilitator notes

**Broken-link note (repo stub):** the Medium “Nodemailer + Mailgun” write-up linked from the old `Emails.md` may be stale. Prefer official docs (Resources above). Do **not** demo from the Medium piece.

- After ACS-4210 same day — keep energy practical; MVP in Lab 1 before perfect templates.
- Voice: write-like-you-talk. Short blocks. GOAL first. Job-sim framing only.
- If signup friction spikes → **force Path A Ethereal** so nobody loses the JS async lesson to OAuth.
- Pete’s Mailgun content is **stretch / after-hours alignment**, not the live primary demo.
- Topic-only file: no roster / attendance / room-mgmt bits on purpose.
- Job-sim standing: dropped academic tokens (`Class/lab`, `homework/stretch`) — use lab / stretch / after-hours instead.

### Node Engineer — open API / stack uncertainties

Flag for follow-up (do not block today’s live block):

1. **Pete’s P06 still Mailgun-shaped** — should the course master / challenge migrate to Resend + Nodemailer SMTP, keep Mailgun, or support both via env (`MAIL_PROVIDER=`)?
2. **`nodemailer-mailgun-transport` maintenance** — still the recommended bridge in 2026, or prefer Mailgun’s official SDK / HTTP API?
3. **Course module system** — examples above lean ESM (`import`); Pete’s historically CJS (`require`). Confirm scaffold default before rewriting P06 snippets.
4. **Resend test sending rules day-of** — confirm whether accounts can use `onboarding@resend.dev` → personal inbox vs only `*@resend.dev` test sinks without a verified domain.
5. **Free-tier numbers** — Resend ~3k/mo & ~100/day cited from 2026 secondary sources; **re-check** [Resend pricing](https://resend.com/pricing) before printing shared handouts.
6. **SES free-tier myths** — old “62k from EC2” lore may not match 2026 accounts; don’t put hard numbers in shared slides without AWS docs confirmation.
7. **dotenv / secret loading** — is the course entry still `dotenv.config()` in `server.js`, or has hosting moved to platform env only?
8. **Handlebars email path** — P06 uses Nodemailer `template: { name, engine, context }` via mailgun transport; plain Nodemailer + Resend may need `handlebars.compile` manually or `nodemailer-express-handlebars`. Which pattern should Wave 2 standardize?
9. **Idempotency / queues** — out of scope for Day 5 MVP; confirm whether a later session mentions BullMQ / SQS for “email after payment” reliability.

</details>

</details>
