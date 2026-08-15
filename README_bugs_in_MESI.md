# MESI Coherence Project — All Bugs, Brief Revision Notes

## Bug 1 — Late c2c transfer never corrected memory (bus_interface.v)
**What:** In `PHASE_DATA`, an "on-time" cache-to-cache transfer (caught during
the SNOOP window) triggered a memory writeback to fix memory's stale copy.
A "late" c2c transfer (arriving one phase later) delivered the same data but
**skipped the writeback** — same protocol event, inconsistent handling.
**Fix:** Mirrored the writeback (`mem_addr/mem_cmd=BUSWB/mem_wdata/mem_req`)
into the two late-c2c branches, same as the on-time branch.

---

## Bug 2 — Dropped memory read requests (memory_controller.v)
**What:** `mem_req` is a 1-cycle pulse, but it was only checked while
`mem_fsm == MEM_IDLE`. If a request arrived while memory was busy, it was
gone the next cycle — silently lost, no retry.
**Fix:** Added `rd_pending`/`rd_addr`/`rd_cmd`, capturing the request
**unconditionally every cycle** (outside the state machine), mirroring how
`wb_pending` already protected writebacks.

---

## Bug 3 — No way to verify a memory response matches the request (bus_interface.v + memory_controller.v)
**What:** `bus_interface` accepted `mem_data_valid` blindly, with no address
tag. If a request was ever dropped/delayed, an unrelated later response
could be silently accepted as the answer to the wrong transaction.
**Fix:** Added `mem_resp_addr` output on `memory_controller`, set alongside
`mem_data_valid`. `bus_interface` now only accepts a response if
`mem_resp_addr == active_addr`.

---

## Bug 4 — Writebacks could silently overwrite each other (memory_controller.v)
**What:** `wb_pending` was a **single slot**. If a second writeback arrived
before the first one drained, it silently overwrote the first — that
writeback's dirty data was lost, no error, no trace.
**Fix:** Replaced the single slot with a proper **FIFO**
(`wb_fifo_addr`/`wb_fifo_data`/`wb_head`/`wb_tail`/`wb_count`,
depth = `WB_FIFO_DEPTH`). Added a priority rule: a pending **read always
drains before a writeback** (reads stall the processor, writebacks don't) —
writebacks only drain in the gaps, via `wb_push`/`wb_pop` conditions.

---

## Bug 5 — A busy core couldn't answer a snoop in time (mesi_controller.v)
**What:** If a core was waiting for its own bus grant (`FSM_WAIT_BUS`, etc.),
the old design deferred its snoop response until it returned to `FSM_IDLE`.
That's fine for updating MESI state, but too late for **supplying c2c data**
— the live snoop window only lasts a few cycles, so the requester ended up
getting stale memory data instead of the busy core's dirty copy.
**Fix ("parallel snooping"):** Gave `cache_array` a **second, dedicated
combinational lookup port** (`snoop_lookup_addr`/`snoop_hit`/etc.), always
driven live by `snoop_addr`. This lets a core answer a snoop **every cycle,
regardless of `fsm_state`**, since it no longer shares the lookup port with
the core's own tag check. (Realistic because this cache is flip-flop based,
like a register file — cheap to add read ports; only the write port stays
single, since duplicating writable storage is genuinely expensive.)

---

## Bug 6 — Parallel snooping then caused a write-port collision (mesi_controller.v)
**What:** Once a busy core could answer snoops live, its own pending write
(`FSM_UPDATE_CACHE`) and a live snoop-driven write could both want the
cache's single write port on the same cycle.
**Fix:** Priority mux — the **snoop write always wins** (hard deadline).
`FSM_UPDATE_CACHE` checks `snoop_wants_write`; if it loses the port, it
**stays in the same state and retries next cycle** — nothing is lost, the
FSM state itself remembers the pending write.

---

## Side fix — Repeated snoop processing (mesi_controller.v)
**What:** `snoop_valid` is a level signal held high for several cycles, but
the old FSM (`IDLE → SNOOP_PROC → IDLE`) treated it like an edge, so it
could re-process the exact same broadcast multiple times.
**Fix:** Resolved as a side effect of Bug 5's fix — removing
`FSM_SNOOP_PROC`/`FSM_SNOOP_RESUME`/`snoop_pending` entirely in favor of the
live, state-independent snoop logic eliminated the ping-pong re-entry.

---

## Net result
Went from **4/22 → 21/22** tests passing after Bugs 1–3 (the memory
request/response chain). Bugs 4–6 are correctness fixes for scenarios this
particular testbench doesn't stress hard enough to fail on today, but are
real and necessary for a design where two cores can genuinely be busy at
once. The one remaining fail is a timing-margin issue (a leftover
speculative read delaying a queued writeback past the testbench's fixed
wait window) — not a logic bug.
