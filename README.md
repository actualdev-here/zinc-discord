# zinc-discord

Experimental Discord API v10 client, written in Zinc, with an optional native
libcurl HTTPS/WSS runtime. The client can execute REST requests, maintain a
single-shard Gateway session and receive events. No Python/Node bridge runs
behind the Zinc API. The pure protocol functions remain available separately.

Requires Linux, the Zinc LLVM backend, libcurl >= 8.2 with WebSocket support,
development headers, OpenSSL libcrypto/headers and a CA trust store. The direct native ELF/Wasm backends do
not implement this runtime. Ordinary Zinc programs do not link libcurl unless
their emitted LLVM IR calls the networking ABI. See [zinc-net](https://github.com/actualdev-here/zinc-net).

## Install from Git

From a Zinc project with a `zinc.toml` manifest:

```sh
zinc add zinc-discord --git https://github.com/actualdev-here/zinc-discord.git --tag v0.1.0
zinc install
zinc check --locked
```

The standalone package resolves tagged `zinc-json` and `zinc-net` Git dependencies
transitively. Use the matching [Zinc compiler/runtime](https://github.com/actualdev-here/Zinc)
with `net.call` and the native networking runtime; installing this library does
not install a compiler or system libraries. `v0.1.0` is experimental, not a
production-readiness or discord.py/discord.js parity claim.

## Local dependency

From a project adjacent to this checkout's `packages` directory:

```toml
[dependencies]
zinc-discord = { path = "../packages/zinc-discord", version = "^0.1.0" }
```

Adjust the path for your checkout. `zinc-json` and `zinc-net` resolve transitively.
The source checkout deliberately keeps sibling-path dependencies for offline
development. Standalone release manifests use version-tagged Git URLs instead.

## REST request construction

```zn
use 'zinc-discord' as discord

discord.Request request = discord.create_message '123456789012345678', 'Hello from Zinc'
# Check request.ok, then pass it to execute with a Client.
```

The base URL is `api_base []`. Helpers construct current-user, Gateway discovery,
channel, get/list/create/edit/delete message, DM, bulk global/guild command registration
requests. Command registration replaces the entire command list; the caller
must opt into executing it. Raw command JSON is syntax/array checked, not
validated against every Discord field. Message helpers reject empty content and
use a conservative 2,000-byte cap. Allowed mentions default to an empty parse
list with replied-user pings disabled. IDs are validated decimal uint64 strings,
never truncated into Zinc's current 32-bit integer.

### Command registration builders

```zn
discord.Definitions options = discord.new_options []
options = discord.add_option options, discord.command_parameter 3, 'text', 'Text to echo', true
discord.Definition echo = discord.slash_command 'echo', 'Echo text privately', options
discord.Definitions commands = discord.new_commands []
commands = discord.add_command commands, echo
discord.Request registration = discord.guild_command_set application_id, guild_id, commands
# Check registration.ok. Executing it replaces ALL commands in this guild.
```

Builders perform no I/O. They return new values; assign each returned list.
Check `ok` / `error`; invalid children poison the resulting list rather than
silently registering a partial command set. `Definition` and `Definitions` keep
their internal state private. `definition_json` / `definitions_json` return
`zinc-json.Text` for inspection, without exposing mutable builder internals.

`command_parameter kind, name, description, required` covers scalar option
types 3–11. `subcommand name, description, options` accepts scalar options;
`command_group name, description, subcommands` accepts nonempty subcommand
lists. Add those branches with `add_option`, then pass the resulting list to
`slash_command`. Scalars cannot mix with branches at the same level. Sibling
names must be unique; required scalar parameters precede optional ones. Lists
are capped at 25 options or 100 top-level slash commands, and serialized
definitions/sets are capped at 60,000 bytes before native transport submission.

This first builder supports lowercase ASCII names (`a-z`, `0-9`, `_`, `-`,
1–32 bytes) and descriptions of 1–100 UTF-8 bytes. These are conservative
subsets of Discord's Unicode rules, not a full Unicode name validator.
Autocomplete configuration, localization, permissions, installation contexts
and context-menu registration builders are not implemented.
Raw `global_commands` / `guild_commands` remain available for
advanced definitions; they validate JSON array syntax only. See
[application command definitions](https://docs.discord.com/developers/interactions/application-commands).

`global_command_set application_id, commands` and `guild_command_set` only
construct bulk-replacement requests. An explicitly empty `new_commands []`
set removes all commands when executed. Nothing is registered automatically;
use a dedicated test guild and opt into replacement deliberately.

#### Parameter bounds and choices

```zn
discord.Definition text = discord.command_parameter 3, 'text', 'Text to echo', true
text = discord.parameter_length text, 1, 2000
discord.Definition count = discord.command_parameter 4, 'count', 'Repeat count', false
count = discord.parameter_range count, 1, 3
count = discord.integer_choice count, 'Once', 1
count = discord.integer_choice count, 'Thrice', 3
# Add each returned Definition to an options list with add_option.
```

`parameter_length` is for string parameters: minimum >= 0, maximum 1–6000,
minimum <= maximum. `parameter_range` is for integer/number parameters and
currently accepts **32-bit integer endpoints** only. `string_choice` accepts
string parameters; `integer_choice` accepts integer/number parameters with
32-bit integer values. Fractional numbers and full 53-bit registration values
still require the raw-JSON API; the invocation decoder remains lossless.

Apply bounds **before** adding choices; a second bounds modifier or a bounds
modifier after choices returns an invalid Definition. Every modifier returns
a new value and preserves the original. There are at most 25 choices with
distinct nonempty labels (up to 100 UTF-8 bytes). String choice values have a
conservative 100-byte cap; local length-bound checks count Unicode codepoints.
Numeric choices must fit the configured range. Wrong types, out-of-range
choices and invalid chains fail closed, without emitting partial definitions
or repeated JSON keys. Use `add_defined_route` below to enforce these same
constraints at invocation. Business rules and permissions still belong to the
application; registration alone does not run runtime validation.

`interaction_reply` returns a type-4 callback JSON payload, optionally ephemeral.
`deferred_reply` produces a type-5 payload. `interaction_callback id, token,
payload` creates the callback request, which `execute` can send. Callback routes
contain a secret interaction token: never log those requests or webhook routes. These helpers do
not verify incoming HTTP interaction signatures. HTTP interaction endpoints must
verify Ed25519 signatures **before** dispatching; no such server is provided.
Gateway-delivered interactions use the authenticated WSS connection instead.

### Message history

`list_messages channel_id, limit, before, after, around` fetches one page.
Choose a limit from 1 to 100 and at most one cursor; use empty strings for the
others. Pagination is explicit: inspect the returned JSON array, keep the last
message ID as text, and request the next page with `before`. Stop on an empty
page, and impose your own page/message budget. No unbounded history crawl runs
inside the client.

```zn
discord.Request page = discord.list_messages '123456789012345678', 25, '', '', ''
discord.Response response = discord.execute client, page
# Check response.ok before parsing response.body with zinc-json.
```

History access depends on channel permissions; message content may also require
the privileged Message Content intent. See [Get Channel Messages](https://docs.discord.com/developers/resources/message#get-channel-messages).

### Deferred replies and followups

Send the initial interaction response within three seconds; interaction tokens
remain usable for followups for fifteen minutes. Choose ephemeral visibility
when deferring: editing the original response does not change it.
See [interaction responses](https://docs.discord.com/developers/interactions/receiving-and-responding).

```zn
discord.Request defer = discord.interaction_callback interaction_id, interaction_token, discord.deferred_reply true
discord.Response acknowledged = discord.execute client, defer
# Check acknowledged.ok before doing the work and completing the reply.
discord.Request complete = discord.edit_original_response application_id, interaction_token, 'Done'
discord.Response completed = discord.execute client, complete
# Check completed.ok before sending a separate followup.
discord.Request extra = discord.create_followup application_id, interaction_token, 'More details', false
discord.Response followed = discord.execute client, extra
```

`original_response` and `delete_original_response` read/delete the initial reply.
`get_followup`, `edit_followup` and `delete_followup` take the returned message ID.
Explicitly complete a deferred response with `edit_original_response` before
creating a separate followup. Content helpers disable automatic mentions;
response bodies remain raw JSON. These snippets assume a live client and IDs
extracted from a Gateway interaction; they are not standalone bot programs.

## Connected client

```zn
str token = process.env 'DISCORD_BOT_TOKEN'
ptr discord.Client client = discord.new_client token, discord.intents_default []
discord.Response response = discord.execute client, discord.current_user []
discord.Event event = discord.next_event client, 1000
discord.close_client client
```

Check for allocation failure (`client` may be null), then `response.ok` /
`event.ok`. A timeout is an event with `ok = true` and empty `name`. Dispatch
events have a name and raw JSON `data`. Errors have `ok = false` and `error`;
`client_error` reports terminal client errors. Never reuse a pointer after
`close_client`. One client per bot/process is the supported model.

`execute` injects Bot authentication only on the fixed Discord HTTPS origin,
and omits it for interaction callbacks and token-authenticated webhook routes.
A webhook 401 does not invalidate the bot session. It keeps per-route/bucket
and global waits, preserves channel/guild/webhook major IDs,
merges learned bucket IDs, rounds fractional delays up, and retries only explicit
429 responses (at most three retries). HTTP transport failures and 5xx responses
are **not** replayed: a message might already have been created. A retry delay
over 60 seconds is returned to the caller rather than blocking indefinitely.
Missing/invalid retry metadata fails closed. Non-2xx status, response body,
error and retry delay remain available in `Response`.

Queries do not create separate rate-limit identities. Interaction/webhook tokens
are replaced in retained route keys, so new tokens cannot exhaust the route
table. Webhook waits conservatively share an application/webhook ID across
tokens; distinct tokens can therefore wait longer than Discord requires.

REST requests run on a bounded native worker; the Zinc client continues pumping
Gateway heartbeats and queues incoming events during both slow HTTP and rate
waits. `next_event` drives the loop; there is no autonomous background Zinc event
loop while application code is busy. Keep handlers short or defer interactions.
Initial discovery/connection can take up to the transport timeout even with a
short event wait; the wait argument is not a hard connection deadline.

Gateway discovery validates the advertised URL and single-shard session-start
budget. Reconnect uses capped exponential backoff with secure jitter, permits
at most five consecutive failures, respects remaining Identify sessions and a
conservative five-second Identify spacing, and resumes when possible. Terminal
authentication/intent errors stop the client. Outgoing Gateway messages have a
conservative 110-per-minute cap (below the protocol ceiling).

The client has bounded capacities: 64 queued events, 128 learned REST routes,
64 KiB HTTP bodies and incoming WSS messages, 4 KiB outgoing WSS messages. Queue
overflow stops the client instead of silently dropping events. REST queries
accept a restricted ASCII alphabet; arbitrary percent-encoded queries and
multipart uploads are not implemented. Single-shard clients,
uncompressed JSON and short handlers are the supported scope.

See the [real /ping bot example](https://github.com/actualdev-here/Zinc/tree/main/examples/packages/discord-bot).
Run its built binary directly: `zinc run` deliberately denies network access and
does not inherit your token environment. Sandbox policy has not been weakened.
`new_test_client` is solely for local fake servers: it requires a `test.*` token
and HTTPS localhost base. Never use it for real credentials.

## Typed event projections

The client still exposes raw event JSON. Two optional decoders cover common
handlers without committing the package to a complete Discord object model:

```zn
discord.Command command = discord.decode_command event.data
if [command.ok and command.command_type eq 1 and str.eq command.name, 'echo'] [
    json.Text text = discord.command_string command.options, 'text'
    if [text.ok] [
        json.Text payload = discord.interaction_reply text.value, true
        discord.Request request = discord.interaction_callback command.id, command.token, payload.value
        discord.Response response = discord.execute client, request
        # Handle response.ok; never print command.token or the request route.
    ]
]
```

Import `zinc-json` as `json` for this snippet. Call `decode_command` on
`INTERACTION_CREATE` data, not the whole Gateway envelope. It accepts application
commands (interaction type 2), including slash, user and message context commands
(command types 1, 2 and 3). Components, autocomplete, modals and unknown command
types return `ok = false`. IDs remain strings. The invoking `user_id` comes from
`member.user` in guilds or `user` in DMs; `resolved` remains raw JSON, not a fetched
or fully validated model. Context commands require a target ID.

`command_option options, name` looks up one level and returns `kind`, raw JSON
`value`, and child `options`. For a group/subcommand, call it again with those
child options. It validates the whole selected option tree, rejects duplicate
names, wrong value types and more than 25 siblings, and bounds nesting to
group → subcommand → scalar. Integer options stay as signed decimal JSON within
Discord's 53-bit range; they are not silently narrowed to Zinc `int`.
Command/option names currently have a conservative 100-byte UTF-8 cap.
`command_string` only accepts string options; missing/wrong-type values return
`ok = false`, while an empty string or boolean `false` remains a present value
in the general option accessor. These are invocation parsers, not registration
builders or autocomplete parsers. See [interaction data](https://docs.discord.com/developers/interactions/receiving-and-responding).

`decode_message` reads a message object from a REST response or Gateway event.
It requires `id` and `channel_id`, validates supplied guild/author/content fields,
and exposes `has_content` / `has_author` for partial updates. Empty content is
distinct from absent content; neither proves the app has the Message Content
intent. `author_bot` defaults to false when the author has no bot flag. This is
a projection, not a complete schema validator or a cache-merging implementation.
See the [message object](https://docs.discord.com/developers/resources/message#message-object).

Check each decoder's `ok` before using fields. Error strings never include the
input payload, but successful `Command` values contain secret tokens: never log
the whole structure. Unknown fields are ignored and remain available in the
original JSON. Decoding does not verify HTTP signatures or grant permissions.

## Interaction controller and command routing

`Event.received_ms` is a monotonic receipt timestamp, captured before queueing.
The new `InteractionContext` owns mutable response state behind one shared
pointer. Create it from an `INTERACTION_CREATE` event, keep its Client alive,
and close it before discarding the Client. Never log the context or the Command
returned by `interaction_command`: both contain interaction secrets.

```zn
ptr discord.InteractionContext context = discord.new_interaction client, event
if [context] [
    discord.Response deferred = discord.interaction_defer context, true
    if [deferred.ok] [
        # Do bounded/cooperative work, then complete the deferred response.
        discord.Response completed = discord.interaction_edit context, 'Done'
        if [completed.ok] [
            discord.Response extra = discord.interaction_followup context, 'More details', false
        ]
    ]
    discord.close_interaction context
]
```

For immediate replies use `interaction_reply_to context, content, ephemeral`.
`interaction_phase` returns pending, sending, replied, deferred, unknown,
rejected, expired or invalid. `interaction_error` describes the last local/API
failure without including tokens. Initial response claims happen **before**
I/O; duplicate reply/defer calls on the same context cannot start another
initial callback. Deferral must be completed with a successful edit before
the controller permits a separate followup. Ephemeral visibility is selected
by the initial callback, not changed by editing.

Pre-dispatch failures and explicit 429s release the in-flight response claim
on the original context while its initial deadline remains. They do not release
the Client's receipt reservation. An attempted request with a lost transport outcome or 5xx
becomes `unknown`; other HTTP rejections become `rejected`. Neither permits
another initial reply. This is conservative, not server-side exactly-once
delivery. **Reuse the original context per interaction ID**. Creating another
context for a replay is now rejected by the Client's receipt store, including
after the original context is closed or its delivery outcome is unknown.

### Bounded replay protection

Each Client reserves `(application_id, interaction_id)` when a valid, unexpired
command controller is created, before handlers or network I/O. For a replayed or
otherwise rejected factory result, the Command returned by `interaction_command`
has `ok = false` and cannot be routed as a valid command. Allocation failure returns null;
other rejection details are available through `interaction_error` on the
returned context. Always close non-null contexts, including rejected ones.

Reservations last 900,000 ms from their first claim, measured by monotonic
client activity. Closing a context does not remove its reservation. Duplicate
events never extend retention. Expired slots are cleared and reused; live
reservations are never evicted to admit new ones. If capacity is reached, new
controllers fail closed without invalidating existing ones. Malformed or
already-expired events do not consume reservations.

`new_client` uses 1,024 slots. `new_client_with_receipts token, intents, capacity`
accepts 1–4,096 slots; account for the full 15-minute arrival window, not merely
the number of concurrently active handlers. The store retains fixed-size
numeric ID chunks, not tokens, payload strings or context pointers, and adds
no per-claim heap allocation. Closing the Client frees its store and slot array.
This does **not** qualify the rest of Zinc's string/object ownership for long
running production bots.

Client polling and REST activity advance retention, including across the clock
wrap. Keep servicing the client; clock rollback or an ambiguous sampling gap
(at least 500,000,000 ms) fails closed rather than discarding live claims.
Recreate the Client after such a clock failure. The pure `ReceiptStore` APIs
(`new_receipts`, `claim_receipt`, `advance_receipts`, `receipt_count`,
`receipt_error`, `close_receipts`) accept explicit clock samples for testing;
the Client's own store remains private and cannot be cleared by application code.

Protection is **per Client and in-memory**, not cross-process or durable.
It does not suppress raw Gateway events, deduplicate followup POSTs, or provide
transactional exactly-once execution of business logic. After retention expires,
the same key can be admitted again. A new Client/process starts a new store;
raw REST helpers bypass it. Construct/check the controller before dispatching
side-effecting handlers, and use durable application-level idempotency keys
when the workload requires stronger guarantees.

The controller budgets initial callbacks from local receipt (including queue
time), with a 150 ms safety margin; later requests use the remaining local
15-minute window, capped to the transport's 15-second request timeout. See
[Discord interaction timing](https://docs.discord.com/developers/interactions/receiving-and-responding).
These are best-effort local budgets, not a guarantee of arrival before Discord's
server deadline. `execute_for client, request, budget_ms` exposes bounded REST
execution (1–15,000 ms) and an `Execution.dispatched` flag: a started worker
means the request **may** have reached the peer. Inspect `Execution.response`;
do not automatically replay an ambiguous POST. Budget accounting includes
rate-limit waits/retries, not just the last HTTP attempt.
Gateway reconnect work and OS scheduling can still delay the calling thread's
return; this is not a hard wall-clock deadline for all client activity.

The pure `ResponseState` functions support deterministic lifecycle tests;
applications should use the shared controller for real requests rather than
copying state values and executing REST separately.

`new_router`, `add_route router, command_type, path, handler_id` and
`resolve_command router, command` provide bounded route lookup. Slash paths
use space-separated names such as `tools tasks run` (root/group/subcommand);
the returned `RouteMatch.options` contains only leaf arguments. Context-menu
names remain intact and are distinguished by command type. Duplicate routes,
invalid paths and more than 128 registrations are rejected. Call `close_router`
when done.

### Definition-backed routes

`add_defined_route router, definition, path, handler_id` accepts the same opaque
slash-command `Definition` used for registration, and binds a real leaf path
to its argument schema. Root/group-only paths, nonexistent branches, parameter
definitions, duplicates and invalid definitions are rejected without consuming
a route slot. A flat command uses its root name; a grouped command uses, for
example, `tools tasks run`. Keep using the same definition values for deployment
and routing rather than maintaining parallel handwritten schemas.

```zinc
discord.Definitions options = discord.new_options []
discord.Definition text = discord.command_parameter 3, 'text', 'Text', true
text = discord.parameter_length text, 1, 2000
options = discord.add_option options, text
discord.Definition echo = discord.slash_command 'echo', 'Echo', options
ptr discord.Router router = discord.new_router []
bool added = discord.add_defined_route router, echo, 'echo', 2
# Check added, then resolve a Command obtained from an accepted controller.
discord.RouteMatch route = discord.resolve_command router, command
# Dispatch route.handler only when route.ok is true.
```

Before exposing a handler, defined routes reject missing required arguments,
unknown/duplicate options, type mismatches, out-of-range strings/numbers and
values outside registered choices. Option order is irrelevant; absent optional
arguments remain absent. String lengths count decoded Unicode codepoints, not
UTF-8 bytes or grapheme clusters; handlers may impose stricter byte limits.
Numeric comparison uses exact JSON decimal values, including exponent forms,
without narrowing 53-bit inputs to int32 or rounding into a range/choice. Thus
`3`, `3.0` and `30e-1` compare equal, but `3.000000000000000001` is strictly
greater than `3` even if a floating-point consumer would round it. NUMBER values
must also fit the +/-2^53 domain. Registration endpoints/choices are still int32
only as described above; this is not a floating-point conversion API.

`validate_command definition, command` provides the pure check without routing,
including root/group/subcommand selection. It returns `ArgumentCheck` with
`ok`, `code`, `error`, and `parameter`. `RouteMatch` exposes the same failure
`code` and `parameter`; failed matches expose no usable handler or leaf arguments.
Codes include `invalid_definition`, `invalid_command`, `command_mismatch`,
`invalid_branch`, `unknown_option`, `missing_required`, `option_type`,
`option_length`, `option_range`, `option_choice`, and `route_missing` (router).
Success has empty error fields. Errors never interpolate raw arguments, unknown
option names or interaction tokens; `parameter` contains a local schema name
only. Never log the entire Command/context/RouteMatch to diagnose rejection.

The original `add_route` is a low-level **schema-free** registry, including
context-menu commands; it does not enforce argument constraints. Unrestricted
`add_defined_route` validates arguments but applies no access/cooldown policy.
Use `add_policy_route` below to opt into those checks. All are **handler-ID
routing**, not automatic callbacks or automatic error replies. Always check
`RouteMatch.ok` before dispatching application code. Definitions do not
verify remote deployment state or resolve per-guild/application command IDs.
The `/ping` / `/echo` example opts into defined routes and sends a fixed private
error reply for rejected commands without echoing untrusted data.

### Access policies and cooldown admission

`new_policy []` returns an unrestricted immutable `CommandPolicy`. Modifiers
return a new value; invalid chains stay invalid. The supported checks are:

- `policy_context policy, context`: `0` any, `1` guild only, `2` DM only.
- `policy_user policy, user_id`: exactly one allowed caller, not inferred bot owner.
- `policy_guild policy, guild_id`: exactly one allowed guild, implies guild-only.
- `policy_permissions policy, member_mask, app_mask`: decimal permission masks;
  use `'0'` to omit either requirement. Nonzero member requirements imply guild-only.
- `policy_cooldown policy, duration_ms, scope`: one admission per 1–900,000 ms;
  scopes `1` user, `2` guild, `3` channel, `4` global. Guild scope implies guild-only.

Contradictory DM/guild/member requirements, malformed IDs/masks, unsupported
contexts/scopes and invalid durations fail construction. A modifier cannot
mutate a policy already copied into a route. There is no allowlist/callback ACL
engine, role-ID check, command deployment permission update or owner discovery.

```zinc
discord.CommandPolicy checks = discord.new_policy []
checks = discord.policy_permissions checks, '8192', '2048'
checks = discord.policy_cooldown checks, 3000, 1
bool added = discord.add_policy_route router, definition, 'moderate', 3, checks
# After checking added, use one accepted InteractionContext per event:
discord.RouteMatch route = discord.resolve_interaction router, context
if [route.ok] [
    # Dispatch the application's handler for route.handler here.
]
```

`Command.member_permissions` and `Command.app_permissions` are separate opaque
`PermissionSet` projections from `member.permissions` and `app_permissions`.
Absent, non-string, malformed, noncanonical or overflowing metadata is unknown,
not a known zero or grant. The command stays replyable so an application can
send a safe rejection. A policy requiring unknown metadata fails closed, without
falling back to another user's resolved object, a role cache or the app's mask.
No synthetic member permissions are created for a DM.

`parse_permissions decimal`, `permission_bit set, bit` (0–63),
`permissions_empty set` and `permissions_include actual, required` preserve
all uint64 bits without int32/float conversion. Zero is valid; leading zeros,
signs, whitespace, exponent/fraction forms and masks wider than uint64 are
rejected, not truncated. Discord masks are variable-width; this implementation
deliberately fails closed beyond 64 bits rather than claiming arbitrary-width
support. Pure containment is literal. Policies apply Administrator expansion
only in guilds, for both member and app masks; DM bit 3 never expands to all
permissions. Administrator does not bypass user/guild filters or cooldowns.
The SDK checks the interaction's effective permission **snapshot**, not role
hierarchies, ownership, channel overwrite recomputation, timeouts, action-specific
restrictions or changes after receipt. Discord still authorizes REST operations.
See [interaction metadata](https://docs.discord.com/developers/interactions/receiving-and-responding)
and [permission semantics](https://docs.discord.com/developers/topics/permissions).

Schema and access checks run **before** cooldown admission; rejections never
consume or refresh a window. A successful timed resolution does consume it
before application work and never refunds it after handler/network failure.
It is not a side-effect-free preview. The key combines route index, application
ID and scope subject, not handler ID or interaction token. User scope spans that
user's guilds; guild/channel scopes share a window among users; global scope
shares one window for that route/application **within this Router only**.
Missing required location data is rejected rather than collapsed into a shared
empty key. Numeric uint64 key chunks are lossless and token-free.

Routers default to 1024 fixed slots. `new_router_with_cooldowns capacity` accepts
1–4096; expired slots are recycled, never live-evicted. Full stores reject new
subjects with `cooldown_capacity` rather than bypassing active limits.
`RouteMatch.retry_ms` reports remaining time for `cooldown` (zero for other
rejections), without sleeping or resending. Policy errors use fixed `code`/`error`
fields such as `user_denied`, `guild_denied`, `guild_required`, `dm_required`,
`member_permissions[_unavailable]`, `app_permissions[_unavailable]`,
`channel_required`, `invalid_policy`, `cooldown_clock` and `cooldown_capacity`.
They expose no handler/arguments or secret-bearing payload fragments.

`resolve_interaction router, context` samples the native monotonic clock at
admission, not the queued event's old timestamp, and refuses expired/rejected/
already-acknowledged contexts. While polling, call `service_router router` even
when idle. Pure deterministic tests use `resolve_command_at router, command, now`
and `advance_router router, now` with the same modulo-1,000,000,000-ms clock
domain as the transport. Rollback, invalid timestamps or an ambiguous gap of
>=500,000,000 ms permanently fail that store closed; create a new router only
as an explicit application lifecycle decision, not to bypass a limit. The
untimed `resolve_command` still checks access, but returns
`cooldown_clock_required` for a configured cooldown rather than silently
ignoring it. `check_policy policy, command` is a pure access-only check; it
does not validate argument schemas or reserve cooldowns.

These stores are process-local, cooperative and not thread-safe/distributed
rate limits, transactional handler execution or durable replay protection.
Rebuilding a Router resets its windows. Keep one long-lived router, check
admissions once before dispatch, and use the Client receipt guard for replay
contexts plus durable application-level idempotency when needed. A raw REST
call or manually dispatched handler bypasses routing policy. Close the Router
with `close_router` when done. The bot example uses independent 2-second `/ping`
and 3-second `/echo` per-user windows, servicing the clock during polling.

Queue draining now services the Gateway even when events are already queued.
This fixes that starvation path, but arbitrary CPU-bound handlers still block
the cooperative Zinc loop. There is no autonomous Zinc scheduler or background
heartbeat while application code runs without yielding. See the
[SDK implementation roadmap](https://github.com/actualdev-here/Zinc/blob/main/docs/DISCORD_SDK_ROADMAP.md).

## Gateway state machine

`new_gateway token, intents` creates a state. `connected state` resets the
connection-local heartbeat state while preserving resumable session metadata.
Pass JSON text frames to `receive state, frame, jitter`; jitter must be a random
integer in 0..1000. `tick state, elapsed_ms` advances heartbeat time. Use elapsed
monotonic time **since the previous tick**, including time spent receiving
frames. The first heartbeat is jittered; subsequent ones require an ACK.
`disconnected state, close_code` handles terminal errors and stale sessions.

Inspect the returned `action` after every call:

- `send`: send `payload` once on the current WebSocket.
- `event`: dispatch `event` and raw JSON `data` once.
- `reconnect`: close the socket, wait at least `retry_ms`, then reconnect.
- `fatal`: stop; fix credentials, intents or configuration before creating a new state.
- `none`: no action.

Do not call the next transition until the current action is consumed. Tick does
not queue actions. A new connection requires `connected`; it must then wait for
HELLO. READY stores session/sequence/resume URL. Reconnect chooses Resume when
possible, otherwise Identify. Non-resumable Invalid Session and close codes
1000/1001/4007/4009 clear session state. Authentication/invalid-shard/version/intent close
codes are terminal. Unknown opcodes are ignored for forward compatibility.
Sequence numbers remain raw decimal text (up to 20 digits), not machine ints;
session IDs are capped at 256 bytes to keep outgoing Identify/Resume bounded.

Use `intents_default []` for guilds only. `intents_messages []` adds guild/direct
message events, **not message content**. Privileged intents require Developer
Portal configuration/approval where applicable. Voice, sharding, compression
and the complete event/model surface are not yet implemented. The connected
client implements single-shard Identify spacing, not a multi-shard concurrency
manager. The low-level state machine is pure;
the connected client above supplies clock, randomness, scheduling and I/O.
Native WSS messages have a 64 KiB cap; large guild events need a larger/streaming
parser before those workloads can be supported.

The state contains a bot token, and Identify/Resume payloads contain it too.
Never log or serialize the whole state/payload. Only bot-token authentication
is intended, not user-account automation. Keep real credentials out of source,
CLI arguments, fixtures and Git. The connected client enforces TLS certificate
and hostname verification, a fixed Discord API origin, no redirects, and a
Gateway/resume hostname allowlist. Native transport rejects raw NUL and invalid
UTF-8 before converting network data to Zinc strings. The low-level pure state
machine alone does not make an origin trust decision.

## Rate-limit gates

Keep one `Gate` per Discord bucket plus one global gate. `block gate, retry_ms`
extends a delay; a shorter new delay cannot bypass an existing wait. `advance`
uses monotonic elapsed milliseconds. `can_send global_gate, bucket` requires
both gates to be ready. Invalid delays fail closed. These pure helpers can also
be used independently; `Client` integrates its own bucket/global timers with
the transport and event queue as described above.

## Verification and remaining work

From the Zinc source checkout (the test harness is not in the standalone library),
`sh tests/discord_package.sh` compiles the consuming package, runs deterministic
REST/Gateway/rate-gate tests and a Python-standard-library JSON differential
corpus, and validates package resolution. Tests use fake tokens and no network.
Receipt tests cover uint64 identity, application namespaces, full-cache behavior,
expiry/reuse, non-refreshing replays, clock wrap and fail-closed rollback.
Builder checks cover nested definitions, bounds and choice constraints, duplicate/order/count/size failures,
sticky invalid values and compile-time private-state boundaries.
Runtime schema tests cover required/unknown/type failures, Unicode lengths,
lossless decimal ranges/choices, nested selections and schema-backed routing.
Permission arithmetic is also checked against Python exact integers across
every bit boundary, deterministic random masks and malformed/overflow cases.
Policy tests cover separate member/app grants, guild-only Administrator,
user/guild/DM restrictions, cooldown scope partitioning, capacity, expiry/reuse,
non-refreshing denials, lossless identities and sticky clock failure.
`sh tests/net_package.sh` additionally runs real local TLS/WSS fake servers:
certificate and hostname rejection, auth/header injection guards, response caps,
masking, fragmented receive across timeout, ping/pong, close/error handling,
READY/Identify, heartbeat during slow HTTP/429 waits, Resume, queued messages,
typed `/echo` interaction handling and fatal close handling. Registration tests
send generated global/guild command sets to the fake HTTPS server and inspect
the authenticated request bodies. Pagination tests traverse 131
pages without exhausting route slots or bypassing a bucket wait. Another 130
interaction tokens reuse a redacted route; deferred/original/followup lifecycle
tests verify scoped authentication and recovery after an expired token.
Controller tests also reject separately-created duplicate contexts before reply,
after closing the owner, and after an unknown 500 outcome; the fake server checks
that each original callback was sent only once.
Definition-backed routing is also exercised on a Gateway interaction; a schema
rejection produces one fixed private callback without dispatching its handler.
Policy integration checks permit one guarded handler admission, reject member/
app permission failures and deny a following event on cooldown; the fake server
checks safe private responses and no duplicate callbacks.
Only test tokens and loopback
connections are used. Python/OpenSSL are test tools, not runtime bridges.

Not yet complete: broad typed endpoint/model coverage, multipart uploads,
sharding, compression, voice/audio, HTTP signed-interaction server, proxy
configuration and a callback framework. Current Zinc string allocation behavior
has not been qualified for long-running production bots. No real-token Discord
smoke test, publication or production-readiness claim is made. Use a dedicated
test bot/guild before considering any deployment.

Protocol references: [API reference](https://docs.discord.com/developers/reference),
[Gateway](https://docs.discord.com/developers/events/gateway),
[Gateway events](https://docs.discord.com/developers/events/gateway-events),
[rate limits](https://docs.discord.com/developers/topics/rate-limits).

## License

[MIT](LICENSE).
