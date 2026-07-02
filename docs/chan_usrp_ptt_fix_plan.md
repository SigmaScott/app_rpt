# Plan: Fix PTT Bit Handling in chan_usrp

## Problem Statement

`chan_usrp.c` does not honor the PTT (Push-To-Talk) `keyup` bit in received USRP
packets. The field exists in the protocol header (`struct _chan_usrp_bufhdr.keyup`)
but the receive path completely ignores it. RX keying is instead determined by a
queue-presence heuristic with a countdown timer.

## Current Behavior

### Packet Header (line 92-101)

The USRP protocol defines a `keyup` field (uint32_t) in every packet header:

```c
struct _chan_usrp_bufhdr {
    char eye[4];        /* "USRP" */
    uint32_t seq;       /* sequence counter */
    uint32_t memory;    /* memory ID or zero */
    uint32_t keyup;     /* tracks PTT state <-- DEFINED BUT IGNORED ON RX */
    uint32_t talkgroup; /* trunk TG id */
    uint32_t type;      /* voice, dtmf, text */
    uint32_t mpxid;     /* future use */
    uint32_t reserved;  /* future use */
};
```

### Receive Path (usrp_xread, lines 460-521)

When a UDP packet arrives, the code:
- Validates the `"USRP"` eye catcher
- Extracts and checks `seq` via `ntohl(bufhdrp->seq)`
- Checks `bufhdrp->type` for TEXT vs VOICE routing
- **Never reads `bufhdrp->keyup`** - no extraction, no conditional logic

Voice frames are unconditionally queued into `pvt->rxq`.

### RX Key Heuristic (usrp_xwrite, lines 567-648)

Instead of using the PTT bit, the driver uses a countdown timer:

1. If `rxq` has data AND `pvt->rxkey == 0` -> signal `AST_CONTROL_RADIO_KEY`
2. Set `pvt->rxkey = MAX_RXKEY_TIME` (4 write cycles, ~80ms)
3. Each `usrp_xwrite()` call decrements `pvt->rxkey`
4. When `pvt->rxkey` reaches 1 -> signal `AST_CONTROL_RADIO_UNKEY`

This means RX is considered "keyed" whenever audio frames are present in the
queue, regardless of the sender's PTT state.

### Transmit Path (usrp_xwrite, line 661)

The TX side correctly sets `keyup`:
```c
bufhdrp->keyup = htonl(1); /* indicates key up */
```

And on unkey (`usrp_indicate`, lines 400-411), sends a header-only packet with
`keyup = 0` (zeroed via memset).

## Consequences of Current Behavior

| Scenario | Expected | Actual |
|----------|----------|--------|
| Remote sends audio with `keyup=1` | RX keyed | RX keyed (accidentally correct) |
| Remote sends audio with `keyup=0` | RX NOT keyed | RX keyed (wrong - queue has data) |
| Remote stops sending (PTT release) | Immediate unkey | ~80ms delayed unkey (countdown) |
| Remote sends header-only unkey packet | Immediate unkey | Ignored (no voice = nothing queued) |

## Proposed Fix: Hybrid Approach

### Design Principles

- **KEY immediately** when PTT transitions 0->1 (prevents clipped audio on repeater)
- **UNKEY after draining voice queue** when PTT transitions 1->0 (preserves frame ordering)
- **Keep a reduced safety timer** for lost UDP packets (prevents stuck-key)

### Rationale

- Frame ordering is the hidden constraint: queuing UNKEY before `xwrite()` drains
  remaining voice causes UNKEY-before-last-audio, which drops repeater tail audio.
- Keeping UNKEY processing in `xwrite()` after drain guarantees correct sequence.
- KEY can signal from `xwrite()` before draining `rxq` since KEY must precede voice.
- A safety timer is mandatory because a lost UDP unkey packet would permanently
  key the repeater without it.

## Implementation Plan

### Step 1: Add State Fields to `struct usrp_pvt`

Add these bit fields to the private channel structure (around line 134):

```c
unsigned int remote_rx:1;       /* last known keyup state from remote */
unsigned int rxkey_pending:1;   /* KEY control frame needs to be queued */
unsigned int rxunkey_pending:1; /* UNKEY control frame needs to be queued */
```

Keep existing `int rxkey` field, repurposed as a safety countdown.

Reduce `MAX_RXKEY_TIME` from 4 to 3 (~60ms) - enough to survive one lost packet,
short enough to not cause noticeable hang time.

### Step 2: Modify `usrp_xread()` - Extract PTT and Detect Transitions

After the sequence number validation (around line 493), add PTT bit extraction:

```c
uint32_t keyup = ntohl(bufhdrp->keyup);

/* Detect PTT transitions */
if (keyup && !pvt->remote_rx) {
    /* 0 -> 1 transition: remote keyed up */
    pvt->rxkey_pending = 1;
}
if (!keyup && pvt->remote_rx) {
    /* 1 -> 0 transition: remote unkeyed */
    pvt->rxunkey_pending = 1;
}
pvt->remote_rx = keyup ? 1 : 0;
```

Handle header-only unkey packets explicitly: when `datalen == 0` and `keyup == 0`,
process the transition without attempting to queue voice data.

Continue queuing voice data into `rxq` as before for voice frames.

### Step 3: Restructure RX Processing in `usrp_xwrite()`

Replace the current queue-presence heuristic (lines 567-648) with flag-driven logic:

1. **If `rxkey_pending` is set**:
   - Queue `AST_CONTROL_RADIO_KEY` to Asterisk
   - Clear `rxkey_pending`
   - Reset safety countdown
   - Do this BEFORE draining voice from `rxq`

2. **Drain `rxq`** and queue voice frames to Asterisk (same logic as current code)

3. **If `rxunkey_pending` is set AND `rxq` is empty**:
   - Queue `AST_CONTROL_RADIO_UNKEY` to Asterisk
   - Clear `rxunkey_pending`
   - Zero the safety countdown

4. **Safety timer fallback**:
   - If `remote_rx == 1` but no new packets arrive for 3 cycles, force UNKEY
   - Reset countdown each time a packet with `keyup=1` is received
   - If countdown expires, queue `AST_CONTROL_RADIO_UNKEY` and reset state

5. **Remove** the old "queue has data = keyed" heuristic.

### Step 4: Edge Case Handling

| Scenario | Handling |
|----------|----------|
| Lost unkey packet (UDP) | Safety timer fires after ~60ms, forces UNKEY |
| Rapid PTT chatter (squelch bounce) | Only signal KEY when `rxq` has >=1 frame (debounce) |
| First packet ever received | `remote_rx` initializes to 0 via `ast_calloc`, first `keyup=1` triggers clean transition |
| Remote sends voice with `keyup=0` | No KEY queued, voice frames still pass but no RX key indication |
| Queue overload (>25 frames) | Keep existing flush logic, also clear pending flags |
| Header-only packet with `keyup=0` | Triggers `rxunkey_pending` without touching `rxq` |

### Step 5: No TX Path Changes

The transmit path is already correct:
- `usrp_indicate()` sets `pvt->txkey` on KEY/UNKEY from Asterisk
- `usrp_xwrite()` sends `keyup = htonl(1)` when keyed
- On unkey, sends header-only packet with `keyup = 0`

No modifications needed.

## Files Modified

Only one file requires changes:

- `channels/chan_usrp.c`

## Specific Line Ranges

| Section | Lines | Change Description |
|---------|-------|--------------------|
| `#define MAX_RXKEY_TIME` | 77 | Reduce from 4 to 3 |
| `struct usrp_pvt` | ~134 | Add `remote_rx`, `rxkey_pending`, `rxunkey_pending` |
| `usrp_xread()` | ~488-516 | Add PTT extraction + transition detection |
| `usrp_xwrite()` | ~567-648 | Replace queue-heuristic with flag-driven KEY/UNKEY |

## Backward Compatibility

Remote endpoints that never set the `keyup` field (always send 0) will experience:
- No `AST_CONTROL_RADIO_KEY` ever signaled via PTT bit
- Safety timer still provides UNKEY if state becomes inconsistent
- Voice frames still queued and delivered

This is a behavior change from current code where any audio in queue = keyed.
If backward compatibility with `keyup=0` senders is required, an additional
fallback could key based on queue presence when `remote_rx` has never been set
to 1 (i.e., the remote never asserts PTT). This should be evaluated during
testing.

## Testing Criteria

1. Remote keys up (sends `keyup=1` + voice) -> Asterisk receives `RADIO_KEY` + audio
2. Remote unkeys (sends `keyup=0` header-only) -> Asterisk receives last audio then `RADIO_UNKEY`
3. Lost unkey packet -> UNKEY fires within ~60ms via safety timer
4. Remote sends audio with `keyup=0` -> No `RADIO_KEY` signaled to Asterisk
5. Rapid key/unkey -> No spurious KEY without actual voice data
6. Existing TX behavior unchanged (outbound `keyup` bit still set correctly)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Lost UDP unkey packet causes stuck key | Medium | High | Safety timer (3 cycles/~60ms) |
| Backward compat with non-PTT remotes | Low | Medium | Evaluate during testing; fallback possible |
| Frame ordering (UNKEY before last audio) | N/A | High | UNKEY only after rxq drain |
| Thread safety issues | None | N/A | xread/xwrite serialized on same channel thread |
