# Cross-Bot Delivery Pipeline Migration (Scope: Option 2)

## Problem

The user has N bots under `c:\Users\Itive Peace Ufuoma\Desktop\TG BOTS\` that share a specific legacy architecture:
- Mongo model `Media` with single fields `fileId: String`, `fileType`, `channelId`, `channelMessageId`, `addedAt`.
- Delivery uses bot-scoped `file_id` captured from whoever (admin user, another bot) posted to the file channel.
- When the admin rotates to a NEW bot token that didn't originally upload the pool, every legacy row's `file_id` fails with `400 Bad Request: wrong file identifier` → corrupted / zero-byte / missing deliveries to user packages even when admin panel shows sufficient quota uploaded.
- Legacy fallback `resolveAndReuploadMedia` downloads via HTTP from Telegram and re-uploads, silently truncating large files (>20MB) and corrupting payloads.
- `syncMediaPool` calls `bot.telegram.getFile(m.fileId)` on every legacy row every 30 min → 400 rate-limit storm → when `failRate <= 0.2` (which often applies when just a portion of pool went bad after partial backfill) it runs `deleteMany` against rows → sudden unexpected data loss of the whole pool.

The exact same fix has just been completed for the "origin/blueprint" bot at `Client Rex\-starstomediabot--master\`. The user wants the EXACT SAME fix copied to all bots under TG BOTS\ that exhibit the legacy pattern, excluding:
- Any directory containing `tradingchart` in the path name (case-insensitive).
- Any directory containing `PaymentBot` or `paymentbots` in the path name (case-insensitive).

All bots that match "same architecture" are defined objectively by these search criteria applied BEFORE this spec was written (grep evidence from the exploratory turn is preserved below):
1. Bot directory contains a `Media.js` model (anywhere under the bot root) that includes `fileId`, `channelMessageId` or `channelId`, and `addedAt` legacy fields.
2. Bot directory contains `Settings.js` model (anywhere under the bot root) for the single `fileManagerChannel` key or equivalent.
3. Bot directory contains `mediaService.js` that calls `resolveAndReuploadMedia` (the legacy download-reupload fallback).
4. Bot directory contains `syncService.js` that performs `getFile(m.fileId)` batch check with failRate-driven deleteMany logic.
5. Bot directory contains `keyboards/admin.js` reply-button `Markup.keyboard` main menu.

Folders meeting all 5 criteria (confirmed via grep 2026-09-21, excluding user's blacklist):
- BOT-A: `TG BOTS\Client Rex\@premiumvc_bot\`
- BOT-B: `TG BOTS\Client Rex\@rexmediatgbot\`
- BOT-C: `TG BOTS\Client Rex\thetamediabot-master\`
- BOT-D: `TG BOTS\Client Rex\vidmatrixbot-master\`

Partial match excluded by rule 2/3/4/5 (no Settings/mediaService/syncService/keyboards/admin — different architecture, out of scope):
- `TG BOTS\Client P\bot\` — no Settings, no keyboard menu, no syncService media pool watcher. Will not be touched.

All other 52+ folders under TG BOTS\ returned 0 matches for criteria 1-5 (confirmed in the full directory sweep of the turn). They are out of scope.

## Users / Stakeholders

- **Admin user** (the human logged into the IDE): operates 4+ sibling bots with identical media-pool / package-redemption / file-channel / keyboard-admin pattern. Cares the fix is applied identically across all bots so behaviour stays uniform; wants no corruption on bot-token rotation; wants no unexpected deleteMany data loss from syncService; wants userbot scene available via reply-keyboard on each bot.
- **End users of each bot**: pay for / refer for package-redemption flows. Cares that N-of-N media in the package is delivered byte-identical to what was posted to the file channel (no shorted, no truncated, no "corrupt video that won't play past 2s").

## Goals

1. **Identical, drop-in fix.** For each of BOT-A through BOT-D, swap the single-`fileId` / `resolveAndReuploadMedia` / `getFile-batch-delete` pipeline for exactly the same 3-tier pipeline + new Media schema + `bot_file_ids[BOT_KEY]` sparse Mixed slots that are now running in `Client Rex\-starstomediabot--master\`.
2. **Uniform admin surface.** Userbot scene entry is a reply-keyboard button `['🤖 Userbot Login']` on each bot's `mainAdminKeyboard()`. Scene steps (phone→code→2FA password) and list-sessions inline sub-menu are identical.
3. **Same channel works.** Each bot keeps its own existing `Settings.fileManagerChannel` id (single channel design). No new channel creation required; forwards into existing channels are valid ingestion.
4. **Zero legacy fallbacks.** After migration, no call to `resolveAndReuploadMedia` exists anywhere in BOT-A/B/C/D; `m.fileId`, `m.fileType`, `m.channelId`, `m.channelMessageId`, `m.addedAt` field reads are zero.
5. **Surgical edits.** Each bot's existing payment, referral, packages, broadcast, gift, advertised-relay, admin-mgmt, user-view-toggle, scenes are preserved verbatim except where they MUST touch legacy media/delivery field names.

## Non-Goals

- **Not adding any multi-`UploadChannel` approval system from Client Pbox.** Keep each bot's single `Settings.fileManagerChannel` design, exactly as done on the blueprint bot.
- **Not implementing real GramJS Tier-3 send.** Only the `hasActiveUserbot` check + userbot login scene + stub `{ok:false, reason:...}` return is required, exactly as on blueprint.
- **Not modifying `Client Rex\-starstomediabot--master\` itself.** It is the read-only source of truth for this migration.
- **Not touching `Client Rex\tradingchartbot\`, `Client Rex\PaymentBot\`, `Client Rex\PaymentBot-Clone\`, or any folder containing `PaymentBot`/`paymentbots`/`tradingchart` anywhere in the path.**
- **Not migrating `Client P\bot\` or any other bot that lacks all 5 architecture-match criteria.**
- **Not running against live Telegram long-poll or Mongo.** Static syntax + require checks only, same verification steps used on blueprint.
- **Not touching git / committing / pushing.** User profile wants auto commit at the very end as a separate single operation after everything passes.

## Functional Requirements

### FR1 — Media schema swap (per bot)

Every bot BOT-A/B/C/D gets its `src/models/Media.js` replaced verbatim with blueprint's Media.js. Specific shape required:
```
source: { channel_id: String, message_id: Number }
metadata: { kind: String enum[photo|video|document], mime_type: String, file_name: String, file_size: Number, uploaded_at: Date }
bot_file_ids: Mixed default {}
file_unique_id: String sparse unique
mtproto: { type: String, id: Number sparse unique, access_hash: String, file_reference: Mixed, dc_id: Number }
last_seen_at: Date
```
4 indexes exactly matching blueprint: compound unique `(source.channel_id, source.message_id)`, sparse unique on `file_unique_id`, sparse unique on `mtproto.id`, descending `last_seen_at:-1`.

### FR2 — `UserbotAccount` model

Each bot gets `src/models/UserbotAccount.js` identical to blueprint. `{ number, username, userId, session, timestamps }`; sparse unique number; sparse index session.

### FR3 — Helpers

Each bot gets `src/helpers/fingerprint.js` (DEVICE_POOL, randomFingerprint 5-field) and `src/helpers/telegram.js` (getDCAddress, sleep, sendCodeWithRetry including PHONE_MIGRATE_X reconnect loop via initialServerAddress) copied verbatim.

### FR4 — cache.js dual export

Each bot's `src/cache.js` must export destructurable `{ adminCache, deliveryCache }`. Add `deliveryCache = new LRUCache({max:2000, ttl:4*60*60*1000, updateAgeOnGet:true})`. Update EVERY consumer of `const adminCache = require('../cache')` in the bot to `const { adminCache } = require('../cache')`. Exact same set of consumers as blueprint (auth.js, start.js, admin.js/handler, user.js/handler, channel.js, syncService.js, all scenes, server.js).

### FR5 — Dependencies

Each bot's `package.json` adds `"lru-cache": "^10.2.0"` and `"telegram": "^2.20.15"` to `dependencies` (pinned versions matching blueprint). `npm install` is run per bot after the edit OR shared node_modules is already present — but npm install step is required to guarantee package-lock reflects new deps.

### FR6 — mediaService.js full rewrite (3-tier)

Each bot's `src/services/mediaService.js` replaced with blueprint's mediaService.js. Must include:
- `BOT_KEY = String(process.env.CURRENT_BOT_KEY || (process.env.BOT_TOKEN||'').split(':')[0] || 'default').trim()` at top.
- `pendingPromiseCache = new LRUCache({max:500, ttl:60*1000})`.
- `withRetry`, `isSkippableTelegramError`, `isBadFileIdentifierError`, `unwrapQueueResult`, `summarizeErr` verbatim.
- `hasActiveUserbot()` → `UserbotAccount.countDocuments({session:{$ne:null,$exists:true}})`.
- `hotSendMedia()` → reads `row.bot_file_ids[BOT_KEY]`, dispatches `sendPhoto/sendVideo/sendDocument` via `tgQueue.enqueue` + `withRetry`, marks skippable errors as `{skippable:true}`.
- `copyForwardErrorLooksLikeDeadMessage(err)` 6-substring heuristic; `lazyDeleteIfDead` deletes Media + invalidates `deliveryCache`.
- `extractFileIdFromSentMessage(msg)` handles video/document/photo[last].
- `coldForwardAndSeed(telegram, chatId, row, replyToMessageId, fileManagerChannelId)`:
  - `copyMessage` first (no forward header),
  - `extractFileIdFromSentMessage`,
  - write back `bot_file_ids[BOT_KEY]` via `findOneAndUpdate` + refresh `deliveryCache`,
  - fall back `forwardMessage` if copyMessage fails (non-dead),
  - on dead-message err → `lazyDeleteIfDead`.
- `userbotDirectFallback(row, chatId)` → stub returns either `{ok:false, reason:'no_userbot_session'}` or `{ok:false, reason:'userbot_engine_stub'}`.
- `deliverMedia(telegram, chatId, count, {excludeIds})` outer sample+loop unchanged in structure; per item runs hot→cold→userbot sequentially, propagates skippable→shouldAbortChat=true. Silent failures add to usedIds continue.
- Function identifier `resolveAndReuploadMedia` must not exist anywhere in the file after rewrite.
- Export exactly `{ deliverMedia, hotSendMedia, coldForwardAndSeed, userbotDirectFallback, BOT_KEY }`.

### FR7 — channel.js ingest

Each bot's `src/handlers/channel.js` rewritten to:
- `BOT_KEY` const at top matching blueprint.
- Only triggers if `channelId === Settings.get('fileManagerChannel')`.
- 3-way detect: photo (last element), video, **document** (AC7 explicit).
- Captures `file_unique_id`, `mime_type`, `file_name`, `file_size` where present.
- `uploaded_at = post.date ? new Date(post.date*1000) : new Date()`.
- Writes `bot_file_ids[BOT_KEY] = fileId` on create.
- Calls `mirrorChannelPost(...)` advertised relay (if that bot has that service; drop the call if that bot doesn't export it — but if blueprint's channel.js calls it and that bot doesn't have it, remove that line rather than breaking require).
- Sends admin notifications via `adminCache`/`enqueue` exactly as blueprint does.

### FR8 — Userbot scene + keyboard button

Each bot:
- Gets `src/bot/userbotLogin.js` (step state machine: beginLogin, phone validation → connect → sendCodeWithRetry → awaiting_code → SESSION_PASSWORD_NEEDED → awaiting_password → computeCheck dynamic import → CheckPassword → saveNewAccount → listLoggedInAccounts; cancel paths + ❌ Cancel hears) copied verbatim.
- Gets `src/scenes/userbotLogin.js` (BaseScene id='USERBOT_LOGIN') copied verbatim.
- `src/scenes/index.js` exports array adds new scene at end.
- `src/keyboards/admin.js` mainAdminKeyboard reply-button rows include a new row `['🤖 Userbot Login']` exactly. Not inline.
- `src/handlers/admin.js`:
  - Import `{ listLoggedInAccounts } from '../bot/userbotLogin'`.
  - Add `bot.hears('🤖 Userbot Login', ... adminGuard → reply menu with 2 inline buttons: `➕ Add Userbot Session` (callback `userbot_add` → `ctx.scene.enter('USERBOT_LOGIN')`), `👁 List Sessions` (callback `userbot_list` → `listLoggedInAccounts(bot, ctx.chat.id)`).
  - If `process.env.API_ID && process.env.API_HASH` missing, prepend a warning banner.
- `src/server.js` top-of-file add explicit `require('./models/Media'); require('./models/UserbotAccount');` before cache/scenes loads.

### FR9 — syncService.js safe stats mode

Each bot's `src/services/syncService.js` stripped of the legacy `getFile(m.fileId)` batch and failRate-driven `deleteMany`. Keep `checkChannelAccess(bot)` exactly (file-channel admin-loss alerts). New `syncMediaPool`:
- `total = Media.countDocuments()`.
- `seeded = Media.countDocuments({['bot_file_ids.'+BOT_KEY]:{$exists:true, $ne:null}})`.
- Single console.log line: `[sync] ${total} media record(s); BOT_KEY=${BOT_KEY} — seeded=${seeded}, will-cold-reseed-on-first-redemption=${total-seeded}`.
- No getFile. No deleteMany. No User.receivedMedia $pull for stale sync rows.

### FR10 — Media list view adapters (admin action `media_list:N`)

Each bot's `src/handlers/admin.js` media_list callback adapter exactly as blueprint:
- Sort change: `.sort({ addedAt:-1 })` → `.sort({ last_seen_at:-1 })`.
- Emoji branch: `m.fileType === 'photo' ? 📷 : 🎬` → `m.metadata.kind === 'photo' ? 📷 : (m.metadata.kind === 'video' ? 🎬 : (m.metadata.kind === 'document' ? 📄 : 📦))`.
- Date: `formatDate(m.addedAt)` → `formatDate(m.metadata.uploaded_at || m.last_seen_at || m.createdAt || new Date())`.

### FR11 — Observer/giftMedia scenes unchanged unless they read removed fields

For every bot, run static grep against file paths under `src/utils/`, `src/scenes/giftMedia.js`, any observers, handlers/user.js for the regex `\.fileId[^s]|\.fileType|\.channelId|\.channelMessageId|\.addedAt|resolveAndReuploadMedia`. Any hit → swap to the new schema field names. If no hits, leave files untouched.

### FR12 — .env.example updates (per bot)

Each bot that ships a `.env.example` file:
- Add empty lines: `API_ID=`, `API_HASH=` (no trailing comments with values — keep empty).
- Add optional comment block + `CURRENT_BOT_KEY=` commented empty line.
- If `.env.example` already contains real values (bot token, mongo URI, etc.) they MUST be stripped to empty keys.

### FR13 — Zero stale references (global post-condition)

After all edits, full recursive grep under `TG BOTS\` (excluding tradingchart/paymentbots/node_modules/.kilo/starstomediabot) must find ZERO matches for the legacy regex:
```
\.fileId[^s]|\.fileType|\.channelId|\.channelMessageId|\.addedAt|resolveAndReuploadMedia
```
Specifically zero matches in any file of BOT-A/B/C/D.

## Non-Functional Requirements

### NFR1 — Blueprint fidelity. No creative deviation

Any changed/added file MUST match the blueprint `Client Rex\-starstomediabot--master\src\...` byte-for-byte unless the sibling bot's own surrounding file (e.g., handlers structure, missing advertisedRelay export, different admin scene exports set) requires a narrow, minimal surgical adaptation. Adaptations must be commented inline with reason.

### NFR2 — Syntax, destructuring, import shape identical per bot

Every consumer of cache.js uses `const { adminCache } = require('../cache')`. No bot has a legacy direct-export. Verify with a per-bot 10-consumer require test.

### NFR3 — Missing API_ID/API_HASH env must not break delivery

Hot/cold tiers of the pipeline must run without those two vars. Only the userbot login menu shows a warning; the flow can enter the scene but will fail at the client-connect step (the normal GramJS error).

### NFR4 — No runtime env secrets written to disk

No credential value (token, mongo pass, api id/hash) written into any .env.example. Always empty.

### NFR5 — Backwards-compat User.receivedMedia

Every bot still stores `Media._id` (ObjectId) into `User.receivedMedia` array; no shape change there. Delete-media action still $pulls by `_id`.

### NFR6 — Deterministic verification (same TR tests as blueprint)

Per bot, after edits:
- `node --check` on every touched file.
- `require(mediaService).deliverMedia` returns function of arity 4.
- Grep of `resolveAndReuploadMedia` returns 0 matches in the bot.
- `require(scenes).length === original_length + 1`.
- UserbotLogin exports keys matches blueprint.
- Full legacy-field regex grep returns 0.
- IDE diagnostics (GetDiagnostics) against the entire TG BOTS\ parent workspace filtered to changed paths → error count 0.

## Constraints & Dependencies

- **Blueprint exists, is the source of truth:** `TG BOTS\Client Rex\-starstomediabot--master\src\*` and `.env.example`.
- **Dependencies for each bot:** `lru-cache@10.2.0`; `telegram@2.20.15` (npm registry).
- **Excluded directories are mandatory:** Any path containing `tradingchart`, `PaymentBot`, or `paymentbots` (case-insensitive substring) → no edit, no verify, no mention.
- **Operating system:** Windows/PowerShell. All commands PowerShell 5 or pwsh compatible with legacy syntax (avoid `? :` ternary in PowerShell command text, use if/else or separate statements).
- **No Mongo connectivity during edits.** All checks are static.
- **Coupling with other migrations in-flight:** None. This migration is self-contained to the 4 sibling bots.

## Assumptions

- BOT-A/B/C/D each run their own independent Mongo database (different DB_NAME per bot). Schema changes won't collide.
- Each bot has `npm` available and a `package.json` with existing telegraf/mongoose deps already installed.
- Admin user will copy their own env values into .env per bot after migration (secrets are never added by this spec).
- User will run actual `node src/server.js` smoke boots with live token/mongo themselves if they want to; we only do static syntax+require tests.

## Open Questions

None. Scope was confirmed via user's Option-2 response ("every bot dir under TG BOTS/ with media pool/file channel pattern except tradingchart and paymentbots"), then narrowed to the 4 exact architecture matches via independent grep evidence (the only bots with Media.js + old fileId pattern + Settings + mediaService + syncService + keyboards/admin). No further ambiguity.

## Acceptance Criteria

### AC1 — rule

For each of BOT-A through BOT-D: `src/models/Media.js` contains the strings `"source"`, `"metadata"`, `"bot_file_ids"`, `"mtproto"`, `"last_seen_at"`, and the 4 index definitions (`compound`, `sparse unique file_unique_id`, `sparse unique mtproto.id`, `last_seen_at:-1`) all present. Pass evidence: node -e require + schema introspection showing the nested paths exist + index spec counts match blueprint.

### AC2 — rule

For each of BOT-A through BOT-D: `src/models/UserbotAccount.js` exports a mongoose.model('UserbotAccount', ...) with fields `number`, `username`, `userId`, `session`, and `{timestamps:true}`. Pass evidence: require+schema path check.

### AC3 — rule

For each of BOT-A through BOT-D: helpers `fingerprint.js` and `telegram.js` require-load without throwing; `randomFingerprint()` returns object with ≥5 fields; `sendCodeWithRetry` is typeof function; `getDCAddress(1..5)` returns a non-empty string.

### AC4 — rule

For each of BOT-A through BOT-D: `src/cache.js` exports an object `{adminCache, deliveryCache}` where both have `.set/.get` methods; every file in the bot that previously required `cache.js` directly now destructures `{ adminCache }`. Pass evidence: 10-file require smoke test identical to blueprint's Task 5 verification.

### AC5 — rule

For each of BOT-A through BOT-D: `package.json dependencies` contains both `lru-cache` ≥10.2 and `telegram` ≥2.20. `npm ls lru-cache telegram` resolves both.

### AC6 — rule

For each of BOT-A through BOT-D: `src/services/mediaService.js`:
  (a) exports `{deliverMedia, hotSendMedia, coldForwardAndSeed, userbotDirectFallback, BOT_KEY}`.
  (b) `deliverMedia.length === 4`.
  (c) file contains `'BOT_KEY'` literal string and `'userbotDirectFallback'` literal string.
  (d) string `'resolveAndReuploadMedia'` appears zero times.
  (e) Tier-2 cold path contains both `copyMessage(` and `forwardMessage(` call sites and a `findOneAndUpdate` write of `bot_file_ids[BOT_KEY]`.

### AC7 — rule

For each of BOT-A through BOT-D: `src/handlers/channel.js` contains all three case-branches: `post.photo`, `post.video`, `post.document`, and writes `bot_file_ids[BOT_KEY]` on Media.create.

### AC8 — rule

For each of BOT-A through BOT-D: mainAdminKeyboard exported from `keyboards/admin.js` renders a Markup.keyboard row of `['🤖 Userbot Login']`; handlers/admin.js has `bot.hears('🤖 Userbot Login', …)` and inline sub-menu with `userbot_add` + `userbot_list` actions.

### AC9 — rule

For each of BOT-A through BOT-D: scenes array exported by `scenes/index.js` has exactly one more scene than before the migration, and the last scene's `.id === 'USERBOT_LOGIN'`.

### AC10 — rule

For each of BOT-A through BOT-D: `src/services/syncService.js` contains NO calls to `bot.telegram.getFile` and NO `deleteMany`. It only reports counts via console.log with a BOT_KEY-tagged line.

### AC11 — rule

For each of BOT-A through BOT-D: handlers/admin.js media_list callback uses `sort({last_seen_at:-1})`, emoji branch reads `m?.metadata?.kind`, and date reads fallback chain.

### AC12 — rule

Cross-bot recursive grep (all 4 bots, exclude node_modules/.kilo/starstomediabot/tradingchart/paymentbots) against legacy regex `\.fileId[^s]|\.fileType|\.channelId|\.channelMessageId|\.addedAt|resolveAndReuploadMedia` → 0 matches in any \*.js or \*.json or \*.md file under the 4 bot roots.

### AC13 — rule

Per bot, 20+ touched files pass `node --check`. GetDiagnostics (IDE) for entire workspace against all changed paths → zero error-level diagnostics.

### AC14 — rubric

Blueprint fidelity. Scale 0-2:
- `2`: every added/rewritten file matches blueprint source byte-for-byte except documented surgical adaptations (e.g., missing advertisedRelay removed, different scene list length); no ad-hoc logic.
- `1`: one minor deviation (e.g., a default value tweak without spec basis).
- `0`: two or more unexplained deviations.
Pass threshold: ≥2.

### AC15 — rubric

Scope containment (no accidental edits elsewhere): Scale 0-2:
- `2`: file diff touches only BOT-A/B/C/D roots, 0 changes in starstomediabot blueprint, 0 changes in tradingchart/PaymentBot folders, 0 changes in Client P\bot\, 0 changes outside of TG BOTS\ (excluding local spec artifacts folder inside starstomediabot\.trae\specs\cross-bot-delivery-migration which is allowed).
- `1`: one accidental stray edit outside target set, quickly reverted in implementation.
- `0`: multiple stray edits or edits to excluded folders.
Pass threshold: ≥2.
