# Code Review

## Fix Status

The findings below describe the original implementation. The Worker fixes have
now been implemented: initialized webhook processing, private-chat enforcement,
explicit error handling, strongly consistent Durable Object state, an encrypted
persistent inbox, account resets, expiring picker authorization, preserved raw
passwords, complete notification diffs, bounded requests, message pagination,
resumable notification delivery, reminder retries, and atomic credential/session
writes. Numeric strings are normalized and pending grades are displayed without
being counted as passed courses. Regression coverage is in `tests/regression.test.ts`.

The official elective, repeat-attempt, and semester-label rules still require
verification against the university's rules. Existing KV data is not imported;
log in again after upgrading. The Python prototype is unchanged.

Reviewed 2026-10-11. Scope: every module under `worker/src`, Worker configuration,
package scripts, and README. The Python prototype in `../main.py` was also read
for context. Application code was not changed.

The findings below describe the current implementation. For checks of bot
handlers, the harness supplies mock bot information to get past finding 1.
These later findings will become visible once initialization is fixed.

## High Priority

### 1. Every webhook update fails before reaching the handlers

Location: `src/index.ts:21`, `src/bot.ts:74`.

`createBot` returns a new, uninitialized grammY bot. `handleUpdate` requires bot
information and immediately throws `Bot not initialized!`. The webhook still
returns HTTP 200, so Telegram considers the update delivered. All incoming
commands fail, even though scheduled API sends can work.

Reproduced against the installed grammY package with a synthetic `/start` update.
Initialize the bot before processing updates, or use the supported webhook
adapter and its initialization behavior.

### 2. Private account data can be sent to a group

Location: `src/bot.ts:78`.

The owner middleware checks the sender ID but not the chat type or chat ID.
If the owner runs `/summary` in a group and enters the PIN, the summary is posted
to that group. Login prompts can also run there, exposing credentials to other
members before any attempted deletion. Conversation state is shared between
chats, so a private prompt can also be answered from a group.

Reproduced with a synthetic supergroup conversation. Require the owner's
private chat for all credential, PIN, result, and callback processing.

### 3. The configured error handler does not handle webhook errors

Location: `src/bot.ts:249`, `src/index.ts:21`.

`bot.catch` applies to grammY polling, not direct `handleUpdate` calls. A KV,
portal, or Telegram error rejects the background promise without sending the
configured error reply. The identical-message suppression also never runs.
Since HTTP 200 has already been returned, Telegram cannot retry these failures.

Reproduced by making KV reads throw during `/status`: `handleUpdate` rejects
and no error reply is sent. Catch errors around actual webhook processing or
use middleware that catches errors from downstream handlers.

### 4. KV cannot reliably coordinate the login and PIN conversations

Location: `src/bot.ts:83`, `src/bot.ts:111`, `src/bot.ts:133`, `src/bot.ts:149`.

Each incoming message reads and writes global conversation keys in separate
Worker invocations. KV is eventually consistent, including cached missing keys.
A matric number or password can therefore read an older conversation step or no
step at all. Returning HTTP 200 before processing also allows updates to overlap;
there is no serialization of state transitions. Fast repeated commands can
additionally hit KV's per-key write limit.

This is a platform-contract issue, not something an in-memory mock can reproduce.
Use strongly consistent storage and serialize updates, for example through a
Durable Object for the owner's conversation. Simply keeping a bot in module
scope does not solve coordination across Worker instances.

### 5. Logout and account changes retain the previous account's results

Location: `src/bot.ts:117`, `src/bot.ts:171`, `src/cache.ts:17`.

Logout deletes the session and credentials but leaves `results`, `snapshot`,
`pin_state`, and `last_check`. `loadResults` returns cached results without
requiring an active login. A correct PIN can still reveal results after logout.
Logging into a different account also leaves the old account's results visible
until a refresh, and its notification snapshot is compared against the old
account's baseline.

Reproduced retained results, snapshot, and pending PIN after logout. Bind cached
data to the student identity and reset appropriate state on logout/account
change. Coordinate in-flight refreshes and logins so they cannot restore state
after logout.

## Medium Priority

### 6. A pending PIN prompt breaks a new login conversation

Location: `src/bot.ts:110`, `src/bot.ts:133`.

`/login` leaves `pin_state` intact. Plain text checks that key first. Example:
`/summary`, `/login`, then a matric number produces "Wrong PIN" and leaves login
waiting for a username. The reverse overlap can leave a password conversation
active after another action finishes.

Reproduced that exact sequence. Starting a flow should cancel incompatible
flows; logout should clear all conversation state.

### 7. Old result buttons bypass later PIN prompts indefinitely

Location: `src/bot.ts:186`, `src/bot.ts:204`.

PIN validation only controls creation of the semester picker. Semester and Back
callbacks read current results without checking an authorization expiry or
linking the button to a successfully authorized request. A picker left in chat
can reveal newly refreshed results indefinitely, including after logout.

Reproduced a semester callback with no PIN state or session. If the PIN is meant
to protect each viewing session, track authorization for the picker message,
expire it, and revoke it on logout/account changes. If permanent picker access
is intentional, document that limitation explicitly.

### 8. Some valid passwords are altered or ignored

Location: `src/bot.ts:129`, `src/bot.ts:131`, `src/bot.ts:172`.

All text is trimmed before interpreting the conversation. Leading/trailing
spaces in a password are lost. Any password starting with `/` is passed to
command handling before checking the password step. An unrecognized slash
password is ignored and never deleted; a recognized command runs that command.

Reproduced a `/secret` password remaining unprocessed and undeleted. Preserve
raw password text and explicitly define command cancellation during password
entry.

### 9. Removed results are never reflected in the notification baseline

Location: `src/notify.ts:40`, `src/notify.ts:83`.

The diff only iterates current courses. A removed course is not a change and,
if no other additions or grade changes occur, the snapshot is not updated.
If the removed course later reappears with its original score and grade, it is
also ignored because the old snapshot still contains it. Title and credit-unit
changes are not tracked either.

Reproduced a removed course remaining in the snapshot after a successful check.
Advance the baseline after every successfully processed check, and define which
removals/corrections should trigger notifications.

### 10. Slow portal requests can permanently lose acknowledged commands

Location: `src/index.ts:21`, `src/portal.ts:27`, `src/portal.ts:66`,
`src/portal.ts:102`.

The webhook acknowledges immediately and runs all work in `waitUntil`. HTTP
Workers allow only 30 seconds of background execution after the response.
Portal calls have no explicit timeout and login/re-login can require several
sequential requests. If they exceed that budget, processing is canceled after
Telegram has already accepted HTTP 200. Password conversation state may
already have been deleted, leaving no retry path.

Confirmed from code and Cloudflare's runtime contract; no slow live requests
were made. Add bounded request timeouts and persist work in a queue when it
must survive slow upstream responses.

### 11. Large change notifications can fail forever

Location: `src/notify.ts:55`, `src/notify.ts:85`, `src/format.ts:12`.

All changed courses are sent in one message without a length bound. Telegram
limits `sendMessage` text to 4096 characters after entity parsing. A large batch
is rejected; the snapshot deliberately stays unchanged, so subsequent checks
retry the same oversized message. Further updates only enlarge it. Semester
tables also have no pagination or fallback when oversized.

Split notifications and large tables into valid message-sized chunks, and
persist delivery progress so partial sends do not repeatedly duplicate earlier
chunks. This is a size-dependent defect; live Telegram rejection was not tested.

### 12. A failed login-reminder send is marked as already handled

Location: `src/notify.ts:102`, `src/notify.ts:106`.

The catch branch writes the login error into `last_check` before sending the
reminder. If Telegram rejects that send, the next cron sees the same error and
suppresses the reminder. The user may never receive the request to reconnect.

Track successful reminder delivery separately from the last portal check, and
only suppress a reminder after delivery succeeds. Identified by tracing failure
ordering; a real Telegram outage was not simulated.

### 13. Session and saved credentials can refer to different accounts

Location: `src/bot.ts:172`, `src/portal.ts:96`, `src/credentials.ts:6`.

Login saves the new session before encryption and credential storage succeed.
If encryption configuration is invalid or the credential write fails, the bot
says "Login failed" but the new session remains. Existing saved credentials can
still belong to the old account; when the new token expires, auto re-login can
silently switch back to the old account.

Validate/encrypt credentials before publishing a new session and commit a
consistent account record using coordinated storage. This was identified from
write ordering, rather than tested against production KV failures.

## Portal Data and Policy Questions

These need a representative, redacted API response or verified university rules
before deciding the exact correction:

- `src/portal.ts:116` asserts API types but never validates or normalizes them.
  If `credit_unit` is a JSON string, `units += course.unit` concatenates strings.
  A mock response containing two 3-unit courses graded A and B produced GPA
  `27 / 33 = 0.818...` instead of `27 / 6 = 4.50`. This failure is reproduced,
  but the live portal's numeric types have not been established.
- Null or blank grades, including pending results, make `totalsFor` throw and
  prevent an entire semester/summary from displaying. Whether such grades
  actually occur in this portal was not verified.
- `isElective` excludes every course whose number starts with zero. The code
  alone cannot establish that all such courses are excluded from official GPA,
  or that all excluded courses follow that pattern. The Python prototype uses
  the same assumption and does not independently verify it.
- `semesterLabel` assumes every `12N` key is a sequential semester number.
  This mapping needs confirmation against the actual API's semester identifiers.
- Summary totals count every course attempt, including repeats. Confirm official
  CGPA and earned-unit rules before changing repeat handling.

## Verification and Limits

- `npm run typecheck`: passed. There is no configured test suite.
- `bun /tmp/oau-review.ts`: all 10 local checks completed. Eight reproduced bot,
  state, or notification defects; one reproduced the conditional numeric-string
  GPA defect; one verified encryption round-trip and tamper rejection.
- The harness mocks KV and Telegram calls. No real messages were sent, secrets
  were not read, and no real student portal login was attempted.
- The Python prototype also has unbounded requests, unvalidated result fields,
  and no recovery from malformed cached session JSON. The Worker findings above
  are the primary scope of this review.
- `wrangler.jsonc:7` still contains a placeholder namespace ID, so this checked-in
  configuration needs the real binding before deployment. No deployment was
  attempted.

## Reference Contracts

- [grammY Bot API: initialization and catch](https://grammy.dev/ref/core/bot)
- [Cloudflare KV consistency](https://developers.cloudflare.com/kv/concepts/how-kv-works/)
- [Cloudflare KV writes and limits](https://developers.cloudflare.com/kv/api/write-key-value-pairs/)
- [Cloudflare waitUntil execution limit](https://developers.cloudflare.com/workers/runtime-apis/context/)
- [Telegram sendMessage limits](https://core.telegram.org/bots/api#sendmessage)
