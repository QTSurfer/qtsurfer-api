# QTScript (beta)

QTScript is a compact way to write a strategy: you keep the part that is yours — indicators, windows,
signals — and leave out the ceremony of a Java class (package, imports, class, base class, property
annotations, listener boilerplate). **Every `{ }` body is plain Java**, so everything the [Java
strategy API](strategy_coding.md) offers inside a body works unchanged. A QTScript file becomes exactly
one Java class and goes through the same server-side compilation as a Java strategy.

It is **in beta** and will gain syntax. Java remains the established route, with the full engine API;
anything QTScript cannot express is written in Java (see [what it does not do](#what-it-does-not-do)).

## Submit it like any strategy

There is no separate endpoint and no header to set: send the raw source to
[`POST /strategy`](strategy.md#compiling-a-strategy) with `Content-Type: text/plain`, exactly as for
Java. The platform tells the two apart by the text — a QTScript source starts with `strategy`, and
whitespace or comments (`//`, `/* */`) before it are ignored, so a description can sit on top of the
file. `.qtscript` is the conventional file extension; only the text is sent.

```bash
curl -X POST https://api.qtsurfer.net/v1/strategy \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: text/plain" \
  --data-binary @rsi-reversion.qtscript
```

The answer is the same as for Java: a `strategyId` and the `declaredProperties` — every `param` you
declared is listed there.

## A whole strategy

This is the complete file: no imports, no class, no listener.

```
strategy "RSI reversion"

param rsiLow  = 30  "Oversold"
param rsiHigh = 70  "Overbought"

instruments */usdt

setup:
  rsi(14) window m1 {
    if (actual < rsiLow)  emitBuy(price);
    if (actual > rsiHigh) emitSell(price);
  }
```

It declares two parameters, accepts any base quoted in USDT, registers a 14-period RSI and watches it
on a one-minute window, emitting a signal on each threshold.

## The parts of a file

| Part | What it does | Example |
|---|---|---|
| `strategy` | The first line: the name and, optionally, the data source (`ticker` by default, `kline`, `funding`) | `strategy "RSI reversion"` · `strategy Bars kline` |
| `param` | A configurable value. The type comes from the literal (`9` → `int`, `0.5` → `double`, `true` → `boolean`, `"text"` → `String`). **The name is the key** that a run's `params` and a sweep axis use | `param rsiLow = 30 "Oversold"` |
| `init { }` | Optional. Plain Java run once when the strategy is built — engine setters | `init { setPercentGain(0.5); }` |
| `instruments` | Optional. Which markets: pairs (`BASE/QUOTE`, either side may be `*`), regular expressions (`~"..."`, matched against the whole symbol), or a Java body for full control | `instruments */usdt` · `instruments btc/usdt, eth/*` |
| `uses` | Optional. Brings the [pro indicators](#the-pro-indicators-uses) into the file, so `setup:` can call them by their bare name. Several `uses` lines add up | `uses pro.*` · `uses pro.adx, pro.ichimoku` |
| `setup:` | The indicators, **one builder call per line** (the free catalogue of the builder a Java strategy uses, [listed below](#what-goes-under-setup)). The lines under `setup:` are indented, and the first line back at column 0 ends the section | `ema(12)` · `bollinger(20, 2)` |
| Windows | Where the logic goes — see below | `rsi(14) window m1 { ... }` |
| `onCommand { }` | Optional, at most one per file. Runs when the run receives a [command](live.md#commands) — see below | `onCommand { if ("flatten".equals($command)) ... }` |

### What goes under `setup:`

Each line is a call on the indicator builder that a Java strategy configures in `setupIndicators`,
[`InstrumentGroupRTIndicator`](https://www.qtsurfer.com/docs/developers/java-api/reference/latest/com/wualabs/qtsurfer/engine/indicators/helpers/group/InstrumentGroupRTIndicator.html),
written without the dot. These are the names that work:

`add` `bollinger` `bollingerBandwidth` `cci` `clamp` `diff` `distance` `distanceMa` `ema` `equal`
`gain` `greatOrEqual` `greatThan` `lessOrEqual` `lessThan` `loss` `macd` `max` `min` `mul` `notEqual`
`percentChange` `rateChange` `round` `rsi` `sma` `stochRsi` `sum`

These are the forms of each name that a QTScript line can write: a `String` is the name of an indicator, either one it reads or
the one it registers, and a `Number` is a whole number or a decimal. What each argument means is in the builder's Javadoc.

| Indicator | The forms QTScript can write |
|---|---|
| `add` | `add(String, String)` · `add(String, String, String)` |
| `bollinger` | `bollinger()` · `bollinger(int)` · `bollinger(int, Number)` · `bollinger(String, int, Number)` |
| `bollingerBandwidth` | `bollingerBandwidth(String)` |
| `cci` | `cci(int)` · `cci(String, int)` |
| `clamp` | `clamp(String, Number, Number)` · `clamp(String, String, Number, Number)` |
| `diff` | `diff(String, String)` · `diff(String, String, String)` |
| `distance` | `distance(String, String)` · `distance(String, String, String)` |
| `distanceMa` | `distanceMa(String)` · `distanceMa(String, String)` |
| `ema` | `ema(int)` · `ema(String, int)` · `ema(String, String, int)` |
| `equal` | `equal(String, String, Number)` · `equal(String, String, Number, String)` · `equal(String, String, Number, String, String)` |
| `gain` | `gain(String)` · `gain(String, String)` · `gain(String, String, boolean)` |
| `greatOrEqual` | `greatOrEqual(String, String, Number)` · `greatOrEqual(String, String, Number, String)` · `greatOrEqual(String, String, Number, String, String)` |
| `greatThan` | `greatThan(String, String, Number)` · `greatThan(String, String, Number, String)` · `greatThan(String, String, Number, String, String)` |
| `lessOrEqual` | `lessOrEqual(String, String, Number)` · `lessOrEqual(String, String, Number, String)` · `lessOrEqual(String, String, Number, String, String)` |
| `lessThan` | `lessThan(String, String, Number)` · `lessThan(String, String, Number, String)` · `lessThan(String, String, Number, String, String)` |
| `loss` | `loss(String)` · `loss(String, String)` · `loss(String, String, boolean)` |
| `macd` | `macd()` · `macd(String)` · `macd(String, int, int)` · `macd(String, String, int, int)` |
| `max` | `max(String)` · `max(String, String)` |
| `min` | `min(String)` · `min(String, String)` |
| `mul` | `mul(String, Number)` · `mul(String, String)` · `mul(String, String, Number)` · `mul(String, String, String)` |
| `notEqual` | `notEqual(String, String, Number)` · `notEqual(String, String, Number, String)` · `notEqual(String, String, Number, String, String)` |
| `percentChange` | `percentChange(String)` · `percentChange(String, String)` |
| `rateChange` | `rateChange(String)` · `rateChange(String, String, boolean)` |
| `round` | `round(String, int)` · `round(String, String, int)` |
| `rsi` | `rsi(int)` · `rsi(int, boolean)` · `rsi(String, int, boolean)` |
| `sma` | `sma(int)` · `sma(String, int)` · `sma(String, String, int)` · `sma(String, int, boolean)` |
| `stochRsi` | `stochRsi(int)` · `stochRsi(String, int)` |
| `sum` | `sum(String)` · `sum(String, String)` |

A call whose arguments match none of them is rejected with
the forms that do exist: `ema(1, 2, 3)` answers that its forms are `ema(int)`, `ema(String, int)` and `ema(String,
String, int)`. The builder's methods that take a function (`predicate`, `fun`, `decorate`, `periodCount`) and `conditional`,
which takes a function and indicators as objects, can only be written in Java. To compare an indicator with a number, use
`equal`, `notEqual`, `greatThan`, `greatOrEqual`, `lessThan` or `lessOrEqual`.

A name that is not on the list, and that no `uses` line brings in ([next section](#the-pro-indicators-uses)), is not an
indicator you can use here, and the strategy is rejected with `unknown indicator '<name>'`: `wma` and `zscore` are
among the names that answer this way, and so are `atr` and `adx` unless the file brings them in with `uses`. The same
goes for `value("name")`, which reads an indicator inside a `{ }` body and has no meaning on a `setup:` line.

### The pro indicators: `uses`

The pro indicators are not in the list above. A file brings them in with a `uses` line, and then calls them in `setup:` by
their bare name, like any other indicator.

- `uses pro.*` brings in all thirteen. `uses pro.adx, pro.ichimoku` brings in only those. Several `uses` lines add up.
  Write them with `param` and `instruments`, before `setup:`.
- Only the names that a `uses` line brings in are pro. With `uses pro.atr`, a call to `adx(14)` is still `unknown
  indicator 'adx'`, reported on the line of the call. The builder's own names (`ema`, `rsi` and the rest of the list
  above) are never pro.
- `pro` is the only set there is. `uses` is a reserved word: a file cannot have a `param` or a section called `uses`.
- Pro indicators read whole bars, so use them in a [`kline`](#candles) strategy. On a `ticker` strategy the values are not
  the bar's and mean nothing, and validating or running the strategy returns an engine notice that says so.
- A pro line is an ordinary `setup:` line: one call per line, with a window after it as usual.
- Backtests, sweeps (walk-forward included), [`validate`](strategy.md#checking-it-actually-runs) and live runs run the pro
  indicators.

| Indicator | Forms | Registers, with the default arguments | Also registers |
|---|---|---|---|
| `atr` | `atr()` · `atr(int)` · `atr(String, int)` | `atr14` | |
| `adx` | `adx()` · `adx(int)` · `adx(String, int)` | `adx14` | `adx14PlusDI` · `adx14MinusDI` |
| `superTrend` | `superTrend()` · `superTrend(int, double)` · `superTrend(String)` · `superTrend(String, int, double)` | `st10_3` | `st10_3Upper` · `st10_3Lower` |
| `ichimoku` | `ichimoku()` · `ichimoku(int, int, int)` · `ichimoku(String)` · `ichimoku(String, int, int, int)` | `ichi9_26_52`, the Tenkan line | `ichi9_26_52Kijun` · `ichi9_26_52SenkouA` · `ichi9_26_52SenkouB` · `ichi9_26_52Chikou` |
| `obv` | `obv()` · `obv(String)` | `obv` | |
| `vwap` | `vwap()` · `vwap(String)` | `vwap` | |
| `mfi` | `mfi()` · `mfi(int)` · `mfi(String, int)` | `mfi14` | |
| `cmf` | `cmf()` · `cmf(int)` · `cmf(String, int)` | `cmf20` | |
| `kurtosis` | `kurtosis(int)` · `kurtosis(String, int)` | `kurt20` (the period is required; it reads the close) | |
| `parkinson` | `parkinson(int)` · `parkinson(String, int)` · `parkinson(String, int, double)` | `park20` | |
| `garmanKlass` | `garmanKlass(int)` · `garmanKlass(String, int)` · `garmanKlass(String, int, double)` | `gk20` | |
| `rogersSatchell` | `rogersSatchell(int)` · `rogersSatchell(String, int)` · `rogersSatchell(String, int, double)` | `rs20` | |
| `yangZhang` | `yangZhang(int)` · `yangZhang(String, int)` · `yangZhang(String, int, double)` | `yz20` | |

The `double` of the four volatility estimators is the annualization factor. With other arguments the name follows the same
pattern: `atr(20)` registers `atr20`, `kurtosis(30)` registers `kurt30`, `ichimoku(7, 22, 44)` registers `ichi7_22_44`, and
`superTrend(10, 2.5)` registers `st10_2.5` (a whole multiplier loses its decimal: `superTrend(7, 3)` registers `st7_3`).
With a name as the first argument the indicator registers under that name, and the other indicators it registers take it
as their prefix: `adx("trend", 14)` registers `trend`, `trendPlusDI` and `trendMinusDI`.

Some of them register more than one indicator under one name, as `bollinger` does (see
[Which indicator a window is on](#which-indicator-a-window-is-on)). A window written on such a line attaches to the main
one, so `actual` is its value, and the body reads the others with `value($indicator + "<suffix>")`, using the suffixes of
the last column. `$indicator` is the name the window is attached to, so it also works when you gave the indicator a name
of your own. The main one of `adx` is the ADX, that of `superTrend` is the SuperTrend line, and that of `ichimoku` is the Tenkan line.

This reads the ADX with the two directional lines beside it:

```
strategy "ADX trend" kline
uses pro.adx, pro.atr

param adxLevel = 25 "ADX level"

setup:
  ema(20)
  atr(14)
  adx(14) window m1 {
    if (actual > adxLevel && value("adx14PlusDI") > value("adx14MinusDI")) emitBuy(close);
    if (actual > adxLevel && value("adx14PlusDI") < value("adx14MinusDI")) emitSell(close);
  }
```

And this one brings in all of them and uses bands and a cloud:

```
strategy "Cloud and bands" kline
uses pro.*

setup:
  superTrend(10, 3) window m5 {
    if (close > value($indicator + "Upper")) emitSell(close);
    if (close < value($indicator + "Lower")) emitBuy(close);
  }
  ichimoku() window m5 {
    if (close > value($indicator + "SenkouA") && close > value($indicator + "SenkouB")) emitBuy(close);
  }
  vwap()
  yangZhang(20)
```

A file that brings in only `atr` and then calls `adx(14)` does not compile. The line of the call is rejected with
`unknown indicator 'adx'`:

```
uses pro.atr

setup:
  adx(14)
```

A `uses` line that is not well formed is rejected, against your file like the other errors of QTScript, with one of
these messages:

- `uses needs the indicators to bring in: uses pro.* or uses pro.adx, pro.ichimoku`, when nothing follows `uses`.
- `expected pro.* or pro.<indicator>, found 'pro'`, for `uses pro`.
- `unknown set 'ta': the only set is pro`, for a set that does not exist.
- `unknown pro indicator 'zscore': the pro indicators are adx, atr, cmf, garmanKlass, ichimoku, kurtosis, mfi, obv, parkinson,
  rogersSatchell, superTrend, vwap, yangZhang`, for a member that does not exist.
- `expected pro.* or pro.<indicator> here`, for a comma with nothing after it, as in `uses pro.adx,`.
- `expected pro.* or pro.<indicator>, found 'pro.adx pro.atr'`, for two members with no comma between them.

### The pro indicators and your plan

The pro indicators come with the Pro and Elite plans (your plan is the `tier` that [`GET /account`](account.md) returns).
Registering a strategy that uses them is accepted on every plan; what a plan without them cannot do is run it.

- **`POST /strategy` says what the strategy needs.** The answer carries `requires`, a list: `["indicators.pro"]` when the
  strategy uses the pro indicators and `[]` when it does not. [`GET /strategy/{strategyId}`](strategy.md#getting-a-strategy)
  does not show it.
- **A plan that does not include them is refused with `402 Payment Required`**, without a `Retry-After` header, since waiting
  does not change the plan. That is the answer to validating the strategy, to starting a backtest with it, to starting a
  sweep with it (walk-forward included) and to starting a live run with it. The message names the first pro indicator the
  strategy asks for, the line of your file and your plan (on the Basic plan it ends `your plan: basic`). The body is the usual
  error, with `retryable` set to `false`, so a client that retries on `retryable` does not retry this one. When the platform
  has a page where you change plan, the body also carries its address as `upgradeUrl`; no other error does:

```json
{"code": 402, "message": "strategy uses pro indicators (pro.adx, QTScript line 7); your plan: free", "retryable": false, "upgradeUrl": "https://example.com/upgrade"}
```

## Windows

A window fires when its period closes, not on every tick, and wraps one indicator. A period is one of
`s1 s5 s10 s30 m1 m3 m5`, or a whole number of seconds (`window 900 { ... }`); omitted, it is `s1`.
Five ways to write one:

```
setup:
  rsi(14) window m1 { ... }      // inline, on the indicator this line registers
  rsi(33) window Oversold        // same, with the body in a named section below
  window price m5 { ... }        // inline, on an indicator by name
  window ema12 Trend             // by name, body in a named section

window Oversold m1 { ... }       // a named section, at column 0
Trend s5 { ... }                 // `window` is optional here
```

A section called `Main` that nothing references watches the primary value (`price`, or `rate` on
funding), which makes the shortest useful file:

```
strategy Simple

Main m1 {
  if (actual > prev) emitBuy(price);
}
```

### The window gives you the previous value

A window fires when its period closes, so `prev` is the value the indicator had when the **previous** window
closed and `actual` is its value now: the `close[1]` of other tools. A crossing, or anything else that compares one
close with the one before, needs nothing kept in the [store](#inside-a-body); the previous value is already in
scope.

This enters when the RSI climbs back through a level, once, on the window where it happens:

```
strategy "RSI re-entry"

param level = 30 "Level to climb back through"

setup:
  rsi(14) window m1 {
    if (prev < level && actual >= level) emitBuy(price);
  }
```

On a `kline` strategy a window on `close` hands you the previous bar's close as `prev`, which is what the true range
needs next to the bar's own `high` and `low`:

```
strategy "True range" kline

setup:
  window close m1 {
    double trueRange = Math.max(high - low, Math.max(Math.abs(high - prev), Math.abs(low - prev)));
    emitInfo("trueRange", trueRange);
  }
```

The first time a window fires there is no earlier window to compare with: `prev` is the value the indicator had when
that first window opened. A rule that needs two real closes should let the first firing pass, and an indicator with a
warm-up period, such as an RSI or an EMA, has not settled in its first windows either. Keep the store for what a
window cannot hand you: a flag, a counter, a value from several windows ago.

### Which indicator a window is on

Inside any window body, `$indicator` is the name of the indicator that window is attached to, as a
`String`: `"ema12"` for a window written on the line `ema(12)`, `"price"` on a `Main` (`"rate"` on
funding). The `$` marks a name QTScript provides; the names you declare cannot start with one. A named
section attached to several indicators sees, each time it fires, the indicator that fired.

It is also how a body reaches an indicator that the same builder call registered beside the one it is on.
`bollinger(20, 2)` registers three under one name: the middle band, `blgr20_2`, and the outer bands, that
name followed by `Upper` and `Lower`. A window written on that line attaches to the middle band, so
`actual` is the middle band's value, and `value($indicator + "Upper")` and `value($indicator + "Lower")`
read the other two:

```
setup:
  bollinger(20, 2) window m1 {
    if (price > value($indicator + "Upper")) emitSell(price);
    if (price < value($indicator + "Lower")) emitBuy(price);
  }
```

A window that names an indicator which is not registered, such as `window nosuch m1 { ... }`, is not
rejected when you register the source. Whether the name exists can depend on your `param`s (a `param
fast` used as `ema(fast)` registers a different name for each value) and on indicators registered from
Java, so it is only found when the strategy is [validated](strategy.md#checking-it-actually-runs), which
then fails on the window's line: `QTScript line 4: unknown indicator 'nosuch'`.

## Inside a body

Your Java, plus what is already in scope — nothing needs importing:

| In scope | What it is |
|---|---|
| `actual`, `prev` | the window's new value and the one it had when the previous window closed — [see above](#the-window-gives-you-the-previous-value) |
| `price` (ticker) · `price open high low close volume` (kline) · `rate` (funding) | the current values, as plain variables |
| `value("name")` | any other indicator's current value |
| `$indicator` | the name of the indicator this window is on, as a `String` — [see above](#which-indicator-a-window-is-on) |
| `store` | the per-instrument state shared by every window of that instrument |
| `emitBuy(price)`, `emitSell(price)`, `emitInfo(key, values…)`, `emitSignal(signal)` | signal emission |
| every `param` | readable by its name |

A position flag in the store, emitting on the crossing only once:

```
strategy "EMA cross"

param fast = 9  "Fast EMA"
param slow = 21 "Slow EMA"

setup:
  ema(fast)
  ema(slow)
  window price s5 {
    double f = value("ema" + fast);
    double s = value("ema" + slow);
    if (f > s && !store.is("long")) {
      store.set("long");
      emitBuy(price);
    }
    if (f < s && store.is("long")) {
      store.unset("long");
      emitSell(price);
    }
  }
```

`value("ema" + fast)` reads any registered indicator by name; `store` is the same store every window of
the instrument shares.

## Candles

`strategy … kline` gives the bar's fields as plain variables. A kline strategy never takes an
interval: the bar width is the `cadence` you [prepare the data at](backtest_execute.md#kline-you-choose-the-bar-width),
so the same file runs at any of them.

```
strategy "Range breakout" kline

param minRange = 0.5 "Minimum range, percent"

setup:
  window close m1 {
    double range = (high - low) / low * 100.0;
    if (range < minRange) return;
    if (close > open) emitBuy(close);
    else              emitSell(close);
  }
```

## Handling a command

`onCommand { }` is a special section, at most one per file, that runs when the platform delivers a
[command](live.md#commands) to a live run — `POST /live/{runId}/commands`, applied without restarting the
strategy. It has no period and no indicator: it is not a window, it runs once per command, not once per
market event, and a file has at most one of it, the same way it has at most one `init { }`.

Inside its body, `$command` is the command's text, as a `String` — nothing else a window body has (`actual`,
`$indicator`, `value(...)`, the ambient `store`) is in scope, because a command is not tied to a market tick
or, unlike a window, to one instrument already chosen for you. Every `param` is still readable and settable.

A command may carry a `properties` object of your own choosing, alongside `command` in the request body.
Read a value from it with `$command.<key>` — a `String`, `null` when the command carried no such key.
`$command.<key>` fires only when `<key>` is not itself a call, so `$command.equals(...)`,
`$command.startsWith(...)` and the rest still read as ordinary `String` methods on `$command` itself.

A window body gets its instrument's `store` handed to it; `onCommand` does not, since a command names no
instrument on its own — but `getStateStore("<symbol>")` takes one directly, so a command whose own
properties name an instrument can still reach that instrument's store:

```
strategy "Manual flatten"

onCommand {
  if ("flatten".equals($command)) {
    getStateStore($command.instrument).set("flattened");
  }
}
```

A strategy with no `onCommand { }` does not implement the engine's `CommandRequestHandler`, so a command sent
to one of its runs is rejected with `409` (see [Commands](live.md#commands)). `$command` and `$command.<key>`
are not visible outside `onCommand`'s body, the same way `$indicator` is not visible outside a window's.

Setting a `param` or writing to a `StateStore` from inside `onCommand` both take effect immediately, but
neither survives a restart: a `StateStore` is memory, gone on a restart the same as a field. Only
[`PUT /live/{runId}/params`](live.md#runtime-parameters) writes something a restarted replica actually
starts from. `getStateStore(...)` here is not about durability — it is how `onCommand` reaches the
per-instrument state a window body already reads, since a command carries no instrument of its own.

A Java strategy implements the same contract directly, through `CommandRequestHandler` — see [Coding Java
strategies](strategy_coding.md#receiving-commands).

## Sizing

A strategy says when and which way, not how much. `emitBuy(price)` and `emitSell(price)` carry a price and nothing
else, and no call puts a quantity or a notional on a signal. The size of each entry is a setting of the run,
`percentAmountToLock`: the percentage of the account's free balance that an entry locks, above 0 and up to 100.

Where you give it depends on the run: `baseConfig` for a [backtest](backtest_execute.md) or a
[sweep](backtest_sweep.md), the `paper` block for a [paper live run](live_paper.md). Left out, a backtest puts the
whole free balance into each entry (all-in) and a live run's paper account locks 10%. To see how a strategy behaves
at another size, run it again with another `percentAmountToLock`; the file does not change.

## Running it

A registered QTScript strategy is prepared, executed and swept like any other:
[`prepare`, `execute`](backtest_execute.md) and [`executeSweep`](backtest_sweep.md), with the
`strategyId` you got back. Sweep axes and `params` use the names of your `param` lines.

| `strategy …` | Prepare | Execute | Sweep |
|---|---|---|---|
| ticker (default) | yes | yes | yes |
| `kline` | yes | yes | yes |
| `funding` | yes | not yet | not yet |

See [Data sources](backtest_execute.md#data-sources) for the details, including why a `funding`
strategy can be registered and its data prepared but not yet run.

## When something is wrong

Errors are reported against **your** file, never the generated Java. Registering a source with a
mistake is a `400` whose message carries `Line N, Column M:` entries, for QTScript's own errors (an
unknown section, a bad period, a duplicate parameter, a malformed instrument pattern) and for Java
errors from inside a body.

**A `200` from `POST /strategy` means the source parsed and compiled, not that it will run.**
Registering does not set the strategy up, so what only shows when it does is found by
[`validate`](strategy.md#checking-it-actually-runs), which reports it against your file the same way. The
case to know is a window on an indicator that is not registered, `window nosuch m1 { ... }`: it
registers, and `validate` ends `failed` on the window's line, `QTScript line 4: unknown indicator
'nosuch'` (see [Which indicator a window is on](#which-indicator-a-window-is-on)). Call `validate`
before you run a strategy you have just written.

A failure while the strategy runs is recorded on the job the same way, on the line the body came
from:

```
QTScript line 6: Index 2 out of bounds for length 1
```

## The strategy id

`POST /strategy` returns the same `strategyId` for the same source. For QTScript the id comes from
the text, because indentation is part of the grammar: a byte-order mark, the style of line endings,
whitespace at the end of a line and blank lines before the first and after the last line are ignored;
anything else — a comment, the indentation, a blank line in between — gives a different id. (Java
strategies are more forgiving; see [Strategies](strategy.md#compiling-a-strategy).)

## What it does not do

By design, and each one is a reason to write the strategy in Java instead:

- **No `update()`.** Windows are the model; a strategy that has to see every tick belongs in Java.
- **No cross-instrument logic** — reading other instruments' indicators needs the full class.
- **No custom indicator classes**, no extra fields or methods beyond what the sections declare, and
  no multi-source strategies.
- **One class.** Anything that wants helper types is a Java strategy.

Switching is never a dead end: a `.qtscript` file is a Java class with the ceremony left out.

## Going further

The maintained **`qtsurfer-qtscript-strategy`** skill has the full language reference — every section,
the instrument filter in detail, reserved names, and more examples — and works with agents that read
skills:

```bash
npx skills add QTSurfer/strategy-skills --skill qtsurfer-qtscript-strategy
```

The bodies are written against the strategy API documented in the
[`qtsurfer-java-strategy`](https://github.com/QTSurfer/strategy-skills) skill and in
[Coding Java strategies](strategy_coding.md).
