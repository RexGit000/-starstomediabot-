# Implementation Tasks: Delivery Pipeline Migration

## Task 1: Add new dependencies (lru-cache, telegram) to package.json

- **Status**: pending
- **Priority**: high
- **AC Coverage**: FR9, AC5 (userbot login requires `telegram` package)

### What to do
1. Read current `package.json`.
2. Add `"lru-cache": "^10.2.0"` and `"telegram": "^2.20.15"` (or latest compatible) to `dependencies`.
3. Run `npm install` from the project root.
4. Confirm `node_modules/telegram` and `node_modules/lru-cache` exist.

### Test Requirements (rule / rubric)

- **TR1.1 (rule)**: `require('lru-cache')` and `require('telegram')` both resolve without throwing from a Node REPL at project root.
- **TR1.2 (rule)**: `node -e "require('package.json').dependencies"` lists both packages.

### Files touched
- `package.json`
- `package-lock.json` (auto)

---

## Task 2: Rewrite Media model (src/models/Media.js) to new schema

- **Status**: pending
- **Priority**: high
- **AC Coverage**: FR1, AC1

### What to do
1. Replace entire contents of `Media.js`:
   - Remove fields: `fileId`, `fileType`, `channelId`, `channelMessageId`, `addedAt`.
   - Add: `source: { channel_id, message_id }` (embedded sub-schema, required).
   - Add: `metadata: { kind, mime_type, file_name, file_size, uploaded_at }` (embedded sub-schema, kind enum video/photo/document).
   - Add: `bot_file_ids: Mixed, default {}`.
   - Add: `file_unique_id: String, sparse unique`.
   - Add: `mtproto: { type, id, access_hash, file_reference, dc_id }` (reserved, default null).
   - Add: `last_seen_at: Date, default Date.now`.
   - Keep `timestamps: true`.
2. Indexes:
   - Unique compound `{ 'source.channel_id': 1, 'source.message_id': 1 }`.
   - Sparse unique `{ file_unique_id: 1 }`.
   - Sparse unique `{ 'mtproto.id': 1 }`.
   - `{ 'metadata.uploaded_at': -1 }` (for admin listing).

### Test Requirements

- **TR2.1 (rule)**: `node -e "const M=require('./src/models/Media'); console.log(Object.keys(M.schema.paths))"` shows `source.channel_id`, `metadata.kind`, `bot_file_ids`, `last_seen_at` as paths.
- **TR2.2 (rule)**: `Media.listIndexes()` equivalent — confirm 3 unique indexes exist (compound source, file_unique_id sparse, mtproto.id sparse).

### Files touched
- `src/models/Media.js`

---

## Task 3: Add UserbotAccount model (src/models/UserbotAccount.js)

- **Status**: pending
- **Priority**: high
- **AC Coverage**: FR6, AC5, AC6

### What to do
1. Create new file `src/models/UserbotAccount.js` matching Pbox schema:
   - `number: String, default null, sparse unique index`
   - `username: String, default null`
   - `userId: String, default null`
   - `session: String, default null, sparse index`
   - `{ timestamps: true }`
2. Export `mongoose.model('UserbotAccount', schema)`.

### Test Requirements

- **TR3.1 (rule)**: `require('./src/models/UserbotAccount')` returns a valid mongoose model.
- **TR3.2 (rule)**: Creating a new UserbotAccount with `session: 'abc'` saves and loads correctly (use in-memory logic check only — actual DB round-trip optional if DB not running).

### Files touched
- `src/models/UserbotAccount.js` (new)

---

## Task 4: Port helpers: telegram.js + fingerprint.js

- **Status**: pending
- **Priority**: high
- **AC Coverage**: FR7 (login flow needs DC migration + random fingerprints)

### What to do
1. Create `src/helpers/fingerprint.js`:
   - Copy from Client Pbox (DEVICE_POOL + `randomFingerprint()`).
2. Create `src/helpers/telegram.js`:
   - Copy `getDCAddress`, `sleep`, `sendCodeWithRetry` from Pbox.
   - Ensure it reads `process.env.API_ID` / `process.env.API_HASH`.

### Test Requirements

- **TR4.1 (rule)**: `require('./src/helpers/fingerprint').randomFingerprint()` returns an object with `deviceModel`, `systemVersion`, `appVersion`, `langCode`, `systemLangCode`.
- **TR4.2 (rule)**: `require('./src/helpers/telegram').sleep(1).then(()=>true)` resolves to true (does not throw).

### Files touched
- `src/helpers/fingerprint.js` (new)
- `src/helpers/telegram.js` (new)

---

## Task 5: Extend cache.js with deliveryCache LRU

- **Status**: pending
- **Priority**: medium
- **AC Coverage**: FR8

### What to do
1. Read current `src/cache.js` (adminCache only).
2. Add `const { LRUCache } = require('lru-cache');` at top.
3. Create `deliveryCache` LRU: `max: 2000, ttl: 4h, updateAgeOnGet: true`.
4. Export alongside `adminCache`: `module.exports = { adminCache, deliveryCache };`.
5. Update all existing `require('../cache')` consumers that only need adminCache to destructure or keep importing `adminCache` explicitly — check files:
   - `src/middleware/auth.js`
   - `src/handlers/start.js`
   - `src/handlers/admin.js`
   - `src/handlers/user.js`
   - `src/handlers/payment.js`
   - `src/handlers/channel.js`
   - `src/services/broadcastService.js`
   - `src/services/syncService.js`
   - `src/services/advertisedRelay.js`
   - `src/server.js`
   - `src/seed.js`
   - `src/utils/mediaSendObserver.js`

### Test Requirements

- **TR5.1 (rule)**: No syntax / require errors on boot. Boot process `node src/server.js` starts (Ctrl+C after 5s) with no `Cannot find module` or destructuring errors.
- **TR5.2 (rule)**: `deliveryCache.set('k','v')` followed by `deliveryCache.get('k')` returns `'v'`.

### Files touched
- `src/cache.js` (modify)
- `src/middleware/auth.js` (update require if needed)
- `src/handlers/start.js` (update require if needed)
- `src/handlers/admin.js` (update require if needed)
- `src/handlers/user.js` (update require if needed)
- `src/handlers/payment.js` (update require if needed)
- `src/handlers/channel.js` (update require if needed)
- `src/services/broadcastService.js` (update require if needed)
- `src/services/syncService.js` (update require if needed)
- `src/services/advertisedRelay.js` (update require if needed)
- `src/server.js` (update require if needed)
- `src/seed.js` (update require if needed)
- `src/utils/mediaSendObserver.js` (update require if needed)

---

## Task 6: Rewrite mediaService.js — 3-tier delivery pipeline

- **Status**: pending
- **Priority**: high
- **AC Coverage**: FR2, FR3, FR10, AC2, AC3, AC4, AC7, AC8

### What to do
1. Add at top of `mediaService.js`:
   - `const BOT_KEY = String(process.env.CURRENT_BOT_KEY || (process.env.BOT_TOKEN || '').split(':')[0] || 'default').trim();`
   - `const { LRUCache } = require('lru-cache');`
   - `const { deliveryCache } = require('../cache');`
   - `const UserbotAccount = require('../models/UserbotAccount');`
   - `const Settings = require('../models/Settings');`
   - Add `pendingPromiseCache = new LRUCache({max:1000, ttl:60*1000})`.
2. Add helpers `hasActiveUserbot()` — findOne `UserbotAccount` with session `$ne:null`.
3. Add `hotSendMedia(telegram, chatId, row, replyToMessageId)`:
   - Reads `row.bot_file_ids[BOT_KEY]`. If missing, `{ok:false, reason:'no_file_id'}`.
   - Dispatch on `row.metadata.kind` → `sendPhoto / sendDocument / sendVideo` (document: use `disable_content_type_detection: false`; video: `supports_streaming: true`).
   - Wrap the actual send in existing `enqueue` + `withRetry`.
4. Add `copyForwardErrorLooksLikeDeadMessage(err)` — match Pbox heuristic.
5. Add `lazyDeleteIfDead(row, err)` — deletes Media row if error matches dead-message heuristic, invalidates from `deliveryCache`.
6. Add `coldForwardAndSeed(telegram, chatId, row, replyToMessageId)`:
   - Gate on `row.source.channel_id === Settings.get('fileManagerChannel')` (both as strings; Settings.get is async so read once before loop).
   - Try `copyMessage(chatId, row.source.channel_id, row.source.message_id)` via `enqueue` + `withRetry`.
   - On fail, try `forwardMessage(...)` as fallback.
   - On success: extract `file_id` from returned `msg.video / msg.photo[last] / msg.document` → upsert `bot_file_ids[BOT_KEY]` + `last_seen_at = new Date()` → refresh `deliveryCache`.
   - On fail: call `lazyDeleteIfDead(row, lastError)`.
7. Add `userbotDirectFallback(row, chatId)`:
   - Check `hasActiveUserbot()`; if no session → `{ok:false, reason:'no_userbot_session'}`.
   - Otherwise → `{ok:false, reason:'userbot_engine_stub'}` (engine NOT implemented per Non-Goal 1).
8. Rewrite `deliverMedia(telegram, chatId, count, {excludeIds})`:
   - Same outer loop: aggregate sample → candidate iteration → exclude tracking.
   - **Per item**: replace old send path with pipeline:
     - `hot = await hotSendMedia(...)` → ok = done.
     - `cold = await coldForwardAndSeed(...)` → ok = done.
     - `ub = await userbotDirectFallback(...)` → ok = done.
     - All failed: `usedIds.add(id)`; skip silently (FR10).
   - Remove `resolveAndReuploadMedia` entirely (deleted function, no calls remain).
9. Keep `summarizeErr`, `unwrapQueueResult`, `isSkippableTelegramError`, `isBadFileIdentifierError` (still useful).

### Test Requirements

- **TR6.1 (rule)**: `require('./src/services/mediaService').deliverMedia` is a function with arity `(telegram, chatId, count, opts)`.
- **TR6.2 (rule)**: No `resolveAndReuploadMedia` identifier exists in file (grep empty).
- **TR6.3 (rule)**: String `'BOT_KEY'` and `'userbotDirectFallback'` present in source (confirm pipeline wired).
- **TR6.4 (rubric)**: Manual integration — post a test video to file channel via human account, redeem with bot, and check that delivery succeeds with correct bytes (same AC2 steps). Score 2 on first attempt = pass. Pass threshold >= 2.

### Files touched
- `src/services/mediaService.js` (rewrite)

---

## Task 7: Rewrite channel_post ingest (handlers/channel.js) for new Media shape

- **Status**: pending
- **Priority**: high
- **AC Coverage**: FR4, AC7

### What to do
1. Read current `handlers/channel.js`.
2. Replace the `Media.create({...})` block:
   - Add `document` detection branch: `post.document?.file_id` → kind `'document'`, use `post.document.mime_type`, `post.document.file_name`, `post.document.file_size`.
   - Photo: use `post.photo[post.photo.length-1]` as before; capture `file_size`.
   - Video: use `post.video`; capture `mime_type`, `file_name` (if present), `file_size`.
   - Capture `file_unique_id` from `post.video.file_unique_id` / `post.document.file_unique_id` / `post.photo[last].file_unique_id` if present.
   - Set `metadata.uploaded_at = new Date(post.date * 1000)` if `post.date` exists, else `Date.now()`.
   - Populate `bot_file_ids[BOT_KEY] = file_id` (need `BOT_KEY` const at top of handler).
   - Populate `source = { channel_id: channelId, message_id: post.message_id }`.
3. Keep admin notification + advertised relay (unchanged).

### Test Requirements

- **TR7.1 (rule)**: Ingest handler covers 3 kind branches (video/photo/document) and sets `source.channel_id` + `source.message_id` on all.
- **TR7.2 (rule)**: Document kind uses `sendDocument` path (i.e. `metadata.kind === 'document'` set). Confirmed by reading code branches.

### Files touched
- `src/handlers/channel.js`

---

## Task 8: Build userbot login module + scene

- **Status**: pending
- **Priority**: high
- **AC Coverage**: FR7, FR6, AC5, AC6

### What to do
1. Create `src/bot/userbotLogin.js`:
   - Port Client Pbox `uploaderLogin.js` structure but adapt to this codebase's **scene-based** flow (Telegraf `WizardScene` / `BaseScene` style, similar to how `addAdmin.js` and `giftMedia.js` work — inspect one existing scene first for pattern).
   - State machine steps:
     - Step 0: Prompt "Send phone number (+CCXXXXXXXX format)." → store in `ctx.wizard.state.phoneNumber` → proceed.
     - Step 1: On receiving text → run `sendCodeWithRetry` using `TelegramClient` + `randomFingerprint`. If success, store `phoneCodeHash`, prompt "Enter verification code:". If `PHONE_MIGRATE_*`, reconnect with new DC address.
     - Step 2: On code → `invoke(Api.auth.SignIn)`. If `SESSION_PASSWORD_NEEDED` error, prompt "2FA enabled. Send password:" → step 3. Otherwise: session save + UserbotAccount create.
     - Step 3 (conditional): On 2FA password → `GetPassword` + `computeCheck` + `CheckPassword` → save account.
   - On success: `ctx.reply('Userbot session saved ✓')` + leave scene to admin main.
   - Cancel button: `ctx.scene.leave()` + admin main.
   - Disconnect + cleanup `authClients` map on error/cancel.
2. Create scene file `src/scenes/userbotLogin.js` that wires the wizard above.
3. Register in `src/scenes/index.js`: add to exported array.
4. Add `const UserbotAccount = require('../models/UserbotAccount')` import where needed.

### Test Requirements

- **TR8.1 (rule)**: `src/scenes/userbotLogin.js` exists and exports a valid Telegraf Scene (WizardScene/BaseScene).
- **TR8.2 (rule)**: Scene appears in `require('./src/scenes')` array (index.js updated).
- **TR8.3 (rubric)**: Code review — login scene handles cancel, invalid number, timeout, and 2FA branch at minimum. Score 2 = all 4 paths covered, 1 = 2-3 covered, 0 = <2. Pass threshold >= 2.

### Files touched
- `src/bot/userbotLogin.js` (new)
- `src/scenes/userbotLogin.js` (new)
- `src/scenes/index.js` (register new scene)

---

## Task 9: Add Userbot Login button to admin keyboard + wire handlers

- **Status**: pending
- **Priority**: high
- **AC Coverage**: FR7, AC5

### What to do
1. Read `src/keyboards/admin.js`.
2. In `mainAdminKeyboard()`, add new row: `['🤖 Userbot Login']` (before/after existing rows, any position fine).
3. In `src/handlers/admin.js`:
   - Add `bot.hears('🤖 Userbot Login', ...)` guard-gated with `adminGuard(ctx)`, then `ctx.scene.enter('USERBOT_LOGIN')` (match scene id).
   - Also add a secondary listener for listing accounts. Under the same `🤖 Userbot Login` flow (or as a separate `👁 List Userbot Accounts` button on the admin keyboard):
     - Query `UserbotAccount.find({ session: { $ne: null, $exists: true } }).sort({createdAt:-1}).select('number username').lean()`.
     - Reply with "No accounts yet" or `"- +CCXXXX @username"` list.
   - (Simpler approach: put both as inline buttons inside a `userbotIntroInlineKeyboard` — OR just keep as hears-buttons on the reply keyboard. Both are fine as long as the main entry is reply-keyboard button as required by FR7.)
4. Update `.env.example` to add `API_ID=` and `API_HASH=` lines (commented/default empty).

### Test Requirements

- **TR9.1 (rule)**: `mainAdminKeyboard()` return value contains a row with button label `'🤖 Userbot Login'`.
- **TR9.2 (rule)**: In `handlers/admin.js`, string `'USERBOT_LOGIN'` or scene-id equivalent appears in a `hears` branch guarded by admin check.
- **TR9.3 (rule)**: `.env.example` contains `API_ID` and `API_HASH` lines.

### Files touched
- `src/keyboards/admin.js`
- `src/handlers/admin.js`
- `.env.example`

---

## Task 10: Adapt syncService.js + mediaSendObserver.js call sites

- **Status**: pending
- **Priority**: medium
- **AC Coverage**: FR5, FR10

### What to do
1. `src/services/syncService.js`:
   - Remove or adapt the `getFile(m.fileId)` batch check — old `fileId` field no longer exists.
   - New strategy: no-op by default (cold path reseeds lazily). Optionally: iterate rows without `bot_file_ids[BOT_KEY]`, log count, return.
   - Keep `checkChannelAccess(bot)` intact (still reads Settings.fileManagerChannel, still alerts admins on lost admin).
   - Keep fail-rate guard conceptually (but applied to whatever new light-weight check we run).
2. `src/utils/mediaSendObserver.js`:
   - Inspect for any references to old `item.fileId` or `item.fileType`. Replace with new shape fields where needed (`item.bot_file_ids` not needed there, but `item.metadata.kind` replaces `fileType` comparisons if any).
3. `src/scenes/giftMedia.js`:
   - Check if it references old media fields; update to new model shape (gifting iterates Media records).
4. `src/handlers/admin.js` media list / delete flows:
   - Columns displayed: currently uses `fileType` emoji + `addedAt`. Replace with `metadata.kind` emoji + `metadata.uploaded_at` or `last_seen_at`.
   - Delete flow: already uses `_id`, fine; but `User.updateMany({receivedMedia})` stays unchanged.

### Test Requirements

- **TR10.1 (rule)**: No string `'fileId'` (old top-level field) exists in `syncService.js` or `mediaSendObserver.js` after edits — grep empty.
- **TR10.2 (rule)**: Admin media list `bot.action(/^media_list/)` uses `m.metadata.kind` or `row.metadata.kind` for emoji.
- **TR10.3 (rule)**: Boot process requires no errors related to undefined paths on Media model.

### Files touched
- `src/services/syncService.js`
- `src/utils/mediaSendObserver.js`
- `src/scenes/giftMedia.js`
- `src/handlers/admin.js` (media list formatting, lines ~160)

---

## Task 11: Wire server.js boot — no regressions

- **Status**: pending
- **Priority**: medium
- **AC Coverage**: All ACs via boot smoke test

### What to do
1. Verify `server.js` imports:
   - No references to removed `resolveAndReuploadMedia`.
   - `deliverMedia` still imported from `mediaService.js`.
   - Ensure Settings and DB models load UserbotAccount too (mongoose auto-loads models on first require — just make sure `require('./models/UserbotAccount')` is triggered somewhere explicit, e.g. at top of `server.js` or in `seed.js`).
2. Verify `mediaSendObserver.js` integration in `server.js` `deliverWithVerification` still works with new return shape.
3. Run a smoke boot:
   - `cd "C:\Users\Itive Peace Ufuoma\Desktop\TG BOTS\Client Rex\-starstomediabot--master"`
   - `node src/server.js`
   - Wait 10 seconds, Ctrl+C.
   - Verify console output contains: "Admin cache loaded", "Bot connected: @...", "Bot commands registered.", "[bot] long-poll launched" lines. No unhandled stack traces.

### Test Requirements

- **TR11.1 (rule)**: No unhandled exceptions during 10-second smoke boot (check exit code of child process after Ctrl+C — any stack trace printed = fail).
- **TR11.2 (rule)**: All 4 expected console lines above appear in output.

### Files touched
- `src/server.js` (ensure UserbotAccount model is required)
- `src/seed.js` (optional — require UserbotAccount)

---

## Task 12: Run lint/type diagnostics

- **Status**: pending
- **Priority**: high
- **AC Coverage**: All (quality gate)

### What to do
1. Run `GetDiagnostics` tool on the workspace (no URI = all files).
2. If any diagnostic errors remain, fix them in the relevant file(s).
3. Alternatively run `npm` scripts if present; current repo has no `lint` script — rely on VSCode diagnostics.

### Test Requirements

- **TR12.1 (rule)**: Zero error-level diagnostics reported by `GetDiagnostics` on all touched files. Warnings are ok.
