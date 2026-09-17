<!-- Run as a slideshow: reveal-md Lessons/WebSocketsIntro.md -w -->
# Intro to WebSockets — Day 6

⭐️ **GOAL:** Leave able to explain why a long-lived socket fits live apps, attach Socket.IO 4 to an `http.Server` with Express 5, choose the right emit audience, and extract connection handlers into a small CommonJS module.

<!-- omit in toc -->
## ⏱ Agenda

- [[**5m**] Attendance &amp; Announcements](#5m-attendance--announcements)
- [[**10m**] ☀️ Warm Up](#10m-️-warm-up)
- [[**40m**] 📚 TT: Overview](#40m--tt-overview)
- [[**10m**] 🌴 Break](#10m--break)
- [[**20m**] 💻 Activity 1: Attach Socket.IO to HTTP Server](#20m--activity-1-attach-socketio-to-http-server)
- [[**30m**] 💻 Activity 2: Emit Across Tabs and Extract Handlers](#30m--activity-2-emit-across-tabs-and-extract-handlers)
- [[**5m**] Wrap Up](#5m-wrap-up)

<!-- > -->

<!-- omit in toc -->
## 🏆 Objectives

*By the end of this session, you'll be able to&hellip;*

1. Explain **bidirectional, full-duplex** traffic in one breath, and name when a socket is the wrong tool (one-shot request/response)
1. Sketch the **HTTP upgrade** (`101 Switching Protocols`) and say that later frames are **WS / WSS**, not HTTP
1. Attach Socket.IO 4 to a Node **`http.Server`** that also serves Express 5 — and say why `app.listen(3000)` fails
1. Write JS that **registers named handlers**, **closes over `socket`**, **guards payloads**, and **exports** a `registerChatHandlers(io)` module
1. Pick the right emit: `socket.emit` / `socket.broadcast.emit` / `io.emit` (and name `socket.join` + `io.to(room).emit` as the next move)


<!-- > -->

## [**5m**] Attendance &amp; Announcements

Tonight is the phone-call mental model and the wiring: Express 5 + `http.Server` + Socket.IO 4. Day 7 is tradeoffs in the wild plus lab time on [Make Chat Tutorial](https://github.com/Tech-at-DU/Make-Chat-Tutorial).


<!-- > -->

## [**10m**] ☀️ Warm Up

<p align="center"><img src="assets/litebrite.gif" alt="LiteBrite live board demo gif"></p>

Products that feel *live* are not refreshing the page. A board lights up. A chat row appears. A cursor moves. That is a **long-lived connection**, not a new HTTP request per click.

**Check out this [LiteBrite Demo](https://litebrite.live/)!** If the live board is down, stay on the gif and the still below.

- Keep one tab on the demo
- In another tab, open **Play Now** if the site offers it
- Watch a cell change from *someone else* — you did not reload

<p align="center"><img src="assets/howitworks.jpg" alt="How a live board updates over a long-lived connection"></p>

> “HTTP is a letter. You send one, you get one back, the clerk hangs up. A WebSocket is a phone call: both sides can talk, and they can talk at the same time.”

| HTTP(S) | WS / WSS |
| --- | --- |
| Request → response, then the conversation is over | One handshake, then frames either way |
| Client usually starts every exchange | Server can **push** without being asked again |
| Great for documents, forms, REST | Great for chat, presence, live boards, collab cursors |

> **📈 PROTIP:** A receipt page that loads once is still HTTP. “Live UI” alone does not prove you need a socket.

Think and jot (30s), then unmute or chat: name one product you used this week that updated **without** you refreshing.

<!-- > -->

## [**40m**] 📚 TT: Overview

**Next action:** Why a socket → upgrade → attach to `http.Server` → JS handlers/modules → the right emit.  
**Done when:** You can sketch “browser `io()` → same-origin Socket.IO → `http.Server` + Express” and say which `emit` reaches whom.  

### 1. Why a long-lived socket (~5m)

<p align="center"><img src="assets/chat-example.gif" width="600" alt="Chat example showing messages arriving without a page reload" /></p>

**Bidirectional:** both parties send and receive.  
**Full-duplex:** send and receive can happen **at the same time** — like a phone call, not a walkie-talkie.

Use it when the **server** has something to say and cannot wait for the next page load:

| Socket is a fit | Stay on HTTP |
| --- | --- |
| Chat, typing indicators, presence | CRUD that already has a request/response |
| Live boards / LiteBrite-style cells | Search, pagination, “submit this form” |
| Collab cursors, shared docs | One-shot webhooks you already poll on a job |

Socket.IO is **not** the WebSocket standard. It is a library: it prefers WebSocket after an HTTP handshake, can fall back to HTTP long-polling, and adds **named events**, **rooms**, **acknowledgements**, and **reconnect**. The protocol underneath is still **WS** or **WSS**.

> “Today the pattern is a live channel. The outcome is JS you would ship: handlers, closures, modules, and an emit you can defend in review.”

<!-- -->

> **💬 ASK QUESTION:** Slack-style live board vs loading a blog post — which needs a long-lived socket?

<details>
<summary>Answer</summary>

**The live board.** The server must push cell/chat updates without waiting for the next page load. A blog post is a one-shot HTTP document.

</details>

### 2. The upgrade — HTTP, then not HTTP (~6m)

The WebSocket standard starts with an **HTTP handshake**, then **switches** to WS / WSS. Later messages are frames, not `GET`/`POST`.

<p align="center">
  <img src="assets/WebSockets-Diagram.png" height="420" alt="WebSocket handshake then frame diagram">
</p>

Same picture, upgrade called out:

<p align="center">
  <img src="assets/WebSockets-Diagram-Explained.png" height="420" alt="WebSocket upgrade annotated: HTTP 101 then WS frames">
</p>

Request (shape from the opening handshake — browsers set `Sec-WebSocket-Key` for you):

```txt
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```

Server response:

```txt
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

1. Client sends `Sec-WebSocket-Key` — a nonce. The browser adds it; you do not invent it in page JS.
1. Server answers `101` plus `Sec-WebSocket-Accept` derived from that key. That proves “I speak WebSocket,” not “this user is logged in.”

> **‼️ BE AWARE:** The upgrade is a protocol switch, not authentication. Identity still lives in **your** handshake checks (cookies, tokens) — out of scope for tonight’s MVP.

Native browser API (MDN `WebSocket`) sends **strings / binary**. You stringify yourself:

```js
const ws = new WebSocket('ws://localhost:3000');
ws.addEventListener('open', () => {
  ws.send(JSON.stringify({ text: 'hello' }));
});
ws.addEventListener('message', (event) => {
  console.log(event.data); // a string unless you chose binary
});
```

Socket.IO is a different client: `io()` from `/socket.io/socket.io.js`, **named events**, objects without `JSON.stringify`. Do not mix the two constructors in one lab.

<!-- -->

### 3. Attach Socket.IO to `http.Server` (~7m)

Official Socket.IO path (current docs, **v4** — latest release **4.8.3** as of Dec 2025; verify day-of): a server that **mounts on** Node’s HTTP server, plus a client that loads from that same origin.

**Documented caution:** `app.listen(3000)` creates a *new* HTTP server. Socket.IO will not be on it.

```js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app); // Socket.IO needs this http.Server
const io = new Server(server);

app.get('/', (req, res) => {
  res.sendFile(__dirname + '/index.html');
});

io.on('connection', (socket) => {
  console.log('a user connected');
  socket.on('disconnect', (reason) => {
    console.log('user disconnected', reason);
  });
});

server.listen(3000, () => {
  console.log('listening on *:3000');
});
```

Client on the page Express just served (same origin — `io()` with no URL):

```html
<script src="/socket.io/socket.io.js"></script>
<script>
  const socket = io();
</script>
```

That script tag is the client the Socket.IO server **serves for you** at `GET /socket.io/socket.io.js`. Opening a raw `file://` HTML file will not hit that endpoint.

| Piece | Job |
| --- | --- |
| `app` | Express routes, `sendFile`, later REST |
| `server` | Node `http.Server` — **listen here** |
| `io` | Socket.IO — upgrade + events on **that** server |
| `io()` in the browser | Client; default URL is the page’s host |

> “Express handles HTTP routes. Socket.IO rides the same port and steals the upgrade. If someone asks why both, that’s the sentence.”

<!-- -->

> **💬 ASK AUDIENCE:** Does Socket.IO attach to the Express `app`, or to the `http.Server`?

<details>
<summary>Answer</summary>

**The `http.Server`.** `const server = http.createServer(app); const io = new Server(server);` then `server.listen(...)`. `app.listen(3000)` will not work — it creates a different HTTP server.

</details>

### 4. Realtime handlers — events, closures, modules (~14m)

Goal: ship **realtime handlers** in Node — named events, closures, a small module. The socket is the vehicle.

Socket.IO’s API is **EventEmitter-shaped**: `on` on one side, `emit` on the other. Any JSON-serializable value is fine. **Do not** `JSON.stringify` objects before `emit` — the library already encodes them.

```js
// BAD
socket.emit('hello', JSON.stringify({ name: 'Ada' }));

// GOOD
socket.emit('hello', { name: 'Ada' });
```

`Date` arrives as an ISO string. `Map` / `Set` must be turned into arrays yourself (`[...myMap.entries()]`). That is JS, not “a socket thing.”

**Closure:** every connection gets its own `socket`. Handlers you register *inside* `connection` close over **that** socket. That is how `broadcast` knows who the sender is.

**Named functions** beat anonymous soup when you need to read the file in five minutes:

```js
function registerChatHandlers(io) {
  function onConnection(socket) {
    const { id } = socket;

    function onChatMessage(msg) {
      if (typeof msg !== 'string') return;
      const text = msg.trim();
      if (text === '') return;

      io.emit('chat message', { id, text, at: Date.now() });
    }

    function onDisconnect(reason) {
      console.log('disconnected', id, reason);
    }

    socket.on('chat message', onChatMessage);
    socket.on('disconnect', onDisconnect);
  }

  io.on('connection', onConnection);
}

module.exports = { registerChatHandlers };
```

Entry file stays thin:

```js
const { registerChatHandlers } = require('./sockets/chat');

registerChatHandlers(io);
```

Same move as a `utils/mailer.js` helper: **extract, export, call from the boot file.** Do not dump every listener into `index.js`.

**Acknowledgements** — last argument of `emit` is a callback. The other side **calls** it. That is Node-style, not a new protocol.

```js
// receiver (server)
socket.on('update item', (itemId, payload, callback) => {
  callback({ status: 'ok' });
});
```

```js
// sender (client)
socket.emit('update item', '1', { name: 'updated' }, (response) => {
  console.log(response.status); // ok
});
```

Timeout (Socket.IO **≥ 4.4.0**):

```js
socket.timeout(5000).emit('my-event', (err, response) => {
  if (err) {
    // the other side did not acknowledge in time
    return;
  }
  console.log(response);
});
```

**Anti-patterns to call out (rookie traps):**

- `.then()` on something that is not a Promise — `on` / `emit` are not `fetch`
- Fire-and-forget `emit` when you needed an ack (and then lying to the UI)
- Mutating a module-level array from every connection without saying who owns it
- Trusting `msg` is a string because “the input is a text box”

Rooms leave themselves on disconnect. Official docs: no special teardown. Use `disconnecting` if you still need `socket.rooms` (a `Set`) before they empty.

### 5. The right emit (~6m)

| Call | Who receives it |
| --- | --- |
| `socket.emit(...)` | **That one** socket |
| `socket.broadcast.emit(...)` | **Everyone else**, not the sender |
| `io.emit(...)` | **Everyone** connected, including the sender |
| `socket.join('room')` then `io.to('room').emit(...)` | Everyone **in that room**, including the sender |
| `socket.to('room').emit(...)` | Everyone in that room **except** the sender |

`join` / `leave` / `to` / `in` (`to` and `in` are the same) are **server-only**. The client does not get a room list.

Tonight’s default (matches the official chat guide): `io.emit('chat message', msg)` so the sender sees their own line via the same event. Stretch: append locally and `socket.broadcast.emit` instead.

```js
io.on('connection', (socket) => {
  socket.on('chat message', (msg) => {
    io.emit('chat message', msg);
  });
});
```

```html
<script src="/socket.io/socket.io.js"></script>
<script>
  const socket = io();
  const form = document.getElementById('form');
  const input = document.getElementById('input');
  const messages = document.getElementById('messages');

  form.addEventListener('submit', (e) => {
    e.preventDefault();
    if (input.value) {
      socket.emit('chat message', input.value);
      input.value = '';
    }
  });

  socket.on('chat message', (msg) => {
    const item = document.createElement('li');
    item.textContent = typeof msg === 'string' ? msg : msg.text;
    messages.appendChild(item);
    window.scrollTo(0, document.body.scrollHeight);
  });
</script>
```

<!-- -->

> **💬 QUICK CHECK:** Server handles `chat message` with `socket.emit('chat message', msg)` — who sees it?

<details>
<summary>Answer</summary>

**Only that one socket** (usually the sender). Use `io.emit` for everyone, or `socket.broadcast.emit` for everyone except the sender.

</details>

### 6. Bridge into practice (~2m)


<!-- -->

> **💬 YOUR TURN:** What is the done-state for Activity 1?

<details>
<summary>Answer</summary>

Terminal shows `a user connected` (and a disconnect if you refresh), plus one inbound `chat message` log from the page form — same origin, not `file://`.

</details>

<!-- > -->

## [**10m**] 🌴 Break

Stand up. Leave the process running if you want — or kill it and restart after.  

<!-- > -->

## [**20m**] 💻 Activity 1: Attach Socket.IO to HTTP Server

> **✅ DONE WHEN:** Terminal prints a connect line and one inbound `chat message`. One browser tab on `http://localhost:3000` (not `file://`).

**Connection + one emit.** Solo · visible checkpoint · artifact.

| | |
| --- | --- |
| **Next action** | Empty folder → `npm install express@5 socket.io` → `http.createServer(app)` → `new Server(server)` → serve `index.html`. |
| **Done when** | Terminal prints `a user connected` and one `chat message` line from the form. |
| **Artifact** | Screenshot or pasted log lines in your notes / shared channel when asked. |
| **Checkpoint (visible)** | “I see a user connected” in chat or unmuted shout-out. |

### Steps

Follow the official [Socket.IO chat getting started](https://socket.io/get-started/chat) through **Emitting events** (log the message on the server). Stay on **Express 5 + CJS** — that is what the guide still ships.

1. `mkdir` a throwaway folder. `npm init -y`. Unique `"name"` (not `socket.io` / `express`).
2. `npm install express@5 socket.io`
3. `index.js`: Express `app`, `http.createServer(app)`, `new Server(server)`, `app.get('/', ... sendFile ...)`, `io.on('connection', ...)`, **`server.listen(3000)`**
4. `index.html`: form + `<script src="/socket.io/socket.io.js"></script>` + `const socket = io();` + submit → `socket.emit('chat message', input.value)`
5. Run `node index.js`. Open `http://localhost:3000` (not the file on disk).
6. Submit once. Confirm the terminal log.


If you finish early, help a peer who’s stuck.

<!-- > -->

> **📈 TIP:** Unique `package.json` `"name"` — not `socket.io` / `express` — avoids install confusion.
> **FINISHED EARLY?** Log `socket.id` on connect, or sketch `socket.broadcast.emit` for Activity 2.

## [**30m**] 💻 Activity 2: Emit Across Tabs and Extract Handlers

> **✅ SHIP WHEN:** Two browser tabs share a message via `io.emit` (or broadcast + local append), and handlers live in a `module.exports` file (or you are one clear extract step away).

**Realtime handlers — modular emit + the right audience.** You do. Same topic, less scaffolding.

| | |
| --- | --- |
| **Next action** | Extract `registerChatHandlers(io)` into `sockets/chat.js`; `io.emit` so two tabs share a line. |
| **Done when** | Two browser tabs on `http://localhost:3000` show the same message; handlers live in a module the entry file `require`s. |
| **Artifact** | Two-tab screenshot **or** pasted `{ id, text }` log + `sockets/chat.js` path. |
| **Checkpoint** | Paste one received payload (no secrets) when both tabs update. |

### Steps

1. Create `sockets/chat.js` exporting `registerChatHandlers` (TT Pattern). Keep CJS unless your folder is already `"type": "module"`.
2. Guard the payload (`typeof` + `trim`). Decide: emit a **string** (official guide) *or* a small object `{ id, text, at }`. Pick **one** and match the client `textContent` line.
3. `io.emit('chat message', ...)` so the sender and everyone else go through the same listener.
4. Wire the client `socket.on('chat message', ...)` to append an `<li>`.
5. Open a **second** tab. Send from tab A. Tab B updates without refresh.
6. **JS stretch (pick one, not all):**
   - Named `onChatMessage` / `onDisconnect` (no anonymous soup)
   - `socket.broadcast.emit` plus append locally (official homework: don’t echo to the sender on the wire)
   - Acknowledgement: client `emit`s with a callback; server calls `callback({ status: 'ok' })`; log it
   - `socket.timeout(5000).emit(...)` and handle `err`

### Stretch (if ahead)

Official homework ideas — do **one**: connect/disconnect broadcast, nicknames, or `{user} is typing`. Full Challenge 2 (“at least two homework bullets”) is **after hours**, not tonight’s gate.

<!-- > -->

> **‼️ WATCH OUT:** Opening `index.html` via `file://` skips the Socket.IO handshake origin. Serve from `http://localhost:3000`.
> **FINISHED EARLY?** Prefer `socket.broadcast.emit` plus local append, or add a typing event from the official homework list.


## [**5m**] Wrap Up

Takeaways to say out loud:

1. HTTP is a letter; **WS / WSS** is a phone call — upgrade first (`101`), then frames.
2. Socket.IO **mounts on `http.Server`**. `app.listen` is the classic miss.
3. **JS you ship:** named handlers, closures over `socket`, payload guards, `module.exports`.
4. **`io.emit` / `broadcast` / `socket.emit`** are different audiences — say which one you meant.
5. Acknowledgements are a **callback as the last `emit` argument**, not a Promise unless you wrap them.


```text
Shipped today:
Stuck on:
Tomorrow's first 15m:
```

- What to finish before the next block: Activity 2 two-tab artifact if unfinished
- Where to submit: per shared channel / notes
- One thing to try if stuck: serve via `http://localhost:3000`, not `file://`; confirm `server.listen`, not `app.listen`

<!-- > -->

## Additional Resources

1. **[Socket.IO — Get started (chat)](https://socket.io/get-started/chat)** — official Express + `http.Server` + emit/broadcast lab.
2. **[Socket.IO — Server initialization](https://socket.io/docs/v4/server-initialization/)** — `new Server(httpServer)`; **`app.listen` will not work**.
3. **[Socket.IO — Server installation](https://socket.io/docs/v4/server-installation/)** — current release notes (v4.8.3 cited Dec 2025; re-check day-of).
4. **[Socket.IO — Emitting events](https://socket.io/docs/v4/emitting-events/)** — `on`/`emit`, no `JSON.stringify`, acknowledgements, `timeout`.
5. **[Socket.IO — Rooms](https://socket.io/docs/v4/rooms/)** — `join` / `leave` / `to` / `in`; rooms are server-only.
6. **[Socket.IO — Server API](https://socket.io/docs/v4/server-api/)** — `connection`, `disconnect` / `disconnecting`, `socket.id`.
7. **[MDN — WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)** — native `WebSocket`, handshake headers, `ws:` / `wss:`.
8. **[High Performance Browser Networking — WebSocket](https://hpbn.co/websocket/)** — protocol and the upgrade’s limits (not auth).

**Do not use as primary:** old PubNub / HTML5 Rocks / CodePen links from the previous stub (stale stacks; they are not the Socket.IO v4 path).

## For Curriculum Authors

<details>
<summary>For Curriculum Authors</summary>

## For curriculum authors

### In Class

| | |
| --- | --- |
| **Next action** | Open this file → skim the agenda table → start Why / Objectives at the hour. |
| **Done when** | Two tabs share a message from a module that exports `registerChatHandlers`, and you can say why `app.listen` fails. |

```text
Feeling (1 word):
Behind | On track | Ahead:
Today's MVP (1 sentence): two tabs share one chat line; handlers live in sockets/chat.js
```

### Facilitator notes

**Broken-link note (repo stub):** PubNub (2015), HTML5 Rocks, and the old CodePen were the previous Resources list. Prefer official Socket.IO v4 + MDN (Resources above). Do **not** demo from those stubs.

- Voice: write-like-you-talk. Short blocks. GOAL first.
- Realtime handlers are the outcome; sockets are the vehicle. If the clock slips, **cut handshake history**, not the module / emit / ack beat.
- Official chat guide still uses **Express 5 + CJS** (`require`). Match it live. ESM only if a room already set `"type": "module"`.
- Live-code the first five minutes of `http.createServer` + `new Server(server)` only. Then get out of the way.
- Two-tab demo is the aha. Do it once on the projector before Activity 2.
- If `litebrite.live` is down, gif + `howitworks.jpg` — do not burn Warm Up on a dead tab.
- `file://` and `app.listen` are the two failure modes you will see. Debrief one in the main room after Activity 1.
- Day 7 owns polling vs sockets and alternatives. Name polling in one sentence tonight; do not steal that block.
- Challenge 2 / Make Chat are **after hours / later days**, not tonight’s gate. Activity 2 stretch points at one official homework bullet.
- Have one extension ready for rooms that finish Activity 2 early (ack or `broadcast` + local append).

### Expert follow-ups

Flag for follow-up (do not block today’s live block):

1. **Guide vs course default** — Day 6 pins **CJS + Express 5** with Socket.IO 4 (smoke-tested). Re-check the official chat guide day-of if its default major changes.
2. **Release pin** — cite `socket.io@4.8.3` (Dec 2025) in handouts, or `npm install socket.io` and re-read [Server installation](https://socket.io/docs/v4/server-installation/) each term?
3. **LiteBrite** — is `litebrite.live` still the Warm Up, or should the gif + a local board replace the live site?
4. **Rooms on Day 6** — `join` / `to` are documented and tempting; confirm they stay **named only** tonight so Day 7 / Make Chat still have a climb.
5. **Auth on handshake** — `socket.handshake` exists; cookie/token checks are out of scope for the MVP. Which later session owns them?
6. **Challenge 2 wording** — still “complete the guide + at least 2 homework bullets,” or retitle now that Activity 2 already starts the guide?
7. **Native `WebSocket` vs Socket.IO** — one contrast slide tonight; confirm we do **not** add a `ws` server lab on Day 6.
8. **Acknowledgements vs Promises** — `emitWithAck` exists on the server API; tonight teaches the **callback-as-last-arg** from the emitting-events guide. Standardize later?

</details>
