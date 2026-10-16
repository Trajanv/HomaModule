# Fix server RPC silence timeouts while waiting for ACKs

## Motivation and finding

Discussion of ACK requirements and server-state retention for the Homa RFC
prompted investigation of the completed server response path. Against upstream
commit `65cecd7831e8b24ba7bcd967288bb7ce26fe8eba`, a server RPC waiting for an
ACK resets `silent_ticks` and returns before the ordinary `timeout_ticks`
check. No further client packets are needed to keep it alive.

If neither an ACK nor an explicit peer/network error arrives, the server
retains the RPC and response fragments and continues sending NEED_ACKs.
A client-process crash is not necessarily sufficient: its kernel can ACK or
return an error. A host failure or partition preventing all feedback is the
motivating case. This is demonstrated behavior, not a claim about production
incident frequency.

An initial version of [PR #117](https://github.com/PlatformLab/HomaModule/pull/117)
added an opt-in absolute retention deadline called `ack_timeout_ticks`.
The maintainer confirmed that completed server RPCs were intended to honor
the existing silence timeout. The revised implementation fixes that bug and
does not add a configuration parameter. Earlier deadline experiments remain
in branch history; the artifacts and diagram here describe the revised fix.

## Revised implementation

- [`homa_timer.c`](./homa_timer.c) identifies a fully transmitted outgoing
  server response as awaiting an ACK. Such RPCs no longer take the server-side
  silence-reset/early-return exemption and reach the existing timeout check.
- After checking timeout, ACK-waiting servers return before the generic
  RESEND-request path: they need an ACK, not incoming response data.
- [`homa_incoming.c`](./homa_incoming.c) resets an outgoing server RPC's
  silence counter when a client RESEND arrives, so response recovery is
  treated as feedback rather than mistaken for client failure.
- Application processing and incomplete local response transmission retain
  their existing exemptions.
- Timeout uses normal server abort/reaping; physical buffer release is not
  synchronous with the timer check.

This is a **silence timeout**, not an absolute response-retention deadline.
Client feedback can postpone expiry. Sending NEED_ACK does not reset silence.
An earlier ACK still ends the RPC normally. After state is discarded,
response retransmission and duplicate suppression for that RPC are no longer
available; sufficiently late duplicates may execute again.

## `timeout_ticks` versus the earlier `ack_timeout_ticks` proposal

The earlier parameter name was `ack_timeout_ticks` (not `ac_timeout_ticks`).
It was introduced in the initial proposal, not as an existing upstream
setting, and is removed from the revised implementation.

| Behavior | Existing `timeout_ticks`, with this bug fix | Earlier proposed `ack_timeout_ticks` |
|----------|--------------------------------------------|--------------------------------------|
| Purpose | Detect loss of peer feedback for eligible RPCs, including completed server responses | Impose a separate absolute limit on completed server response retention |
| Clock | Consecutive silent timer ticks for the RPC | Elapsed timer ticks after response completion is observed |
| Default | 100 ticks, nominally about 100 ms | 0, meaning the additional deadline was disabled |
| Client RESEND for the same RPC | Resets that RPC's silence counter and postpones its timeout | Does not restart the absolute retention deadline |
| Traffic for another RPC | Does not reset this RPC's silence counter | Does not restart the deadline |
| Server sends NEED_ACK | Does not reset silence | Does not restart the deadline |
| ACK received | Normal RPC cleanup | Normal RPC cleanup |
| Expiration | Normal timeout abort and response-buffer reaping | Retire the completed server RPC at the separate deadline |
| Status in revised PR | Reused; the ACK-wait path now reaches its existing timeout check | Removed; no new sysctl |

For example, with `timeout_ticks=100`, a client RESEND received after 80
silent ticks resets the same RPC's `silent_ticks` to zero. If no further
feedback arrives, it can then time out after another 100 silent ticks.
An absolute `ack_timeout_ticks=100` deadline would still expire near its
original completion-based deadline, even if recovery was actively ongoing.
These are timer-tick examples, not exact wall-clock guarantees.

The reset in [`homa_resend_pkt`](./homa_incoming.c) applies to the outgoing
server RPC matched by the packet's RPC identity. It does not treat activity
from unrelated calls as evidence that this RPC is alive. A RESEND requests
missing response data; it is not an ACK of the complete response.

The maintainer's intended behavior is the silence-based policy. Fixing the
early return implements that policy without introducing a second timeout
knob or imposing an absolute deadline on active response recovery.
`request_ack_ticks` remains separate: it controls when the server starts
requesting an ACK, not when the RPC expires or how many attempts it makes.

## Existing defaults

| Setting | Default | Purpose |
|---------|---------|---------|
| `request_ack_ticks` | 2 ticks | Delay before requesting an ACK after observed response completion |
| `timeout_ticks` | 100 ticks | Abort eligible RPCs whose silence reaches this threshold |
| `resend_ticks` | 5 ticks | Begin client/message recovery after silence |
| `resend_interval` | 5 ticks | Spacing between recovery attempts |

Ticks run nominally every millisecond; scheduling affects wall-clock timing.
For a scheduled client request with no grant, START_MSG retry uses the
existing recovery path. This fix does not change client request retry policy.
The RPC silence counter and ACK-request clock have different origins: the
live fixed trace shows 99 elapsed ACK-clock ticks when silence reaches 100.

## Potential server impact

Retained payload is approximately the sum of outstanding response lengths,
plus allocation rounding and metadata. For maximum-size 1,000,000-byte
responses, 100 / 1,000 / 10,000 RPCs represent approximately 100 MB / 1 GB /
10 GB of response payload. These are estimates, not measured capacity.

The baseline sends NEED_ACK on each timer invocation after the threshold,
without ACK-request backoff. At nominal 1 ms cadence, 1,000 stuck RPCs could
attempt roughly one million NEED_ACK transmissions per second, before
scheduling and transmit limits. The fix bounds this no-feedback behavior
using the existing silence timeout; it does not introduce retry backoff.

## Validation on H1 and H2 (2026-10-15)

Both hosts ran x86-64 Linux 6.17.8.

| Validation | H1 | H2 |
|------------|----|----|
| Kernel module build | Pass | Pass |
| Timer/incoming unit suites, IPv6 default | 156/156 | 156/156 |
| Timer/incoming unit suites, IPv4 default | 156/156 | 156/156 |

Four new regressions cover server ACK timeout, incomplete response
transmission, client feedback postponing timeout, and RESEND resetting server
silence. The in-service exemption test was strengthened. Existing suites cover
ACK handling and client recovery. Two new regressions were run against
unmodified upstream and failed as expected. Test summary markers were checked
explicitly because this harness can exit zero despite failed assertions.

AddressSanitizer stayed enabled. Unit builds used a test-only forced-include
header that includes the kernel's `linux/kconfig.h`, undefines
`CONFIG_CC_HAS_NAMED_AS` and `CONFIG_DEBUG_LIST`, and defines
`CONFIG_TRACE_PREEMPT_TOGGLE=1`, plus `-fcf-protection=branch`.
These settings adapt kernel headers to user-space mocks, not production
module source. H2 used an isolated matching ASan runtime from H1; GCC and
glibc versions matched. `git diff --check` passed.

## Live packet comparison

H1 temporarily loaded unmodified upstream, then the revised module. H2's
existing client module was unchanged. Both cases used the existing defaults
`request_ack_ticks=2` and `timeout_ticks=100`; neither module exposed an
`ack_timeout_ticks` setting.

Client OUTPUT rules blocked explicit ACKs and relevant ICMP errors only for
the test flow, simulating an unreachable client without crashing a host.
Blocking ICMP is necessary because a closed client socket can otherwise
produce an error that legitimately cleans up the server RPC.

The client utility ran ten warm-up RPCs plus one measured 100-byte RPC.
Warm-ups can be acknowledged by subsequent requests; the comparison isolates
the final RPC with no later request to piggyback its ACK.

| Final RPC evidence | Unmodified baseline | Silence-timeout fix |
|--------------------|---------------------|---------------------|
| Client / server RPC ID | 8620 / 8621 | 8642 / 8643 |
| Captured NEED_ACK packets | 794 | 98 |
| First NEED_ACK after response | 2.357 ms | 2.360 ms |
| Last saved NEED_ACK after response | 810.346 ms | 101.705 ms |
| Kernel trace | Still requesting ACK at elapsed tick 1,174 | Ordinary timeout and server abort with `-ETIMEDOUT` |

Counts are saved packets, not asserted exact transmission totals. Baseline
capture/filter counts differ. The fixed capture remained running at least a
second after client completion with no further NEED_ACKs for the expired RPC.
The unit test independently checks expiry at `silent_ticks=100`; live traces
confirm the normal timeout/abort path. Physical memory reclamation was not
measured.

Artifacts:

- [Before pcap](./test/ack_retention_evidence/before.pcap) and
  [decoded packets](./test/ack_retention_evidence/before-packets.csv).
- [After pcap](./test/ack_retention_evidence/after.pcap) and
  [decoded packets](./test/ack_retention_evidence/after-packets.csv).
- [Trace excerpts and method](./test/ack_retention_evidence/observations.txt).
- [Updated before/after diagram](./ACK_RETENTION_BEFORE_AFTER.drawio).

Published evidence is anonymized: addresses use documentation-only
`192.0.2.1` / `192.0.2.2`; link-layer identifiers, interface indices, and DATA
payload patterns are cleared; pcap times are relative to the first packet.
IPv4 checksums are recomputed. Packet order, RPC IDs, counts, and intervals
are preserved. Trace peer addresses and command paths are anonymized.

Captures use Linux cooked v2 framing. Receive offload means these are not
complete bidirectional captures; client OUTPUT counters independently verify
blocked feedback. Homa opcode is transport byte 11 (`0x17` NEED_ACK, `0x18`
ACK), and the sender RPC ID is the network-order 64-bit field at offset 20.

All test rules and H1's temporary module were removed afterward. H2's original
module remained loaded. No actual host was crashed, and no raw evidence was
published.
