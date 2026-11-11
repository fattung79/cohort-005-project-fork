# How established systems model events so in-app and email can fan out later

Research for [#3](https://github.com/fattung79/cohort-005-project-fork/issues/3), under the map [#1](https://github.com/fattung79/cohort-005-project-fork/issues/1). This file collects facts and trade-offs. It does not choose Cadence's model; that happens in the grilling ticket [#4](https://github.com/fattung79/cohort-005-project-fork/issues/4).

Researched 2026-11-11 against official docs and source code. Each claim links to its source.

## The question

How do notification systems separate three things:

1. the **thing that happened** (the event),
2. **who is told** (one Notification per Recipient), and
3. **by which channel** (in-app, email, …)?

What does each layer store? Where do read state and per-channel preferences live? How does one event reach many Recipients? Which patterns let an in-app-only first version grow into in-app + email without a data migration?

## Summary

| System | Event layer | Per-Recipient layer | Per-channel layer | Read state lives on | Preferences |
| --- | --- | --- | --- | --- | --- |
| Rails Noticed v2 | `noticed_events` row (type, record, params) | `noticed_notifications` row (event, recipient, read_at, seen_at) | Background jobs only. No table. | Per-Recipient row | App code (`if`/`unless` per delivery method) |
| Laravel Notifications | None. The notification is an in-memory object. | `notifications` row, written only by the `database` channel | One queued job per Recipient × channel. No table. | Per-Recipient row | App code (`via()` per notifiable, `shouldSend`) |
| Novu | Trigger event (workflow + payload + subscribers) | One *notification* per subscriber per trigger | One *message* per channel step, with its own status | Per message | Stored: workflow defaults → subscriber global → subscriber per-workflow |
| Knock | Workflow trigger | Workflow run per recipient | One *message* per recipient per channel step | Per message (engagement statuses) | Stored `PreferenceSet`: channel_types, categories, workflows |
| GitHub | None exposed | One *thread* per user per subject (issue/PR/commit) | Not exposed | Per thread (`unread`, `last_read_at`, "done") | Per-thread subscription (`ignored`) plus watch settings |

There are three shapes:

- **Two tables: event plus per-Recipient row.** Noticed v2. The event row holds what happened once. Each Recipient row points to it and holds read state. Channels are jobs, not rows.
- **One per-Recipient row, with the in-app inbox as just another channel.** Laravel. No event row. Each Recipient's `data` JSON is written separately, and the `database` channel is opt-in per notifiable.
- **Three layers: event, per-Recipient notification, per-channel message.** Novu, Knock. Each channel delivery is a row with its own status, and in-app read state sits on the in-app message.

## Rails Noticed (v2)

**Records.** The migration creates two tables ([migration source](https://github.com/excid3/noticed/blob/main/db/migrate/20231215190233_create_noticed_tables.rb); [README](https://github.com/excid3/noticed/blob/main/README.md)):

- `noticed_events`: `type` (the Notifier class), polymorphic `record` ("the record/object this notification is *about*"), `params` (json/jsonb), timestamps.
- `noticed_notifications`: `type`, `event_id` (required), polymorphic `recipient` (required), `read_at`, `seen_at`, timestamps.

**Fan-out.** `NewCommentNotifier.with(record: @comment).deliver(recipients)` creates one Event and one Notification per Recipient. A background job then enqueues one job per Recipient for each `deliver_by` method, and one job for each `bulk_deliver_by` method, such as one Slack post for everyone ([README](https://github.com/excid3/noticed/blob/main/README.md)).

**The database row is not a channel.** A Notifier with no delivery methods "will only create new database records". The in-app row is always written, and email or other channels are added later as `deliver_by :email` with no schema change ([README](https://github.com/excid3/noticed/blob/main/README.md)).

**Per-channel rules** go in code. `config.if`, `config.unless`, `config.wait` and `config.wait_until` apply to each delivery method. The README's example is an email fallback: `deliver_by :email { config.wait = 15.minutes; config.unless = -> { read? } }`. Email is sent only if the in-app Notification is still unread after 15 minutes ([README](https://github.com/excid3/noticed/blob/main/README.md)).

**Read state** is `read_at` on the per-Recipient row, with `mark_as_read`/`mark_as_unread` and `read`/`unread` scopes. `seen_at` exists as a column, but the README documents no API for it ([README](https://github.com/excid3/noticed/blob/main/README.md)).

**Trade-off.** The event's params are stored once, and every Recipient row is small. Rendering a Notification needs a join to the event. Noticed stores no per-channel delivery status, so it has no "was the email sent or did it bounce" record.

## Laravel Notifications

**Records.** The `make:notifications-table` stub creates `id` (uuid), `type`, `notifiable` morphs, `data` (text, JSON), `read_at` (nullable) and timestamps ([stub source](https://github.com/laravel/framework/blob/12.x/src/Illuminate/Notifications/Console/stubs/notifications.stub)). Laravel has no event table.

**Channels per Recipient.** `via($notifiable)` returns the channels for each notifiable, for example `return $notifiable->prefers_sms ? ['vonage'] : ['mail', 'database'];`. `database` is one channel among others. `toDatabase`/`toArray` shape the stored `data`, and `toMail` shapes the email ([docs](https://laravel.com/docs/12.x/notifications)).

**Fan-out.** `Notification::send($users, new InvoicePaid($invoice))`. When the notification is queued, "a queued job will be created for each recipient and channel combination… six jobs will be dispatched to the queue if your notification has three recipients and two channels." `viaQueues` routes each channel to its own queue ([docs](https://laravel.com/docs/12.x/notifications)).

**Late veto.** `shouldSend($notifiable, $channel)` runs when the job is processed and can cancel one Recipient × channel. `NotificationSending` and `NotificationSent` events fire for each send ([docs](https://laravel.com/docs/12.x/notifications)).

**Read state** is `read_at` on the row: `$user->unreadNotifications->markAsRead()`, or a mass `update(['read_at' => now()])` ([docs](https://laravel.com/docs/12.x/notifications)).

**Trade-off.** Laravel is the simplest: one table, and adding mail is a one-word change to `via()`. Payload data is copied into every Recipient's row, and nothing links the rows from one event. Grouping by event, or answering "who else was told", needs a key inside `data`.

## Novu

**Records.** An event names a workflow, carries a payload, and lists subscribers. Novu creates "a **notification** for each subscriber involved" ("One event is one notification"). Each workflow step runs as a job, and each channel step produces **messages**, each with its own ID, channel, delivery status and provider response ([notifications concept](https://docs.novu.co/platform/concepts/notifications)).

**Fan-out.** A trigger to a topic looks up the topic's subscriptions and creates "a separate workflow event … for each subscriber whose subscription passes". The limit is 100,000 subscribers per topic ([topics](https://docs.novu.co/platform/concepts/topics)).

**Preferences** are stored and layered: workflow channel defaults (code), then dashboard, then subscriber global, then subscriber per-workflow. "The most specific layer that defines a channel wins." Each channel resolves on its own. A *critical* workflow ignores both subscriber layers and is hidden from the preferences UI ([preferences](https://docs.novu.co/platform/concepts/preferences)).

## Knock

**Records.** A Message "represents a notification delivered to a User or an Object on a particular channel" ([messages](https://docs.knock.app/concepts/messages)). "Any channel step within that workflow generates a Message for each recipient" ([message statuses](https://docs.knock.app/send-notifications/message-statuses)). The in-app feed is "a specialized view of messages delivered to an in-app feed channel" ([messages](https://docs.knock.app/concepts/messages)).

**Two kinds of status on each message** ([message statuses](https://docs.knock.app/send-notifications/message-statuses)):

- *Delivery status*, exactly one at a time: queued, sent, delivered, delivery attempted, undelivered, bounced, not sent.
- *Engagement statuses*, any number: seen (fetched for display in the feed), read, link clicked, interacted, archived.

**Cross-channel rule.** A step condition can check an earlier message's status. The documented example is a delay step, then an email step that runs only if the in-app message "has not been seen". The delay is needed because conditions are checked immediately ([step conditions](https://docs.knock.app/designing-workflows/step-conditions)).

**Preferences** are a stored `PreferenceSet` with `channel_types`, `categories` (each with nested `channel_types`) and `workflows` (each with nested `channel_types`). "Knock only sends a notification if all preference combinations that exist on the recipient evaluate to true." Anything unset defaults to true ([preferences overview](https://docs.knock.app/preferences/overview)). Example from the docs:

```json
{
  "channel_types": { "sms": false },
  "categories": { "billing": { "channel_types": { "email": true, "in_app_feed": false } } },
  "workflows": { "invoice-issued": { "channel_types": { "email": false } } }
}
```

**Batching.** A batch step collects later triggers for the same recipient, optionally split by a batch key, into one open batch for a time window. The later runs are terminated. When the window closes, the workflow continues once, with `activities` (up to 10), `total_activities` and `total_actors` available to templates ([batch function](https://docs.knock.app/designing-workflows/batch-function)).

Note the difference from Novu: Novu's preference layers resolve by "most specific wins", while Knock requires every applicable level to be true.

## GitHub notifications (inbox style)

**Records.** A notification is a **thread**, one per user per subject. A thread has `subject` (`title`, `url`, `latest_comment_url`, `type`), `reason` (`mention`, `review_requested`, `subscribed`, …), `unread`, `updated_at` and `last_read_at`. The `reason` can change over the thread's life ([REST: notifications](https://docs.github.com/en/rest/activity/notifications)).

**Read state.** `PATCH /notifications/threads/{id}` marks one thread read. `PUT /notifications` with `last_read_at` marks read everything updated before that time, and a large batch can return `202` and finish asynchronously. `DELETE …/threads/{id}` marks a thread "done", a separate state from read ([REST: notifications](https://docs.github.com/en/rest/activity/notifications)).

**Preferences** are per thread: `PUT …/subscription {ignored: true}` mutes it, and `DELETE` mutes it until you are mentioned or comment ([REST: notifications](https://docs.github.com/en/rest/activity/notifications)).

**Delivery to the client** is polling. Requests use `If-Modified-Since`, a `304` doesn't count against the rate limit, and `X-Poll-Interval` sets the minimum interval, which can grow under load ([REST: notifications](https://docs.github.com/en/rest/activity/notifications)). This is close to Cadence's accepted 10–15 s lag.

**Trade-off.** One row per user per subject, updated in place, gives grouping ("5 new replies") for free. That row cannot stand for one email per event, so email is driven from events, not from the thread row.

## Patterns that let in-app-only grow into in-app + email without a data migration

These come from the systems above. Each lists what it costs.

1. **Record the event once, separate from the Recipient rows.** Seen in Noticed (`noticed_events`), Novu (trigger/notification) and Knock (workflow trigger). A later email channel reads the same event, and the payload isn't copied per Recipient. *Cost:* a second table and a join on every inbox read. Laravel omits this and copies `data` into each row.
2. **Treat the in-app row as the per-Recipient record, not as "the in-app channel".** In Noticed the row is always written and channels are added beside it. In Laravel `database` is one channel in `via()`, so a Recipient who chooses email-only gets no inbox row. *Trade-off:* "always a row" keeps one inbox history per Recipient. "Inbox is a channel" makes per-channel opt-out of in-app uniform.
3. **Put per-channel delivery state in its own place when the second channel arrives.** Knock and Novu store a message per channel with delivery status. Noticed and Laravel store nothing and rely on job logs. Adding a `deliveries` table later is a new table, not a migration of existing rows. Existing in-app rows need no backfill if "has an in-app row" already implies "delivered in-app".
4. **Keep read state on the per-Recipient (or in-app message) row.** All five do this. A later email fallback can then read it, as in Noticed's `unless: read?` with `wait`, and Knock's delay + "has not been seen". Knock and Noticed also separate *seen* (shown in the list) from *read* (opened). GitHub adds *done*.
5. **Key preferences by (event type or category) × channel type, with unset meaning on.** Knock (`channel_types` at the top level, per category and per workflow; default true; all must pass) and Novu (global and per-workflow per channel; most specific wins; critical workflows bypass). An in-app-only v1 can store `{type → {in_app: false}}` and add `email` keys later with no migration. *Decision for later:* "all must pass" (Knock) or "most specific wins" (Novu).
6. **Fan out in the background, one unit of work per Recipient × channel.** Laravel queues one job per Recipient × channel. Noticed writes all Recipient rows and then enqueues per-Recipient-per-method jobs. Novu topics create one event per subscriber, capped at 100k. Large fan-outs like Cadence trigger (d) are the case where this matters.
7. **Grouping is either a delivery-time batch or an in-place thread.** Knock batches by recipient (plus an optional key) inside a time window and sends once with `total_activities`. GitHub keeps one thread per subject and bumps `updated_at`/`unread`. The batch keeps every event row. The thread throws away per-event detail in the inbox.
8. **Check rules when delivering, not only when triggering.** Laravel's `shouldSend`, Noticed's `if`/`unless` (run after `wait`), and Knock's step conditions all run when the job executes. Rules such as "never notify someone about their own action" and "skip if already read" can then live in one place for every channel.

## Relevance to Cadence's current code (facts only)

- Cadence stores everything in one SQLite file through Drizzle, with one writer at a time and one machine (ADR 0006, `docs/adr/0006-one-sqlite-file-through-drizzle.md`). Background fan-out has no external queue today. Any "one job per Recipient × channel" pattern would run in-process or as a table that the process polls.
- `app/lib/mailer.server.ts` exposes one `MailTransport.send({to, subject, body})`, with Postmark or a `.outbox/` file. It is a ready "email channel" sink for a later fan-out. It keeps no per-send record.
- `GLOSSARY.md` defines **Notification** as a message to *one* Recipient. That matches the per-Recipient layer in every system above. The glossary has no term yet for the event layer or the per-channel layer.
