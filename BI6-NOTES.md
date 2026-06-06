# Blue Iris 6 Compatibility Notes

> Living document. bi-mcp was built and tested against Blue Iris **5.x**
> (5.9.9.71). These are field notes from running it against a live **Blue
> Iris 6** install, recorded as issues are found. The README still lists
> BI 6 as "untested" — this file tracks what's actually been observed.

## Test environment

| | |
|---|---|
| Blue Iris version | **6.0.7.4** (`system name` NTS-IRIS) |
| Web server | port **80** |
| Cameras | 16 |
| Auth | single user with admin (read + admin-gated paths both exercised) |
| Mutations | disabled (`BI_MCP_ALLOW_MUTATIONS=0`) |
| Method | each tool invoked via the CLI (`bi-mcp-server <tool> [--key=value]`) against the live server |
| Date | 2026-06-06 |

## Status summary (read-only tools)

Legend: ✅ works · ⚠️ works but output looks changed/thin · ❌ fails on BI 6 · ⏭️ not yet verified (blocked by environment, not BI 6)

| Tool | Status | Notes |
|---|:--:|---|
| `bi_get_session` | ✅ | Returns version `6.0.7.4`, license, tzone, capabilities. |
| `bi_get_status` | ✅ | CPU/GPU/RAM, folders, profile/schedule all present. |
| `bi_get_sysconfig` | ✅ | `archive`/`schedule`/`manrecsec` returned (admin path). |
| `bi_list_cameras` | ✅ | `isOnline`, `type`, trigger/alert counts, dual-stream fields all present. |
| `bi_get_camera_config` | ✅ | Deep `camconfig` path works with admin (note: exposes stream URLs incl. embedded camera creds, as on 5.x). |
| `bi_get_camera_motion_config` | ✅ | `setmotion`/`setpost` subtrees returned (admin path). |
| `bi_get_camera_snapshot` | ✅ | `GET /image/<short>` returns JPEG; base64 payload intact. |
| `bi_list_alerts` | ✅ | Requires `--camera=<short>` (or `Index`). AI memo/zones/clip fields present. |
| `bi_list_clips` | ✅ | Path, duration, res, flags, `filetype` (e.g. `bvr H265 New`) present. |
| `bi_get_clip_info` | ✅ | Works with `--clip=` or `--path=` (alert path). |
| `bi_get_ptz_status` | ✅ | Presets 1–20, position/IR/power fields returned. |
| `bi_list_log` | ✅ | Recent log entries with level/obj/msg (admin path). |
| `bi_get_timeline` | ⚠️ | See [Finding 1](#finding-1-bi_get_timeline-returns-a-thin-response). |
| `bi_get_alert_tracks` | ❌ | See [Finding 2](#finding-2-bi_get_alert_tracks-access-denied). |
| `bi_get_reg` | ⏭️ | Needs a `<short>.reg` export on disk; not BI 6-specific. BI 6's export format itself is unverified. |
| `bi_get_actionset` | ⏭️ | Same `.reg` dependency. |
| `bi_audit_actions` | ⏭️ | Same `.reg` dependency (returned empty `cohorts` with no exports present). |
| `bi_explain_alert_chain` | ⏭️ | Reaches BI, but also needs the camera's `.reg` export to decode action rows. |

**Net: 12/14 testable read tools work unchanged on BI 6.0.7.4.** The 4 `.reg`
parser tools are blocked by a missing-export environment, not by BI 6.

## Findings

### Finding 1: `bi_get_timeline` returns a thin response

On BI 6, the shaped output is just:

```json
{ "colors": [ 8938299 ] }
```

`--raw=true` shows BI 6 returns:

```json
{ "colors": [ 8938299 ], "alerts": [], "clips": [] }
```

The tool is documented as a "24-hour activity timeline (motion/trigger/alert
buckets)". On this install `colors` carries a single entry and `alerts`/`clips`
come back empty. This may be a BI 6 response-shape change to the `timeline`
cmd, or simply low activity in the window — **needs confirmation against a
camera with known recent activity and/or a BI 5.x baseline** before deciding
whether the shaper needs updating.

- Repro: `bi-mcp-server bi_get_timeline --camera=front-door --raw=true`

### Finding 2: `bi_get_alert_tracks` — "Access denied"

The `tracks` cmd is rejected by Blue Iris itself, even with an **admin** user
and a valid alert path:

```json
{ "error": "Blue Iris cmd=tracks failed: Access denied", "kind": "bi_error" }
```

Same result with `--raw=true`. Since the account has admin and every other
admin-gated cmd works, this looks like a BI 6 change to how the `tracks` cmd is
gated or named (or it was removed). **Needs investigation** against the BI 6
JSON API docs / changelog.

- Repro: `bi-mcp-server bi_get_alert_tracks --path=@<alert>.bvr --raw=true`
- Alert used: had `zones=1`, `flags=196608` (a real AI alert with a clip).

## Open questions / next steps

- [ ] Confirm whether BI 6 changed the `timeline` cmd response shape (Finding 1).
- [ ] Confirm BI 6 gating/renaming of the `tracks` cmd (Finding 2); update the
      tool or its error mapping accordingly.
- [ ] Export a camera `.reg` from BI 6 and verify the four `.reg` parser tools
      (`bi_get_reg`, `bi_get_actionset`, `bi_audit_actions`,
      `bi_explain_alert_chain`) — BI 6's export format may differ from 5.x.
- [ ] Verify the mutating tools on BI 6 (not exercised; mutations were disabled).
