# Dante Route Automation

Declarative, scriptable route management for [Dante](https://www.audinate.com/)
audio networks — treat your Dante subscription map like infrastructure-as-code.

> **Note:** This is the public overview of a private project. The implementation
> (route maps, CLI, tests) lives in a private repository because it targets a
> live client deployment. Reach out if you'd like to talk shop.

## The problem

Dante Controller is a GUI with no public API. Commissioning a room means
clicking every subscription crosspoint by hand — and after a device swap,
factory reset, or firmware surprise, doing it all again from memory or
screenshots. Presets help, but they're GUI-applied, opaque XML, and don't
diff against reality.

## The approach

1. **A declarative route map (TOML)** — the desired state of every Dante
   subscription in a room: `RX channel @ device ← TX channel @ device`, with
   routes marked active or pending, and unknowns documented inline. The route
   map is code-reviewed, version-controlled, and readable by humans.

2. **A Python CLI** wrapping the open-source
   [`netaudio`](https://pypi.org/project/netaudio/) tool (which speaks Dante's
   routing protocol directly):

   | Command | What it does |
   |---|---|
   | `discover` | Enumerate devices/channels on the Dante VLAN, snapshot to JSON |
   | `plan` | Show desired routes and which are actionable |
   | `apply` | Create subscriptions — **dry-run by default**, `--execute` to act, auto-verifies after |
   | `verify` | Diff desired vs. live: `OK` / `WRONG-source` / `MISSING` |
   | `snapshot` | Timestamped backup of current subscriptions |

3. **Safety rails, learned in the field:**
   - Dry-run is the default; nothing touches the network without `--execute`.
   - A Dante Controller preset is saved before any change — netaudio is
     unofficial (reverse-engineered protocol), so the sanctioned rollback path
     stays warm.
   - Preflight detects VPN/overlay interfaces (Tailscale et al.) that hijack
     mDNS and silently break Dante discovery — a gotcha that cost us a site
     visit before we automated the check.
   - A green subscription isn't audio: the workflow ends at the DSP's meters,
     not at the routing matrix.

## Under the hood

The tool is ~350 lines of dependency-free Python 3.11 (stdlib `tomllib`,
`subprocess`, `argparse`) plus a pytest suite covering route-map parsing,
command construction, output parsing (netaudio emits JSON *or* human text
depending on state), and desired-vs-live diff logic. Subscription identity
keys on the receive side — one transmitter per RX channel — so a channel
subscribed to the *wrong* source is flagged distinctly from one that's simply
missing.

It sits on top of a broader private knowledge base covering Dante network
engineering: PTP clocking and leader election, QoS/DSCP switch configuration
(CS7 for PTP events, EF for audio, strict-priority queuing), IGMP
snooping/querier discipline, EEE pitfalls, and vendor control protocols
(Shure ASCII over TCP 2202, among others).

## Status

Bench-verified (unit tests + dry-run + discovery behavior). Live apply against
production hardware is scheduled as a supervised on-site step — this touches
real rooms, so the last mile is deliberately human-in-the-loop.

---

*AV systems integration & automation — [@jscales4000](https://github.com/jscales4000)*
