# Signal channels

A **signal channel** carries **notices** from a provider to live runs. A provider is any user, typically a service
that spots an instrument about to move. It publishes a notice saying "this pair, or this pool, deserves attention
now". Any user can **subscribe** one of their live runs to the channel. Every notice is then delivered to the run by
**attaching** the instrument to it, as `POST /live/{runId}/attachments` would, either at once or after the delay the
subscription chose.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/signals/channels` | Create a channel of yours |
| `GET` | `/signals/channels` | List your channels |
| `GET` | `/signals/channels/public` | Browse public channels (no token) |
| `GET` | `/signals/channels/{channelId}` | Read a channel |
| `PATCH` | `/signals/channels/{channelId}` | Change a channel of yours, or replace its webhook address |
| `DELETE` | `/signals/channels/{channelId}` | Delete a channel of yours |
| `POST` | `/signals/channels/{channelId}/notices` | Publish a notice |
| `GET` | `/signals/channels/{channelId}/notices` | List your notices and how they were delivered |
| `POST` | `/signals/hook/{hookKey}` | Publish a notice through the channel's webhook address (no token) |
| `POST` | `/signals/channels/{channelId}/subscriptions` | Subscribe a run of yours |
| `GET` | `/signals/subscriptions` | List your subscriptions |
| `DELETE` | `/signals/subscriptions/{subscriptionId}` | End a subscription |
| `GET` | `/signals/subscriptions/{subscriptionId}/deliveries` | What a subscription received |

The scopes these operations need are in [Authentication and scopes](auth.md#the-scope-each-operation-requires).

## How it works

1. The provider creates a channel and says which **delays** it offers, for example `[0, 60]`: delivered at once, or
   a minute later.
2. A subscriber starts a run, usually with no `sources`, so that it waits (see
   [Attaching and detaching a source](live.md#attaching-and-detaching-a-source)). Then it subscribes the run to the
   channel, choosing one of the delays.
3. The provider publishes a notice about an instrument.
4. When the notice is due to a subscription, at its publication plus the subscription's delay, the instrument is
   attached to the run. A run that was waiting starts on it; a run already reading takes it as one more source, at
   the attachment's `effectiveAtMs`. The strategy learns of it in `onAttach`, as with any attachment (see
   [QTScript](qtscript.md) and [Coding Java strategies](strategy_coding.md)).

A delivery is an ordinary attachment of the subscriber's run: the subscriber's plan, and the rules of the run's
sources, decide whether it fits. A notice never costs the subscriber more than an attachment it made itself.
Nothing is detached when a later notice comes; detach an attachment yourself when you are done with it.

## Creating a channel

```json
POST /signals/channels
{
  "name": "Early pool alerts",
  "description": "Pools about to move, seconds ahead.",
  "visibility": "public",
  "delays": [0, 60]
}
```

```json
{
  "channelId": "7aQk2mVn9pXr4sTw1yZb3c",
  "name": "Early pool alerts",
  "description": "Pools about to move, seconds ahead.",
  "visibility": "public",
  "delays": [0, 60],
  "createdAtMs": 1758330000000,
  "webhookUrl": "https://api.qtsurfer.net/v1/signals/hook/7aQk2mVn9pXr4sTw1yZb3c.Jp3kQ0aZx9LmB2vT7nWc4e",
  "subscriptions": 0
}
```

| Field | Meaning |
|---|---|
| `name` | Required, 1 to 64 characters. What subscribers see. Your own user is never shown to anyone. |
| `description` | Optional, up to 500 characters. |
| `visibility` | `unlisted` (the default): found only by its `channelId`, which you hand to whoever you want. `public`: also listed in `GET /signals/channels/public`. |
| `delays` | 1 to 4 different whole seconds from `0` to `3600`; left out, `[0]`. Each subscription chooses one. |

`webhookUrl` and `subscriptions` (how many are active; never who holds them) are shown only to you, the owner.

`PATCH /signals/channels/{channelId}` changes any of these. A subscription keeps the delay it chose even if `delays`
no longer offers it. `"rotateWebhook": true` replaces `webhookUrl`, and the old address stops publishing at once.
`DELETE` ends every subscription, and the notices still due are not delivered.

## Publishing a notice

```json
POST /signals/channels/{channelId}/notices
{
  "source": {
    "venueType": "dex",
    "exchange": "robinhood",
    "segment": "uniswap-v3",
    "instruments": ["0x3f4e0b2c8e1a4b6d9c0f1a2b3c4d5e6f708192a3"]
  },
  "ref": "alert-8841",
  "expiresIn": 300,
  "data": { "direction": "up", "confidence": "high" }
}
```

```json
{
  "noticeId": "2Hn8sKp4vQx7mRt1wYz6Ab",
  "deliveries": 12,
  "receivedAtMs": 1758330000000,
  "expiresAtMs": 1758330300000
}
```

| Field | Meaning |
|---|---|
| `source` | Required. The instrument: a source as a start takes it (see [Sources](live.md#sources)), without `type` and with the instruments named one by one. For a `cex` source, 1 to 10 pairs, with no `*`. For a `dex` source, exactly one pool. |
| `ref` | Optional, up to 64 characters. The same `ref` to the same channel answers the notice it already made, so a retried request or a repeated alert is safe. A repeat does not count against the channel's rate. |
| `expiresIn` | Optional, `1` to `3600` seconds, `300` by default. A delivery not made by then is `EXPIRED`. A subscription whose delay is longer than this never receives the notice. |
| `data` | Optional. Text for subscribers to read with the notice: up to 16 entries. Keys are lower case (letters, digits, `_`, parts separated by `.`), up to 64 characters. Values are 1 to 256 characters with no control characters. It does not reach the strategy. |

`deliveries` is how many subscriptions the notice goes to: those active when it is published. A subscription made
afterwards does not receive it. What each delivery did is the subscriber's to read; `GET
/signals/channels/{channelId}/notices` shows you the counts by status.

### Through the webhook address

A sender that cannot get a token, or set a header, publishes with the same body at the channel's `webhookUrl`:

```bash
curl -X POST "https://api.qtsurfer.net/v1/signals/hook/7aQk2mVn9pXr4sTw1yZb3c.Jp3kQ0aZx9LmB2vT7nWc4e" \
  -H "Content-Type: application/json" \
  -d '{"source":{"venueType":"cex","exchange":"binance","segment":"spot","instruments":["SOL/USDT"]},"ref":"a-17"}'
```

The address is the credential: keep it secret, and replace it if it leaks. It answers as publishing with a token
does, with your plan's limits. An address that is not a channel's current one answers `404`.

## Subscribing a run

```json
POST /signals/channels/{channelId}/subscriptions
{ "runId": "5t5oAmQ4PD0lQRoCU58uE0", "delay": 60 }
```

| Field | Meaning |
|---|---|
| `runId` | Required. One of your live runs. Each run is subscribed to a channel once. |
| `delay` | Optional. One of the channel's `delays`; left out, the smallest. |
| `type` | Optional, `ticker` or `kline`, `kline` by default. How the run reads a centralized exchange's instrument; it must be one the run's strategy consumes. A pool is always read as `kline`. |
| `warmFrom` | Optional, `0` to `3600`, `0` by default: the warm-up of each attachment a delivery makes (see [Warming up](live.md#warming-up-warmfrom)). With `0` the strategy reads from the moment of the delivery, which is the fastest way to act on a notice. A pool delivered to a run that already reads another is read from that moment, whatever this says. |

Any channel can be subscribed to, `unlisted` or `public`, yours or anyone's, as long as you have its `channelId`.
`DELETE /signals/subscriptions/{subscriptionId}` ends a subscription. What it already attached stays attached.

## Reading deliveries

`GET /signals/subscriptions/{subscriptionId}/deliveries` lists every notice the subscription was due to receive,
most recent first, and what its delivery did. A delivery carries the notice's `source` and `data` from its `dueAtMs`
on; before that it says that a notice is coming and when, not what it is about, so a subscription at a delayed level
learns of the instrument no earlier than its level does.

| `status` | Meaning |
|---|---|
| `PENDING` | Not due yet, or being made. |
| `ATTACHED` | The instrument was attached to the run; `attachmentId` names the attachment. Read the run's `attachments` for how it stands. A notice about an instrument the run already has attached counts as delivered, with that attachment. |
| `REFUSED` | The run could not take it; `reason` says why (below). |
| `EXPIRED` | Not made before the notice expired. |
| `SKIPPED` | Not made: `NOT_RUNNING` (the run was not running), `UNSUBSCRIBED` or `CHANNEL_DELETED`. |

The `reason` of a `REFUSED` delivery:

| `reason` | When |
|---|---|
| `PLAN_LIMIT` | A limit of your plan: the instruments a run may read, the pools you may follow at once, or the runs you may have reading at once. |
| `PLAN_REQUIRED` | Your plan no longer includes live runs, or subscriptions. |
| `OTHER_FEED` | The run reads another market feed: every source attached to a run at the same time is on the same one. |
| `NOT_ACCEPTED` | The run's strategy does not accept that instrument or that `type`. |
| `CAPACITY` | The platform had no room. |

The sets may grow: read an unknown `status` as not delivered, and an unknown `reason` as a refusal.

## Limits by plan

This table is the one place that lists them.

| Plan | Channels you own | Active subscriptions | Notices per channel, per 60 seconds |
|---|---|---|---|
| free | 0 | 0 | 0 |
| basic | 1 | 2 | 6 |
| pro | 3 | 10 | 30 |
| elite | 10 | 50 | 60 |

A plan with `0` for something does not include it: the operation is refused with `402`. Past the limit, creating a
channel or subscribing is refused with `429` and `reason: PLAN_LIMIT`, with no `Retry-After`. Publishing past the
rate is refused with `429` and `reason: NOTICE_RATE_LIMIT`, `retryable: true`, and `Retry-After` says when the next
notice fits.

## What can refuse a request

| Status | When |
|---|---|
| `400` | A field outside its rules; a notice's `source` that is not one instrument or pool of a market that can be followed live; a subscription's `type` that the run's strategy does not consume. |
| `402` | Your plan does not include it (see [Limits by plan](#limits-by-plan)). |
| `404` | No such channel (or it was deleted), no such run or subscription of yours, or a webhook address that is not current. Publishing on, changing or deleting a channel that is not yours is also `404`. |
| `409` | The run is already subscribed to the channel, or `delay` is not one the channel offers. |
| `429` | `PLAN_LIMIT` or `NOTICE_RATE_LIMIT`, as above. |

## Related guides

- [Live execution](live.md): runs that wait, and attaching a source.
- [QTScript](qtscript.md) and [Coding Java strategies](strategy_coding.md): `onAttach` and `onDetach`.
- [Authentication and scopes](auth.md): the `signals` scope.
