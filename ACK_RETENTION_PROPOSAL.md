# Proposal: bound server RPC state awaiting ACKs

## Finding

The concern that a server can retain a completed RPC while waiting for its ACK
is valid in the current implementation. This was checked against upstream
`main` at commit `65cecd7831e8b24ba7bcd967288bb7ce26fe8eba` (2026-10-08).

After a server finishes transmitting a response, `homa_timer_check_rpc` sends
`NEED_ACK` after `request_ack_ticks`. It then enters the server-side
`else if (!homa_is_client(rpc->id))` branch, resets `silent_ticks`, and
returns. This happens for server RPCs in `RPC_OUTGOING` too, so they never
reach the `timeout_ticks` check or `homa_rpc_abort`. The ordinary silence
timeout therefore does **not** bound the lifetime of a completed server RPC
waiting for its ACK. The ACK handler ends that RPC when an ACK arrives.

`NEED_ACK` does not itself reset `silent_ticks` (the incoming dispatcher
resets it for DATA, GRANT, and BUSY), but that fact does not help here: the
server-side early return prevents the timeout check from being reached.
`timeout_ticks=100` is not a completed-server-ACK retention limit.

For a client request whose `START_MSG` receives no grant or other response,
the ordinary client RPC timer provides recovery. The defaults in
`homa_utils.c` are `resend_ticks=5`, `resend_interval=5`, and
`timeout_ticks=100`; timer ticks run nominally every 1 ms. Thus the client
starts retransmitting the `START_MSG` after about 5 ms, repeats it at about
5 ms intervals while silent, and aborts the RPC after about 100 ms without a
response. Actual timing can vary with timer scheduling and incoming traffic;
received packets can change the RPC's silence state. These client-side
settings are separate from the proposed server-side `ack_timeout_ticks`.

Relevant code:

- [`homa_timer.c`](./homa_timer.c): sends NEED_ACK and applies the independent
  silence-timeout logic; the server branch returns before that timeout check.
- [`homa_incoming.c`](./homa_incoming.c): resets silence for DATA, GRANT, and
  BUSY packets; handles NEED_ACK and ACK.
- [`homa_rpc.c`](./homa_rpc.c): aborting a server RPC ends it; receiving its
  ACK also ends it. Reaping frees the outgoing message fragments.
- [`homa_outgoing.c`](./homa_outgoing.c): outgoing message data remains in
  Homa's transmit page pool so it is available for retransmission.
- [`homa_utils.c`](./homa_utils.c): initializes `timeout_ticks` to 100 and
  `request_ack_ticks` to 2.

There is already a regression test that NEED_ACK does not reset the client's
silence timer (`test/unit_homa_incoming.c`,
`homa_dispatch_pkts__dont_reset_silent_ticks_on_NEED_ACK`). The timeout test
in `test/unit_homa_timer.c` covers a client RPC; neither test establishes a
cleanup deadline for the server's completed-response/ACK-wait case.

## Server-side impact

Each unACKed completed server RPC remains active and retains RPC metadata and
the outgoing response fragments in the transmit page pool. The response
payload is a first-order estimate of retained memory; page rounding and
per-RPC metadata add overhead. Homa's maximum message size is 1,000,000 bytes,
so a maximum-size response can retain about 1 MB of response data, plus
rounding and metadata, until the RPC is ended and reaped.

For `N` completed RPCs awaiting ACKs, with response lengths `L_i`, payload
retention is approximately `sum(L_i)` bytes, plus page rounding and metadata.
Assuming one maximum-size response for each affected client:

| Stuck RPCs | Response payload retained (approx.) |
|-----------:|------------------------------------:|
| 100 | 100 MB |
| 1,000 | 1 GB |
| 10,000 | 10 GB |

These are payload-only estimates, not a prediction that a host can sustain
that many such RPCs: memory pressure, buffer limits, and workload constraints
may intervene first.

Once `request_ack_ticks` elapses (default 2 timer ticks, nominally 2 ms), the
timer sends a `NEED_ACK` on every invocation while the RPC remains active.
The timer is intended to run every millisecond, giving a nominal rate of up to
about 1,000 `NEED_ACK`s/s per stuck RPC. Thus 1,000 stuck RPCs could produce
about 1 million `NEED_ACK`s/s; 10,000 could produce about 10 million/s,
before accounting for scheduling, packet drops, or transmit limits. There is
no ACK-request backoff in this path.

In the ACK-wait state, the server sends `NEED_ACK`, not `RESEND`. `RESEND` is
used when the receiver of an incomplete message is missing response data; a
live client can request retransmission from the server. Retaining the RPC and
its outgoing fragments enables that recovery.

### Failure scenario

Suppose the server has completed responses for 1,000 clients, each response
is 1 MB, and the clients become unreachable before their kernels can
acknowledge receipt (for example, a rack-level partition or power/network
failure during the transaction). The server can retain about 1 GB of response
payload, plus RPC/page metadata, and attempt roughly 1 million `NEED_ACK`s
each second. Neither `timeout_ticks` nor receipt of `NEED_ACK` bounds this
state in the current server timer path. Explicit peer/network errors or
socket shutdown can still end the RPC; the concern is loss of all such
feedback during a host failure or partition.

A client *process* crash alone is not equivalent to a client-host failure:
the kernel may still receive the response and answer `NEED_ACK`. A client-host
crash or partition can prevent the kernel ACK from reaching the server. If
the client has not received the complete response, it may send `RESEND` while
it is alive; loss of all communication leaves the server unable to distinguish
a failed client from a delayed or partitioned one.

## Semantics and design decision

The ACK lets the server free retransmission data and RPC state after the
client has received the response; retaining that state also supports duplicate
suppression. If the server expires the state without an ACK, a later retry of
the same RPC ID may arrive after its duplicate-suppression record is gone.
Re-executing the request may violate at-most-once semantics.

A finite retention timeout must therefore define the at-most-once guarantee's
horizon and the treatment of late retries. If at-most-once must hold
indefinitely, dropping the response context alone is insufficient; a durable
or longer-lived deduplication record is needed. If a finite
duplicate-suppression window is acceptable, specify its duration and make
late-retry behavior explicit. Idempotent operations can tolerate some
re-execution, but that is not a transport-level substitute for the ACK.

## Implemented change

The before/after flow is illustrated in
[ACK_RETENTION_BEFORE_AFTER.drawio](./ACK_RETENTION_BEFORE_AFTER.drawio).

The implementation adds `ack_timeout_ticks`, a separate server-side retention
deadline measured from completion of response transmission. Its default is
zero, preserving current behavior unless an operator opts into expiry. A
positive value causes the server to stop requesting ACKs and retire the RPC at
the deadline. Response buffers are queued for normal reaping, and physical
release follows that schedule. The deadline should exceed
`request_ack_ticks` if the server should first have time to request an ACK.
Expiry also ends duplicate suppression, so a late retry may be processed as a
new request after its duplicate record has expired. NEED_ACK can also prompt a
client with an incomplete response to send RESEND; expiry limits the time
available for that response recovery.

The parameter is exposed at `/proc/sys/net/homa/ack_timeout_ticks` and
documented in `man/homa.7` and `protocol.md`. Tests cover opt-in expiry and
default-disabled retention beyond `timeout_ticks`; the existing ACK-handler
test covers cleanup when the ACK arrives.

## Validation on H1 and H2 (2026-10-14)

Both hosts run Linux 6.17.8 on x86-64. Isolated copies of the changed source
were used; running Homa modules and sysctl values were not replaced or changed.

| Validation | H1 | H2 |
|------------|----|----|
| Kernel module build against running-kernel headers | Pass | Pass |
| Focused unit tests, IPv6 default | 10/10 pass | 10/10 pass |
| Focused unit tests, IPv4 default | 10/10 pass | 10/10 pass |

The focused tests cover ACK requests, enabled ACK-retention expiry,
default-disabled retention, client timeout, retransmission scheduling,
START_MSG retry without a grant, NEED_ACK not resetting client silence,
explicit and piggybacked ACK cleanup, and existing sysctl clamping behavior.
The expiry test's expected log was corrected to include the normal
`homa_rpc_end invoked` event.

The unit harness required host-specific compatibility settings: include the
kernel's `linux/kconfig.h`, undefine `CONFIG_CC_HAS_NAMED_AS` and
`CONFIG_DEBUG_LIST`, and define `CONFIG_TRACE_PREEMPT_TOGGLE=1` in a test-only
forced-include header. C compilation used `-fcf-protection=branch`; the
existing AddressSanitizer instrumentation remained enabled. These settings
adapt the kernel headers to the user-space mocks, not the production module.
H2 lacked the AddressSanitizer runtime and its configured package repository
was unavailable, so the matching runtime from H1 (same GCC and glibc versions)
was supplied only in H2's isolated test directory.

These results verify compilation and mocked RPC behavior, not live
network-partition handling, measured memory reclamation, packet rates, or
wall-clock timeout accuracy. No live module deployment or partition test was
performed during that unit-test validation.

## Live packet evidence on H1/H2 (2026-10-14)

A subsequent live experiment loaded the changed module temporarily on H1,
using H2's existing module as the client. Both cases used the changed binary:
`ack_timeout_ticks=0` reproduces the old retention policy; the second case
sets it to `3`. `request_ack_ticks` remained at its default of `2`.

**The timeout counts ticks, not NEED_ACK attempts.** With these settings,
the server requests an ACK at elapsed tick 2 and expires at elapsed tick 3,
before sending a second NEED_ACK. To allow three requests with the default
ACK-request threshold, a deadline of 5 ticks would ordinarily allow requests
at ticks 2, 3, and 4 before expiry at tick 5. A 3-tick deadline is a diagnostic
example, not a recommended production retention period.

The test simulated client-host unreachability, rather than crashing a host:
H2's client received responses and exited, while narrowly scoped OUTPUT rules
blocked explicit Homa ACKs and ICMP destination-unreachable errors for H1's
test port 29000. Blocking ICMP was important: with only ACK loss, this live
client's kernel eventually returned an error after the socket closed, and
Homa aborted the server RPC with `-ENOTCONN`. Thus retention is not literally
unconditional until ACK or shutdown: an explicit peer/network error can also
end it. A partition that prevents those errors from arriving leaves the
server without that cleanup signal.

The existing `homa_test --count 1 --length 100 rtt` utility sends 10 warm-up
RPCs plus one measured RPC. Later requests can piggyback ACKs for warm-up RPCs;
the comparison below isolates the final RPC, with no subsequent request to
piggyback its ACK.

| Evidence for final RPC | Default: timeout 0 | Enabled: timeout 3 |
|------------------------|-------------------|--------------------|
| Client / server RPC ID | 8554 / 8555 | 8576 / 8577 |
| NEED_ACK packets saved in pcap | 1,621 | 1 |
| First NEED_ACK after response packet | 2.692 ms | 2.292 ms |
| Last captured NEED_ACK after response packet | 1,652.802 ms | 2.292 ms |
| Kernel trace | Still retained until configuration switched; then expired at age 2,241 ticks | NEED_ACK at tick 2; discarded at tick 3 |

Counts are packets actually saved, not exact transmission totals. The capture
remained running for at least one second after each client invocation. In the
enabled case no further NEED_ACK was captured, and the kernel trace confirms
expiry rather than merely assuming an ACK arrived. Across the two phases,
H2's rules counted 19 blocked ACK packets and 8 blocked ICMP errors.

Artifacts:

- [Before capture](./test/ack_retention_evidence/before.pcap) and
  [decoded packets](./test/ack_retention_evidence/before-packets.csv).
- [After capture](./test/ack_retention_evidence/after.pcap) and
  [decoded packets](./test/ack_retention_evidence/after-packets.csv).
- [Trace excerpts and drop counters](./test/ack_retention_evidence/observations.txt).

Captures use Linux cooked v2 framing (`tcpdump -i any`). Homa's receive
offload consumed incoming traffic before it appeared in these captures, so
they show server transmissions, not a complete bidirectional exchange.
ACK-loss evidence comes from H2's OUTPUT-rule counters. The Homa opcode is
byte 11 of the transport header: `0x17` is NEED_ACK and `0x18` is ACK.
The sender RPC ID is the network-order 64-bit value at transport offset 20.

Published captures and CSVs are anonymized: server/client addresses use
documentation-only `192.0.2.1` and `192.0.2.2`, link-layer addresses and
interface identifiers are cleared, DATA payload patterns are cleared, and
timestamps are relative to the first packet in each capture. IPv4 header
checksums were recomputed. Packet counts, ordering, RPC IDs, and timing
intervals are preserved. Trace peer addresses and command paths in the
published excerpts are also anonymized.

All test firewall tables were removed, H1's temporary module was unloaded,
and H2's original module was left loaded. This validates continued ACK
requests versus deadline-driven retirement under simulated unreachability;
it does not measure physical buffer reclamation or reproduce an actual
host crash.
