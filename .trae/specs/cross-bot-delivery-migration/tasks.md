# Cross-Bot Delivery Pipeline Migration — Tasks

**Scope**: BOT-A = `Client Rex\@premiumvc_bot\`, BOT-B = `Client Rex\@rexmediatgbot\`, BOT-C = `Client Rex\thetamediabot-master\`, BOT-D = `Client Rex\vidmatrixbot-master\` (4 total). Blueprint = `Client Rex\-starstomediabot--master\`. Excludes: `tradingchart`, `PaymentBot*`, `paymentbots`, and `Client P\bot\` (different architecture).

All tasks repeat identically across all 4 bots unless noted. Task numbering is global but grouped by bot in the checklist at bottom.

---

## Task 1 — Blueprint reference copy + environment audit (per bot × 4)

Copy the latest blueprint source files into a transient staging area and verify each bot already has the expected surrounding scaffold (models, services, handlers, scenes index, keyboards, server). Verify package.json has telegraf/mongoose before adding deps.

**Status**: pending

### Test Requirements (TR)
- **TR1.1 rule**: For each bot, paths `src/models/Admin.js`, `src/models/Settings.js`, `src/models/User.js`, `src/models/Order.js`, `src/db.js`, `src/bot.js`, `src/server.js`, `src/seed.js` all exist.
- **TR1.2 rule**: Each bot `package.json` `dependencies` already contains `telegraf` and `mongoose`.

---

## Task 2 — Add deps (lru-cache + telegram) to each bot (×4)

Edit `package.json` dependencies to append `"lru-cache": "^10.2.0"` and `"telegram": "^2.20.15"`. Run `npm install` in each bot root. Verify via `npm ls` both resolve.

**Status**: pending

### Test Requirements
- **TR2.1 rule**: `package.json dependencies` includes both strings.
- **TR2.2 rule**: `npm ls lru-cache --depth 0 2>&1` shows version ≥10.2.
- **TR2.3 rule**: `npm ls telegram --depth 0 2>&1` shows version ≥2.20.

---

## Task 3 — Rewrite Media model to new schema (×4)

Replace each bot's `src/models/Media.js` exactly with blueprint's `src/models/Media.js` (source/metadata/mtproto/bot_file_ids/file_unique_id/last_seen_at + 4 indexes). No creative edits.

**Status**: pending

### Test Requirements
- **TR3.1 rule**: Require passes. Schema introspection shows nested paths `source.channel_id`, `metadata.kind`, `mtproto.id` all present.
- **TR3.2 rule**: Schema has exactly 4 declared indexes (call `schema.indexes()` in node). Compound `(source.channel_id, source.message_id)` unique, sparse unique on `file_unique_id`, sparse unique on `mtproto.id`, `last_seen_at:-1`.

---

## Task 4 — Add UserbotAccount model (×4)

Copy blueprint `src/models/UserbotAccount.js` verbatim.

**Status**: pending

### Test Requirements
- **TR4.1 rule**: `require(…/models/UserbotAccount)` → mongoose.model('UserbotAccount') exports.
- **TR4.2 rule**: Schema has fields `number`, `username`, `userId`, `session` and `{timestamps:true}`.

---

## Task 5 — Port helpers fingerprint.js + telegram.js (×4)

Copy blueprint `src/helpers/fingerprint.js` and `src/helpers/telegram.js` verbatim into each bot's `src/helpers/` directory (mkdir helpers if absent).

**Status**: pending

### Test Requirements
- **TR5.1 rule**: `require(…/helpers/fingerprint).randomFingerprint()` returns object with ≥5 keys including `deviceModel`, `systemVersion`, `appVersion`.
- **TR5.2 rule**: `typeof require(…/helpers/telegram).sendCodeWithRetry === 'function'`; `getDCAddress(1..5)` returns non-empty string for each; `sleep(1)` resolves.

---

## Task 6 — cache.js dual export + update all consumers (×4)

Replace each bot's `src/cache.js` with blueprint's export shape `{ adminCache, deliveryCache }`. Add `deliveryCache = LRUCache(max 2000, ttl 4h, updateAgeOnGet true)`. Update every file in the bot that previously used `const adminCache = require('../cache')` to `const { adminCache } = require('../cache')`.

Consumer inventory to update per bot (mirror blueprint; cross-check with grep):
- `middleware/auth.js`
- `handlers/start.js`
- `handlers/admin.js`
- `handlers/user.js`
- `handlers/channel.js`
- `services/syncService.js`
- `scenes/addAdmin.js`
- `scenes/removeAdmin.js`
- `scenes/giftMedia.js`
- `server.js`

**Status**: pending

### Test Requirements
- **TR6.1 rule**: `node -e "const {adminCache, deliveryCache} = require('./src/cache'); console.log(typeof adminCache.set, typeof deliveryCache.get);"` prints `function function`.
- **TR6.2 rule**: All 10 consumers above (or equivalent set on each bot) require-load with destructured import and no TypeError thrown on `.set` access.
- **TR6.3 rule**: Grep for `require('../cache')` OR `require('./cache')` in src/ — any hit is destructured form. Zero legacy direct-non-destructured hits except inside the cache module itself.

---

## Task 7 — Rewrite mediaService.js with 3-tier pipeline (×4)

Replace each bot's `src/services/mediaService.js` with blueprint's file exactly.

**Status**: pending

### Test Requirements
- **TR7.1 rule**: Exports `{deliverMedia, hotSendMedia, coldForwardAndSeed, userbotDirectFallback, BOT_KEY}`.
- **TR7.2 rule**: `deliverMedia.length === 4`.
- **TR7.3 rule**: String `resolveAndReuploadMedia` occurs zero times in file (grep empty).
- **TR7.4 rule**: File contains literal strings `'BOT_KEY'` and `'userbotDirectFallback'`.
- **TR7.5 rule**: Syntax check passes `node --check`.

---

## Task 8 — Rewrite channel.js ingest for new shape + document kind (×4)

Replace each bot's `src/handlers/channel.js` with blueprint's version. Minimal adaptation allowed only if the sibling bot does NOT export `mirrorChannelPost` from `services/advertisedRelay` → drop that line and the require rather than failing.

**Status**: pending

### Test Requirements
- **TR8.1 rule**: File has 3 if-branches for `post.photo`, `post.video`, `post.document`.
- **TR8.2 rule**: `Media.create` call populates `source`, `metadata`, `bot_file_ids`, `last_seen_at`, optionally `file_unique_id`.
- **TR8.3 rule**: File gates on `Settings.get('fileManagerChannel')` equality check.
- **TR8.4 rule**: Syntax check passes.

---

## Task 9 — Build userbot login module + scene (×4)

Add to each bot:
- `src/bot/userbotLogin.js` copied verbatim from blueprint.
- `src/scenes/userbotLogin.js` copied verbatim.
- Append `userbotLoginScene` to array in `src/scenes/index.js`.

**Status**: pending

### Test Requirements
- **TR9.1 rule**: `require(…/bot/userbotLogin)` exports exactly keys `beginLogin, handleCancel, handleTextMessage, listLoggedInAccounts, getSession, clearSession`.
- **TR9.2 rule**: `require(…/scenes/userbotLogin).id === 'USERBOT_LOGIN'`.
- **TR9.3 rule**: `require(…/scenes/index).length === previous_length + 1` (capture before/after counts via node one-liner).
- **TR9.4 rule**: Both files pass `node --check`.

---

## Task 10 — Add Userbot Login keyboard button + wire handlers + .env.example (×4)

Per bot:
- Add row `['🤖 Userbot Login']` to `src/keyboards/admin.js` mainAdminKeyboard() Markup.keyboard rows.
- In `src/handlers/admin.js`:
  - Add import `const { listLoggedInAccounts } = require('../bot/userbotLogin');`.
  - Add `bot.hears('🤖 Userbot Login', … adminGuard → reply inline menu with Add Session + List Sessions buttons + optional API_ID/API_HASH warning banner).
  - Add `bot.action('userbot_add', … enter USERBOT_LOGIN scene)` and `bot.action('userbot_list', … listLoggedInAccounts)`.
- In the bot's `.env.example` file (create if missing):
  - Strip any real values (tokens, mongo URI, api id/hash).
  - Add empty lines `API_ID=`, `API_HASH=`.
  - Add comment block + `# CURRENT_BOT_KEY=` commented empty line.

**Status**: pending

### Test Requirements
- **TR10.1 rule**: `mainAdminKeyboard().reply_markup.keyboard` flatten includes row `['🤖 Userbot Login']` (node test).
- **TR10.2 rule**: handlers/admin.js contains both `bot.hears('🤖 Userbot Login'` and two action handlers `userbot_add` and `userbot_list`.
- **TR10.3 rule**: `.env.example` has no secrets (empty keys). Contains `API_ID=`, `API_HASH=` and a commented `CURRENT_BOT_KEY=` line.
- **TR10.4 rule**: Syntax check passes admin.js + keyboards/admin.js.

---

## Task 11 — Adapt syncService.js, admin media list, giftMedia/observers (×4)

Per bot:
1. Replace `syncMediaPool(bot)` in `src/services/syncService.js` with blueprint's lightweight stats version (no getFile, no deleteMany). Keep `checkChannelAccess` verbatim.
2. In handlers/admin.js media_list `bot.action(/^media_list:(\d+)$/)`:
   - Sort swap `{addedAt:-1}` → `{last_seen_at:-1}`.
   - Emoji swap → metadata.kind 3-way.
   - Date swap → fallback chain of `uploaded_at || last_seen_at || createdAt || new Date()`.
3. Grep giftMedia scenes, mediaSendObserver, handlers/user.js, any observer util files for legacy field regex; replace if hits; skip if none.

**Status**: pending

### Test Requirements
- **TR11.1 rule**: syncService.js contains zero calls to `getFile(`, zero `deleteMany(`, zero `pull: { receivedMedia`.
- **TR11.2 rule**: handlers/admin.js media_list callback uses `sort({last_seen_at:-1})`, `metadata.kind`, and fallback date chain (grep strings present).
- **TR11.3 rule**: Cross-file legacy regex grep on the bot root yields 0 matches before moving to Task 12 final sweep.

---

## Task 12 — Wire server.js boot require + per-bot smoke (×4)

Per bot, edit `src/server.js` top-of-file (before cache/scenes loads) to add explicit:
```js
require('./models/Media');
require('./models/UserbotAccount');
```
Then run a 16-key-module require-load smoke script per bot (models/cache/services/helpers/scenes/keyboards) to ensure no syntax/shape errors.

**Status**: pending

### Test Requirements
- **TR12.1 rule**: require-load sweep of ≥16 key modules passes with zero throws.
- **TR12.2 rule**: UserbotAccount model registers (mongoose connection stub test doesn't need real DB — just require without throwing).

---

## Task 13 — Cross-bot final grep + GetDiagnostics

Run across all 4 bots:
- Legacy regex full grep.
- `node --check` on every touched file (≥80 files total).
- IDE `GetDiagnostics` against all touched files; fix any error-level issues.

**Status**: pending

### Test Requirements
- **TR13.1 rule**: Full recursive legacy regex → 0 matches in any \*.js/ts/md file under BOT-A/B/C/D roots.
- **TR13.2 rule**: `node --check` pass rate 100%.
- **TR13.3 rule**: GetDiagnostics error count on all changed paths → 0.

---

## Task 14 — Scope containment final check (single task)

Compare git status (or a recursive file-hash diff against expected set). Ensure zero files changed outside:
- BOT-A, BOT-B, BOT-C, BOT-D roots.
- The allowed spec artifacts directory `Client Rex/-starstomediabot--master/.trae/specs/cross-bot-delivery-migration/`.
Specifically confirm: no edits inside blueprint starstomediabot/src (it's the source of truth), no edits in any excluded folder names.

**Status**: pending

### Test Requirements
- **TR14.1 rule**: Blueprint src files (hash check) identical to last committed state hash. No writes inside `src/` of starstomediabot during Tasks 1-13.
- **TR14.2 rule**: grep `resolveAndReuploadMedia` across all 4 sibling bot roots → 0.
- **TR14.3 rule**: grep `resolveAndReuploadMedia` inside blueprint src → 0 (idempotent sanity, blueprint already passed this in previous session).

---

# Checklist by bot (same tasks repeated across each)

### BOT-A: `TG BOTS\Client Rex\@premiumvc_bot\`

| Task | Status | Notes | Evidence |
|------|--------|-------|----------|
| 1 (audit) | pending | | |
| 2 (deps)  | pending | | |
| 3 (Media) | pending | | |
| 4 (UserbotAccount) | pending | | |
| 5 (helpers) | pending | | |
| 6 (cache + consumers) | pending | | |
| 7 (mediaService) | pending | | |
| 8 (channel.js) | pending | | |
| 9 (userbot scene) | pending | | |
| 10 (kb + admin.js + .env.example) | pending | | |
| 11 (sync + media list + observers) | pending | | |
| 12 (server + load smoke) | pending | | |
| 13 (final grep + diagnostics) | pending | shared | |
| 14 (scope) | pending | shared | |

### BOT-B: `TG BOTS\Client Rex\@rexmediatgbot\`

| Task | Status | Notes | Evidence |
|------|--------|-------|----------|
| 1 | pending | | |
| 2 | pending | | |
| 3 | pending | | |
| 4 | pending | | |
| 5 | pending | | |
| 6 | pending | | |
| 7 | pending | | |
| 8 | pending | | |
| 9 | pending | | |
| 10 | pending | | |
| 11 | pending | | |
| 12 | pending | | |
| 13 | pending | shared | |
| 14 | pending | shared | |

### BOT-C: `TG BOTS\Client Rex\thetamediabot-master\`

| Task | Status | Notes | Evidence |
|------|--------|-------|----------|
| 1 | pending | | |
| 2 | pending | | |
| 3 | pending | | |
| 4 | pending | | |
| 5 | pending | | |
| 6 | pending | | |
| 7 | pending | | |
| 8 | pending | | |
| 9 | pending | | |
| 10 | pending | | |
| 11 | pending | | |
| 12 | pending | | |
| 13 | pending | shared | |
| 14 | pending | shared | |

### BOT-D: `TG BOTS\Client Rex\vidmatrixbot-master\`

| Task | Status | Notes | Evidence |
|------|--------|-------|----------|
| 1 | pending | | |
| 2 | pending | | |
| 3 | pending | | |
| 4 | pending | | |
| 5 | pending | | |
| 6 | pending | | |
| 7 | pending | | |
| 8 | pending | | |
| 9 | pending | | |
| 10 | pending | | |
| 11 | pending | | |
| 12 | pending | | |
| 13 | pending | shared | |
| 14 | pending | shared | |
