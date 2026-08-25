# Analysis

## Overview

### Facts found in the code

This repository is an AprilTag-based configuration and scanning project for Robotito / ESP32 BLE workflows. The reviewed change range is `a4067c2..HEAD`, using `a4067c2` as the base because the checkout only exposes `main` and `origin/main`, and both point at the same commit.

The commit sequence reviewed is:

- `33195be` - restructures the project so the mapper/configurator lives at the root, splits the tag reader into `tag-reader.html`, and adds reserved tag validation.
- `f8f67ea` - adds `vercel.json` for static deployment.
- `8f839e8` - adjusts the Vercel configuration.
- `1b78ae4` - ignores `.env*.local` files.
- `f1c2d9e` - adds a `/scanner` route for the standalone tag reader.
- `be9e679` - improves the UI, adds API persistence support, adds package metadata, updates BLE-related behavior, and adds a sample tag SVG.
- `a0311c5` - adds the activities library and expands root UI support for activity presets and interchangeable tags.

At a high level, the branch changes the project from a more direct AprilTag mapper/scanner setup into a richer route and activity configurator. The root `index.html` now acts as the main user interface for building tag catalogs and sequences, reading physical AprilTags with an embedded scanner, sending sequences to an ESP32 over BLE, and optionally persisting the last configured sequence through a Vercel API.

### Assumptions and inferences

The project appears intended for educational or assistive robotics activities where physical AprilTag cards represent stations, steps, letters, or ordered tasks that Robotito should recognize or follow. This is inferred from names such as "Robotito", the BLE firmware integration, and presets like hand washing, tree growth, tooth brushing, and word-building activities.

## Main Changes

### Root route configurator

### Facts found in the code

`index.html` is now the primary application screen. It manages:

- `tagCatalog`, the list of known physical tags with aliases, colors, and interchangeability metadata.
- `sequences`, the ordered tag routes that can be sent to the robot.
- `RESERVED_TAGS`, currently defined as `new Set([1, 2, 3, 4, 5, 6])`.
- BLE connection state through `bleDevice`, `bleServer`, `notifyCharacteristic`, `seqWriteCharacteristic`, and `seqReadCharacteristic`.
- UI rendering through functions such as `renderTagCatalog()`, `renderGroups()`, `renderSequencesEditor()`, `renderAllSequencesModal()`, `updatePayloads()`, and `renderAll()`.

The configurator validates that catalog IDs are finite byte-sized values, are not repeated, are not reserved, and that each sequence uses known, non-reserved, non-repeated tags. It limits the number of sequences to `MAX_SEQS = 10`.

### Embedded AprilTag reader

### Facts found in the code

`tag-reader.html` is separated from the root app and supports both standalone and embedded operation. Embedded mode is enabled with `tag-reader.html?embedded=1`.

The scanner uses camera frames, converts them to grayscale in `getGrayscale()`, runs the AprilTag detector in `detectFrame()`, draws overlays in `drawDetections()`, updates panel state in `updatePanel()`, and sends detections to a parent page through `emitTagToParent()`.

When exactly one unique tag is detected, it posts:

```js
{
  type: 'apriltag-detected',
  id: mappedId,
  originalId: detectedId,
  ts: Date.now()
}
```

When multiple unique tag IDs are visible, it posts:

```js
{
  type: 'apriltag-error',
  code: 'multiple-ids',
  ids: uniqueIds,
  ts: Date.now()
}
```

### Sequence and catalog workflow

### Facts found in the code

The root UI embeds the scanner in a popover with `openTagReader()`. It waits for a stable single tag for 5 seconds before accepting it. The stability state is tracked through `stableTagId`, `stableStart`, `lastDetectionAt`, `tagReaderTimer`, and `tagReaderLocked`.

`handleTagReaderMessage()` rejects reserved tags and duplicate catalog tags before allowing progress. `completeTagAssignment()` adds an accepted tag to `tagCatalog`, uses a pending alias from `tagAliasInput`, assigns a color, then re-renders the UI.

Sequences are edited from the tag catalog. The user adds known tags to sequence cards, and `updatePayloads()` converts valid sequences into a BLE byte buffer shaped as:

```text
[sequence_count, len, tag_ids..., len, tag_ids...]
```

### Interchangeable tags

### Facts found in the code

The branch introduces interchangeable tag groups in `index.html`. Each catalog tag can list `interchangeableWith` IDs. The implementation treats the links as an undirected graph:

- `buildInterchangeableComponents()` builds connected components across tag links.
- `normalizeInterchangeableLinks()` rewrites each component into mutual relationships.
- `applyInterchangeableSelection()` updates a selected group and disconnects removed members.
- `getExpandedSequenceVariants()` generates sequence variants by permuting tags that belong to the same interchangeable component when multiple members of that component appear in a sequence.

These expanded variants are shown in the "all sequences" modal.

### Activity presets

### Facts found in the code

`activities-library.json` adds preset activities. Each activity has an `id`, `name`, `description`, `tags`, and `sequences`. Current activities use physical IDs `7`, `8`, `9`, and `10`, avoiding the reserved range used by the root UI.

`index.html` loads this file through `loadActivityLibrary()`, normalizes each entry with `normalizeActivity()`, renders a dropdown with `renderActivityLibraryDropdown()`, and applies a selected preset with `applyActivity()`.

### BLE sequence integration

### Facts found in the code

The root UI connects to the robot through Web Bluetooth in `connectBLE()`. It validates configured UUID strings, searches by Robotito-related device names and service UUID, then gets the notify, sequence write, and sequence read characteristics.

`sendAllToESP32()` validates the current data, serializes sequences to the `[count][len][ids...]` format, writes the result to `seqWriteCharacteristic`, and then calls `saveLastSequence()`.

`loadSequenceFromESP32()` reads `seqReadCharacteristic`. It supports both a legacy single-sequence shape and the current multi-sequence shape, then ensures each loaded tag exists in the catalog with `ensureTagInCatalog()` and updates `sequences`.

`startMonitoring()` and `stopMonitoring()` manage BLE notifications. `handleDetection()` parses notifications with a flags byte, a count byte at index `2`, and physical/logical ID pairs beginning at index `3`. Chunked notifications are logged but ignored by the assignment flow.

### Deployment and persistence

### Facts found in the code

`vercel.json` adds two rewrites:

- `/scanner` -> `/tag-reader.html`
- non-file, non-API paths -> `/`

`scanner/index.html` also redirects static-server requests from `/scanner/` to `/tag-reader.html`, preserving query string and hash.

`api/sequence.js` adds Vercel Blob persistence for `last-sequence.json` using `get()` and `put()` from `@vercel/blob`. `GET()` reads the private blob and returns JSON. `POST()` accepts request JSON and overwrites the private blob without a random suffix.

`package.json` adds ESM mode and the `@vercel/blob` dependency.

### Firmware-facing behavior

### Facts found in the code

`c/robotito_ble.c` changes the BLE sequence read handler for `SPP_IDX_MAPPER_SEQ_READ_VAL`. Instead of allocating separate `lens` and `seqs` buffers and reading via `huskylens_mapper_get_sequences()`, it:

- Calls `mapper_nvs_load_seq()` if `s_mapper_seq_loaded` is false.
- Allocates one 512-byte response buffer.
- Locks sequence state with `mapper_seq_lock()`.
- Packs directly from `s_mapper_seqs.count`, `s_mapper_seqs.lens`, and `s_mapper_seqs.ids`.
- Unlocks with `mapper_seq_unlock()`.

The code comment states this keeps reads in sync with NVS writes, unlike the older `huskylens_mapper` path.

### Test script update

### Facts found in the code

`test_ble.lua` still registers mapper callbacks, parses mapper config and sequence payloads, and applies the first received sequence because the Lua API only supports one sequence. The branch adds periodic printing of the saved sequence using `rble.mapper_get_sequence()` during the heartbeat loop.

## Implementation

### Facts found in the code

The browser application is mostly implemented as static HTML files with inline JavaScript and CSS.

The main runtime pieces interact as follows:

1. `index.html` owns the application state: catalog, sequences, activity presets, BLE connection, and scanner popover state.
2. `tag-reader.html` owns camera access and AprilTag detection. In embedded mode, it does not directly mutate the parent state; it reports detections through `window.parent.postMessage()`.
3. The parent `index.html` listens for scanner messages in `handleTagReaderMessage()`, applies reserved/duplicate/stability validation, and then mutates `tagCatalog`.
4. Sequence editing derives from the catalog. A sequence can only contain known tags, and duplicate tags inside a sequence are rejected by validation.
5. `updatePayloads()` displays the exact BLE sequence payload that `sendAllToESP32()` will write.
6. BLE write/read functions communicate with ESP32 characteristics. Firmware in `c/robotito_ble.c` handles the sequence read side by packaging sequence state from `s_mapper_seqs`.
7. Optional cloud persistence is implemented by `saveLastSequence()` and `loadLastSequence()` calling `/api/sequence`, but automatic startup loading is currently disabled.

`apriltag-map-builder.html` remains as a simpler physical-to-logical mapping tool. It still supports BLE map write/read, embedded tag reading, duplicate validation, and reserved tag checks, but the root UI is now centered on catalogs and ordered sequences rather than only direct physical/logical mapping.

### Assumptions and inferences

The split between `index.html` and `tag-reader.html` appears intentional to reuse the scanner both as a standalone page and as an embedded capture component. This keeps camera/detection concerns separate from route configuration concerns.

The firmware change suggests there was a real sync issue between the sequence state used for NVS persistence and the sequence state returned over BLE reads. The code comment explicitly says the direct `s_mapper_seqs` path stays in sync with NVS writes.

## Execution Flow

### Facts found in the code

### Opening the application

1. The browser loads `index.html`.
2. Initial buttons for BLE-dependent actions are disabled.
3. `applyDebugMode()` hides debug-only UI unless `?debug=true` is present.
4. `renderAll()` renders empty catalog/sequence state.
5. `renderActivityLibraryDropdown()` initializes the activity selector.
6. `loadActivityLibrary()` fetches `activities-library.json`, normalizes valid activities, and updates the dropdown.
7. `loadLastSequence()` exists but is commented out, so persisted sequence state is not automatically loaded.

### Applying an activity preset

1. The user selects an activity in the dropdown.
2. `applyActivity()` locates the activity and calls `normalizeActivity()`.
3. The activity is rejected if it has no valid tags/sequences, uses reserved tags, or contains duplicate tag IDs.
4. `tagCatalog` is replaced with the preset tags.
5. `normalizeInterchangeableLinks()` rebuilds mutual interchangeable relations.
6. `sequences` is replaced with the preset sequences.
7. `renderAll()` refreshes the catalog, groups, editor, and payload preview.

### Reading a physical tag into the catalog

1. The user enters an optional alias and clicks the read tag button.
2. `openTagReader()` opens the popover and loads `tag-reader.html?embedded=1` into an iframe.
3. `tag-reader.html` initializes camera and WASM detection, then runs `detectFrame()` around 15 FPS.
4. Detections are sent to the parent by `emitTagToParent()`.
5. `handleTagReaderMessage()` ignores invalid messages, rejects multi-tag errors, rejects reserved IDs, rejects duplicates, and tracks stable detection time.
6. After 5 seconds on one accepted tag, `completeTagAssignment()` adds the tag to `tagCatalog`, assigns alias/color metadata, re-renders, and closes the popover.

### Creating and previewing sequences

1. The user creates a sequence with the add sequence button.
2. `renderSequencesEditor()` displays available catalog tags as add buttons for each sequence.
3. Each sequence is represented as ordered physical tag IDs.
4. `validateData()` checks catalog and sequence constraints.
5. `updatePayloads()` either displays validation errors or serializes the sequence buffer.
6. `renderAllSequencesModal()` can show expanded variants produced by interchangeable tag groups.

### Sending sequences to Robotito

1. The user connects through `connectBLE()`.
2. The browser requests a BLE device by Robotito names, by service UUID, or finally through an all-devices debug fallback.
3. The app obtains notify, sequence write, and sequence read characteristics.
4. `sendAllToESP32()` validates data and serializes all sequences.
5. The app writes the serialized buffer with `writeValueWithoutResponse()` when available, otherwise `writeValue()`.
6. On successful write, `saveLastSequence()` attempts to persist the current state through `/api/sequence`.

### Loading sequences from Robotito

1. The user triggers `loadAllFromESP32()`, which calls `loadSequenceFromESP32()`.
2. The app reads the sequence read characteristic.
3. The parser accepts either the legacy single-sequence format or the multi-sequence format.
4. Loaded physical IDs are added to the catalog if missing.
5. `sequences` is replaced with unique-in-order versions of the loaded sequences.
6. The UI re-renders.

### Monitoring detections

1. `startMonitoring()` enables notifications on the notify characteristic.
2. `handleDetection()` parses each notification.
3. Non-chunked notifications produce detection entries from physical/logical ID pairs.
4. The UI log displays detected logical IDs.
5. Chunked notifications are currently logged and ignored.

## File Changes

### Facts found in the code

- `index.html`: became the main root configurator. It now includes catalog management, sequence editing, interchangeable groups, activity loading, embedded tag reading, BLE sequence write/read, detection monitoring, payload previews, and optional persistence calls.
- `tag-reader.html`: added as the standalone and embeddable AprilTag camera scanner. It handles camera/WASM detection, drawing overlays, mapped IDs, embedded styling, and `postMessage` output.
- `apriltag-map-builder.html`: remains a simpler physical-to-logical mapping utility and was updated with embedded scanner behavior, validation, and UI refinements.
- `apriltag-mapper.html`: removed after the root restructure moved the main mapper/configurator role into `index.html`.
- `activities-library.json`: new preset activity library with tags, aliases, interchangeable relationships, and sequences.
- `api/sequence.js`: new Vercel serverless API for reading/writing `last-sequence.json` through private Vercel Blob storage.
- `package.json`: new package metadata and `@vercel/blob` dependency for the API route.
- `vercel.json`: new deployment rewrites for `/scanner` and app-style routes.
- `scanner/index.html`: static fallback redirect to `/tag-reader.html`.
- `c/robotito_ble.c`: updates sequence readback to pack directly from `s_mapper_seqs` after loading from NVS if needed.
- `test_ble.lua`: adds periodic saved-sequence logging to the BLE test loop.
- `tag36h11-5.svg`: adds an AprilTag SVG asset.
- `.gitignore`: adds ignored local environment files and other ignore rules.
- `README.md`: updates project documentation to describe the root route configurator, tag reader, BLE protocol, and deployment/usage expectations.
- `.DS_Store`: remains tracked and changed in the diff.

## Before vs After

### Facts found in the code

Before this branch, the project already included AprilTag scanning, WASM detector assets, ESP32 BLE firmware code, a Lua BLE test script, and HTML tools for mapper/scanner workflows.

After this branch:

- The root page is the route and activity configurator instead of a lower-level mapper-only interface.
- The tag scanner is split into `tag-reader.html` and can be embedded by other pages.
- The root UI uses scanner messages to populate a validated tag catalog.
- Ordered sequences are first-class data and can be sent to/read from ESP32 BLE characteristics.
- Interchangeable tag groups can expand displayed sequence variants.
- Preset educational activities can populate tags and sequences.
- Vercel deployment support and a Vercel Blob API exist for scanner routing and optional state persistence.
- Firmware BLE sequence reads now use the same sequence state structure that is kept in sync with NVS writes.

### Assumptions and inferences

The architectural direction changed from "configure AprilTag mappings" toward "author activities or routes for a robot using physical AprilTag cards." The old mapping tool still exists, so the branch does not fully remove lower-level mapper use cases.

## Technical Details

### Facts found in the code

The current root UI reserves tag IDs `1-6`. Activity presets use IDs `7-10`.

The root sequence BLE payload format is:

```text
[seq_count:1] + seq_count * ([len:1] + len * [tag_id:1])
```

Example for two sequences, `[7, 8]` and `[9, 10]`:

```text
[0x02, 0x02, 0x07, 0x08, 0x02, 0x09, 0x0A]
```

The root UI validates:

- Catalog IDs must be finite numbers between `0` and `255`.
- Catalog IDs cannot repeat.
- Catalog IDs cannot be reserved.
- At least one sequence must exist before sending.
- Sequence count must not exceed `MAX_SEQS`.
- Sequence tags must exist in the catalog.
- Sequence tags cannot be reserved.
- A sequence cannot repeat the same tag ID.

The scanner has two ID concepts:

- `originalId`: the actual detected AprilTag ID.
- `id`: the mapped/display ID sent as the primary value to the parent.

`tag-reader.html` currently posts messages with `'*'` as the target origin.

`api/sequence.js` persists this browser state shape:

```js
{
  version: 1,
  tagCatalog: [
    {
      physicalId,
      alias,
      color,
      interchangeableWith
    }
  ],
  sequences: [...]
}
```

`applyPersistedState()` contains backward compatibility for an older persisted `group` string by migrating common group names into `interchangeableWith` links.

The Vercel API stores data in a private blob named `last-sequence.json` and overwrites the same blob on every `POST`.

## Potential Issues or Limitations

### Facts found in the code

- `loadLastSequence()` exists but is commented out at startup, so `/api/sequence` persistence is written after sending but not automatically restored when the app opens.
- `@vercel/blob` depends on a Vercel-compatible runtime and blob configuration. A simple static server such as `python -m http.server` will serve the static files but will not provide `/api/sequence`.
- Web Bluetooth and camera access require supported browsers and secure context. The UI explicitly checks for Web Bluetooth and secure context.
- `tag-reader.html` posts messages to `'*'`, which is convenient for embedding but does not restrict the receiver origin.
- The root UI reserves tags `1-6`, while README text in the current checkout mentions reserved tags `1-5` in at least one place. That documentation and implementation differ.
- The scanner sends both `id` and `originalId`, but the parent root UI uses `data.id`. Because `id` is mapped, future mapping changes could affect assignment behavior unless consumers intentionally use mapped IDs.
- `getExpandedSequenceVariants()` uses permutations for interchangeable groups. Larger groups or many repeated interchangeable positions could create many displayed variants.
- `handleDetection()` logs chunked BLE notifications and returns without parsing them.
- `.DS_Store` is tracked and changed in the repository history, which is usually not desirable for source control.

### Assumptions and inferences

The current implementation appears optimized for small activity sequences and a small number of physical tags. The byte-oriented BLE protocol, `MAX_SEQS = 10`, fixed reserved IDs, and activity presets using four tags all point toward a compact classroom/demo workflow rather than a large route-planning system.

The optional Vercel persistence looks like a convenience feature rather than the source of truth, because the app does not auto-load it and the robot sequence read path can reconstruct sequences from BLE.

## Summary

### Facts found in the code

The branch turns the project into a fuller Robotito AprilTag activity configurator. The root app now lets a user scan physical AprilTags, build a validated catalog, assign aliases/colors, define interchangeable tags, create ordered sequences, preview the exact BLE payload, send sequences to an ESP32, read sequences back, and monitor detections.

The scanner is factored into `tag-reader.html`, which can run standalone or embedded and communicates with the parent page through `postMessage`. Deployment support is added for Vercel, including `/scanner` routing and an optional Vercel Blob API for saving the last sequence state. Firmware readback in `c/robotito_ble.c` is adjusted so BLE sequence reads are packed from the sequence state synchronized with NVS.

### Assumptions and inferences

For external explanation, this work can be described as moving from a technical AprilTag/BLE mapper toward an activity authoring tool for a robot: educators or operators can define what each tag means, arrange tags into expected orders, and send that behavior to Robotito over BLE.
