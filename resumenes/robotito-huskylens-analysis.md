# robotito-huskylens Branch Analysis

## Overview

This document analyzes branch `huskylens_mapper` against `origin/robotito`, per request.

Facts found in Git:

- Base branch used for comparison: `origin/robotito`
- Merge base: `dd12c44e50f355f4bfc8bfad7aa6420ade64c738`
- Branch tip: `f13b76e6362af0f72e3d2ede7ac984131af290b6`
- Net diff: 12 files changed, 5583 insertions, 600 deletions

Commits reviewed:

- `6a1b7fc8` `funciona modulo husky`
- `f099e296` `update huskylens module`
- `31b58345` `complete module`
- `8f41c48a` `add HUSKYLENS_SPEED config and new huskylens sdkconfig`
- `c1cf906c` `updat sdkconfig`
- `35bb103a` `update husky c protocol`
- `2b236301` `switch and wait until getting a response`
- `77c2d91a` `remove error from omni`
- `a51c246f` `comment errors`
- `3a3e5b40` `big refactor v2`
- `35e2bd41` `Add mapper`
- `eee68bcf` `fix husky lens mapper`
- `971af025` `Merge branch 'robotito-huskylens' into huskylens_mapper`
- `66e26aab` `add logging`
- `d22ca8be` `new fixes`
- `78f79e14` `mas fixes`
- `b77dff5c` `first version bluetooth`
- `492db158` `wip`
- `0bdf0dae` `minor change`
- `f325d943` `add`
- `f2bf913e` `fix robotito ble`
- `f13b76e6` `husky_mapper eliminado`

Overall purpose, based on the final code: the branch adds a Lua-accessible DFRobot HuskyLens camera driver to Robotito and extends the Robotito BLE module so an external BLE client can configure persistent ID mappings and ID sequences used by HuskyLens-based workflows.

Inference: the branch appears to support a robot behavior where HuskyLens-detected physical IDs can be remapped to application-level logical IDs, and where an external phone/app/controller can update that mapping over BLE without reflashing firmware or editing Lua scripts.

## Main Changes

### HuskyLens Lua Module

The branch adds a new Lua module named `huskylens`, implemented by `components/lua/modules/hw/huskylens_lua.c` and registered with `MODULE_REGISTER_RAM(HUSKYLENS, huskylens, luaopen_huskylens, 1)`.

The module is compiled when `CONFIG_LUA_RTOS_LUA_USE_HUSKYLENS` is enabled.

The Lua API exposes:

- Initialization and diagnostics: `init`, `test`
- Data acquisition: `request`, `request_by_id`, `request_blocks`, `request_arrows`, learned-object variants, and ID-filtered variants
- Result access: `available`, `get_results`, `get`, `get_by_id`, `get_block`, `get_arrow`, learned-result variants, and count functions
- Camera operations: `set_algorithm`, `learn`, `forget`, `is_learned`
- Advanced HuskyLens commands: custom sensor writes, custom names/text, save/load model, save picture/screenshot, Pro-version check, and firmware-version check/write
- Algorithm constants such as `ALGORITHM_OBJECT_TRACKING`, `ALGORITHM_LINE_TRACKING`, and `ALGORITHM_COLOR_RECOGNITION`

The module stores one static device instance:

```c
static bool initialized = false;
static huskylens_t husky_instance;
```

Most Lua entry points reject calls before `init()` with `nil, "not initialized"`.

### HuskyLens C Driver

The branch adds `components/lua/modules/hw/huskylens.c` and `components/lua/modules/hw/huskylens.h`.

The driver implements:

- Initialization of a `huskylens_t` object
- Attachment to I2C address `0x32`
- HuskyLens knock/handshake during initialization
- Raw I2C frame writes and reads
- Response waiting and frame parsing
- Dynamic storage of the most recent result set
- Count and filter helpers for result type, ID, and learned status
- Commands for algorithm selection, learning, forgetting, custom text/name operations, SD-card model/picture operations, firmware checks, and Pro-version detection

Result values are represented as `huskylens_result_t`:

```c
typedef struct {
    uint8_t command;
    int16_t first;
    int16_t second;
    int16_t third;
    int16_t fourth;
    int16_t fifth;
} huskylens_result_t;
```

Facts from the Lua wrapper show these fields are exposed for block-like results as `x`, `y`, `width`, `height`, and `id`.

Inference: for arrow results, the same five integer fields likely represent the HuskyLens arrow coordinate/id tuple rather than `x`, `y`, `width`, and `height`.

### HuskyLens Protocol Core

The branch adds `components/lua/modules/hw/HuskyLensProtocolCore.c` and `.h`.

This layer contains the byte-level protocol framing:

- Frame headers: `0x55`, `0xAA`
- Address byte: `0x11`
- Command byte
- Payload size
- Payload bytes
- One-byte checksum

It provides typed payload write/read helpers and parser functions such as `husky_lens_protocol_receive()`, `husky_lens_protocol_read_begin()`, and `husky_lens_protocol_read_end()`.

### I2C Integration

The HuskyLens driver uses raw I2C transfers rather than register-based reads/writes. Comments in `huskylens.c` state this is meant to match Arduino `Wire` behavior for HuskyLens.

`huskylens_init()` uses `CONFIG_APDS9960_I2C_CHANNEL`, so HuskyLens shares the APDS9960 I2C bus. The comments identify the shared-bus addresses as:

- APDS9960: `0x39`
- HuskyLens: `0x32`

If the first `i2c_attach()` fails, the driver retries with speed `0`, with a comment saying this means attaching without reconfiguring an already-used bus.

The branch also modifies `components/sys/drivers/i2c.c` so `i2c_attach()` records the selected bus speed in `i2c[unit].speed` when the bus is first set up.

### Build and Configuration

`components/sys/Kconfig` is extended with:

- `HUSKYLENS_SPEED`, default `100000`, range `30000..3400000`
- `LUA_RTOS_LUA_USE_HUSKYLENS`, default `y`

The APDS9960 speed range minimum is lowered from `100000` to `30000`.

`sdkconfig.robotito` now enables `CONFIG_LUA_RTOS_LUA_USE_HUSKYLENS=y`.

The branch adds `sdkconfig.huskylens`, which enables HuskyLens, Robotito BLE, BLE CLI, I2C, and NVS, disables Robotito SPP/SPPCLI, sets `CONFIG_ROBOTITO_BLE_LINEBUFFER=512`, and sets both `CONFIG_APDS9960_SPEED` and `CONFIG_HUSKYLENS_SPEED` to `30000`.

### Robotito BLE Mapper

The largest final change is in `components/lua/modules/hw/robotito_ble.c`.

The branch extends the existing BLE SPP-like GATT service with four mapper characteristics:

- `0xABF6`: mapper config write
- `0xABF7`: mapper config read
- `0xABF8`: mapper sequence write
- `0xABF9`: mapper sequence read

`components/lua/modules/hw/robotito_ble.h` extends the GATT attribute enum with matching entries.

The mapper stores two kinds of data:

- ID map: physical HuskyLens ID -> logical/application ID
- Sequences: one or more ordered lists of IDs

The final code embeds mapper constants directly in `robotito_ble.c`:

```c
#define HUSKYLENS_MAP_MAX 50
#define HUSKYLENS_SEQ_MAX 50
#define HUSKYLENS_SEQ_MAX_SEQS 10
```

The comment says these were previously provided by `huskylens_mapper.h` and were copied into `robotito_ble.c` because `robotito_ble` no longer depends on that module. The commit history confirms this: `huskylens_mapper.c` and `huskylens_mapper.h` were added in earlier commits and deleted in the final commit `f13b76e6`.

Mapper data is persisted in NVS:

```c
#define MAPPER_NVS_NAMESPACE  "apriltag"
#define MAPPER_NVS_KEY_MAP    "map"
#define MAPPER_NVS_KEY_SEQ    "seq"
```

Inference: the NVS namespace name `apriltag` suggests the mapping concept may have originally been shared with, or inspired by, AprilTag-based workflows, even though this branch uses it in the HuskyLens/BLE path.

The module adds Lua APIs for mapper initialization, reading/writing pairs, reading/writing one or many sequences, remapping an ID, clearing map/sequence state, mapper callbacks, connect/disconnect callbacks, BLE deinit, and heap memory info.

The previous `robotito_ble` module from `origin/robotito` exposed only `init`, `send`, `set_rcv_callback`, and `set_line_callback`.

### BLE Lifecycle Changes

The BLE module now tracks more runtime state, including connect/disconnect callbacks, mapper callbacks, task handles, mapper mutexes, and a mapper event ringbuffer.

`robotito_ble_init()` now creates a Lua mutex, releases classic Bluetooth memory only once, initializes BLE, creates the stream ringbuffer, starts the receive task, creates the mapper event ringbuffer, starts the mapper event task, and starts the command task.

`robotito_ble_deinit()` is added. It stops advertising, disconnects any active client, deletes BLE tasks, tears down Bluedroid and the BLE controller, frees ringbuffers/queues and advertising/name allocations, deletes mutexes, clears connection state, clears Lua callback refs, runs Lua GC, and returns `true`.

Fact: the printed message says reboot is required, but the line that would enforce permanent shutdown is commented out:

```c
// robotito_ble_permanently_down = true;
```

So the final code does not actually mark BLE as permanently down after deinit.

### Omni H-Bridge PID Behavior

`components/lua/modules/hw/omni_hbridge.c` changes `omni_drive()` by commenting out the reset of `accum_error` for the three motors.

Before this branch, every `omni_drive()` call reset the accumulated PID error after computing new target wheel velocities.

After this branch, the accumulated error is preserved across target updates.

Inference: this may have been intended to avoid control discontinuities during repeated movement updates, possibly for HuskyLens-guided motion. The code does not include a comment tying this directly to HuskyLens.

## Implementation

The final implementation is organized into two main areas: camera access and remote mapper configuration.

The HuskyLens access stack is:

1. Lua module: `huskylens_lua.c`
2. C device driver/state: `huskylens.c` and `huskylens.h`
3. Protocol encoder/parser: `HuskyLensProtocolCore.c` and `.h`
4. Transport: existing I2C driver functions in `components/sys/drivers/i2c.c`
5. Build selection: `Kconfig`, `sdkconfig.robotito`, and `sdkconfig.huskylens`

Lua code calls `huskylens.init()`, which initializes the single static `husky_instance`. Later Lua calls invoke C driver functions against that same instance.

The C driver writes a command frame using the protocol core, sends it with raw I2C, waits for a matching response command where needed, then parses returned data into `husky->protocolPtr`.

The result cache is replaced on each successful data request. `process_return()` reads `COMMAND_RETURN_INFO`, stores `protocolSize`, `knowledgeSize`, and `frameNum`, allocates a `protocolPtr` array, and reads each returned `COMMAND_RETURN_BLOCK` or `COMMAND_RETURN_ARROW` frame.

The mapper path is implemented inside `robotito_ble.c`. Map state is an array of `{physical_id, logical_id}` entries. Sequence state is an array of up to 10 sequences, each with up to 50 IDs. Both are protected by FreeRTOS mutexes.

The BLE binary formats are:

- Map: `[count, physical1, logical1, physical2, logical2, ...]`
- Legacy single sequence: `[len, id1, id2, ...]`
- Multi-sequence: `[count, len1, ids..., len2, ids..., ...]`

When the BLE client writes mapper config or sequence characteristics, `gatts_profile_event_handler()` validates the incoming byte length, updates in-memory mapper state, writes it to NVS, and enqueues a mapper event. The mapper event task later calls the registered Lua callback with the raw payload bytes.

When the BLE client reads mapper config or sequence characteristics, the handler lazily loads the data from NVS if needed, packs the current in-memory state, and returns it using `esp_ble_gatts_send_response()`.

The Lua API accesses the same in-memory state and NVS persistence functions. For example, `mapper_remap(physical)` returns the mapped logical ID if present, or the original physical ID if no mapping exists.

## Execution Flow

### HuskyLens Initialization

1. Lua calls `huskylens.init()`.
2. `l_huskylens_init()` calls `huskylens_init(&husky_instance)`.
3. `huskylens_init()` clears the `huskylens_t` structure, sets address `0x32`, default timeout `100 ms`, and default result values of `-1`.
4. The driver attaches to I2C channel `CONFIG_APDS9960_I2C_CHANNEL` at `CONFIG_HUSKYLENS_SPEED`.
5. If attach fails, it retries `i2c_attach()` with speed `0`.
6. The driver sends `COMMAND_REQUEST_KNOCK`.
7. It waits for `COMMAND_RETURN_OK`.
8. Lua receives `true` on success or `nil, "init failed"` on failure.

### HuskyLens Request and Result Read

1. Lua calls a request function such as `huskylens.request()` or `huskylens.request_blocks_by_id(id)`.
2. The Lua wrapper validates initialization and arguments.
3. The driver builds and sends the matching protocol command frame.
4. `process_return()` waits for `COMMAND_RETURN_INFO`.
5. It reads result count, learned-object count, and frame number.
6. It allocates result storage and reads returned block or arrow frames.
7. Lua can call `get_results()`, `get()`, count functions, or filtered getters to consume the cached result set.

### Algorithm Change

1. Lua calls `huskylens.set_algorithm(algorithm)`.
2. The driver sends `COMMAND_REQUEST_ALGORITHM` with the selected algorithm ID.
3. It waits for `COMMAND_RETURN_OK`.
4. It then repeatedly calls `huskylens_request()` with a shorter `25 ms` timeout until a request succeeds or the algorithm-ready timeout expires.
5. On success, it resets cached results and returns `true`.

Inference: the extra request loop exists because HuskyLens can acknowledge an algorithm switch before the new algorithm is ready to return normal frames.

### BLE Mapper Write

1. A BLE client writes to `0xABF6` for map config or `0xABF8` for sequences.
2. `gatts_profile_event_handler()` resolves the handle to a mapper attribute.
3. For map config, it expects at least a count byte and enough pair bytes for that count.
4. For sequences, it detects either the legacy single-sequence format or the newer multi-sequence format.
5. The mapper state is updated under mutex.
6. The new state is saved to NVS.
7. A mapper event is sent to `mapper_evt_buffer_handle`.
8. `mapper_evt_task()` receives the event and calls either `robotito_ble_mapper_cfg_callback` or `robotito_ble_mapper_seq_callback` with the raw payload.

### BLE Mapper Read

1. A BLE client reads `0xABF7` or `0xABF9`.
2. The handler lazily loads map or sequence data from NVS if it has not already been loaded.
3. The current state is packed into the binary format.
4. The handler sends an explicit GATT response because these mapper values use `ESP_GATT_RSP_BY_APP`.

### Lua Mapper Use

1. Lua calls `robotito_ble.mapper_init()`.
2. Mapper mutexes are created.
3. Map and sequence data are loaded from NVS.
4. Lua can call `mapper_remap(id)` to translate a detected physical ID to a logical ID.
5. Lua can read or update map/sequence data directly through the mapper Lua functions.

## File Changes

### `components/lua/modules/hw/HuskyLensProtocolCore.c`

Adds the HuskyLens protocol parser and frame builder. It maintains static send/receive buffers, validates checksums, and provides typed payload read/write helpers.

### `components/lua/modules/hw/HuskyLensProtocolCore.h`

Declares the protocol core API used by the HuskyLens driver.

### `components/lua/modules/hw/huskylens.c`

Adds the C driver for HuskyLens. It owns I2C transport operations, handshake, command sending, response waiting, result parsing, result filtering, algorithm selection, learning/forgetting, and miscellaneous HuskyLens commands.

### `components/lua/modules/hw/huskylens.h`

Adds protocol command constants, algorithm constants, result/state structs, and function declarations for the HuskyLens driver.

### `components/lua/modules/hw/huskylens_lua.c`

Adds the Lua-facing `huskylens` module. It wraps the C driver, exposes constants, converts result structs into Lua tables, and returns Lua-style errors.

### `components/lua/modules/hw/robotito_ble.c`

Extends the existing Robotito BLE module with mapper GATT characteristics, NVS-backed physical-to-logical ID mapping, NVS-backed ID sequence storage, Lua mapper APIs, mapper BLE callbacks, connect/disconnect callbacks, BLE deinitialization, heap memory info, and broader lifecycle cleanup.

It also incorporates constants that had temporarily lived in the deleted standalone `huskylens_mapper` module.

### `components/lua/modules/hw/robotito_ble.h`

Extends the BLE GATT attribute enum to include the mapper characteristics.

### `components/lua/modules/hw/omni_hbridge.c`

Stops resetting the PID accumulated error on every `omni_drive()` target update by commenting out the three assignments.

### `components/sys/Kconfig`

Adds HuskyLens configuration options and lowers the APDS9960 minimum I2C speed to allow 30 kHz operation.

### `components/sys/drivers/i2c.c`

Stores the configured bus speed in `i2c[unit].speed` during first-time bus setup.

### `sdkconfig.huskylens`

Adds a dedicated build configuration for the HuskyLens/BLE setup.

### `sdkconfig.robotito`

Enables `CONFIG_LUA_RTOS_LUA_USE_HUSKYLENS=y` in the Robotito configuration.

## Before vs After

### Before

Facts from `origin/robotito`:

- There was no `huskylens` Lua module.
- There was no HuskyLens C driver or HuskyLens protocol core.
- Robotito BLE exposed only `init`, `send`, `set_rcv_callback`, and `set_line_callback`.
- The BLE GATT service had SPP-style data, notify, command, and status characteristics, with optional heartbeat support.
- There were no mapper GATT characteristics.
- There was no NVS-backed mapper state in `robotito_ble`.
- `omni_drive()` reset each motor's `accum_error` on every drive command.
- APDS9960 I2C speed had a minimum Kconfig value of `100000`.

### After

Facts in `huskylens_mapper`:

- Lua scripts can initialize and control a HuskyLens camera through the new `huskylens` module.
- The firmware can request blocks/arrows/results from HuskyLens and inspect cached results from Lua.
- HuskyLens uses raw I2C at address `0x32` and is configured to share the APDS9960 I2C channel.
- A new `sdkconfig.huskylens` enables HuskyLens plus Robotito BLE.
- Robotito BLE exposes mapper GATT read/write characteristics.
- Mapper map/sequence data persists in NVS under namespace `apriltag`.
- Lua can read, write, clear, and use mapper state through `robotito_ble.mapper_*` functions.
- BLE clients can update mapper state and trigger Lua callbacks.
- `omni_drive()` preserves accumulated PID error across target updates.

## Technical Details

### Result Cache Semantics

The HuskyLens driver stores only the latest returned frame's results. Request functions overwrite the previous cache. Lua getters read from `husky_instance.protocolPtr`; they do not request fresh camera data by themselves.

### Indexing

The C driver uses zero-based indexes internally. Lua-facing result arrays are one-based where a table is returned. Direct single-item wrappers such as `l_huskylens_get()` pass the checked Lua integer directly into the C getter.

Inference: direct `get(index)` functions may expect zero-based indexes despite being exposed to Lua, which is unusual for Lua APIs.

### Mapper Limits

The final code enforces fixed limits:

- Up to 50 map entries
- Up to 50 IDs per sequence
- Up to 10 sequences

Extra entries are ignored or truncated by the setter/packing logic.

### BLE Attribute Response Strategy

The mapper read/write values use `ESP_GATT_RSP_BY_APP`, so the code is responsible for explicit `esp_ble_gatts_send_response()` calls for reads. The regular SPP data/notify/command/status values still use the older auto-response style where configured.

### Concurrency

The mapper uses separate mutexes for map and sequence state. BLE events, Lua calls, and the mapper event task can all interact with mapper state.

Lua callbacks are protected by `lua_mutex`, and the mapper-specific callbacks are invoked from `mapper_evt_task` rather than directly from the GATT event handler.

Inference: queueing mapper callbacks through a ringbuffer reduces the amount of Lua work done inside the BLE stack event callback.

### Temporary Standalone Mapper Removed

The commit history shows a standalone `huskylens_mapper.c/.h` was created, iterated on, and then deleted by `f13b76e6` (`husky_mapper eliminado`). The final branch state has no standalone mapper module; its remaining functionality is integrated into `robotito_ble.c`.

## Potential Issues or Limitations

Facts found:

- `git diff --check origin/robotito..huskylens_mapper` reports many trailing-whitespace issues in the added HuskyLens files and a few in `robotito_ble.c`.
- `robotito_ble_deinit()` prints that reboot is required, but the permanent-down flag assignment is commented out. The message and behavior do not match.
- `huskylens_init()` returns `false` if the knock handshake fails, but the visible code does not detach the I2C device on that failure path.
- The protocol core uses static global send/receive buffers, so it is not reentrant.
- `huskylens_lua.c` exposes a single static `huskylens_t` instance, so the Lua module supports one HuskyLens device instance.
- `mapper_init()` pushes `nil, "mapper: failed to create mutexes"` if mutex creation fails, but does not immediately return in that branch before continuing to load NVS and push `true`. That can leave an inconsistent Lua stack/result.
- `mapper_map_pack()` calculates `(out_len - 1)` when the output buffer is too small. Current call sites pass nonzero fixed-size buffers, but the helper itself does not guard `out_len == 0`.
- BLE mapper config and sequence formats use `uint8_t` counts and IDs, limiting IDs to `0..255`.
- The mapper NVS namespace is named `apriltag`, which may be confusing in a HuskyLens feature unless there is project context outside this diff.

Inferences and risks:

- The direct Lua `get(index)` APIs may be off by one for typical Lua callers because they pass the supplied index directly to zero-based C functions.
- The BLE read handlers allocate buffers and response structs on each read. That is simple, but repeated reads under low-memory conditions can fail and return `ESP_GATT_OUT_OF_RANGE`.
- The algorithm-switch readiness loop improves reliability, but it adds blocking wait time to `set_algorithm()`.
- Sharing the APDS9960 bus and lowering speed to 30 kHz may improve HuskyLens compatibility, but it can slow APDS9960 communication and any other device on that I2C bus.
- Preserving `accum_error` in `omni_drive()` changes controller dynamics. It may reduce discontinuities, but it can also preserve integral windup when targets change abruptly.

## Summary

The `huskylens_mapper` branch turns Robotito into a firmware build that can use a DFRobot HuskyLens camera from Lua and configure HuskyLens-related ID behavior over BLE.

The camera side adds a complete stack: Lua API, C driver, HuskyLens protocol encoder/parser, raw I2C transport, and Kconfig/sdkconfig integration. Lua code can select HuskyLens algorithms, request detections, read blocks/arrows/results, train or forget IDs, and use several advanced device commands.

The mapper side extends the existing Robotito BLE GATT service with read/write characteristics for physical-to-logical ID maps and ID sequences. The mapper state is stored in RAM, protected with mutexes, persisted to NVS, exposed to Lua, and updateable by a BLE client. Earlier commits introduced a standalone `huskylens_mapper` module, but the final branch deletes it and keeps the mapper inside `robotito_ble.c`.

The main behavior change for an external explanation is: before this branch, Robotito had BLE data transport but no HuskyLens integration and no BLE-configurable ID mapper. After this branch, Robotito can talk to HuskyLens over I2C, expose HuskyLens data to Lua, and let an external BLE client update persistent ID mappings/sequences used by robot behavior.
