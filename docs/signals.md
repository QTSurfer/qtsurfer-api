# Signal channels

A **signal channel** carries **notices** from a provider to live runs. A provider is any user, typically a service
that spots an instrument about to move. It publishes a notice saying "this pair, or this pool, deserves attention
now". What it sells is access to those notices, as **offers**: a delay, a number of runs, a duration. Each sale is a
single-use **code**. Whoever redeems a code gets a **subscription** that feeds their live runs: every notice reaches
those runs at the offer's delay, by **attaching** the instrument to them, as `POST /live/{runId}/attachments` would.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/signals/channels` | Create a channel of yours |
| `GET` | `/signals/channels` | List your channels |
| `GET` | `/signals/channels/public` | Browse public channels and their offers (no token) |
| `GET` | `/signals/channels/{channelId}` | Read a channel |
| `PATCH` | `/signals/channels/{channelId}` | Change a channel of yours, or replace its webhook address |
| `DELETE` | `/signals/channels/{channelId}` | Delete a channel of yours |
| `POST` | `/signals/channels/{channelId}/notices` | Publish a notice |
| `GET` | `/signals/channels/{channelId}/notices` | List your notices and how they were delivered |
| `POST` | `/signals/hook/{hookKey}` | Publish a notice through the channel's webhook address (no token) |
| `POST` | `/signals/channels/{channelId}/offers` | Create an offer of a channel of yours |
| `GET` | `/signals/channels/{channelId}/offers` | List its offers, with what they sold |
| `POST` | `/signals/offers/{offerId}/codes` | Mint codes for an offer, one per sale |
| `GET` | `/signals/offers/{offerId}/codes` | List the codes of an offer and their state |
| `DELETE` | `/signals/codes/{codeId}` | Revoke a code, and the subscription it gave |
| `POST` | `/signals/subscriptions` | Redeem a code |
| `GET` | `/signals/subscriptions` | List your subscriptions |
| `GET` | `/signals/subscriptions/{subscriptionId}` | Read a subscription |
| `DELETE` | `/signals/subscriptions/{subscriptionId}` | Cancel a subscription |
| `POST` | `/signals/subscriptions/{subscriptionId}/renew` | Renew a subscription with another code |
| `POST` | `/signals/subscriptions/{subscriptionId}/runs` | Feed one of your runs from a subscription |
| `DELETE` | `/signals/subscriptions/{subscriptionId}/runs/{runId}` | Stop feeding a run |
| `GET` | `/signals/subscriptions/{subscriptionId}/deliveries` | What the runs of a subscription received |

The scopes these operations need are in [Authentication and scopes](auth.md#the-scope-each-operation-requires).

## How it works

1. The provider creates a channel, and the **offers** it sells, for example 0 s for 1 run and 60 s for 3 runs, 30 days
   each. It mints **codes** for an offer, one per sale, and hands each code to whoever bought it, wherever it sells
   (the offer's `purchaseUrl` says where).
2. The buyer **redeems** the code and gets a **subscription** with the offer's terms. It starts runs, usually with no
   `sources`, so that they wait (see [Attaching and detaching a source](live.md#attaching-and-detaching-a-source)),
   and feeds up to the subscription's `maxRuns` of them from it.
3. The provider publishes a notice about an instrument.
4. When the notice is due, at its publication plus the subscription's delay, the instrument is attached to each run
   the subscription feeds. A run that was waiting starts on it; a run already reading takes it as one more source, at
   the attachment's `effectiveAtMs`. The strategy learns of it in `onAttach`, as with any attachment (see
   [QTScript](qtscript.md) and [Coding Java strategies](strategy_coding.md)).

A delivery is an ordinary attachment of the buyer's run: the buyer's plan, and the rules of the run's sources, decide
whether it fits. A notice never costs the buyer more than an attachment it made itself. Nothing is detached when a
later notice comes, nor when a subscription ends; detach an attachment yourself when you are done with it.

## Creating a channel

```json
POST /signals/channels
{
  "name": "Early pool alerts",
  "description": "Pools about to move, seconds ahead.",
  "visibility": "public"
}
```

```json
{
  "channelId": "7aQk2mVn9pXr4sTw1yZb3c",
  "name": "Early pool alerts",
  "description": "Pools about to move, seconds ahead.",
  "visibility": "public",
  "delays": [],
  "offers": [],
  "createdAtMs": 1758330000000,
  "webhookUrl": "https://api.qtsurfer.net/v1/signals/hook/7aQk2mVn9pXr4sTw1yZb3c.Jp3kQ0aZx9LmB2vT7nWc4e",
  "subscriptions": 0
}
```

| Field | Meaning |
|---|---|
| `name` | Required, 1 to 64 characters. What buyers see. Your own user is never shown to anyone. |
| `description` | Optional, up to 500 characters. |
| `visibility` | `unlisted` (the default): found only by its `channelId`. `public`: also listed in `GET /signals/channels/public`, with its offers. |

`delays` are the delays of the channel's offers, and `offers` what it sells. `webhookUrl`, `subscriptions` (how many
are active and not expired; never who holds them) and the ids and counts of the offers are shown only to you, the
owner.

`PATCH /signals/channels/{channelId}` changes any of these. `"rotateWebhook": true` replaces `webhookUrl`, and the old
address stops publishing at once. `DELETE` ends every subscription of the channel, and the notices still due are not
delivered.

## Selling: offers and codes

```json
POST /signals/channels/{channelId}/offers
{ "delay": 60, "maxRuns": 3, "durationDays": 30, "purchaseUrl": "https://shop.example/early-alerts" }
```

| Field | Meaning |
|---|---|
| `delay` | Required, `0` to `3600` seconds: how long after its publication a notice reaches the runs of a subscription of this offer. |
| `maxRuns` | Required, `1` to `100`: how many of the buyer's runs one subscription feeds at once. |
| `durationDays` | Optional, `1` to `3650`: how long a subscription lasts from the moment its code is redeemed, and what a renewal adds. Left out, it does not expire. |
| `purchaseUrl` | Optional, an `https` address of at most 500 characters: where you sell the offer. It is shown with the channel and with every subscription of the offer. |

A channel has at most 10 offers, and several may share a delay. `GET /signals/channels/{channelId}/offers` lists them
with what they sold: codes minted and redeemed, subscriptions active now, and the runs they feed.

```json
POST /signals/offers/{offerId}/codes
{ "count": 2 }
```

```json
{
  "codes": [
    { "codeId": "8cWv3nLq6rTz1yPk5mXb2a", "code": "qsc_K7pQ2mZr9xVb4nLt8wYc3dGh6jFs1aEu", "prefix": "qsc_K7pQ" },
    { "codeId": "1dXr6mNp3sQv8zWk2yTc7b", "code": "qsc_R2vT8nWq4kMz7yXp1cLb6dGf3hJs9aNe", "prefix": "qsc_R2vT" }
  ]
}
```

**This answer is the only time the codes are shown**: the platform keeps only a fingerprint of each, and `GET
/signals/offers/{offerId}/codes` lists their prefixes and states (`UNUSED`, `REDEEMED`, `REVOKED`), never who
redeemed them. Hand each code to the buyer it was sold to, and keep the others secret until then. Up to 100 codes a
call, and 10 000 an offer.

`DELETE /signals/codes/{codeId}` revokes a code, for a refund for example. If it was redeemed, **the whole
subscription it created or renewed is revoked**, including any time paid before with other codes: its runs stop
being fed and the deliveries not yet made are `SKIPPED` with `ACCESS_REVOKED`.

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
| `data` | Optional. Text for buyers to read with the notice: up to 16 entries. Keys are lower case (letters, digits, `_`, parts separated by `.`), up to 64 characters. Values are 1 to 256 characters with no control characters. It does not reach the strategy. |

`deliveries` is how many runs the notice goes to: those fed, when it is published, by a subscription that is active
and not expired. What each delivery did is the buyer's to read; `GET /signals/channels/{channelId}/notices` shows you
the counts by status.

### Through the webhook address

A sender that cannot get a token, or set a header, publishes with the same body at the channel's `webhookUrl`:

```bash
curl -X POST "https://api.qtsurfer.net/v1/signals/hook/7aQk2mVn9pXr4sTw1yZb3c.Jp3kQ0aZx9LmB2vT7nWc4e" \
  -H "Content-Type: application/json" \
  -d '{"source":{"venueType":"cex","exchange":"binance","segment":"spot","instruments":["SOL/USDT"]},"ref":"a-17"}'
```

The address is the credential: keep it secret, and replace it if it leaks. It answers as publishing with a token
does, with your plan's limits. An address that is not a channel's current one answers `404`.

## Buying: redeem, feed your runs, renew

```json
POST /signals/subscriptions
{ "code": "qsc_K7pQ2mZr9xVb4nLt8wYc3dGh6jFs1aEu" }
```

```json
{
  "subscriptionId": "9cLm3nQp5rSt7uVw1xYz2A",
  "channelId": "7aQk2mVn9pXr4sTw1yZb3c",
  "channelName": "Early pool alerts",
  "delay": 60,
  "maxRuns": 3,
  "expiresAtMs": 1760922000000,
  "state": "ACTIVE",
  "purchaseUrl": "https://shop.example/early-alerts",
  "runs": [],
  "createdAtMs": 1758330000000
}
```

A code is used once. Sending the same code again yourself answers the subscription it gave you (`200`), so a retried
request is safe; a code someone else redeemed is `409` with `reason: CODE_USED`. A code that does not exist, was
revoked, or whose channel was deleted is `404`.

```json
POST /signals/subscriptions/{subscriptionId}/runs
{ "runId": "5t5oAmQ4PD0lQRoCU58uE0", "type": "ticker" }
```

| Field | Meaning |
|---|---|
| `runId` | Required. One of your live runs. |
| `type` | Optional, `ticker` or `kline`, `kline` by default. How the run reads a centralized exchange's instrument; it must be one the run's strategy consumes. A pool is always read as `kline`. |
| `warmFrom` | Optional, `0` to `3600`, `0` by default: the warm-up of each attachment a delivery makes (see [Warming up](live.md#warming-up-warmfrom)). With `0` the strategy reads from the moment of the delivery, which is the fastest way to act on a notice. A pool delivered to a run that already reads another is read from that moment, whatever this says. |
| `resolveInstrument` | Optional, `true` by default. For a pool: whether the strategy is told of it when the pool delivers its first candle (`true`) or at once, with a stand-in instrument (`false`), as for [an attachment](live.md#the-instrument-of-a-pool). |

A subscription feeds at most its `maxRuns` runs at once (`429` with `reason: SUBSCRIPTION_RUNS_LIMIT` past it), and a
run is fed by one subscription of a channel at a time (`409` with `reason: RUN_FED_BY_ANOTHER`). `DELETE
/signals/subscriptions/{subscriptionId}/runs/{runId}` stops feeding a run; `DELETE
/signals/subscriptions/{subscriptionId}` cancels the subscription. What was already attached stays attached.

A subscription past its `expiresAtMs` is `EXPIRED`: it feeds nothing and does not count against your plan, and its
runs can be fed by another subscription of the channel. `POST /signals/subscriptions/{subscriptionId}/renew` with
another code of **the same offer** moves its expiry by the offer's duration, from its current expiry or, if it has
already expired, from now; an expired subscription resumes with the runs it fed, and counts against your plan again
(`429` with `reason: PLAN_LIMIT` if you are at its limit). A code of another offer is `409`
with `reason: OTHER_OFFER`; a revoked or cancelled subscription cannot be renewed (`SUBSCRIPTION_INACTIVE`), nor one
whose offer does not expire (`DOES_NOT_EXPIRE`).

## Reading deliveries

`GET /signals/subscriptions/{subscriptionId}/deliveries` lists every notice due to the runs the subscription feeds,
one delivery per run (`runId`), most recent first, and what each did. A delivery carries the notice's `source` and
`data` from its `dueAtMs` on; before that it says that a notice is coming and when, not what it is about, so a
subscription at a delayed level learns of the instrument no earlier than its level does.

| `status` | Meaning |
|---|---|
| `PENDING` | Not due yet, or being made. |
| `ATTACHED` | The instrument was attached to the run; `attachmentId` names the attachment. Read the run's `attachments` for how it stands. A notice about an instrument the run already has attached counts as delivered, with that attachment. |
| `REFUSED` | The run could not take it; `reason` says why (below). |
| `EXPIRED` | Not made before the notice expired. |
| `SKIPPED` | Not made: `NOT_RUNNING` (the run was not running), `UNSUBSCRIBED` (the run is no longer fed, or the subscription was cancelled), `ACCESS_REVOKED` (the provider revoked its code), `SUBSCRIPTION_EXPIRED` or `CHANNEL_DELETED`. |

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

| Plan | Channels you own | Subscriptions you hold at once | Notices per channel, per 60 seconds |
|---|---|---|---|
| free | 0 | 0 | 0 |
| basic | 1 | 2 | 6 |
| pro | 3 | 10 | 30 |
| elite | 10 | 50 | 60 |

A plan with `0` for something does not include it: the operation is refused with `402`. A subscription that expired,
was revoked or was cancelled does not count. Past the limit, creating a channel, redeeming a code or renewing an
expired subscription is refused with `429` and `reason: PLAN_LIMIT`, with no `Retry-After`. Publishing past the rate
is refused with `429` and `reason: NOTICE_RATE_LIMIT`, `retryable: true`, and `Retry-After` says when the next notice
fits. How many runs a subscription feeds is the offer's, not the plan's.

## What can refuse a request

| Status | When |
|---|---|
| `400` | A field outside its rules; a notice's `source` that is not one instrument or pool of a market that can be followed live; a run's `type` that its strategy does not consume; a `purchaseUrl` that is not `https`. |
| `402` | Your plan does not include it (see [Limits by plan](#limits-by-plan)). |
| `404` | No such channel, offer, code, run or subscription of yours (or the channel was deleted), or a webhook address that is not current. |
| `409` | `CODE_USED`, `OTHER_OFFER`, `SUBSCRIPTION_INACTIVE`, `DOES_NOT_EXPIRE`, `RUN_FED_BY_ANOTHER`; a channel with 10 offers (`OFFER_LIMIT`) or an offer with 10 000 codes (`CODE_LIMIT`). |
| `429` | `PLAN_LIMIT`, `SUBSCRIPTION_RUNS_LIMIT` or `NOTICE_RATE_LIMIT`, as above. |

## Related guides

- [Live execution](live.md): runs that wait, and attaching a source.
- [QTScript](qtscript.md) and [Coding Java strategies](strategy_coding.md): `onAttach` and `onDetach`.
- [Authentication and scopes](auth.md): the `signals` scope.
