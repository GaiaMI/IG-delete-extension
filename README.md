# Instagram — Bulk Unsend

Browser extension that deletes **your own messages** from an Instagram conversation, in bulk, using the web client's internal API.

No click simulation: no hovering over messages, no opening the "…" menu, no confirmation dialog to dismiss. The extension sends the same requests the site itself sends, which makes it far faster and much less brittle than a script that drives the interface.

---

## ⚠️ Before you start

- **Unsending is permanent.** An unsent message disappears for you *and* for the other person. There is no undo.
- **Always start with Dry run.** It counts the matching messages and reports their date range without touching anything.
- **Instagram may rate limit your account** if you delete too much too fast. The extension stops on its own when it detects a block, but the "Turbo" preset is not advisable across thousands of messages.
- **Automating the site goes against Instagram's Terms of Use.** Use it on your own account, at your own risk.
- The extension only targets messages **you** sent (`is_sent_by_viewer`). The other person's messages are never touched.

---

## Installation

The extension is not published on any store — it loads in developer mode. It works on Chrome, Brave, Edge, Vivaldi and Opera: any recent Chromium browser (Chrome 111+, required for `world: "MAIN"` support).

1. Download or clone this repository.
2. Open the extensions page:
   - Chrome → `chrome://extensions`
   - Brave → `brave://extensions`
   - Edge → `edge://extensions`
3. Enable **Developer mode** (toggle in the top right).
4. Click **Load unpacked** and select the repository folder.
5. Pin the icon to the toolbar for easy access.

After editing any file, come back to this page and click ↻ to reload the extension.

---

## Usage

1. Open `instagram.com` and **enter the conversation** you want to clean up (the URL should look like `instagram.com/direct/t/...`).
2. If you just installed or reloaded the extension, **reload the page** so the engine gets injected.
3. Click the extension icon.
4. Leave **Dry run** checked and click **Start**. You get the exact number of matching messages and their date range.
5. If the count looks right: uncheck Dry run, set **Max** to 5 for a first real attempt, then set it back to 0 to process everything.

A badge appears in the bottom-right corner of the page while it runs, showing progress and a **Stop** button. It stays active even if you close the popup.

### Settings

| Setting | Effect |
|---|---|
| **Speed preset** | Delay between two unsends: Careful 800 ms, Normal 350 ms, Fast 150 ms, Turbo 60 ms. |
| **Custom delay** | Editing the value by hand switches the preset to "Custom". |
| **From / To** | Only process messages in that range. Empty means the whole conversation. Dates come from API timestamps, so they are exact. |
| **Max** | Cap on the number of unsends. 0 means no limit. Useful for a first run. |
| **Dry run** | Counts and reports, deletes nothing. |

If you set a **From** bound, reading stops as soon as it goes past that date instead of crawling the entire conversation.

### Automatic stops

The extension stops by itself when:

- Instagram returns HTTP 429 or a `feedback_required` / `spam` style response → wait a few hours before resuming;
- 5 unsends fail in a row;
- you click **Stop**.

Messages are unsent from **oldest to newest**, so stopping midway leaves a contiguous remaining block.

---

## How it works

### Architecture

| File | Role |
|---|---|
| `engine.js` | The whole engine. Runs in the **MAIN world**, i.e. the page's own context. |
| `bridge.js` | The extension's isolated world. Relays popup commands to the engine over `postMessage` and reports state back. |
| `popup.html` / `popup.js` | Settings and controls. |

Why the MAIN world: in an MV3 extension, `fetch` calls issued from a regular content script carry a `chrome-extension://…` origin, which Instagram may reject. Running in the page's context makes the engine's requests indistinguishable from the official client's. The trade-off is that `chrome.runtime` is not reachable there, hence the `bridge.js` relay.

### Request chain

**1. Resolving the thread**

The id in the `/direct/t/<id>/` URL is **not** the API's `thread_id` — it is the `messaging_thread_key` (the recipient's messaging id). Passing it straight to the threads endpoint returns HTTP 500.

The real `thread_id` is a 39-digit number, found by walking the inbox:

```
GET /api/v1/direct_v2/inbox/?...&limit=40[&cursor=…]
```

The inbox is paginated and the conversation you want can be far down it — in one real case it sat on page 7. The extension walks up to 40 pages.

**2. Reading the messages**

```
GET /api/v1/direct_v2/threads/<thread_id>/?direction=older&limit=50[&cursor=…]
```

Cursor pagination (`oldest_cursor` + `has_older`) walks the whole conversation server-side: it is the equivalent of scrolling, without any scrolling. Each item exposes `is_sent_by_viewer`, `timestamp` (in **microseconds**) and `message_id`.

Every message is collected **before** the first deletion: cursors become inconsistent if you delete while paginating.

**3. Unsending**

This is not a REST endpoint but a **GraphQL mutation**:

```
POST /api/graphql          (application/x-www-form-urlencoded)

  doc_id    = 26948700068153789
  variables = {"message_id":"mid.$…","send_data":{"thread_id":"3402…"}}
  av, fb_dtsg, jazoest, lsd, fb_api_caller_class=RelayModern, server_timestamps=true
```

Expected response:

```json
{"data":{"direct_unsend_message":true}}
```

Two gotchas:

- **`message_id` is not `item_id`.** The mutation expects the `mid.$…` form, not the 35-digit number. Both are present on the item returned by the REST API.
- **`fb_dtsg` and `lsd` are required.** They are injected into the page's HTML and extracted with a regular expression. `jazoest` is derived from them: `"2"` followed by the sum of the character codes of `fb_dtsg`. If those tokens expire mid-run, the extension fetches fresh ones and retries.

---

## Troubleshooting

**"conversation not found in the inbox"**
The conversation sits beyond inbox page 40. Open it, send or receive a message to bump it up, or raise the limit in `resolveThreadId()`.

**"session tokens not found"**
Reload the Instagram page. If it persists, sign out and back in.

**The popup says "No Instagram tab found"**
The engine is not injected: reload the Instagram tab after (re)loading the extension.

**Everything fails at once, when it used to work**
Most likely the `doc_id` changed — it is tied to the current version of the web client. To recover the new one:

1. Open DevTools (F12) → **Network** tab, filter on `graphql`.
2. Unsend **one** message by hand ("…" menu → *Unsend*).
3. Open the `POST /api/graphql` request that just fired, **Payload** tab.
4. Read off `doc_id` and check the shape of `variables`.
5. Put the value into the `UNSEND_DOC_ID` constant at the top of `engine.js`.

---

## Privacy

Nothing leaves `instagram.com`.

- No third-party server, no telemetry, no service worker or background script.
- Requested permissions: `storage` (to remember your settings) and `host_permissions` scoped to `https://www.instagram.com/*`.
- Session cookies and tokens are read at runtime and used only to sign requests to Instagram. They are never stored or sent anywhere else.
- No personal identifier is hardcoded. The only two numeric constants are public and identical for every user: the Instagram web client's app id and the mutation's `doc_id`.

---

## Known limitations

- The endpoints and the `doc_id` are internal and undocumented: an Instagram update can break the extension overnight.
- Message types without a usable `message_id` are skipped and counted separately.
- One conversation at a time — whichever is currently open.
- Not tested on group conversations.

---

## License

MIT — see [LICENSE](LICENSE).
