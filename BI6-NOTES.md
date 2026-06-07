# Blue Iris 6 Compatibility Notes

> Living document. bi-mcp was built and tested against Blue Iris **5.x**
> (5.9.9.71). These are field notes from running it against a live **Blue
> Iris 6** install. **Headline: no BI 6 regressions found** among the
> testable read-only tools — every tool that works on 5.x works the same
> on 6.0.7.4.

## Test environment

| | |
|---|---|
| Blue Iris version | **6.0.7.4** (`system name` NTS-IRIS) |
| Web server | port **80** |
| Cameras | 16 |
| Auth | single user with admin (read + admin-gated paths both exercised) |
| Mutations | disabled (`BI_MCP_ALLOW_MUTATIONS=0`) |
| Method | tools invoked via the CLI (`bi-mcp-server <tool>`), plus direct JSON API probes (MD5 login handshake) to isolate tool vs. server behavior |
| Date | 2026-06-06 |

## Status summary (read-only tools)

Legend: ✅ works · ⚠️ pre-existing cross-version limitation (not BI 6) · ⏭️ not verified (blocked by environment, not BI 6)

| Tool | Status | Notes |
|---|:--:|---|
| `bi_get_session` | ✅ | Returns version `6.0.7.4`, license, tzone, capabilities. |
| `bi_get_status` | ✅ | CPU/GPU/RAM, folders, profile/schedule all present. |
| `bi_get_sysconfig` | ✅ | `archive`/`schedule`/`manrecsec` returned (admin path). |
| `bi_list_cameras` | ✅ | `isOnline`, `type`, trigger/alert counts, dual-stream fields all present. |
| `bi_get_camera_config` | ✅ | Deep `camconfig` path works with admin (exposes stream URLs incl. embedded camera creds, as on 5.x). |
| `bi_get_camera_motion_config` | ✅ | `setmotion`/`setpost` subtrees returned (admin path). |
| `bi_get_camera_snapshot` | ✅ | `GET /image/<short>` returns JPEG; base64 payload intact. |
| `bi_list_alerts` | ✅ | Requires `--camera=<short>` (or `Index`). AI memo/zones/clip fields present. |
| `bi_list_clips` | ✅ | Path, duration, res, flags, `filetype` (e.g. `bvr H265 New`) present. |
| `bi_get_clip_info` | ✅ | Works with `--clip=` or `--path=` (alert path). |
| `bi_get_ptz_status` | ✅ | Presets 1–20, position/IR/power fields returned. |
| `bi_list_log` | ✅ | Recent log entries with level/obj/msg (admin path). |
| `bi_get_timeline` | ✅ | Works. See [Note 1](#note-1-bi_get_timeline-needs-a-date-range) for why the default call looks empty. |
| `bi_get_alert_tracks` | ⚠️ | Access-denied — but on **5.x and 6.x alike**. See [Note 2](#note-2-bi_get_alert_tracks-is-a-cross-version-limitation). |
| `bi_get_reg` | ⏭️ | Needs a `<short>.reg` export on disk; not BI 6-specific. BI 6's export format itself is unverified. |
| `bi_get_actionset` | ⏭️ | Same `.reg` dependency. |
| `bi_audit_actions` | ⏭️ | Same `.reg` dependency (returned empty `cohorts` with no exports present). |
| `bi_explain_alert_chain` | ⏭️ | Reaches BI, but also needs the camera's `.reg` export to decode action rows. |

**Net: 13/14 testable read tools work on BI 6.0.7.4. The 14th
(`bi_get_alert_tracks`) is denied by BI on 5.x too — not a BI 6 regression.**
The 4 `.reg` parser tools are blocked by a missing-export environment, not BI 6.

## Notes

### Note 1: `bi_get_timeline` needs a date range

Calling the tool with only a camera returns what looks like a near-empty result:

```json
{ "colors": [ 8938299 ] }
```

This is **not** a BI 6 problem. Two things combine:

1. `bi_get_timeline` forwards `startdate`/`enddate` only if you pass them, and
   sends none by default. With no range, BI 6 returns empty span arrays.
2. The `shape_timeline` shaper runs `_drop_empty`, which removes those empty
   `alerts`/`clips` arrays — leaving just `colors`.

Pass a range and BI 6 returns full span data. Direct JSON probe (`front-door`,
last 24h):

```json
{ "colors": [8938299],
  "alerts": [],
  "clips":  [ {"x1": -1692, "x2": 1909, "track": 0},
              {"x1": 19908, "x2": 86392, "track": 0} ] }
```

A busier camera (`frontyard`, last 24h) returns alert spans too:

```json
{ "alerts": [ {"x1": 6942, "x2": 7005, "type": 204800, "record": "@4631", "tracks": 1} ],
  "clips":  [ ... three spans ... ] }
```

So the BI 6 `timeline` shape is `{colors, alerts, clips}` where `alerts`/`clips`
are arrays of `{x1, x2, ...}` **spans** (not hourly buckets). Two small,
non-urgent improvement ideas (not bugs):

- Default `bi_get_timeline` to a trailing 24h window when no range is given, so
  the tool matches its "24-hour timeline" description out of the box.
- Consider not dropping empty `alerts`/`clips` in `shape_timeline`, so callers
  can tell "no activity" from "field absent."

- Repro: `bi-mcp-server bi_get_timeline --camera=front-door --startdate=<epoch-24h> --enddate=<epoch> --raw=true`

### Note 2: `bi_get_alert_tracks` is a cross-version limitation

The `tracks` cmd is rejected by Blue Iris itself:

```json
{ "result": "fail", "data": { "reason": "Access denied" } }
```

This is **already documented in the tool's own description** as "KNOWN BROKEN on
BI 5.9.9.71" — so it is **not** a BI 6 regression. BI 6.0.7.4 behaves
identically. New characterization from direct JSON probing on 6.0.7.4:

- Denied for **every** parameter shape tried: `path=@<alert>.bvr`,
  `alert=@<alert>.bvr`, `camera=<short>` + `path`, `path=@<clip>.bvr`, and the
  numeric `record` id. → the gate is **not** parameter-dependent.
- Denied even though the connected user has **admin** (`login_data.admin == true`).
  → **not** fixable by supplying admin creds.
- The data clearly exists: the `timeline` alert span above reports `"tracks": 1`
  for that alert. The `tracks` JSON cmd simply can't retrieve it over the
  authenticated `/json` POST interface.

**Conclusion:** `tracks` is gated server-side in a way the JSON API can't
satisfy on either 5.x or 6.x. There is no client-side fix that makes it work;
the realistic options upstream are to (a) keep it documented as unavailable and
return a clearer error, or (b) drop the tool until BI exposes the cmd.

- Repro: `bi-mcp-server bi_get_alert_tracks --path=@<alert>.bvr --raw=true`

## Open questions / next steps

- [ ] Export a camera `.reg` from BI 6 and verify the four `.reg` parser tools
      (`bi_get_reg`, `bi_get_actionset`, `bi_audit_actions`,
      `bi_explain_alert_chain`) — BI 6's export format may differ from 5.x.
- [ ] Verify the mutating tools on BI 6 (not exercised; mutations were disabled).
- [ ] (Optional, upstream) timeline default-range + keep-empty-arrays tweaks (Note 1).
- [ ] (Optional, upstream) clearer `bi_get_alert_tracks` error / deprecation (Note 2).
