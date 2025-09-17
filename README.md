**Goal:** Build a minimal one-to-one random chat (Omegle-like) app. Requirements:

* Randomly pair two strangers for a 1:1 text chat.
* Show the total number of online strangers (people currently waiting or connected).
* Simple UI built with **shadcn/ui** and **Tailwind** (Tailwind is already used by shadcn).
* Use **Next.js (app router OK)** and **TypeScript**.
* Realtime communication via **WebSocket** (Node `ws`) or `next` WebSocket solution.
* Keep scope minimal — no accounts, no persistent DB (in-memory for matching/online count).
* Basic safety: rate limiting & ephemeral IDs; do **not** log chat messages to disk.

---

## Features (MVP)

1. **Landing page** — shows a big "Start Chat" button and the live counter `Total online: N`.
2. **Matchmaking** — on "Start Chat" the client opens a WebSocket & enters a waiting queue; when matched, they go to the chat UI.
3. **Chat UI** — simple message list, input box, send button, "Next" button to disconnect and find a new partner.
4. **Presence counter** — shows how many users are currently online (waiting + connected).
5. **Disconnect handling** — when a user disconnects, their partner is returned to the waiting queue (or sees "Partner disconnected").

---

## Suggested architecture

* **Server**: Next.js API route (or a small separate Node server) that upgrades to WebSocket using `ws` (or `uWebSockets.js`), keeps an **in-memory** map:

  * `clients: Map<clientId, WebSocket>`
  * `waitingQueue: string[]` (client IDs waiting)
  * `pairs: Map<clientId, partnerClientId>`
  * Broadcast presence to all clients when counts change.
* **Client**: Next.js + React + TypeScript. Connect to ws endpoint and exchange JSON messages (join, msg, leave, heartbeat, presence).
* **Messaging protocol** (JSON over WS):

  ```ts
  // from client
  { type: 'join', id: string }
  { type: 'msg', text: string }
  { type: 'next' } // user wants a new partner
  { type: 'heartbeat' }

  // from server
  { type: 'paired', partnerId: string }
  { type: 'message', from: string, text: string }
  { type: 'partner-disconnected' }
  { type: 'presence', totalOnline: number }
  { type: 'waiting' }
  ```

---

## Minimal TypeScript interfaces

```ts
type ClientId = string;

interface ServerMessage {
  type: 'paired' | 'message' | 'partner-disconnected' | 'presence' | 'waiting';
  partnerId?: ClientId;
  from?: ClientId;
  text?: string;
  totalOnline?: number;
}

interface ClientMessage {
  type: 'join' | 'msg' | 'next' | 'heartbeat';
  id?: ClientId;
  text?: string;
}
```

---

## UI / Component spec (use shadcn/ui)

* **Header**: small brand + `Total online: N` (use `Card` or `Badge`)
* **Landing**: `Card` with big `Button` -> Start Chat
* **Chat screen**:

  * `Card` containing message list (auto-scroll).
  * `Input` field and `Button` to send.
  * `Button` (secondary) for "Next".
  * Small status line: "Connected — Partner: anonymous" or "Waiting for partner..."
* Use accessible semantic elements (aria labels).

---

## File structure (suggested)

```
/app
  /page.tsx         // landing + online counter
  /chat/page.tsx    // chat UI (client connects to ws when loaded)
  /components/      // shadcn components + small helpers
/pages
  /api/ws.ts        // websocket upgrade and server logic (or /app/api)
lib/
  wsServer.ts        // server matching logic (if extracted)
types/
  index.ts
```

---

## Minimal server matching algorithm (pseudo)

1. When a client connects and sends `join`:

   * Add to `clients` map.
   * If `waitingQueue` has an ID, pop it and pair them:

     * `pairs[a] = b`, `pairs[b] = a`
     * Notify both via `{type: 'paired', partnerId: ...}`
   * Else push client ID into `waitingQueue` and send `{type:'waiting'}`
2. On `msg` from client:

   * If paired, forward `{type:'message', from: id, text}` to partner.
3. On `next` or disconnect:

   * If paired, notify partner with `partner-disconnected`, unpair, and put the partner back into the waiting queue (attempt immediate rematch).
4. Broadcast presence (`totalOnline = clients.size`) whenever clients join/leave.

Notes: keep a periodic heartbeat to drop stale connections.

---

## Example minimal server snippet (Node + ws)

(put this in `pages/api/ws.ts` or a tiny separate server)

```ts
// server-side (simplified)
import { WebSocketServer } from 'ws';
const wss = new WebSocketServer({ noServer: true });

const clients = new Map<string, WebSocket>();
const waitingQueue: string[] = [];
const pairs = new Map<string, string>();

function broadcastPresence() {
  const msg = JSON.stringify({ type: 'presence', totalOnline: clients.size });
  for (const ws of clients.values()) ws.send(msg);
}

function tryPair(aId: string, bId: string) {
  pairs.set(aId, bId);
  pairs.set(bId, aId);
  clients.get(aId)?.send(JSON.stringify({ type: 'paired', partnerId: bId }));
  clients.get(bId)?.send(JSON.stringify({ type: 'paired', partnerId: aId }));
}

wss.on('connection', (ws) => {
  let id = randomId();
  clients.set(id, ws);
  broadcastPresence();

  ws.on('message', (raw) => {
    const msg = JSON.parse(raw.toString());
    if (msg.type === 'join') {
      if (waitingQueue.length > 0) {
        const other = waitingQueue.shift()!;
        tryPair(id, other);
      } else {
        waitingQueue.push(id);
        ws.send(JSON.stringify({ type: 'waiting' }));
      }
    }
    if (msg.type === 'msg') {
      const partner = pairs.get(id);
      if (partner) {
        clients.get(partner)?.send(JSON.stringify({ type: 'message', from: id, text: msg.text }));
      }
    }
    if (msg.type === 'next') {
      // handle next: disconnect pair and requeue partner
    }
  });

  ws.on('close', () => {
    // cleanup: remove from clients, waitingQueue, pairs
    clients.delete(id);
    // if paired, inform partner...
    broadcastPresence();
  });
});
```

---

## Client-side notes (chat/page.tsx)

* On mount: open WebSocket to `/api/ws`. Send `{type: 'join', id}`.
* Maintain `state`:

  * `status: 'waiting' | 'paired' | 'disconnected'`
  * `messages: {from, text}[]`
  * `totalOnline: number`
* Render presence in header. When server emits `presence`, update counter.
* When receiving `paired`, set `status = 'paired'`.
* When sending message: `ws.send(JSON.stringify({type:'msg', text}))`.
* Implement "Next" button to send `{type:'next'}` and clear `messages`.

---

## UX/Design tips (shadcn)

* Use `Card` for central chat container, `Textarea` (or Input) for message entry, and `Button` variants for primary/secondary actions.
* Keep colours neutral; show system messages (waiting/partner left) as `Badge` or `Alert`.

---

## Security & Privacy (important)

* Use ephemeral client IDs (random UUID) stored only in memory.
* **Do not** store messages on server. If a DB is added later, encrypt or delete logs frequently.
* Rate-limit message sending to prevent spam (simple client-side throttle + server check).
* Use HTTPS + secure WebSocket (wss\://) in production.
* Consider WebRTC for P2P later (for audio/video), but text over ws is fine for MVP.

---

## Acceptance criteria

* [ ] Clicking Start opens a chat or shows waiting state.
* [ ] Two users are paired and can exchange messages in realtime.
* [ ] "Total online" updates as users connect/disconnect.
* [ ] "Next" disconnects current pairing and tries to find another partner.
* [ ] No persistent storage of chat messages.

---

## Dev commands / setup

* `npx create-next-app@latest --ts` (or the boilerplate you prefer)
* Add `tailwindcss` + `shadcn/ui` per their docs.
* Add `ws` (`npm i ws`) for server-side WebSocket if you use a Node server approach.

---

## Optional extras (if you want)

* Show approximate waiting time (simple heuristic).
* Add a small emoji pack or topic selection (match with same topic).
* Add profanity filter before forwarding messages (basic bad-words list).


