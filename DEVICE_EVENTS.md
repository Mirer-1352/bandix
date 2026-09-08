# Device event payloads

The configured device event URL receives HTTP POST JSON. Existing fields
(`ts_ms`, `event`, `mac`, `ip`, `hostname`, `connection_type`) are retained.

Additional fields:

| Field | Meaning | Unknown / not applicable |
| --- | --- | --- |
| `uplink` | Bridge port observed for the device, e.g. `eth2` or `phy1-ap0` | `""` |
| `wifi_channel` | Local Wi-Fi channel | `0` |
| `wifi_frequency_mhz` | Local Wi-Fi frequency from `iw dev … info` | `0` |
| `wifi_band` | `2.4 GHz`, `5 GHz`, `6 GHz`, or `60 GHz`, determined from frequency | `""` |

When a device remains online and its known bridge port changes, a separate
`uplink_changed` event includes `previous_uplink` and the new `uplink`:

```json
{
  "ts_ms": 1788744035758,
  "event": "uplink_changed",
  "mac": "02:00:00:00:00:02",
  "ip": "192.168.1.100",
  "hostname": "example-device",
  "connection_type": "wired",
  "previous_uplink": "eth2",
  "uplink": "eth3",
  "wifi_channel": 0,
  "wifi_frequency_mhz": 0,
  "wifi_band": ""
}
```

Startup and the first known port only establish a baseline. Missing forwarding
database entries do not generate changes; the last known port is retained while
the device remains online. Going offline clears that baseline, so reconnecting
does not generate an additional port-change event. `previous_uplink` is omitted
from online/offline events.

## Limits

- Sampling currently occurs every 30 seconds. Intermediate moves can be missed;
  timestamps indicate observation time. HTTP delivery order is not guaranteed.
- This observes the router's bridge port, not an AP/BSSID association. Clients of
  external APs can appear as `wired`. Mesh changes behind an unchanged ingress
  port are invisible; a backhaul path change can also change the observed port.
- Local Wi-Fi details may be unavailable after disconnection. A channel change
  alone does not trigger an event.
- Existing IPv4-neighbor online detection is unchanged, including its treatment
  of STALE entries as online. This change does not fix stale online indicators.
- Receivers with strict schemas must allow the additional fields and event type.

## Validation

With the project's Rust/eBPF build dependencies installed:

```sh
cargo test -p bandix --bin bandix device::tests
```

Tests cover port changes, missing entries, repeated observations, disconnects,
reconnects, radio frequency parsing, and serialization of device details.
