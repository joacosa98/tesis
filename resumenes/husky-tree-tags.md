# robotito-huskylens Branch Analysis

## Overview

This analysis compares the current branch tip `husky-tree-tags` at `1965e20` against the requested base commit `9e39a87c55ada0756da4d2877abb8b2f257b58da` (`add test`).

Facts from the repository:

- The exact comparison range reviewed is `9e39a87c55ada0756da4d2877abb8b2f257b58da..husky-tree-tags`.
- No local or advertised remote branch named `robotito-huskylens` was available in this checkout when checked.
- The analyzed branch adds 65 commits over the requested base.
- The diff adds 4 new files under `source/bt/`, adds `source/tests/test_PID.lua`, rewrites `source/tests/test_ble.lua`, removes `source/tests/test_facu.lua` and `source/tests/test_husky.lua`, and changes `update.sh`.

The branch changes the firmware from a set of mostly standalone HuskyLens and movement experiments into a more complete Robotito game runtime. The new runtime is centered on `source/bt/huskylens.lua`, which uses a small behavior tree library to coordinate line following, AprilTag command handling, checkpoint validation, LED feedback, Bluetooth sequence configuration, and fault recovery.

Inference:

- The purpose of the branch is to turn HuskyLens line and tag detection into a deployable autonomous route game: the robot follows a line, reacts to command tags, validates checkpoint tags against configured sequences, and exposes Bluetooth configuration for route/checkpoint data.

## Main Changes

### Behavior tree runtime

The branch adds `source/bt/bt.lua`, a compact behavior tree implementation with these node types:

- `bt.sequence(children)`: runs children in order until one fails or runs.
- `bt.selector(children)`: runs children in order until one succeeds or runs.
- `bt.action(fn)`: wraps stateful Lua actions.
- `bt.condition(fn)`: turns a predicate into `SUCCESS` or `FAILURE`.
- `bt.repeater(child, repeat_count)`: repeats a child a fixed number of times, or indefinitely when count is omitted.
- `bt.interrupt(condition_fn, child)`: defines an interrupt predicate and handler.
- `bt.interruptible(child, interrupt_list, reset_on_interrupt)`: wraps a subtree and checks interrupts before and after normal execution.
- `bt.create_tree(root_node)`: runs the root node with a shared context and performs garbage collection every 500 ticks.

This module is used by `source/bt/huskylens.lua` to model the robot flow as reusable actions, conditions, selectors, sequences, repeaters, and a fault interrupt.

### Huskylens game controller

The branch adds `source/bt/huskylens.lua`, which is the main runtime script. It requires:

- `bt`
- `huskylens`
- `omni`
- `led_ring`
- `proximity`
- `robotito_ble`

It introduces centralized configuration for camera size, HuskyLens algorithms, tag IDs, timing values, line-following thresholds, LED behavior, and motion speeds.

Important runtime state is stored in `run_context`, including:

- Current drive state: `drive_direction`.
- Current HuskyLens algorithm cache: `husky_algo`.
- Last valid line arrow: `last_arrow`.
- Checkpoint results: `checkpoints_feedback` and `checkpoints_read`.
- Fault latches: `checkpoint_scan_error`, `proximity_lost`, and `arrow_fault`.
- Behavior gates: `route_loop_skip`, `waiting_for_start_skip`, and `config_mode_done`.
- Bluetooth-provided valid route sequences: `valid_sequences`.
- Mission state: `is_running`.

### Tag and checkpoint handling

The runtime uses AprilTag/block recognition mode for command and checkpoint detection. The configured command tags are:

- `1`: RESUME
- `2`: STOP
- `3`: END
- `4`: RESET
- `5`: EXIT_CONFIG
- `6`: CONFIG
- `0`: invalid tag

The code treats these IDs as operator/system tags through `operator_tag_set`. During checkpoint scanning, `scan_checkpoint_action` ignores operator tags and accepts only non-operator tag IDs as checkpoint cards.

Checkpoint validation is incremental. When a non-operator checkpoint tag is found, the code checks whether its ID matches the same position in any configured sequence:

- If at least one configured sequence has that tag at the current checkpoint position, the checkpoint is marked `LedColor.CORRECT`.
- Otherwise it is marked `LedColor.WRONG`.
- The result is appended to `ctx.checkpoints_feedback`.
- The read tag ID is appended to `ctx.checkpoints_read`.

### Line following

The controller switches the HuskyLens into line-tracking mode and reads arrow detections. Important functions:

- `read_line_frame(ctx)` switches to algorithm `3`, reads frames with retries, and filters detections to `COMMAND_RETURN_ARROW` (`43`).
- `line_arrow_skip_reason(detection, prev_arrow)` rejects arrows that are too short or jump too far in angle from the previous accepted arrow.
- `arrow_detection_to_drive_command(detection)` combines line position error and heading error into discrete drive commands.
- `drive_omni(ctx, command)` translates drive commands into `omni.drive(vx, vy, vz)` values, avoiding duplicate hardware calls when the command has not changed.
- `line_follow_step(ctx, frame, beh)` applies the line-following logic and raises `ctx.arrow_fault` after too many rejected or missing arrow frames.

The added drive commands include forward, left/right, semi-hard turns, hard turns, mega-hard turns, ultra-hard turns, and stopped. Motion parameters live under `config.motion`.

### Bluetooth configuration

The production runtime uses `robotito_ble` directly inside `source/bt/huskylens.lua`:

- `init_ble()` initializes BLE as `robotito`, initializes the mapper, and registers connect, disconnect, and mapper sequence callbacks.
- `on_mapper_seq(bytes)` parses a sequence payload, writes it through `rble.mapper_set_sequences(seqs)`, gives LED feedback, and reloads the active sequences.
- `load_sequences()` reads `rble.mapper_get_sequences()` into `run_context.valid_sequences`.
- `deinit_ble()` shuts BLE down after the EXIT_CONFIG tag is detected.

The added `source/bt/bluetooth.lua` is not required by `huskylens.lua`. It is packaged by `update.sh`, but its contents are a long-running BLE mapper test/helper script: it initializes HuskyLens tag recognition, sets an initial mapper, starts `robotito_ble`, monitors mapper changes, and prints instructions for testing with `bluetoothctl`.

### Deployment/update script

`update.sh` now defines `BT_DIR="source/bt"` and deploys:

- `bt.lua`
- `huskylens.lua`
- `bluetooth.lua`

At the same time, the file list is narrowed. Many older test and AHSM state files that were previously copied/uploaded by `update.sh` are removed from the update list. The script still copies selected files into `fs/` and uploads newer files to the board using `wcc -p /dev/ttyUSB0 -up`.

### Test and experiment files

The branch removes:

- `source/tests/test_facu.lua`: a simple omni movement test.
- `source/tests/test_husky.lua`: a continuous HuskyLens print/debug test.

The branch adds:

- `source/tests/test_PID.lua`: a standalone HuskyLens line-following experiment using adaptive steering and anti-oscillation thresholds.

The branch rewrites:

- `source/tests/test_ble.lua`: from a simple BLE echo/send loop into a mapper configuration test that initializes `robotito_ble`, parses mapper pair and sequence payloads, registers mapper callbacks, and runs a 600-second heartbeat loop.

## Implementation

### Main behavior tree structure

`source/bt/huskylens.lua` builds the runtime tree in `build_tree()`:

```text
root interruptible
└── root sequence
    ├── config mode gate selector
    ├── waiting-for-start gate selector
    └── route loop gate selector
```

The root sequence is wrapped in `bt.interruptible(...)` with `fault_recovery_interrupt`. The wrapper is created with `reset_on_interrupt = false`, so the interrupt handler can choose targeted resets rather than always resetting the full root sequence.

The runtime entry point is `run_forever()`:

1. `init_hardware()` enables omni drive, initializes HuskyLens, and registers a proximity callback.
2. `load_sequences()` loads configured Bluetooth mapper sequences into `run_context.valid_sequences`.
3. `build_tree()` constructs the behavior tree.
4. The tree is bound to `run_context`.
5. `tree:run()` is called forever.

### HuskyLens mode switching and frame filtering

The code caches the active HuskyLens algorithm in `ctx.husky_algo`.

- `ensure_tag_mode(ctx)` switches to tag recognition algorithm `5`.
- `ensure_line_mode(ctx)` switches to line tracking algorithm `3`.
- `set_husky_algo(ctx, algo_id)` no-ops when the requested algorithm is already active.

Frames are normalized through `get_frame()`, which protects `huskylens.request()` with `pcall`, checks `huskylens.available()`, and returns `{ ok, count, payload }`.

Two separate read paths then filter the normalized frame:

- Tag reads keep only block detections with command `42`.
- Line reads keep only arrow detections with command `43`.

This separation matters because the same physical HuskyLens device is reused for both navigation and command/checkpoint detection.

### Line-following implementation

`line_follow_repeater` runs `line_follow_action` `config.behavior.line_follow_ticks_per_cycle` times per route cycle. The default is `5`.

Each line-following action:

1. Reads a line frame with `read_line_frame(ctx)`.
2. Rejects invalid frames or non-arrow detections.
3. Rejects arrows shorter than `config.line.arrow_filter.min_length`.
4. Rejects arrows whose angle changes by more than `config.line.arrow_filter.max_angle_vs_previous_deg` from the previous accepted arrow.
5. Computes a blended error from the arrow tip position and arrow heading.
6. Maps that error to one of the discrete `Drive` commands.
7. Sends a motion command through `omni.drive`.

If line data is missing or rejected repeatedly, `ctx.bad_arrow_frames` increments. Once it reaches `config.behavior.arrow_rejected_frame_limit`, the code sets `ctx.arrow_fault = true`, which can trigger fault recovery while the mission is running.

### STOP checkpoint implementation

After the line-following phase, `tag_detection_repeater` runs `tag_detection_sequence` twice. It captures a tag snapshot with `tag_snapshot_action`, then uses a selector:

1. `stop_checkpoint_sequence`
2. `end_route_sequence`
3. `no_stop_or_end_tag_action`

When a STOP tag is present:

1. `scan_checkpoint_action` stops the robot and rotates to search for a non-operator checkpoint tag.
2. The detected checkpoint ID is checked against `ctx.valid_sequences` at the current checkpoint position.
3. The checkpoint result is stored and shown through the LED ring.
4. `checkpoint_led_action` runs a white chase pattern and then holds green or red.
5. `align_stop_action` rotates until the STOP tag's x position passes `config.behavior.stop_align_min_x`.
6. `resume_line_action` returns to line mode and runs line-settling for `post_checkpoint_line_settle_ticks`.

If scanning or alignment exceeds `config.timing.checkpoints_scan_limit_sec`, `ctx.checkpoint_scan_error` is set and the action fails.

### END route implementation

When an END tag is present:

1. `finish_route_action` waits briefly, stops the robot, and evaluates all accumulated checkpoint feedback.
2. If no checkpoint was marked wrong, the route passes: the ring turns green and the robot rotates for `all_ok_hold_ms`.
3. If any checkpoint was wrong, the route fails: accumulated checkpoint segments blink against an off ring.
4. `ctx.is_running` is set to `false`.
5. `wait_reset_action` waits for the RESET tag while proximity contact is true, then calls `reset_run(ctx)`.

The pass/fail decision is based on the stored per-checkpoint colors, not on comparing the full read sequence at END time.

### Fault recovery implementation

The fault interrupt is active only when:

```lua
(ctx.proximity_lost == true or ctx.checkpoint_scan_error or ctx.arrow_fault == true)
    and ctx.is_running == true
```

When active, `fault_recovery_action`:

1. Stops the robot.
2. Blinks the LED ring yellow.
3. Waits until proximity contact is restored.
4. Reads tag frames.
5. If RESUME is shown, it restores route LED segments, stops motion, resets the route loop sequence if available, and clears fault flags.
6. If RESET is shown, it calls `reset_run(ctx)`, resets the root sequence if available, and clears fault flags.

The proximity callback sets `run_context.proximity_contact` to the current input value and latches `run_context.proximity_lost = true` whenever contact is lost.

## Execution Flow

### Boot/config/start flow

1. `huskylens.lua` runs immediately through `run_forever()`.
2. Hardware initializes: omni drive is enabled, HuskyLens is initialized, and proximity callback is registered.
3. Bluetooth sequences are loaded into `run_context.valid_sequences`.
4. The behavior tree starts ticking.
5. Because `run_context.config_mode_done` defaults to `false`, the first root pass enters `config_mode_action`.
6. Config mode initializes BLE, blinks the ring blue, and waits for EXIT_CONFIG tag `5`.
7. After EXIT_CONFIG, BLE is deinitialized and `config_mode_done` becomes `true`.
8. The waiting-for-start action blinks the ring white.
9. RESUME tag `1` enables proximity, clears `route_loop_skip`, sets `waiting_for_start_skip`, plays the welcome LED pattern, and starts the route loop.

### Normal route flow

1. `start_game_action` marks `ctx.is_running = true`.
2. The robot follows the line for a fixed number of ticks.
3. The robot switches to tag mode and checks for STOP or END tags.
4. If neither tag is found, the route cycle succeeds and repeats.
5. If STOP is found, the checkpoint scan/feedback/realignment flow runs.
6. If END is found, route validation and reset-wait flow runs.

### Configuration flow after startup

The waiting-for-start action also recognizes CONFIG tag `6`. When detected, it sets `ctx.config_mode_done = false` and returns success. On the next root sequence pass, the config-mode gate can enter Bluetooth configuration again.

### Fault flow

Faults can be latched from:

- Proximity loss.
- Too many missing/rejected line arrows.
- Checkpoint scan or stop-alignment timeout.

While the route is active, the interrupt wrapper diverts execution to `fault_recovery_action`. Recovery requires proximity contact and either RESUME or RESET.

## File Changes

### `source/bt/bt.lua`

Adds the behavior tree framework used by the new runtime. It is intentionally small and stateful: nodes store current child indexes, counters, and started flags directly on node tables.

### `source/bt/huskylens.lua`

Adds the main integrated robot behavior. This file contains configuration, runtime context, HuskyLens reads, line-following logic, LED helpers, checkpoint validation, BLE sequence handling, fault recovery, behavior tree assembly, and the infinite runtime loop.

### `source/bt/bluetooth.lua`

Adds a BLE mapper test/helper script. It initializes HuskyLens tag recognition and `robotito_ble`, sets an initial mapper, monitors mapper changes, and documents manual `bluetoothctl` testing. It is deployed by `update.sh`, but it is not imported by `source/bt/huskylens.lua`.

### `source/bt/README.md`

Adds documentation and examples for the behavior tree library, including actions, conditions, sequences, selectors, repeaters, interrupts, reset behavior, and memory guidance.

### `source/tests/test_PID.lua`

Adds an experimental line-following script with adaptive curve handling and anti-oscillation logic. It is separate from the behavior tree runtime.

### `source/tests/test_ble.lua`

Replaces the earlier simple BLE echo/sender test with a mapper-oriented BLE test. It registers connect/disconnect callbacks and mapper config/sequence callbacks.

### `source/tests/test_facu.lua`

Removed. In the base commit, this was a short omni movement test.

### `source/tests/test_husky.lua`

Removed. In the base commit, this was a continuous HuskyLens debug script that printed block and arrow detections.

### `update.sh`

Adds deployment of the new behavior-tree/HuskyLens/Bluetooth files and removes many older state and test files from the deployment list.

## Before vs After

### Before

Facts from the base commit:

- There was no `source/bt/` behavior tree runtime in the compared base.
- `source/tests/test_husky.lua` was a standalone continuous HuskyLens diagnostic script.
- `source/tests/test_facu.lua` was a standalone omni movement test.
- `update.sh` copied and uploaded a broad set of AHSM state files and test scripts.
- BLE test behavior in `source/tests/test_ble.lua` was a simple echo/send loop.

### After

Facts from the target branch:

- The main new Robotito logic lives in `source/bt/huskylens.lua`.
- Runtime control is modeled as a behavior tree instead of one procedural test loop.
- The robot alternates between line-tracking and tag-recognition HuskyLens algorithms.
- STOP, END, RESUME, RESET, CONFIG, and EXIT_CONFIG tags affect runtime state.
- Checkpoints are validated against configured sequences and visualized on the LED ring.
- Bluetooth mapper sequences can update `run_context.valid_sequences`.
- Faults interrupt normal route execution and require operator recovery.
- `update.sh` focuses deployment on the new BT/HuskyLens/BLE files plus selected core modules.

Inference:

- The branch moves the project from proof-of-concept HuskyLens scripts toward a field workflow for a route/checkpoint game, where external operators use printed tags and BLE configuration to control the robot.

## Technical Details

- HuskyLens algorithm IDs are hardcoded in `config.huskylens.algorithm`: line tracking `3`, tag recognition `5`.
- HuskyLens detection command IDs are hardcoded as block `42` and arrow `43`.
- The camera center is derived from `config.camera.frame_width / 2`, currently `160`.
- Line steering uses a weighted blend: `position_err * 0.6 + heading_err * 0.4`.
- The default route sequence in `run_context.valid_sequences` is `{{6,7,8,9}}`.
- LED checkpoint summaries divide the 24-pixel ring into one colored segment per checkpoint result, with separators when there is more than one segment.
- `reset_run(ctx)` resets mission state, clears checkpoint lists, stops the robot, clears LEDs, and disables proximity.
- `clear_faults(ctx, node)` clears fault latches but preserves route/checkpoint progress unless the caller also invokes `reset_run(ctx)`.
- `bt.create_tree(...):run()` uses `self.object` as the behavior tree context after `tree:set_object(run_context)`.

## Potential Issues or Limitations

Facts from the code:

- There is no available ref named `robotito-huskylens` in this checkout; this document analyzes `husky-tree-tags` because that is the current branch containing the Huskylens work.
- `run_context.config_mode_done` defaults to `false`, so the runtime enters Bluetooth config mode immediately on first boot and waits for EXIT_CONFIG before reaching the RESUME/start wait.
- Tag ID `6` is used both as the CONFIG command and as the first ID in the default valid checkpoint sequence `{{6,7,8,9}}`. Because `scan_checkpoint_action` ignores operator tags, a checkpoint card with ID `6` will be ignored by the default code path.
- `source/bt/bluetooth.lua` uses `huskylens.init()`, `huskylens.set_algorithm(...)`, and mapper functions without requiring `huskylens` locally in that file. It depends on `huskylens` being globally available or preloaded by the runtime environment.
- `source/tests/test_ble.lua` defines `total_bytes`, `total_lines`, and `total_packets`, but the active callbacks shown in the final file do not increment all of those counters.
- `update.sh` removes many previous tests and AHSM state files from the deployment list. That may be intentional for this branch, but it changes what gets copied to `fs/` and uploaded to the board.
- `finish_route_action` treats a route with no wrong checkpoint feedback as passing. From the code alone, an empty checkpoint list would also satisfy that condition because `all_ok` starts as `true`.
- `on_mapper_seq` in `huskylens.lua` uses `rble.mapper_set_sequences(seqs)` and `rble.mapper_get_sequences()`, while `test_ble.lua` uses singular API names in some code paths. The correct API shape depends on the `robotito_ble` module, which is not changed in this diff.

Inference:

- Some commit messages and inline comments show the branch was developed iteratively, and there are signs of unfinished cleanup: mixed Spanish/English messages, test-style code deployed as `bluetooth.lua`, and inconsistent mapper API naming between files.
- The default CONFIG/checkpoint ID overlap is likely accidental unless physical checkpoint ID `6` is never used in practice or is remapped through BLE before running the route.

## Summary

The branch introduces a behavior-tree-based Robotito runtime for a HuskyLens-guided route game. Compared with base commit `9e39a87`, the code now follows lines using HuskyLens arrow detections, switches into AprilTag recognition to process command and checkpoint tags, validates checkpoints against Bluetooth-configurable sequences, reports progress and outcomes on the LED ring, and interrupts normal execution for proximity, line-tracking, or checkpoint-scan faults.

The central implementation is `source/bt/huskylens.lua`, supported by the new lightweight behavior tree library in `source/bt/bt.lua`. `update.sh` is changed so those runtime files are deployed to the board. The branch also adds BLE mapper test/config scripts and replaces older standalone HuskyLens and omni movement tests.

The most important concerns are the missing `robotito-huskylens` ref in this checkout, the boot flow entering config mode by default, the tag ID `6` conflict between CONFIG and the default checkpoint sequence, and the fact that the deployed `bluetooth.lua` appears to be a test/helper script rather than a module consumed by the production runtime.
