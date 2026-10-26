# SIP/RTP Call Flow Analysis using Wireshark and SIPp

Hands-on analysis of SIP call signaling — both a successful call and a failed
call — captured locally with Wireshark, using the open-source SIPp tool to
generate real SIP traffic.

## Objective

Understand and document how a SIP call is established, negotiated, and torn
down at the packet level, and how a failed call looks different from a
successful one when there's no listener on the target port.

## Tools Used

- **Wireshark 4.6.9** — packet capture and protocol analysis
- **SIPp** (built from source as `sipp-rs`, the official Rust port, via
  Rust/Cargo on Windows) — SIP traffic generator (UAC/UAS roles)

## Lab Setup

- Local UAC (caller) and UAS (callee), both bound to `127.0.0.1`, communicating
  over UDP.
- Captured on the Npcap Loopback Adapter.
- Display filter `udp.port==5060 or udp.port==5061` used to isolate SIP
  signaling from unrelated local loopback traffic.

```
UAC (127.0.0.1:5061) <----UDP----> UAS (127.0.0.1:5060)
```

## Scenario 1 — Successful Call

Command used:
```
# UAS
sipp-rs -sn uas -i 127.0.0.1 -p 5060

# UAC
sipp-rs -sn uac 127.0.0.1:5060 -i 127.0.0.1 -p 5061 -m 1
```

| # | Frame | Message | Description |
|---|-------|---------|-------------|
| 1 | 898 | INVITE | Call setup request. SDP offer: G.711 PCMU, port 6000 |
| 2 | 899 | 180 Ringing | Provisional response |
| 3 | 900 | 200 OK | Call accepted. SDP answer: G.711 PCMU, port 6000 (matched) |
| 4 | 901 | ACK | Handshake complete, call established |
| 5 | 902 | BYE | Call teardown initiated |
| 6 | 903 | 200 OK | Teardown confirmed |

![Successful call flow filtered in Wireshark](screenshots/successful-call-filtered.png)

![SDP codec offer in INVITE](screenshots/sdp-codec-invite.png)

![SDP codec answer in 200 OK](screenshots/sdp-codec-200ok.png)

**Finding**: The UAC offered G.711 PCMU and the UAS accepted it exactly as
offered — a clean codec negotiation with no mismatch or renegotiation
required.

## Scenario 2 — Failed Call (no listener on target port)

Command used:
```
sipp-rs -sn uac 127.0.0.1:5062 -i 127.0.0.1 -p 5061 -m 1 -timeout 10s -timeout_error
```
(Port 5062 deliberately had no UAS listening.)

![Failed call retransmissions in Wireshark](screenshots/failed-call-filtered.png)

- Wireshark shows **5 total INVITE attempts** (1 original + 4 retransmissions)
  sent at increasing intervals, matching SIP's standard exponential backoff
  retransmission timer (RFC 3261 Timer A).
- Each INVITE triggered an **ICMP "Destination Unreachable (Port
  Unreachable)"** response, confirming at the network layer that nothing was
  bound to that port.
- SIPp logged `uac timed out after '10.005' seconds` once the configured
  global timeout was reached.

```
2026-10-26      17:35:39.470198 1793016339.470198: uac timed out after '10.005' seconds

                                 Messages  Retrans   Timeout   Unexpected-Msg
0 :      INVITE ---------->         1         4         0
...
------------------------------ Test Terminated --------------------------------
```

**Finding**: This distinguishes a *port-unreachable failure* (explicit ICMP
rejection from the OS) from a *silent timeout* (e.g. a firewall dropping
packets with no response at all) — an important distinction when
troubleshooting real VoIP call failures.

## Key Takeaways

- Practiced isolating SIP signaling from unrelated background traffic using
  Wireshark display filters.
- Verified SDP codec negotiation directly from captured packets rather than
  assuming it from configuration.
- Observed SIP's retransmission/backoff behavior on an unanswered INVITE, and
  learned to distinguish a network-layer rejection (ICMP port unreachable)
  from an application-layer timeout.
