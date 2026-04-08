# Hard-Won Integration Lessons

## Action names MUST include trigger/ or set/ prefix
Scene config actions like `"unmute_track"` produce `/ralf/act/unmute_track`. Translator parses `trigger/` or `set/` prefix and drops messages without one. Always use `"trigger/unmute_track"` or `"set/filter_cutoff"` in scene intents.

## `continuous` mode lives on the intent ENTRY, not the reading
A reading with `"continuous": true` does nothing. The intent entry needs `{ "intent": "name", "mode": "continuous" }`. Different intents in the same reading can have different modes.

## `readdir()` order is NOT alphabetical — sort scene lists explicitly
`fs.readdir()` returns filesystem order (creation order), not alphabetical. The runtime loaded `house-session` instead of `crowd-demo` because it was created first. Fixed by adding `.sort()` in `loader.ts`. Always sort when order matters.

## Wrong scene loaded = silent failure with no errors
If the runtime loads a scene with `dancer1` but the adapter sends `imu_1`, the runtime silently ignores all incoming qualities (unknown dancer ID). No errors anywhere. The Performance Console shows nothing. Always verify which scene is loaded via WS `getScene` when debugging data flow.

## Adapter's WS port is the CAPTURE port, not the runtime port
Adapter connects to mediapipe-test (:3100), NOT the runtime (:8765). The env var is `RALF_CAPTURE_PORT` (default 3100). Previously `RALF_WS_PORT` collided with the runtime's same-named var.

## Bun WebSocket receives ArrayBuffer, not strings
`ws` library sends Buffer, Bun receives as ArrayBuffer. `typeof event.data === "string"` returns false. Fix: `new TextDecoder().decode(raw)`. This silently drops 60fps of data.

## Dancer ID must match scene config
Adapter defaults to `dancer1`. Scene must also use `dancer1` (or set `RALF_DANCER_ID`). Mismatch = runtime silently ignores all qualities.

## Browser tab backgrounding kills WebSocket frames
If the camera page is backgrounded or the server restarts, the WS connection goes stale. Refresh the browser page to re-establish.

## Stale processes are the #1 debugging timesink
Old bun/node processes survive Ctrl+C. Always `lsof -i :<port>` and kill stale PIDs. Multiple adapters running = confusing behavior.

## Silent catch blocks hide everything
`catch {}` swallowed ArrayBuffer parsing errors. Always log in catch blocks during development.

## Socket.io ≠ raw WebSocket
Different wire protocols, not interoperable.

## Tone.js AutoFilter `type`
`type` is oscillator type, not filter type. Use `type: "sine"` + `filter: { type: "bandpass" }`.

## Scene data model: `gate` is a Record, not an array
`reading.gate` is `Record<string, {above?, below?}>` (keyed by quality name), NOT an array. The console editor initially built UI for `gates[]` which silently failed.

## Intents have two shapes — normalize early
`scene.intents[name]` can be `IntentOption[]` (bare array) or `IntentPoolConfig {pool, deterministic?}`. UI code should normalize to `IntentPoolConfig` on load so it always accesses `.pool` and `.deterministic`. The bare array form has no `.actions` property — it IS the actions.

## Signal strip: never rebuild innerHTML on every frame
Rebuilding a DOM region via innerHTML at 30fps destroys scroll position, text selection, and expanded panels. Split into `rebuildHTML()` (only on structure change — dancers join/leave, expand toggle) and `updateValues()` (in-place via querySelector/textContent every frame). Track structural changes with a dirty flag.

## Bun.serve() for static files is trivial and reliable
No need for a separate HTTP framework. ~15 lines serves static files with content-type mapping. Lives alongside the existing `ws` WebSocket server on a different port.

## UI terminology must match the data model
Renaming Reading→Moment, Intent→Response, Quality→Body Input caused confusion — the user couldn't map UI labels back to the JSON they were editing. Reverted to Reading/Intent/Quality. Lesson: creative labels can diverge from the data model, but for a tool aimed at technical-creative users, alignment with the underlying model reduces cognitive load.

## Scene Editor has two sections mirroring the data model
Readings section (input config: qualities, weights, thresholds, wired intent names with mode) and Intents section (output config: action pools, weights, args, deterministic toggle). Intent names in readings are clickable links to scroll to the intent card. This matches the actual JSON structure and avoids confusion about shared vs inline intent editing.

## Revert must reload from disk, not from memory snapshot
The old approach cached `savedScene` at page load and restored from it. But if `updateScene` patches had already modified the runtime's in-memory scene, the snapshot was stale. Fix: `reloadScene` WS command makes the runtime re-read the JSON file and broadcast to all clients.

## Readings dashboard: deduplicate across dancers
Runtime evaluates each reading once per dancer, producing N entries per reading in state broadcasts. Dashboard must aggregate by reading ID (max value, OR active flag) to show one row per reading.

## File contention with multi-agent edits on single files
Two agents editing `console/index.html` simultaneously causes edit failures (string matching breaks when the other agent changes the file mid-edit). For single-file work, use one agent. Teams work best when agents own different files.
