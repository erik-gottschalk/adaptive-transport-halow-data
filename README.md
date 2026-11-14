# Measurement data: adaptive transport layer on Wi-Fi HaLow

Hardware captures behind the paper *Paying Only for the Reliability You Need: An Adaptive Transport
Layer for Energy-Constrained IoT* (E. Gottschalk, M. N. Petersen, S. Ruepp, M. S. Berger; under
review). The code that produced and analyzes them is in the code repository, where this repository
is a submodule at `examples/adaptive_transport/scripts/eval/measurement_data/`; every analysis
script reads it from there. The code repository's README maps each table, figure and number of
the paper to the folders below.

## Setup in brief

A Morse Micro `mm-mm8108-ekh05` devkit (MM8108 HaLow module) sends a 32-byte message once per
second (10 s in the deep-sleep captures) to a receiver on the internet, through a Morse Micro
HaLowLink 2 access point that injects packet loss at the network layer: independently with
probability *p*, or in runs of mean length *b* (Gilbert–Elliott). A Nordic Power Profiler Kit II
records the HaLow module's current at 10 kHz with two debug pins that mark every send and its
transport (UDP, TCP, TCP carrying a delivery report). Each capture runs one protocol (`udp`, `tcp`,
`tcp_reply`, `udp_ack`) or the adaptive controller (`adp`) at one loss level or schedule.

## Files

Per capture `pwr_<protocol>_L<budget>_p<loss>_r<replicate>`:

| File | Content |
|---|---|
| `.json` | run manifest: protocol, budget L, loss schedule, timing, the run settings pushed to the device and those it confirmed, the build where recorded, the command that ran it (`argv`) |
| `.console.log` | the device's console: every transport decision, estimator update, delivery report and send result, timestamped |
| `.server.log` | the receiver: one JSON event per received message or report |
| `.channel.log` | the loss the access point injected, over time |
| `.cycles.csv.gz` | from the current trace, per send: the sample indices of its start, of its end and of the next send, the sum of the current samples [µA] up to the next send, and the debug-pin code; the first line records the trace's sample count and D1 channel. `power_common.load_cycles` turns it into energy per send; every energy figure of the paper is computed from it |

and in some folders `flashed_info.txt` / `build.log` (the firmware build that was on the device).

| Path | Content |
|---|---|
| `captures.csv` | one row per capture: folder, protocol, loss, the firmware's build options (`build`) and the controller's settings (`L_budget` to `debt_failed_sends`, as the device confirmed them), sends, energy per message sent and delivered, delivered loss, quality checks, the raw trace's SHA-256 |
| `derived/` | the few results that read samples within a send (the packet-trace figure's windows, the deep-sleep idle floor, the mean current profile after a send), computed from the raw traces |
| `TRACES.sha256` | SHA-256 of every raw trace (`<capture>.csv`, 10 kHz current and debug pins, 55 GB for the 188 captures), available from the corresponding author on request |

## Folders

One top-level folder per experiment of `run_paper.sh` in the code repository, which makes these
captures again:

| Folder | Captures | In the paper |
|---|---|---|
| `baselines` | UDP, TCP, TCP with reply at p = 0, 5, 10, 20 %, 3 replicates | Table I, Fig. 3, Fig. 6 |
| `grid/debt` | controller with the loss debt, L = 2.5, 5, 10 %, p = 0–20 % | Fig. 3, Section IV-C/E |
| `grid/no_debt` | the same with the loss debt off | S steady overload |
| `grid/fixed_probe` | controller without the loss debt and with the fixed probe cycle, incl. L = 0 | Fig. 3 (L = 0); S steady overload (no debt below the binding cells) |
| `grid/binding_cell` | L = 5 %, p = 20 %, failed sends charged to the debt | Fig. 3, tables |
| `step` | step schedule, L = 5 % | Fig. 4 |
| `mixed` | three message classes, p = 15 %, 12 × 2 h | Fig. 5 |
| `bursty/baselines`, `bursty/controller` | bursty, changing loss over 4 h: UDP and TCP; the controller | Fig. 7, Table II |
| `bursty/binding_cell`, `bursty/binding_cell_tcp` | L = 5 %, p = 20 % with runs of 3: the controller; TCP | Section IV-F |
| `udp_ack` | acknowledged UDP (CoAP-style retransmission) | Table I, Section IV-E |
| `deep_sleep/power_save`, `deep_sleep/wnm_sleep` | 10 s interval, 802.11 power save and WNM sleep | S deep sleep |

## Firmware and settings

Every capture was made with the firmware released with the paper (`examples/adaptive_transport/`
in the code repository). `captures.csv` lists, per capture, the build options (`build`, empty for
the default build) and the controller's settings, as the device confirmed them at boot.

The captures span two months, during which the firmware gained settings (the loss debt, the random
probe, charging failed sends to the debt) and larger lwIP buffer pools. A setting added after a
capture is listed as off for it, which is how that capture ran; the larger pools change nothing,
since the smaller ones never ran out in any published capture. For the release, code that no
published capture ran (options that were off or compiled out) was removed. We checked the released
firmware against each earlier version: given the same messages, its controller makes the same
decisions, and the code that ran is otherwise the same. The acknowledged-UDP captures ran a build
with one more option (`DEMO_RELIABLE_UDP_ACK`), since removed, which only changed what the
controller sends as its reliable message; these captures do not run the controller, and their send
path is the same as in the released firmware.

The manifests also record the commit of the development repository at capture time, which is not
public, and in `argv` the folder the capture was first written to; the folders were renamed for
publication.

## Analysis choices

- **Failed connections:** a TCP message whose connection never opened leaves no send in the
  current trace; it is counted as sent and lost from the device's console.
- **The steady-state grid's cell L = 5 %, p = 20 %** comes from `grid/binding_cell`, in the
  figure and the tables alike.

## Timestamps and anonymization

Timestamps are local time of the capture (`tz` in the manifest) unless marked UTC. Public IP
addresses are replaced by addresses from the documentation ranges (the receiver's is `192.0.2.1`),
and the Wi-Fi network's name and passphrase that the firmware logs at boot by `REDACTED-…`.
