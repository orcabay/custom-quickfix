# Design: Fast Path Parser for Market Data FIX Messages (35=W)

**Date:** 2026-04-10  
**Status:** Approved  
**Scope:** quickfix fork + exchangeconnector

---

## Goal

Eliminate the FieldMap allocation overhead for received `35=W` (MarketDataSnapshotFullRefresh) messages. The application only needs tags 55, 52, 269, 270, 271 from these messages — parsing all 552+ fields into a map wastes ~35–40µs per message.

---

## Background

The existing `doParsing` function inserts every body field into `Body.FieldMap` (a `map[Tag]field`). For a 138-entry MD snapshot, that is ~552 `Body.add()` map insertions. The application then calls `GetGroup` (re-scans the map) and per-entry `GetString` (more map lookups) to extract the 3–4 fields it actually needs.

The `bodyBytes` field already exists on `Message` and is already populated by `doParsing` — it is just unexported. Adding a `RawBody()` accessor and a fast path that skips `Body.add()` gives the application raw bytes it can scan directly.

---

## Architecture

### Part 1 — quickfix fork (`message.go`)

#### 1a. Add `RawBody()` method

Add after the `String()` method (~line 583):

```go
// RawBody returns the raw bytes of the message body as received on the wire.
// Populated for inbound messages; nil for outbound messages or messages with no body.
// For 35=W messages, body FieldMap is not populated — use RawBody() for body field access.
func (m *Message) RawBody() []byte {
    return m.bodyBytes
}
```

#### 1b. Fast path in `doParsing` for `35=W`

Inserted immediately after line 228 (where MsgType is added to Header).

Behaviour:
- Checks `string(mp.parsedFieldBytes.value) == "W"`
- Runs a mirror of the normal parse loop with one difference: body fields are NOT passed to `Body.add()`
- `extractField` is still called for every field (required to advance the byte cursor and for body-length validation)
- Header fields are still added to `Header.FieldMap` (session management reads tag 8, 9, 35, 49, 56, 34, 52)
- Trailer fields are still added to `Trailer.FieldMap`
- `bodyBytes` is populated and trimmed identically to the normal path
- Body length is validated using the already-extracted `mp.msg.fields` array
- Function returns early so the normal loop is not re-executed

**Session management impact:** none. The session layer reads only Header and Trailer fields. The fast path is invisible to session state machines.

**Body length validation:** preserved. All fields are still extracted into `mp.msg.fields`; the existing length summation loop works unchanged.

---

### Part 2 — exchangeconnector (`marketDataMethods.go`)

#### 2a. `onMarketDataSnapshot`

Replace:
```go
rawMsg := msg.ToMessage().String()
isTrade := strings.Contains(rawMsg, "\x01269=2\x01") || strings.Contains(rawMsg, "|269=2|")
```

With:
```go
rawBody := msg.ToMessage().RawBody()
isTrade := bytes.Contains(rawBody, []byte("\x01269=2\x01"))
```

Pass `rawBody` downstream instead of the typed `msg`.

Get tag 55 (Symbol) from raw body using `scanTag` helper (see below) instead of `msg.Body.FieldMap.GetString(55)`, since Body FieldMap is empty in the fast path.

Header access for tag 52 (SendingTime) is unchanged — Header FieldMap is still fully populated.

#### 2b. `parseOrderBookSnapshotRaw(rawBody []byte, ...)`

Replaces `parseOrderBookSnapshot`. Scans `rawBody` byte-by-byte via `\x01` delimiters:
- Detects group entry boundaries on tag 269 (MDEntryType delimiter)
- Collects tags 270 (price) and 271 (size) per entry
- Converts directly to `float64` with `strconv.ParseFloat`
- Calls `UpsertBid` / `UpsertAsk` based on entryType value

No allocation for field maps. One pass through raw bytes.

#### 2c. `parseTradeSnapshotRaw(rawBody []byte, ...)`

Same byte-scanner pattern, collecting trade-relevant fields only.

#### 2d. `scanTag(raw []byte, tag int) ([]byte, bool)`

Locates the first occurrence of a specific tag in raw body bytes. Used to extract tag 55 (pair symbol). Returns the value bytes and a found boolean.

```
\x01<tag>=<value>\x01
```

Scans left-to-right using `bytes.IndexByte`. Returns a slice of the original buffer (no allocation).

---

## Data Flow

```
TCP bytes → doParsing
  → Header FieldMap populated (tags 8,9,35,49,56,34,52)  [unchanged]
  → Body: extractField × N (for checksum validation only)  [fast path]
  → bodyBytes = raw body slice                              [fast path]
  → Trailer FieldMap populated (tag 10)                   [unchanged]

onMarketDataSnapshot
  → Header.GetTime(52)        ← FieldMap lookup (fast)
  → scanTag(rawBody, 55)      ← byte scan (fast)
  → bytes.Contains(rawBody, 269=2) ← trade detection
  → parseOrderBookSnapshotRaw OR parseTradeSnapshotRaw
      → byte scanner: one pass, collect 269/270/271
      → UpsertBid / UpsertAsk
```

---

## Testing Strategy

1. **Existing quickfix tests** — run full test suite after adding the fast path. No regressions expected since session layer is unaffected.
2. **New unit test** — `TestFastPathRawBody`: parse a synthetic `35=W` message, assert:
   - `msg.RawBody()` is non-nil and contains expected fields
   - `msg.Body.FieldMap` is empty (fast path skipped Body.add)
   - `msg.Header.FieldMap` contains tags 8, 9, 35, 52
   - Body length validation passes
3. **Integration smoke test** — run exchangeconnector against a replay of real MD messages, assert identical orderbook state produced by raw vs. typed parsers.
4. **Benchmark** — `BenchmarkParseW` before/after, confirm ~35–40µs reduction.

---

## Success Criteria

- All existing quickfix tests pass
- `RawBody()` returns correct body bytes for `35=W` messages
- `Body.FieldMap` is empty for `35=W` messages (fast path active)
- `onMarketDataSnapshot` produces identical orderbook/trade output as before
- Benchmark shows ≥70% reduction in body processing time for `35=W`

---

## Files Changed

| File | Change |
|------|--------|
| `quickfix/message.go` | Add `RawBody()` method; add fast-path block in `doParsing` |
| `quickfix/message_test.go` | Add `TestFastPathRawBody` test |
| `exchangeconnector/wsengine/crossover/marketDataMethods.go` | Replace typed parsing with raw byte scanners |
