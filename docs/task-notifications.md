# Per-user task notifications

Telegram notifications extend the existing task inbox. They run in the control
service and use its shared Postgres/SQLite adapter. No hostd, guest image, snapshot,
network or VM-performance changes are required.

## Use it

Open **Settings → Task notifications**, choose **Connect Telegram**, open the
link and press **Start** in the bot's private chat. Return to Settings and confirm
the displayed Telegram identity. Connecting enables alerts, with separate choices
for finished tasks, failures and tasks needing attention. **Send test** queues a
test message; Settings distinguishes queued delivery from a successful send.

A notification contains its outcome, the task's title, and a button linking to
`#/inbox/task/<id>`. Prompts, commands, results, terminal output, diffs and account
credentials are not sent to Telegram. The task title itself is sent, as disclosed
in Settings; choose titles accordingly. Existing inbox ownership checks still
apply when a recipient opens the link.

Supported events:

- Success: a task finished successfully.
- Failure: task execution or its verification failed.
- Attention: `needs_input`, interruption or a confirmed preparation failure.
- Cancellation does not notify. An uncertain dispatch is not labelled a failure;
  the existing reconciliation checks its ID without replaying execution.

Alerts work without an open dashboard: the existing workflow background refresh
observes completion and persists the delivery intent. Existing terminal-only
sessions and host health/auto-pause alerts are outside this feature. They retain
their current ntfy behavior. No task scheduler, email or group-chat support is
introduced.

## Account isolation and linking

A signed-in request creates a 192-bit one-time link, valid for ten minutes. Only
its SHA-256 digest is stored. A new link invalidates the account's old pending
link. Telegram callbacks require a separate constant-time-checked webhook secret,
a bounded request body, a private chat and matching sender/chat IDs. They can
propose a chat but cannot name a Substrate user or enable notifications. The
signed-in browser must confirm before the destination changes. A chat cannot be
connected to two Substrate accounts simultaneously.

Preferences and connections are per account, not browser storage. Disconnecting
invalidates pending links and cancels queued messages. Reconnecting creates a new
destination generation so old deliveries cannot be redirected to a newly linked
chat. Preference changes are checked again at delivery. A request already sent
to Telegram cannot be recalled by disconnecting.

The preferred configuration stores the bot token and webhook secret in Parameter
Store SecureStrings. Where the deployment identity cannot create parameters,
`telegram:install` provides an operator fallback: it generates a one-use RSA key
on control, transfers only an RSA-OAEP-encrypted payload through the existing S3
artifact bucket, and writes `/etc/substrate/telegram.env` as root with mode 0600.
The private key and temporary plaintext are removed. Ciphertext is deleted from
S3 when allowed, otherwise normal artifact retention removes it; its decryption
key no longer exists. Tokens never enter SSM command text or command arguments.

The fallback survives app deployments and reboots, but must be reinstalled on a
replacement control instance. It is read before Parameter Store's generated
environment, so managed configuration takes precedence when available. It does
not change any IAM policy or other service credential. Parameter Store remains
the preferred durable configuration once the operator can write that prefix.

Secrets never go to the browser or notification payload. Errors omit Telegram's
token-bearing API URLs and provider response text. Only chat labels, preferences
and sanitized delivery status are returned by the account API. The chat ID is
stored in encrypted RDS storage and is not delivered to guests.

## Delivery and persistence

Migration 3 adds `notification_settings`, `telegram_links` and
`notification_outbox`. It is additive; task tables and snapshot formats are
unchanged. Local SQLite uses the same queries as RDS Postgres. The earlier
control release can run against the additional tables, but does not process them.

An active-to-terminal task save and its outbox entry commit in one transaction.
An event key unique to the task suppresses repeated refresh alerts. Historical
completed tasks are not backfilled, and tasks completed while a category is off
are not replayed when it is turned on. Comparison approaches notify independently
using their task IDs and owner.

A background worker polls every three seconds and processes up to ten due rows
per pass. Each row has an atomic 30-second lease, so multiple workers do not
normally send it simultaneously and abandoned work can resume after restart.
Requests time out after ten seconds. Transient failures back off at 15 s, 1 min,
5 min, 15 min and 1 h; Telegram's `retry_after` is respected when longer. Delivery
stops after six attempts or 24 hours. Telegram 400/403 errors are permanent for
that message. Expired links and final delivery records older than 30 days are
pruned. Test messages are limited to one per account per 30 seconds.

Delivery is **at least once**, not exactly once: if Telegram accepts a message
but its response or the database acknowledgement is lost, a retry can duplicate
it. Repeated normal inbox reads do not duplicate notifications. A delivery failure
never reruns or cancels the agent task, and its saved result remains in the inbox.

## Configure the dedicated bot

The operator creates **one shared Substrate bot** with
[BotFather](https://t.me/BotFather). Individual users do not create bots; each
connects their private chat to this one bot in Settings.
Use an AWS identity authorized for `ssm:GetParameter` and `ssm:PutParameter` on
`/substrate/telegram/*` and to read `/substrate/public-url`. From this branch:

```sh
bun run secrets --telegram
bun run deploy:control
bun run telegram:setup
```

The first command reads `TELEGRAM_BOT_TOKEN` from the environment (including Bun's
root `.env` support), or prompts privately when it is unset. `.env` and `.env.*`
are already excluded from Git. It validates the token with `getMe` and
stores these parameters:

- `/substrate/telegram/bot-token` — SecureString.
- `/substrate/telegram/bot-username` — String.
- `/substrate/telegram/webhook-secret` — independent SecureString.

The existing control instance role already reads `/substrate/*`; no CDK change
or host deployment is needed. The control bootstrap maps the parameters to
`SUBSTRATE_TELEGRAM_BOT_TOKEN`, `SUBSTRATE_TELEGRAM_BOT_USERNAME` and
`SUBSTRATE_TELEGRAM_WEBHOOK_SECRET`. `SUBSTRATE_PUBLIC_URL` must be HTTPS.
Without that configuration, the Settings section explains that Telegram is
unavailable, and task execution/inbox behavior continues normally.

When the supplied AWS identity has deployment/SSM access but cannot create the
new Parameter Store prefix, use the same shared bot through the protected
control-file fallback instead:

```sh
bun run telegram:install
bun run deploy:control
bun run telegram:setup --control
```

Bun loads `TELEGRAM_BOT_TOKEN` from the root `.env`; `--env-file=/path/to/.env`
can select another file without copying it into this worktree. The installer
also supports a private prompt. `--control` registers the webhook from the
running control service's effective environment, without returning its secrets
to the deployment caller.

The final setup command checks the deployed secret-authenticated endpoint, then
registers `/api/telegram/webhook` with Telegram. It retains pending updates and
refuses to replace a different webhook on a reused bot. An existing bot used by
another application must not be repurposed. After setup, each user connects their
own private chat and confirms it in Settings. Never paste the token into chat or
pass it as a CLI flag.

Telegram's [deep-link contract](https://core.telegram.org/bots/features#deep-linking)
and [Bot API](https://core.telegram.org/bots/api#setwebhook) define the Start
payload and authenticated callback; the API's
[error parameters](https://core.telegram.org/bots/api#responseparameters) define
rate-limit delays.

## API

All account paths require existing browser/CLI authentication and return
`Cache-Control: no-store`. The authenticated identity determines the account;
there is no user-ID or chat-ID parameter to set a destination.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/notifications` | Own connection, preferences and last delivery |
| PUT | `/api/notifications` | Save all four boolean preferences |
| POST | `/api/notifications/telegram/connect` | Issue a pending connection link |
| POST | `/api/notifications/telegram/confirm` | Confirm the proposed private chat |
| DELETE | `/api/notifications/telegram` | Disconnect and cancel queued deliveries |
| POST | `/api/notifications/test` | Queue a test; returns 202 |
| POST | `/api/telegram/webhook` | Telegram only; requires webhook-secret header |

None of these paths is available through the separate preview origin.

## Validation

- Full suite: 222 passed, one Linux-only PTY test skipped on macOS. Typecheck,
  production web build and changed bootstrap shell syntax passed.
- Real local Postgres 16: 24 notification/workflow/migration tests passed,
  including row leases, concurrent workers, transaction rollback, legacy-store
  migration and existing task/comparison execution.
- Tests cover owner scoping, callback authentication, private-chat/browser
  confirmation, expiry/replay, preference checks, reconnect/disconnect,
  duplicate refresh, retry/restart, rate limits, permanent failures and expiry.
- Real local command execution was observed by background reconciliation with
  no inbox request; its notification contained the title/link and no output.
- Browser checks use mock account identity and a mock Telegram transport with
  real local API/database/queue/task logic. Native input covered connect,
  confirmation, test and disconnect. Desktop (1440 px) and phone (390 px)
  captures show pending, confirmation, connected and unconfigured states without
  horizontal overflow. The mechanical design detector returned no findings.

Live activation used the provided ignored `.env` token. Telegram validated the
bot; the supplied AWS identity could not write the new Parameter Store prefix
or inspect/manage IAM. The encrypted control-file installer therefore supplied
the fallback configuration without an IAM change. Root ownership/mode 0600 and
removal of the one-use transfer keys were verified on control.

Control/UI release `af0f7a9` is deployed. Migration 3, RDS IAM/TLS, the existing
CLI credential/history, SSH/Windows stores and KMS checks passed on AWS, with
all synthetic acceptance rows cleaned up. The secret-authenticated callback was
probed through the public URL and Telegram accepted webhook registration.
Anonymous settings/connect requests return 401; unauthenticated callbacks return
404. Telegram reported no pending incoming updates or provider error.

The operator completed the browser/Telegram flow and reported receiving the
test. Read-only RDS checks confirmed two successful sends and no pending
deliveries after a control redeploy. The chat connection was subsequently
disconnected; that preference was retained rather than overridden. No paid agent
or extra VM was started, and hostd/performance code was not deployed.

Another deployment briefly restored older main during validation. The notification
release was restored and the public routes rechecked. Future control deployments
must include this feature to retain its routes; the root-only fallback file is
retained even if an older application release temporarily ignores it.
