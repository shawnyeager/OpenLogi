# openlogi-device / openlogi-hid — the HID++ device layer

This guide covers the two-crate seam; it lives here because `openlogi-device`
owns the layer. `crates/openlogi-hid/AGENTS.md` points back here.

- The HID++ layer is split at `openlogi_device::backend::HidBackend`.
  `openlogi-device` holds everything that knows the protocol and nothing about
  a host — enumeration policy, the probe, the write layer, sessions, pairing —
  and is handed a backend. `openlogi-hid` is the backend for this machine
  (`async-hid`, the Windows composite channel, Input Monitoring, the on-disk
  probe cache) plus `host`, which supplies it to the entry points so the
  public API still reads `set_dpi(route, dpi)`. A change that makes
  `openlogi-device` depend on a host breaks CI's `wasm (portable crates)` job,
  which is the point of that job.

- `openlogi-hidpp` (lib name `hidpp`, 0BSD) is a **hard fork**, not a tracked vendor
  copy — read `crates/openlogi-hidpp/AGENTS.md` before touching that crate. Its own
  rules (protocol facts from official specs, typed wire values end to end) live there
  now, not here, to keep this file to the `openlogi-hid` side only.
- Device "kind" flows through four incompatible vocabularies (Bolt pairing register,
  feature `0x0005` `DeviceType` — defined in `openlogi-hidpp` — the assets-registry
  string, and `openlogi_core::device::DeviceKind`) — the same small integers mean
  different things in each. Never cross them by raw value; convert at the boundary.
  Live measured capabilities, or the last-good measured value retained by the cache,
  are authoritative. Do not add long-lived capability gates based on `kind`. The only
  kind-derived exception is the centralized `Capabilities::presumed_from_kind`, used
  only for a currently offline device that has never been probed.
- A hotplug event is only a wake-up hint. A fresh enumeration is the authority after
  every event or coalesced burst. If the event stream is unavailable or ends, periodic
  polling must continue to preserve liveness.
- Cache and ledger grace preserve last-good identity and capabilities through partial
  or transient probe failures. Changes to probing or reconciliation must preserve that
  behavior and cover coalesced bursts, event-stream loss, partial failure, and recovery
  in focused inventory/watcher tests.
