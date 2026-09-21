# Specification: Delivery Pipeline Migration (Bot-Scoped file_id + Userbot Fallback)

## Problem

The current starstomediabot codebase stores a single `fileId` string per Media row. Bot API `file_id` strings are **bot-scoped**: a `file_id` produced by a different uploader (human user, another bot, even the same bot after a token rotation) cannot be reused by this bot's token. Result:

- Media uploaded to the file channel by a non-bot actor (admin user manually posting) delivers corrupted / empty files.
- After any `BOT_TOKEN` rotation, every legacy row's `fileId` becomes invalid until the row is re-seeded.
- The current "fix" (`resolveAndReuploadMedia`) downloads bytes via `getFile` and re-uploads — which is slow, bandwidth-heavy, and can corrupt files on large media.

## Users / Actors

1. **End users** — receive media via point/stars redemption (`handlers/user.js` + `server.js` payment-success path).
2. **Admins** — upload media to the file channel, manage settings, optionally log in a userbot account.
3. **System / Bot** — channel_post ingest listener, periodic sync, payment delivery worker.

## Goals

1. Eliminate file corruption for files not uploaded by this bot token.
2. Support `BOT_TOKEN` rotation without data loss or manual DB edits (auto-reseed on first cold query).
3. Add a GramJS userbot login UI (accessed via a keyboard button, following this codebase's keyboard-button conventions) so the final fallback tier (Plan 4/5) has the session it depends on.
4. Apply the three-tier pipeline to the file channel delivery path.
5. Keep delivery silent on routine skips (no user-visible error spam).

## Non-Goals

- Do not implement full GramJS Plan 4 direct-media send engine; keep the delivery-pipeline shape and the `userbotDirectFallback` hook stubbed (session presence check only) per the Client Pbox design. A real engine can be spliced in later without re-touching hot/cold layers.
- Do not rework payment flow, packages, referral system, broadcast, admin-management scenes, or advertised relay.
- Do not change the Settings-model based single "file channel" concept (we keep it, unlike Pbox's multi-channel approval system — but the file channel is implicitly the approved channel).

## Functional Requirements

### FR1 — Media Model Shape

The `Media` mongoose model SHALL store:

| Field | Type | Purpose |
|---|---|---|
| `source.channel_id` | String (required) | `-100XXXX` id of the file channel where this post lives |
| `source.message_id` | Number (required) | `channel_post.message_id` of the original post |
| `metadata.kind` | `'video' \| 'photo' \| 'document'` | selects `sendVideo / sendPhoto / sendDocument` on hot path |
| `metadata.mime_type` | String | from `msg.video.mime_type` etc. at ingest |
| `metadata.file_name` | String | from `msg.document.file_name` etc. at ingest |
| `metadata.file_size` | Number | byte size at ingest |
| `metadata.uploaded_at` | Date | when post hit the channel |
| `bot_file_ids` | Mixed (Map) | sparse `{ [BOT_KEY]: file_id_string }` — one slot per bot key |
| `file_unique_id` | String (optional, sparse unique) | Telegram cross-bot stable id |
| `mtproto.id / access_hash / file_reference / dc_id` | reserved | null by default, Plan 4/5 slot |
| `last_seen_at` | Date | updated on every successful hot/cold delivery |

Indexes:
- unique compound on `(source.channel_id, source.message_id)`
- sparse unique on `file_unique_id`
- sparse unique on `mtproto.id`

Existing fields `fileId`, `fileType`, `channelId`, `channelMessageId`, `addedAt` SHALL be removed and replaced by the new shape. A one-time migration (on boot, if legacy rows exist) is not required by the spec but data can be re-seeded from the channel.

### FR2 — BOT_KEY Derivation

A `BOT_KEY` constant SHALL be derived at require-time from env:
`String(process.env.CURRENT_BOT_KEY || (process.env.BOT_TOKEN || '').split(':')[0] || 'default').trim()`

The numeric token prefix (`123456` from `123456:ABC…`) is the default. `CURRENT_BOT_KEY` overrides for same-bot token rotations.

### FR3 — Three-Tier Delivery Pipeline

`deliverMedia(telegram, chatId, count, { excludeIds })` SHALL pipeline every item through three tiers in order, advancing to the next only on failure:

1. **Tier 1 (hot)** — use `row.bot_file_ids[BOT_KEY]` to call `sendPhoto/sendDocument/sendVideo` based on `metadata.kind`.
2. **Tier 2 (cold + reseed)** — use `copyMessage(chatId, source.channel_id, source.message_id)`, falling back to `forwardMessage` on failure. On success, extract the returned message's `file_id` and upsert it into `row.bot_file_ids[BOT_KEY]` (plus `last_seen_at = now`). This is the reseed step that fixes bot-token rotation and non-bot uploads.
3. **Tier 3 (userbot)** — stub that checks `hasActiveUserbot()` for a non-null session in `UserbotAccount`. Returns a sentinel `{ ok: false, reason }`; actual GramJS send is NOT in scope (Non-Goal 1).

On Tier 2 errors matching "message not found / message to copy not found / CHANNEL_PRIVATE / message_id_invalid", the dead row SHALL be lazily deleted via `Media.deleteOne`.

### FR4 — Channel Post Ingest

The `channel_post` handler (`handlers/channel.js`) SHALL:

- Only react to posts on the configured `fileManagerChannel` (from `Settings`).
- Detect kind: photo (largest size), video, or document. Ignore text/audio/etc. silently.
- Populate `source.channel_id`, `source.message_id`, `metadata.{kind,mime_type,file_name,file_size,uploaded_at}`, and `bot_file_ids[BOT_KEY]` from the incoming `msg.*.file_id`.
- Capture `file_unique_id` if the Telegram message exposes it.
- Notify admins (existing behavior) with kind emoji + count.

### FR5 — Sync Service Changes

`syncMediaPool` (which currently calls `getFile` on every `fileId`) SHALL be replaced or adapted to:

- Iterate rows, grouped per `source.channel_id`.
- For each row without `bot_file_ids[BOT_KEY]`: attempt one `copyMessage` to a throwaway `Saved Messages` style approach is NOT required. Instead: skip. Cold Tier 2 will lazily reseed on first user query.
- Keep the 20% fail-rate guard against deleting rows during Telegram outages.

### FR6 — UserbotAccount Model

Add a new model `UserbotAccount` matching Client Pbox:

```
{ number, username, userId, session (StringSession string), timestamps }
```

Indexes: sparse unique on `number`, sparse on `session`.

### FR7 — Userbot Login UI (Keyboard-Button Style)

Add a new admin-reply keyboard entry **"🤖 Userbot Login"** in `mainAdminKeyboard()` (reply keyboard, matching this codebase's convention of `Markup.keyboard` rows, NOT inline buttons).

When pressed by an admin:

1. A scene-based or session-based state machine SHALL step through:
   - ask phone number (`+CCXXXXXXXX` format)
   - send verification code via GramJS `auth.SendCode` (with `PHONE_MIGRATE_*` DC redirect support)
   - ask verification code
   - if `SESSION_PASSWORD_NEEDED`, ask 2FA password and use `account.GetPassword` + `computeCheck` + `auth.CheckPassword`
   - persist resulting `StringSession` to `UserbotAccount`
2. Cancel button on every step returns user to admin main keyboard.
3. A secondary **"👁 List Userbot Accounts"** menu entry (or inline keyboard under the userbot button) lists saved accounts (number + @username) if any exist.

This login module SHALL use the GramJS `telegram` package (`TelegramClient`, `StringSession`, `Api`) and the helpers `randomFingerprint` + `sendCodeWithRetry` patterns ported from Client Pbox.

### FR8 — cache.js Additions

The existing `cache.js` (adminCache) SHALL additionally export a `deliveryCache` LRU for Media rows keyed by `_id` (or similar pattern, per `queryCache` in Pbox), used on `deliverMedia` hot path to avoid repeated `Media.findOne` calls. TTL 4 hours, max 2000 entries.

### FR9 — package.json Dependencies

Add:
- `telegram` (GramJS, for userbot login)
- `lru-cache` (for delivery and pending-promise caches)

If they are not already present.

### FR10 — Silent Sentinel Handling

All delivery call sites (`handlers/user.js` `executeRedemption`, `server.js` `payment-success` worker) SHALL consume the pipeline's sentinel reasons without user-visible errors for `all_paths_exhausted`, `no_file_id`, `copy_or_forward_failed`, `channel_unapproved`.

User-visible response text remains: "Delivered X items!" for actual deliveries.

## Non-Functional Requirements

1. **Backward compatibility**: Existing `User.receivedMedia` (which stores `Media._id`) SHALL keep working. The `executeRedemption` filter-by-exclude pattern in `handlers/user.js` SHALL remain unchanged.
2. **No UI regressions**: All existing reply-keyboard and inline-keyboard flows (admin mgmt, packages, broadcast, gift-media, referral) SHALL continue working with no text changes.
3. **Rate limiting**: Existing `queue.js` `enqueue` / `enqueueBroadcast` concurrency controls SHALL remain the outer gate for all delivery sends. Inside a single send, Tier 1/2 calls MAY bypass internal rate-limiting of Pbox's `rateLimited` drain-timer since `queue.js` already serializes.
4. **Silent degradation**: Missing `API_ID` / `API_HASH` (GramJS env vars) MUST NOT break hot or cold delivery. Only the userbot login button and userbot fallback stub become no-ops.

## Constraints

- Single file channel id only (Settings `fileManagerChannel`). Multi-channel upload approval (Client Pbox `UploadChannel` / `channelCache.isApproved`) is NOT adopted because this codebase has a single file-channel concept. We keep the Settings-based single channel and treat it as implicitly approved.
- No deletion of channel posts (Clear Storage) in this spec; Client Pbox `pruneStale` periodic GramJS sweep is Non-Goal.
- `document` support added (current bot only supports photo/video).
- Payment / stars / referral / broadcast / gift / admin scenes are untouched.

## Dependencies / Assumptions

- Node 18+ for `fetch()` (used nowhere now; `resolveAndReuploadMedia` is removed entirely so no concern).
- GramJS env vars `API_ID` and `API_HASH` are optional; admin userbot UI errors nicely if missing.
- MongoDB document schema change is applied cleanly; old `fileId`/`fileType` rows are ignored (no automatic migration required; re-upload or live cold reseed handles it).

## Open Questions

1. **Do we need automatic one-time migration of legacy Media rows (populate `source` from legacy `channelId`/`channelMessageId`)?** Default: no, let cold path reseed lazily.
2. **Should `document` files be restricted to specific mime types?** Default: allow any document (admin controls what is posted).

## Acceptance Criteria

### rule: AC1 — Media model stores bot_file_ids map

The `Media` schema has `bot_file_ids: Mixed default {}` and unique compound index on `(source.channel_id, source.message_id)`.

### rule: AC2 — Non-bot uploads deliver without corruption

When a human admin posts a video directly to the file channel (so incoming raw `file_id` is not bot-scoped to our token), the first end-user redemption of that media:

1. Tier 1 fails (`no_file_id` on empty `bot_file_ids[BOT_KEY]`).
2. Tier 2 succeeds via `copyMessage`.
3. Returned `file_id` is written to `bot_file_ids[BOT_KEY]`.
4. Second redemption hits Tier 1 with no errors.

### rule: AC3 — Token rotation reseeds automatically

After changing `.env` `BOT_TOKEN` to a new token (different `BOT_KEY`):

1. First redemption of every legacy row hits Tier 2 cold path.
2. After one successful cold delivery, `bot_file_ids[NEW_BOT_KEY]` is populated.
3. Second delivery uses Tier 1 hot path.

### rule: AC4 — Dead rows cleaned lazily

Deleting a channel post manually results in `copyMessage` failure matching the dead-message heuristic, and the corresponding `Media` document is deleted within one redemption attempt.

### rule: AC5 — Userbot login reachable via keyboard

An admin sees **"🤖 Userbot Login"** as a new button on the admin reply keyboard. Pressing it starts a phone → code → (optional password) GramJS login flow. Successful login produces one `UserbotAccount` row with non-null `session`.

### rule: AC6 — No userbot session = silent skip

With zero `UserbotAccount.session` rows saved: Tier 3 of the pipeline returns `{ ok:false, reason:'no_userbot_session' }` and the call site produces no user-visible error message or alert.

### rule: AC7 — document media kind works

Ingesting a `.zip` / `.pdf` / other `document` from the file channel stores `metadata.kind='document'` and on hot/cold delivery routes through `sendDocument`.

### rubric: AC8 — Delivery correctness (0-2)

- 2: 10/10 manual redemption flows deliver valid, uncorrupted media files identical to the channel originals.
- 1: minor cosmetic issues (e.g. caption stripped) but media bytes valid.
- 0: corrupt or empty files in >1/10 cases.

Pass threshold: >= 2.
