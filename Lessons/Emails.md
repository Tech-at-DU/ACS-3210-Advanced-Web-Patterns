# Sending Emails

<!-- > -->

<!-- omit in toc -->
## ⏱ Agenda {docsify-ignore}

1. [[**05m**] 🏆 Learning Outcomes](#%5B%2a%2a05m%2a%2a%5D-%F0%9F%8F%86-learning-outcomes)
1. [[**40m**] 💬 **TT**: How to Send Emails](#%5B%2a%2a40m%2a%2a%5D-%F0%9F%92%AC-tt%3A-how-to-send-emails)
1. [[**15m**] 💻 **Activity**: Lab I — MVP send](#%5B%2a%2a15m%2a%2a%5D-%F0%9F%92%BB-activity%3A-lab-i--mvp-send)
1. [[**10m**] 🌴 BREAK {docsify-ignore}](#%5B%2a%2a10m%2a%2a%5D-%F0%9F%8C%B4-break-%7Bdocsify-ignore%7D)
1. [[**30m**] 💻 **Activity**: Lab II — modular mailer](#%5B%2a%2a30m%2a%2a%5D-%F0%9F%92%BB-activity%3A-lab-ii--modular-mailer)
1. [[**15m**] 💻 **Homework / stretch**: Pete’s P06 Mailgun](#%5B%2a%2a15m%2a%2a%5D-%F0%9F%92%BB-homework--stretch%3A-pete%E2%80%99s-p06-mailgun)
1. [[**05m**] **Recap**: Today's Takeaways](#%5B%2a%2a05m%2a%2a%5D-recap%3A-todays-takeaways)

<!-- > -->

## [**05m**] 🏆 Learning Outcomes

By the end of this lesson, you should be able to...

1. Explain why transactional email still matters and when *not* to run your own SMTP relay.
1. Compare Resend / SES / Mailgun / SendGrid at a beginner on-the-job level, and use **one** primary path in lab (**Resend**, Ethereal fallback).
1. Keep API keys in server env only — never commit them, never ship them to the browser.
1. Send a message with both `html` and `text` bodies, and name **SPF / DKIM / DMARC** in one sentence each.
1. Write a small async `sendMail` helper that surfaces failures and wire it from an Express route.

**How you’ll know:** Lab I produces a `messageId` (or Ethereal preview URL) in the terminal. Lab II exports `utils/mailer.js` and a POST (or purchase) path that awaits send and returns a clear status.

⭐️ **GOAL:** Leave able to pick a 2026 provider, keep secrets server-side, send HTML+text via Nodemailer (or Resend SDK), and talk deliverability without hand-waving.

**Class primary:** **Resend** + **Nodemailer** · **Ethereal** as the zero-signup fallback.  
**Pete’s Pets P06 (Mailgun)** stays **homework / stretch** — do not make Mailgun the live demo.

<!-- > -->

## [**40m**] 💬 **TT**: How to Send Emails

**GOAL:** Sketch `browser → Express → mailer → provider` and say where the API key lives.

**Next action:** Destination → providers → Nodemailer → secrets → templates → auth headers → async / error shapes.  
**Done when:** You can pick Resend *or* Ethereal for Lab I and explain why the key never touches the client.

People have been trying to kill email for years. It still wins for **receipts, password resets, magic links, “your order shipped,” and “something weird happened on your account.”** Those are **transactional** emails — triggered by an app event, expected by one user, time-sensitive.

> Chat apps come and go. Email is the address book every product already has. If your Node API can’t send a trustworthy transactional message, the product feels unfinished.

### When NOT to roll your own SMTP

| Roll a provider | Don’t DIY raw SMTP |
| --- | --- |
| Deliverability (IP reputation, bounce handling) | You’re not an ISP; residential/VPS IPs get spam-foldered |
| Auth headers + domain verification workflows | SPF/DKIM/DMARC are easy to get wrong |
| Bounce/complaint webhooks | Silently failing sends = silent money loss |
| Rate limits + retries | Your `for` loop will get you blocked |

**Rookie trap:** pointing `nodemailer` at `smtp.gmail.com` with a personal password is a demo, not a product.

<!-- v -->

### 1. Transactional vs marketing (~4m)

| | Transactional | Marketing / bulk |
| --- | --- | --- |
| Trigger | User/app event | Campaign / list |
| Expectation | “I did a thing; email confirms it” | Promo, newsletter |
| Examples | Reset password, receipt, 2FA | Sale blast |
| Risk if broken | Locked out / charge unclear | Unsub / spam |

> Today is transactional. If you mix promo into the same stream without consent plumbing, deliverability tanks — and lawyers get interested.

**Pulse check 1 (≤60s):** Receipt for a purchase vs weekly newsletter — which is transactional?  
Expected: **purchase receipt**.

<!-- v -->

### 2. Providers 2026 — pick one primary (~8m)

Whiteboard the field; **lab primary = Resend**.

| Provider | 2026 vibe (beginner on-the-job) | Free-tier reality (verify day-of) | Good fit when… |
| --- | --- | --- | --- |
| **Resend** | DX-first API; React Email sibling; Nodemailer SMTP supported | **3,000/mo · 100/day · 3 domains** ([Resend quotas](https://resend.com/docs/knowledge-base/account-quotas-and-limits); confirm [pricing](https://resend.com/pricing) day-of) | Labs, modern Node apps |
| **Amazon SES** | Cheap at scale; AWS IAM surface area | Often generous if you already live in AWS — check current docs, ignore old EC2-era myths | You already live in AWS |
| **Mailgun** | Battle-tested API; Pete’s Pets **P06 homework / stretch** | Trial/plan names change — don’t trust old Flex Trial notes | Existing Mailgun accounts / legacy homework |
| **SendGrid** | Twilio ecosystem; marketing + transactional | Free tiers shrink/shift — verify | Marketing tools already in Twilio |

**Nodemailer** is a transport **abstraction**. You configure *how* to talk (SMTP / provider transport); app code stays `transporter.sendMail({...})`.

> **Pete’s scaffold is CJS.** Use `require` / `module.exports` unless the project has `"type": "module"`. ESM `import` is fine only if the course entry is already ESM.

```js
// Shape only — secrets come from env. CJS to match Pete’s.
const nodemailer = require('nodemailer');

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

**Alternate (same provider, SDK):** `const { Resend } = require('resend')` → `await resend.emails.send({...})` returns `{ data, error }` (does **not** throw on typical API errors). Cover both shapes; pick **one** for the MVP so nobody bikesheds.

**Pete’s Pets note:** challenge **P06 stays homework / stretch** on Mailgun. Live primary = Resend / Ethereal. `nodemailer-mailgun-transport` last published 2022 — a later rewrite to `mailgun.js` / HTTP is out of scope for Day 5 live.

<!-- v -->

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

> If the key is in the client, it’s not a secret anymore — it’s a public payment method for someone else’s spam.

**Beginner on-the-job tip:** treat config as a small object you validate once at boot.

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

**Pulse check 2 (≤60s):** Where does `RESEND_API_KEY` live — browser bundle or server env?  
Expected: **server env only**.

<!-- v -->

### 4. Templates: HTML + text; Handlebars vs React Email (~8m)

Some clients render HTML, some are text-only, some strip styles. **Always send both** when you can.

```js
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
| **Handlebars** | Pete’s P06 path — `{{name}}` in `.handlebars`; good for Express-centric stacks |
| **React Email** | Component → HTML; pairs naturally with Resend; stretch, not required for the ACS-3210 MVP |
| **Handlebars on the Resend path** | `handlebars.compile` → pass `html` / `text`, **or** `nodemailer-express-handlebars` on a Nodemailer transporter. Do **not** use Mailgun `template:` on the Resend path. |

### Spoofing / SPF / DKIM / DMARC — one-slide rookie tips

| Acronym | One-liner |
| --- | --- |
| **SPF** | DNS allowlist: which servers may send as your domain |
| **DKIM** | Cryptographic signature so receivers can verify the message wasn’t tampered |
| **DMARC** | Policy that ties SPF+DKIM together and tells receivers what to do on fail (`none` / `quarantine` / `reject`) |

> Anyone can put `from: ceo@yourbank.com` in SMTP. Auth headers + a verified domain are why providers exist.

**P0 — Resend sandbox `from` / 403:** `from: onboarding@resend.dev` may only go **to the email on your Resend account**. Any other `to:` → **403**. Event sinks (`delivered@` / `bounced@` / `complained@` / `suppressed@resend.dev`) need a **verified-domain** `from`, or use Ethereal. Production wants *your* verified domain. Do **not** promise `onboarding@resend.dev` → `delivered@resend.dev` works for every account.

**Pulse check 3 (≤60s):** Can you trust `from:` alone? Why can anyone set `from: ceo@yourbank.com` in raw SMTP?  
Expected: **SMTP doesn’t prove domain ownership** — SPF/DKIM/DMARC + a verified domain do.

<!-- v -->

### 5. Async send + failure surfaces (~14m)

Practice **real Node**, not just paste provider snippets.

#### Pattern A — Nodemailer (throws / Promise rejection)

```js
// utils/mailer.js — CJS to match Pete’s
const nodemailer = require('nodemailer');

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

#### Pattern B — Resend SDK (`{ data, error }` — often does *not* throw on API errors)

```js
const { Resend } = require('resend');

async function sendTransactionalEmail({ to, subject, html, text }) {
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

module.exports = { sendTransactionalEmail };
```

Say the contrast out loud:

- Nodemailer → `try/catch` around `await sendMail`
- Resend SDK → check `error` on the returned object; `try/catch` only for network / unexpected
- Routes should **await** and decide status codes (`202` / `502`), not fire-and-forget without logging
- Prefer a **module** (`utils/mailer.js`) over dumping transport setup into `server.js`

**Rookie traps:**

- `.then()` chain with empty `.catch(() => {})`
- Sending email *before* the DB commit without a failure story (at least log — don’t pretend success)
- Interpolating user-controlled HTML without escaping

**Pulse check 4 (≤60s):** Nodemailer `sendMail` fails — do you `try/catch`, or check a returned `{ error }`?  
Expected: **`try/catch`** (it throws). Resend SDK → check **`{ data, error }`** (often no throw on API errors).

<!-- > -->

## [**15m**] 💻 **Activity**: Lab I — MVP send

**GOAL:** One real *or* Ethereal test send with `html` + `text`, secrets in env only, and a visible success artifact (`messageId` / preview URL).

| | |
| --- | --- |
| **Next action** | Path A (Ethereal — zero signup) *or* path B (Resend — closer to production). |
| **Done when** | Terminal shows `messageId` **or** Ethereal preview URL. |
| **Artifact** | Pasted log line in notes / shared channel when asked. |

### Path A — Ethereal (fastest; fake SMTP inbox)

1. `npm install nodemailer`
2. Create `scripts/send-test.js` that:
   - `const testAccount = await nodemailer.createTestAccount()`
   - builds a transporter with `testAccount.smtp`
   - `await transporter.sendMail({ from, to, subject, text, html })`
   - logs `nodemailer.getTestMessageUrl(info)`
3. Run it: `node scripts/send-test.js`
4. Open the preview URL — that’s your artifact.

```js
// scripts/send-test.js — CJS
const nodemailer = require('nodemailer');

async function main() {
  const testAccount = await nodemailer.createTestAccount();
  const transporter = nodemailer.createTransport({
    host: testAccount.smtp.host,
    port: testAccount.smtp.port,
    secure: testAccount.smtp.secure,
    auth: { user: testAccount.user, pass: testAccount.pass },
  });

  const info = await transporter.sendMail({
    from: 'ACS-3210 <lab@example.com>',
    to: 'you@example.com',
    subject: 'Ethereal MVP',
    text: 'It works (text).',
    html: '<strong>It works (html).</strong>',
  });

  console.log('messageId', info.messageId);
  console.log('preview', nodemailer.getTestMessageUrl(info));
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

### Path B — Resend (preferred primary)

1. Create a Resend account → API key → set `RESEND_API_KEY` in `.env`
2. Sending rules for lab (**avoid 403**):
   - **Sandbox:** `from: 'Acme <onboarding@resend.dev>'` → `to:` **only** the email on your Resend account
   - **Event sinks** (`delivered@` / `bounced@` / `complained@` / `suppressed@resend.dev`): use after you have a **verified domain** `from`, *or* stick to Ethereal if signup/DNS is too slow
   - Do **not** promise `onboarding@resend.dev` → `delivered@resend.dev` works for every Resend account
3. Nodemailer SMTP **or** `resend` SDK — one path only
4. Log `messageId` / `data.id`

**MVP definition of done (≤15m):** one successful send logged. Template polish waits for Lab II.

If signup friction spikes, **force Path A Ethereal** so nobody loses the async lesson to OAuth.

<!-- > -->

## [**10m**] 🌴 BREAK {docsify-ignore}

<!-- > -->

## [**30m**] 💻 **Activity**: Lab II — modular mailer

**GOAL:** Extract transport into `utils/mailer.js` and call it from a route.

| | |
| --- | --- |
| **Next action** | Pattern A or B from the TT → `utils/mailer.js` → `POST /api/send-test`. |
| **Done when** | The route awaits send and returns JSON `{ ok, messageId }` or `{ ok: false, error }`. |
| **Artifact** | `curl` / Thunder Client body + server log line (redact keys). |

### Steps

1. Add `utils/mailer.js` exporting `sendTransactionalEmail` (Pattern A or B).
2. Load env **once** at boot (`dotenv.config()` in `server.js` / entry).
3. Wire a minimal route:

```js
// sketch — adapt to course router style (CJS)
const { sendTransactionalEmail } = require('./utils/mailer');

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

4. Hit the route; confirm the artifact.
5. **JS stretch in the same block:** keep `text` + `html`; escape any user-provided name if you interpolate into HTML; return stable error codes (`email_send_failed`) instead of raw provider dumps to the client.

### Stretch (if ahead)

- Handlebars template file for the body (Pete’s-shaped)
- Idempotency key (Resend) for retry-safe sends
- Don’t block the HTTP response forever — log failures; optional queue mention (name only)

<!-- > -->

## [**15m**] 💻 **Homework / stretch**: Pete’s P06 Mailgun

Class primary tonight is **Resend / Ethereal**. Pete’s Pets **[P06 — Send Emails](https://github.com/Tech-at-DU/Proud-Petes-Pet-Emporium/tree/master/P06-Send-Emails)** stays **homework / stretch**.

Do **not** require a Mailgun signup mid-block if Resend or Ethereal already worked.

If you continue P06 after class:

1. On purchase success, call your mailer with buyer email + pet name.
2. Keep redirects / UX working even if email fails — **log** the failure. Beginner tip: a silent catch that still redirects is OK for UX, bad if you never log.
3. Mailgun + `nodemailer-mailgun-transport` — verify **current** Mailgun dashboard steps; ignore outdated Flex Trial blog dates.
4. Pete’s snippets are **CJS** (`require` / `module.exports`). Match that unless the scaffold is already ESM.
5. Prefer [Mailgun docs](https://documentation.mailgun.com/) over the old Medium “Nodemailer + Mailgun” write-up (likely stale).

<!-- > -->

## [**05m**] **Recap**: Today's Takeaways

Say out loud:

1. Transactional email is product-critical; **don’t DIY SMTP** for real deliverability.
2. **Resend** (class primary) / SES / Mailgun / SendGrid — pick for DX vs AWS vs legacy; **Nodemailer** abstracts transport. **Ethereal** is the zero-signup fallback.
3. **Secrets stay on the server** — env vars, never the client bundle.
4. Send **html + text**. Know SPF / DKIM / DMARC as beginner on-the-job vocabulary.
5. **Await + surface failures** — JS that lies about send success is worse than no email.
6. **P0 sandbox rule:** `onboarding@resend.dev` → **account email only** (other `to:` → 403). Free tier: **3,000/mo · 100/day · 3 domains**.
7. Pete’s **P06 Mailgun** is **homework / stretch**, not the live primary.

<!-- > -->

<!-- omit in toc -->
## 📚 Resources & Credits

1. **[Nodemailer](https://nodemailer.com/)** — transport abstraction, Ethereal test accounts, message options.
1. **[Resend — Node.js SDK](https://resend.com/docs/send-with-nodejs)** — primary lab API docs.
1. **[Resend — Nodemailer SMTP](https://resend.com/docs/send-with-nodemailer-smtp)** — SMTP credentials (`smtp.resend.com:465`).
1. **[Resend — 403 on `resend.dev`](https://resend.com/docs/knowledge-base/403-error-resend-dev-domain)** — sandbox `from` may only send to the account email.
1. **[Resend quotas](https://resend.com/docs/knowledge-base/account-quotas-and-limits)** / **[pricing](https://resend.com/pricing)** — verify free-tier numbers day-of (limits change).
1. **[Amazon SES](https://docs.aws.amazon.com/ses/)** — scale / AWS path.
1. **[Mailgun docs](https://documentation.mailgun.com/)** — prefer over Medium; Pete’s P06 homework / stretch.
1. **[SendGrid docs](https://docs.sendgrid.com/)** — Twilio SendGrid.
1. **[React Email](https://react.email)** — stretch only.
1. **[Pete’s Pets P06](https://github.com/Tech-at-DU/Proud-Petes-Pet-Emporium/tree/master/P06-Send-Emails)** — Mailgun + Handlebars homework / stretch.

**Do not use as primary:** Medium “Nodemailer + Mailgun” (likely stale; linked from the old stub).

<!-- > -->

## Facilitator notes

- Voice: write-like-you-talk. Short blocks. **GOAL** first. Job-sim framing; rookie / beginner on-the-job tips (not a “bar”).
- After a long day — keep energy practical; MVP in Lab I before perfect templates.
- If signup friction spikes → **force Path A Ethereal** so nobody loses the JS async lesson.
- Pete’s Mailgun content is **homework / stretch**, not the live primary demo.
- Sidebar already points at `Lessons/Emails.md` — no nav change required.
