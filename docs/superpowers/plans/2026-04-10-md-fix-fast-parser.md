# Fast Path Parser for 35=W Messages — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Skip `Body.FieldMap` population for received `35=W` messages, exposing raw body bytes via `RawBody()` so the application can parse only the fields it needs.

**Architecture:** Two additions to `message.go`: a one-line `RawBody()` accessor (exposes the already-populated private `bodyBytes` field) and a fast-path block in `doParsing` that mirrors the normal parse loop but omits `Body.add()` for `35=W` messages. All other behaviour (header, trailer, body-length validation) is unchanged.

**Tech Stack:** Go, `github.com/quickfixgo/quickfix` (this repo), `testify/suite`

---

## File Map

| File | Change |
|------|--------|
| `message.go` | Add `RawBody()` method after `String()` (~line 583); add fast-path block in `doParsing` after line 228 |
| `message_test.go` | Add `TestRawBodyExposed` and `TestParseWFastPath` to `MessageSuite`; add `BenchmarkParseMessageW` |

---

## Task 1 — Expose `RawBody()` with a failing test first

**Files:**
- Modify: `message_test.go`
- Modify: `message.go` (~line 583)

- [ ] **Step 1: Write the failing test**

Add this method to `MessageSuite` in `message_test.go`, after `TestParseMessage`:

```go
func (s *MessageSuite) TestRawBodyExposed() {
	// Given a parsed non-W message (normal path)
	rawMsg := bytes.NewBufferString("8=FIX.4.29=10435=D34=249=TW52=20140515-19:49:56.65956=ISLD11=10021=140=154=155=TSLA60=00010101-00:00:00.00010=039")
	s.Nil(ParseMessage(s.msg, rawMsg))

	// RawBody should return the body bytes already populated by doParsing
	raw := s.msg.RawBody()
	s.NotNil(raw, "RawBody should not be nil after parsing a message with body fields")
	s.True(bytes.Contains(raw, []byte("55=TSLA")), "RawBody should contain tag 55, got: %s", string(raw))
}
```

- [ ] **Step 2: Run test to verify it fails**

```
go test -run TestMessageSuite/TestRawBodyExposed -v ./...
```

Expected: `FAIL` — `s.msg.RawBody undefined`

- [ ] **Step 3: Add `RawBody()` to `message.go`**

In `message.go`, after the closing brace of `String()` (after line 583), insert:

```go
// RawBody returns the raw bytes of the message body as received on the wire.
// Populated for inbound messages; nil for outbound messages or messages with no body.
// For 35=W messages parsed with the fast path, Body.FieldMap is empty — use RawBody() instead.
func (m *Message) RawBody() []byte {
	return m.bodyBytes
}
```

- [ ] **Step 4: Run test to verify it passes**

```
go test -run TestMessageSuite/TestRawBodyExposed -v ./...
```

Expected: `PASS`

- [ ] **Step 5: Commit**

```bash
git add message.go message_test.go
git commit -m "feat: expose RawBody() accessor for raw body bytes"
```

---

## Task 2 — Write failing test for 35=W fast path

**Files:**
- Modify: `message_test.go`

- [ ] **Step 1: Add the failing fast-path test**

Add this method to `MessageSuite` in `message_test.go`, after `TestRawBodyExposed`:

```go
func (s *MessageSuite) TestParseWFastPath() {
	// Build a valid 35=W message using normal construction so body length
	// and checksum are computed automatically.
	outMsg := NewMessage()
	outMsg.Header.SetField(tagBeginString, FIXString("FIX.4.4"))
	outMsg.Header.SetField(tagMsgType, FIXString("W"))
	outMsg.Header.SetField(Tag(49), FIXString("EXCH"))   // SenderCompID
	outMsg.Header.SetField(Tag(56), FIXString("CLIENT")) // TargetCompID
	outMsg.Header.SetField(Tag(34), FIXInt(1))           // MsgSeqNum
	outMsg.Header.SetField(Tag(52), FIXString("20260410-10:00:00")) // SendingTime
	outMsg.Body.SetField(Tag(55), FIXString("EURUSD"))   // Symbol
	outMsg.Body.SetField(Tag(268), FIXInt(2))            // NoMDEntries
	outMsg.Body.SetField(Tag(269), FIXString("0"))       // MDEntryType
	outMsg.Body.SetField(Tag(270), FIXString("1.08500")) // MDEntryPx
	outMsg.Body.SetField(Tag(271), FIXString("1000000")) // MDEntrySize

	rawBytes := bytes.NewBufferString(outMsg.String())

	inMsg := NewMessage()
	s.Nil(ParseMessage(inMsg, rawBytes))

	// Fast path: Body FieldMap must be empty — no Body.add() calls for 35=W.
	s.False(inMsg.Body.Has(Tag(55)), "Body FieldMap should be empty for 35=W (fast path); tag 55 should not be present")
	s.False(inMsg.Body.Has(Tag(268)), "Body FieldMap should be empty for 35=W (fast path); tag 268 should not be present")

	// RawBody must contain the body fields as raw bytes.
	rawBody := inMsg.RawBody()
	s.NotNil(rawBody, "RawBody should not be nil for 35=W")
	s.True(bytes.Contains(rawBody, []byte("55=EURUSD")), "RawBody should contain tag 55, got: %s", string(rawBody))
	s.True(bytes.Contains(rawBody, []byte("268=2")), "RawBody should contain tag 268, got: %s", string(rawBody))

	// Header FieldMap must still be fully populated.
	msgType, err := inMsg.MsgType()
	s.Nil(err)
	s.Equal("W", msgType)
	s.True(inMsg.Header.Has(Tag(49)), "Header should contain SenderCompID (49)")
	s.True(inMsg.Header.Has(Tag(52)), "Header should contain SendingTime (52)")

	// Trailer must still be populated.
	s.True(inMsg.Trailer.Has(tagCheckSum), "Trailer should contain CheckSum (10)")
}
```

- [ ] **Step 2: Run test to verify it fails**

```
go test -run TestMessageSuite/TestParseWFastPath -v ./...
```

Expected: `FAIL` — `Body.Has(Tag(55))` returns `true` (fast path not yet implemented; normal parse populates Body FieldMap)

---

## Task 3 — Implement the fast path in `doParsing`

**Files:**
- Modify: `message.go` (after line 228)

- [ ] **Step 1: Insert fast-path block in `doParsing`**

In `message.go`, locate this line (line 228):

```go
	mp.msg.Header.add(mp.msg.fields[mp.fieldIndex : mp.fieldIndex+1])
```

This is the third `Header.add` call, where MsgType (tag 35) is added. Immediately after it (before `// Start parsing.`), insert:

```go
	// Fast path for 35=W (MarketDataSnapshotFullRefresh): skip Body.FieldMap population.
	// extractField is still called per field for body-length validation.
	// Header and Trailer FieldMaps are populated normally.
	// Access body fields via msg.RawBody().
	if string(mp.parsedFieldBytes.value) == "W" {
		mp.fieldIndex++
		mp.trailerBytes = []byte{}
		mp.foundBody = false
		mp.foundTrailer = false
		for {
			mp.parsedFieldBytes = &mp.msg.fields[mp.fieldIndex]
			if mp.rawBytes, err = extractField(mp.parsedFieldBytes, mp.rawBytes); err != nil {
				return
			}
			switch {
			case isHeaderField(mp.parsedFieldBytes.tag, mp.transportDataDictionary):
				mp.msg.Header.add(mp.msg.fields[mp.fieldIndex : mp.fieldIndex+1])
			case isTrailerField(mp.parsedFieldBytes.tag, mp.transportDataDictionary):
				mp.msg.Trailer.add(mp.msg.fields[mp.fieldIndex : mp.fieldIndex+1])
				mp.foundTrailer = true
			default:
				// Body field: intentionally not added to Body.FieldMap.
				// trailerBytes tracks the cursor after each body field so trim below is correct.
				mp.foundBody = true
				mp.trailerBytes = mp.rawBytes
			}
			if mp.parsedFieldBytes.tag == tagCheckSum {
				break
			}
			if !mp.foundBody {
				mp.msg.bodyBytes = mp.rawBytes
			}
			mp.fieldIndex++
		}
		if mp.foundTrailer && !mp.foundBody {
			mp.trailerBytes = mp.rawBytes
			mp.msg.bodyBytes = nil
		}
		// Trim bodyBytes: subtract trailer length to get body-only slice.
		if len(mp.msg.bodyBytes) > len(mp.trailerBytes) {
			mp.msg.bodyBytes = mp.msg.bodyBytes[:len(mp.msg.bodyBytes)-len(mp.trailerBytes)]
		}
		// Validate body length using already-extracted field bytes (same as normal path).
		length := 0
		for _, field := range mp.msg.fields {
			switch field.tag {
			case tagBeginString, tagBodyLength, tagCheckSum:
			default:
				length += field.length()
			}
		}
		bodyLength, blerr := mp.msg.Header.getIntNoLock(tagBodyLength)
		if blerr != nil {
			err = parseError{OrigError: blerr.Error()}
		} else if length != bodyLength {
			err = parseError{OrigError: fmt.Sprintf("Incorrect Message Length, expected %d, got %d", bodyLength, length)}
		}
		return
	}
```

- [ ] **Step 2: Run the fast-path test**

```
go test -run TestMessageSuite/TestParseWFastPath -v ./...
```

Expected: `PASS`

- [ ] **Step 3: Run the full quickfix package test suite**

```
go test -v ./...
```

Expected: all tests pass except the pre-existing `log/mongo` failure (requires live MongoDB, unrelated to this change). The `quickfix` package itself must show `ok`.

- [ ] **Step 4: Verify `TestRawBodyExposed` still passes (non-W path unchanged)**

```
go test -run TestMessageSuite/TestRawBodyExposed -v ./...
```

Expected: `PASS`

- [ ] **Step 5: Commit**

```bash
git add message.go message_test.go
git commit -m "feat: fast path in doParsing for 35=W — skip Body.FieldMap, expose RawBody()"
```

---

## Task 4 — Add benchmark

**Files:**
- Modify: `message_test.go`

- [ ] **Step 1: Add benchmark for 35=W parsing**

Add after `BenchmarkParseMessage` (after line 35 in `message_test.go`):

```go
func BenchmarkParseMessageW(b *testing.B) {
	// Build a realistic 35=W message once, then benchmark parsing it.
	outMsg := NewMessage()
	outMsg.Header.SetField(tagBeginString, FIXString("FIX.4.4"))
	outMsg.Header.SetField(tagMsgType, FIXString("W"))
	outMsg.Header.SetField(Tag(49), FIXString("EXCH"))
	outMsg.Header.SetField(Tag(56), FIXString("CLIENT"))
	outMsg.Header.SetField(Tag(34), FIXInt(1))
	outMsg.Header.SetField(Tag(52), FIXString("20260410-10:00:00"))
	outMsg.Body.SetField(Tag(55), FIXString("EURUSD"))
	outMsg.Body.SetField(Tag(268), FIXInt(2))
	outMsg.Body.SetField(Tag(269), FIXString("0"))
	outMsg.Body.SetField(Tag(270), FIXString("1.08500"))
	outMsg.Body.SetField(Tag(271), FIXString("1000000"))
	raw := outMsg.String()

	msg := NewMessage()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		buf := bytes.NewBufferString(raw)
		_ = ParseMessage(msg, buf)
	}
}
```

- [ ] **Step 2: Run benchmark to confirm it executes**

```
go test -bench=BenchmarkParseMessageW -benchmem -count=3 ./...
```

Expected: benchmark runs and completes. Note the `ns/op` value for comparison with the non-fast-path baseline (`BenchmarkParseMessage`).

- [ ] **Step 3: Commit**

```bash
git add message_test.go
git commit -m "test: add BenchmarkParseMessageW for fast-path perf baseline"
```

---

## Verification

After all tasks complete:

```
go test -run TestMessageSuite -v -count=1 .
```

All `MessageSuite` tests must pass. Confirm in output:
- `TestRawBodyExposed` — PASS
- `TestParseWFastPath` — PASS
- All pre-existing `MessageSuite` tests — PASS (normal path unaffected)
