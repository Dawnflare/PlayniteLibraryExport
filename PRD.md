# PRD: Playnite Library JSON Exporter

## 1. Product Summary

**Product name:** Playnite Library JSON Exporter
**Product type:** Playnite C# Generic Plugin
**Primary purpose:** Automatically export the user’s current Playnite game library to a local JSON file whenever the Playnite library is updated, with an additional manual export command available from the Playnite UI.

The plugin should use the official Playnite SDK/database API rather than directly parsing Playnite’s internal library files. The exported JSON should provide a stable, machine-readable list of all games in the Playnite library, including each game’s name and Steam AppID when the Steam AppID can be confidently inferred from Playnite’s existing metadata.

------

## 2. Background and Problem Statement

The user maintains a Playnite library containing games from Steam and other libraries. They want a reliable JSON file that can be consumed by other scripts or applications, without manually exporting the library each time it changes.

Playnite already maintains the authoritative game library. The safest and most maintainable approach is to create a Playnite plugin that runs inside Playnite, accesses `PlayniteApi.Database.Games`, and writes an export file after Playnite finishes updating the library.

The plugin should avoid direct parsing of Playnite’s internal database/library files because those files are implementation details and may change. The plugin should also avoid external Steam matching in the MVP because title-based matching can create false positives.

------

## 3. Goals

### 3.1 Primary Goals

1. Export all Playnite library games to a JSON file.
2. Automatically refresh the export after Playnite completes a library update.
3. Provide a manual “Export now” action from the Playnite UI.
4. Include the game name for every exported game.
5. Include a Steam AppID when it can be confidently inferred from Playnite metadata.
6. Use the official Playnite SDK/database API.
7. Make the output deterministic, stable, and easy for downstream tools to consume.
8. Provide basic settings for export path, automatic export behavior, and JSON formatting.
9. Log errors clearly without disrupting normal Playnite usage.

### 3.2 Secondary Goals

1. Include useful provider/source metadata for each game.
2. Include export metadata such as schema version, timestamp, total game count, and plugin version.
3. Support atomic file writes to avoid partial/corrupt JSON.
4. Keep automatic exports quiet unless there is an error.
5. Show clear confirmation after manual export.
6. Minimize performance impact on Playnite startup and library updates.

------

## 4. Non-Goals for MVP

The MVP should **not** attempt to:

1. Query Steam’s public app list.
2. Fuzzy-match non-Steam games to Steam AppIDs.
3. Parse Playnite’s raw library/database files directly.
4. Provide a command-line interface.
5. Register custom `playnite://` URI commands.
6. Export cover images, icons, screenshots, trailers, or other media files.
7. Sync to cloud services directly.
8. Upload data to a web service.
9. Modify Playnite library data.
10. Act as a general-purpose backup tool.

These may be considered future enhancements.

------

## 5. Target User

The initial target user is an advanced Windows/Playnite user who wants to keep an automatically updated machine-readable game list for personal automation.

Secondary future users may include Playnite users who want JSON exports for dashboards, game recommendation tools, inventory comparisons, launchers, web apps, or local AI workflows.

------

## 6. Target Environment

### 6.1 Platform

- Windows 10 or Windows 11.
- Playnite Desktop mode.
- Current supported Playnite SDK for C# plugin development.
- Plugin implemented as a Playnite `GenericPlugin`.

### 6.2 Development Environment

- C#.
- Visual Studio or compatible .NET IDE.
- Playnite SDK package/reference.
- Use the Playnite extension structure with an `extension.yaml` manifest.
- Use the Playnite Toolbox utility if appropriate for generating the extension skeleton and packaging.

------

## 7. High-Level Design

The plugin should run inside Playnite and subscribe to Playnite lifecycle events.

### 7.1 Automatic Flow

1. Playnite starts.
2. Playnite updates one or more libraries, either automatically or manually.
3. Playnite fires the plugin’s library-updated lifecycle event.
4. The plugin schedules a JSON export.
5. The plugin reads a snapshot of `PlayniteApi.Database.Games`.
6. The plugin serializes the library to the configured JSON format.
7. The plugin writes the file atomically to the configured output path.
8. The plugin logs success or failure.

### 7.2 Manual Flow

1. User opens the Playnite main menu.
2. User chooses the plugin’s “Export library now” menu item.
3. Plugin immediately performs the export.
4. Plugin displays success/failure feedback to the user.
5. Plugin logs details.

### 7.3 Optional Startup Flow

The plugin may include a setting to export on Playnite startup. Recommended default behavior:

- If the export file does not exist yet, export once on startup.
- Otherwise, rely primarily on library update events.

This prevents the user from having no JSON file after first installation, while avoiding unnecessary exports on every launch.

------

## 8. Functional Requirements

### FR-001: Export All Games

The plugin must export all games visible to the Playnite database API.

The default behavior should be to include every game in the Playnite library, including installed, uninstalled, hidden, manually added, emulated, and third-party-library games.

A future setting may allow filtering, but MVP should export the complete library.

------

### FR-002: Automatic Export After Library Update

The plugin must automatically export JSON when Playnite finishes a library update.

The automatic export should be triggered by Playnite’s library-updated lifecycle event.

Automatic export should be enabled by default.

The plugin should not display a success dialog for automatic exports by default. It should log success silently.

If automatic export fails, the plugin should log the error. User-facing error notifications may be enabled via a setting but should not be overly intrusive.

------

### FR-003: Manual Export Menu Item

The plugin must provide a manual export command in Playnite Desktop mode.

Recommended menu location:

```text
Extensions → Playnite Library JSON Exporter → Export library now
```

The menu item should trigger the same export logic used by automatic export.

After manual export:

- On success, show a short success notification or dialog including the output path.
- On failure, show an error message with a brief explanation and direct the user to Playnite logs for details.

------

### FR-004: Settings UI

The plugin must provide a settings UI through Playnite’s extension settings.

Recommended settings:

| Setting                                      | Type           | Default                                 | Description                                                  |
| -------------------------------------------- | -------------- | --------------------------------------- | ------------------------------------------------------------ |
| Enable automatic export after library update | Boolean        | true                                    | Export whenever Playnite finishes updating libraries         |
| Export once on startup if file is missing    | Boolean        | true                                    | Creates the initial JSON file after installation             |
| Export on every Playnite startup             | Boolean        | false                                   | Optional full startup export                                 |
| Output directory                             | Directory path | User documents or plugin data directory | Folder where JSON file is written                            |
| Output file name                             | String         | `playnite-library.json`                 | Name of exported JSON file                                   |
| Pretty-print JSON                            | Boolean        | true                                    | Human-readable indentation                                   |
| Include export metadata envelope             | Boolean        | true                                    | Use top-level object with metadata and games array           |
| Include only core fields                     | Boolean        | false                                   | Future-compatible option; default should include useful metadata |
| Show notification after automatic export     | Boolean        | false                                   | Quiet by default                                             |
| Show notification after manual export        | Boolean        | true                                    | Confirm manual action                                        |

Settings should be validated before saving or before export.

Validation requirements:

- Output directory must be non-empty.
- Output file name must be non-empty.
- Output file name should end in `.json`; if not, either warn or append `.json`.
- Plugin should create the output directory if it does not exist.
- Plugin should reject invalid path characters.
- Plugin should handle permission failures gracefully.

------

### FR-005: Export File Format

The default export should use a top-level metadata envelope:

```json
{
  "schemaVersion": 1,
  "generatedAtUtc": "2026-06-05T00:00:00Z",
  "generator": {
    "name": "Playnite Library JSON Exporter",
    "version": "1.0.0"
  },
  "playnite": {
    "databaseGameCount": 123
  },
  "export": {
    "totalGames": 123,
    "steamAppIdCount": 80
  },
  "games": [
    {
      "playniteId": "00000000-0000-0000-0000-000000000000",
      "name": "Example Game",
      "sortingName": "Example Game",
      "providerGameId": "123456",
      "pluginId": "00000000-0000-0000-0000-000000000000",
      "sourceId": "00000000-0000-0000-0000-000000000000",
      "sourceName": "Steam",
      "steamAppId": 123456,
      "steamAppIdSource": "playnite-provider-game-id",
      "steamAppIdConfidence": "exact"
    }
  ]
}
```

The plugin should keep the schema stable. Future breaking changes should increment `schemaVersion`.

------

### FR-006: Required Game Fields

Each exported game object must include:

| Field                  | Type                | Required | Description                                         |
| ---------------------- | ------------------- | -------- | --------------------------------------------------- |
| `playniteId`           | String GUID         | Yes      | Playnite’s internal game ID                         |
| `name`                 | String              | Yes      | Game title                                          |
| `sortingName`          | String or null      | Yes      | Playnite sorting name, if available                 |
| `providerGameId`       | String or null      | Yes      | Playnite provider/library game ID                   |
| `pluginId`             | String GUID or null | Yes      | Playnite library plugin ID associated with the game |
| `sourceId`             | String GUID or null | Yes      | Playnite source ID                                  |
| `sourceName`           | String or null      | Yes      | Playnite source name                                |
| `steamAppId`           | Integer or null     | Yes      | Steam AppID if confidently known                    |
| `steamAppIdSource`     | String or null      | Yes      | Method used to derive the Steam AppID               |
| `steamAppIdConfidence` | String              | Yes      | `exact`, `none`, or future values                   |

------

### FR-007: Recommended Optional Game Fields

The plugin should include these fields if they are readily available from the Playnite SDK and do not add meaningful complexity:

| Field              | Type                     | Description                                  |
| ------------------ | ------------------------ | -------------------------------------------- |
| `isInstalled`      | Boolean                  | Whether Playnite marks the game as installed |
| `hidden`           | Boolean                  | Whether the game is hidden                   |
| `favorite`         | Boolean                  | Whether the game is marked favorite          |
| `installDirectory` | String or null           | Install directory if available               |
| `platforms`        | Array of strings         | Platform names if available                  |
| `genres`           | Array of strings         | Genre names if available                     |
| `categories`       | Array of strings         | Category names if available                  |
| `tags`             | Array of strings         | Tag names if available                       |
| `lastActivity`     | String datetime or null  | Last played/activity timestamp if available  |
| `playtimeSeconds`  | Integer or null          | Playtime if available                        |
| `added`            | String datetime or null  | Date added if available                      |
| `modified`         | String datetime or null  | Date modified if available                   |
| `releaseDate`      | String or object or null | Release date if readily serializable         |

Optional fields should not block MVP completion if the SDK properties require more nuanced handling.

------

### FR-008: Steam AppID Detection Logic

The plugin must distinguish between:

1. Playnite’s internal game ID.
2. The provider/library `GameId`.
3. The Steam AppID.

For MVP, the plugin should set `steamAppId` only when the Steam AppID can be derived confidently from Playnite metadata.

Recommended MVP logic:

- If the game appears to be from Steam and `providerGameId` is numeric, set:
  - `steamAppId` = parsed integer provider game ID.
  - `steamAppIdSource` = `playnite-provider-game-id`.
  - `steamAppIdConfidence` = `exact`.
- Otherwise:
  - `steamAppId` = null.
  - `steamAppIdSource` = null.
  - `steamAppIdConfidence` = `none`.

The plugin should avoid title-based or fuzzy Steam matching in MVP.

Steam detection should be conservative. A false null is preferable to an incorrect Steam AppID.

Recommended Steam detection signals:

1. `sourceName` equals `Steam`, case-insensitive.
2. The importing plugin/source is identifiable as Steam.
3. `providerGameId` is a positive integer string.

Implementation should avoid hardcoding fragile assumptions when possible. If a known Steam plugin identifier is available from Playnite metadata or documentation, it may be used as an additional signal, but `sourceName == "Steam"` plus numeric provider ID is acceptable for MVP.

------

### FR-009: Deterministic Output Ordering

The exported `games` array must be sorted deterministically.

Recommended sorting:

1. Case-insensitive `sortingName` if present.
2. Case-insensitive `name`.
3. `playniteId`.

This makes diffs cleaner and avoids unnecessary churn in version-controlled exports.

------

### FR-010: Atomic File Writes

The plugin must avoid leaving a partially written JSON file.

Recommended behavior:

1. Serialize JSON into memory or a temporary file.
2. Write to a temporary file in the target directory.
3. Replace the target JSON file atomically or as close to atomically as practical on Windows.
4. If replacement fails, preserve the previous export file when possible.
5. Clean up stale temporary files when safe.

------

### FR-011: Debounce Automatic Exports

The plugin should avoid excessive repeated exports during bursts of library changes.

Recommended debounce behavior:

- When an automatic export event occurs, schedule export after a short delay, such as 2–5 seconds.
- If another export-triggering event occurs during that delay, reset or coalesce the pending export.
- Never run two exports concurrently.
- Manual export should either:
  - Run immediately after any current export completes, or
  - Cancel/replace a pending automatic export and run now.

------

### FR-012: Concurrency and UI Responsiveness

The plugin must not freeze Playnite’s UI during normal use.

Recommended implementation approach:

- Capture the required Playnite game data safely.
- Perform JSON serialization and file writing off the UI thread when appropriate.
- If Playnite SDK access must occur on the UI thread, take a lightweight snapshot first, then serialize/write in the background.
- Use a lock, semaphore, or single-flight export queue to prevent concurrent writes.

------

### FR-013: Logging

The plugin must log:

- Plugin startup.
- Settings load/save failures.
- Automatic export trigger.
- Manual export trigger.
- Export start.
- Export success, including output path and game count.
- Export failure, including exception details.
- Validation failures.

Use Playnite’s logging facilities rather than writing a separate custom log file in MVP.

------

### FR-014: Error Handling

The plugin must handle common errors gracefully:

| Error                          | Expected behavior                                        |
| ------------------------------ | -------------------------------------------------------- |
| Output directory missing       | Create it if possible                                    |
| Invalid output path            | Show settings validation error or export failure message |
| Permission denied              | Log error and show clear message for manual exports      |
| File locked by another process | Log error and show clear message for manual exports      |
| Serialization error            | Log full exception                                       |
| Playnite API data missing/null | Export nulls where appropriate; do not crash             |
| Duplicate game names           | Preserve distinct `playniteId` entries                   |
| Non-numeric provider IDs       | Do not infer Steam AppID                                 |

Automatic export failures should not crash Playnite.

------

### FR-015: Privacy and Security

The plugin must operate locally.

MVP must not:

- Send library data over the network.
- Query external APIs.
- Upload logs.
- Require credentials.
- Modify game metadata.
- Delete user files except temporary files created by this plugin.

The exported JSON may include local install paths if optional fields are enabled. If install paths are included by default, the settings UI should make this clear.

Recommended default:

- Include local install path only if readily available and useful.
- Consider a setting to exclude local paths for privacy.

------

## 9. User Experience Requirements

### UX-001: Manual Export Discoverability

Manual export should be easy to find in Desktop mode:

```text
Extensions → Playnite Library JSON Exporter → Export library now
```

Optional additional menu items:

```text
Extensions → Playnite Library JSON Exporter → Open export folder
Extensions → Playnite Library JSON Exporter → Open settings
```

### UX-002: Manual Export Feedback

After manual export success, show concise feedback:

```text
Playnite library exported successfully.

Games exported: 123
Steam AppIDs found: 80
Output: C:\Users\<User>\Documents\PlayniteExports\playnite-library.json
```

After manual export failure, show concise feedback:

```text
Playnite library export failed.

Reason: Access to the output path was denied.
Please check the plugin settings and Playnite logs.
```

### UX-003: Automatic Export Quietness

Automatic exports should be silent by default.

The plugin may log success but should not interrupt the user after every library update unless the user enables automatic export notifications.

------

## 10. Settings Defaults

Recommended defaults:

```text
Automatic export after library update: Enabled
Export once on startup if missing: Enabled
Export on every startup: Disabled
Output directory: %USERPROFILE%\Documents\PlayniteLibraryExport
Output file name: playnite-library.json
Pretty-print JSON: Enabled
Include metadata envelope: Enabled
Show automatic export notifications: Disabled
Show manual export notifications: Enabled
Include local install paths: Enabled or configurable
```

If environment variable expansion is supported, paths like `%USERPROFILE%` should be expanded before validation.

------

## 11. Data Model

### 11.1 Export Envelope

| Field            | Type            | Description               |
| ---------------- | --------------- | ------------------------- |
| `schemaVersion`  | Integer         | Export schema version     |
| `generatedAtUtc` | String datetime | UTC timestamp             |
| `generator`      | Object          | Plugin name/version       |
| `playnite`       | Object          | Playnite-related metadata |
| `export`         | Object          | Export summary            |
| `games`          | Array           | Exported games            |

### 11.2 Generator Object

| Field     | Type   | Description    |
| --------- | ------ | -------------- |
| `name`    | String | Plugin name    |
| `version` | String | Plugin version |

### 11.3 Export Summary Object

| Field             | Type    | Description                                                  |
| ----------------- | ------- | ------------------------------------------------------------ |
| `totalGames`      | Integer | Number of exported games                                     |
| `steamAppIdCount` | Integer | Number of games with non-null Steam AppID                    |
| `automatic`       | Boolean | Whether the export was automatic                             |
| `trigger`         | String  | `manual`, `library-updated`, `startup-missing-file`, or `startup` |

### 11.4 Game Object

Required fields:

```json
{
  "playniteId": "string-guid",
  "name": "string",
  "sortingName": "string-or-null",
  "providerGameId": "string-or-null",
  "pluginId": "string-guid-or-null",
  "sourceId": "string-guid-or-null",
  "sourceName": "string-or-null",
  "steamAppId": 123456,
  "steamAppIdSource": "playnite-provider-game-id",
  "steamAppIdConfidence": "exact"
}
```

For games without a Steam AppID:

```json
{
  "steamAppId": null,
  "steamAppIdSource": null,
  "steamAppIdConfidence": "none"
}
```

------

## 12. Technical Requirements

### TR-001: Plugin Type

Implement as a Playnite C# `GenericPlugin`.

### TR-002: Manifest

Include a valid `extension.yaml` manifest with:

- Unique extension ID.
- Name.
- Author.
- Version.
- Module DLL name.
- Type: `GenericPlugin`.
- Optional icon and links.

### TR-003: SDK Usage

Use the Playnite SDK to access:

- Plugin lifecycle events.
- Game database collection.
- Game model metadata.
- Main menu integration.
- Settings integration.
- Logging.

Do not read Playnite library files directly.

### TR-004: JSON Serialization

Use a reliable JSON serializer compatible with the plugin’s target framework and Playnite environment.

Requirements:

- Stable property names.
- Null values included for required fields.
- Pretty print setting.
- UTF-8 output.
- Safe handling of nullable values.
- No circular reference serialization from raw Playnite objects.

The plugin should map Playnite SDK objects into simple export DTOs before serialization.

### TR-005: Build and Packaging

The project should build into a Playnite extension package or loadable extension directory.

Deliverables:

- Source code.
- Project file.
- Manifest.
- Settings view files if needed.
- README.
- Build/package instructions.
- Example output JSON.

### TR-006: No Direct Playnite Source Modification

Do not modify the Playnite source repository. This should be a standalone plugin project.

------

## 13. Acceptance Criteria

### AC-001: Plugin Loads

Given the plugin is installed in Playnite, when Playnite starts, the plugin loads without errors and appears in the extension settings list.

### AC-002: Manual Export Works

Given Playnite is open, when the user selects “Export library now,” then the plugin writes the configured JSON file and shows a success message.

### AC-003: Automatic Export Works

Given automatic export is enabled, when Playnite finishes updating the library, then the plugin writes an updated JSON file without requiring manual user action.

### AC-004: JSON Contains All Games

Given the user has N games in the Playnite library, when the plugin exports JSON, then the exported `games` array contains N game objects unless a filter setting is explicitly enabled.

### AC-005: Steam AppID Detection Works for Steam Games

Given a Playnite game imported from Steam with a numeric provider `GameId`, when exported, then the game object contains that value as `steamAppId`, with `steamAppIdConfidence` set to `exact`.

### AC-006: Non-Steam Games Are Not Incorrectly Matched

Given a non-Steam game with no confident Steam metadata, when exported, then `steamAppId` is null and `steamAppIdConfidence` is `none`.

### AC-007: Output Is Deterministic

Given the same Playnite library state, when the export runs multiple times, then the games appear in the same order and the output is stable except for timestamp metadata.

### AC-008: Invalid Path Handling

Given the output path is invalid or inaccessible, when manual export is triggered, then the plugin shows an error and logs details without crashing Playnite.

### AC-009: Atomic Write Behavior

Given an existing valid export file, when an export fails during writing, then the previous export file should remain intact whenever practical.

### AC-010: Settings Persist

Given the user changes plugin settings and restarts Playnite, then the settings persist and are applied to subsequent exports.

------

## 14. Testing Plan

### 14.1 Unit-Level Tests, If Practical

Test pure logic functions separately where possible:

- Steam AppID inference.
- Export DTO mapping.
- Path validation.
- Sorting.
- JSON serialization.
- Settings defaulting/migration.

### 14.2 Manual Integration Tests

Test with a Playnite library containing:

1. Several Steam games.
2. Several GOG/Epic/Xbox/manual games.
3. Hidden games.
4. Installed games.
5. Uninstalled games.
6. Games with duplicate or similar names.
7. Games with missing provider IDs.
8. Games with non-numeric provider IDs.
9. Games with special characters in names.
10. Large library, ideally 1,000+ games if available.

### 14.3 Export Trigger Tests

Verify export occurs after:

- Manual menu action.
- Manual library update.
- Automatic Playnite library update.
- Startup when file is missing, if enabled.
- Startup always, if setting enabled.

### 14.4 Failure Tests

Verify behavior when:

- Output directory does not exist.
- Output directory cannot be created.
- Output file is locked.
- Output path is invalid.
- User lacks write permission.
- JSON serialization encounters unexpected nulls.

------

## 15. Performance Requirements

The plugin should handle large Playnite libraries without noticeable UI degradation.

Targets:

- For a library of 5,000 games, export should complete within a few seconds on a modern PC.
- Manual export may briefly show progress or block the command, but should not freeze Playnite for an extended period.
- Automatic export should be debounced and non-intrusive.
- Plugin should not repeatedly serialize the library during a burst of changes.

------

## 16. Compatibility Requirements

The plugin should target the current Playnite SDK and supported Playnite release.

Compatibility goals:

- Current Playnite 10-compatible SDK.
- Windows 10/11.
- Desktop mode support for menu items.
- Should not depend on a specific installation path.
- Should work with installed and portable Playnite installations.

The implementation should avoid hardcoded assumptions about the user’s Playnite data directory.

------

## 17. README Requirements

The repository should include a README with:

1. Project purpose.
2. Installation instructions.
3. Build instructions.
4. Playnite version/SDK requirements.
5. How to configure output path.
6. How to manually export.
7. When automatic export runs.
8. Explanation of Steam AppID detection.
9. Example JSON output.
10. Troubleshooting section.
11. Limitations and future enhancements.

Suggested troubleshooting topics:

- Plugin does not appear.
- Export file is not created.
- Permission denied writing JSON.
- Steam AppIDs missing for non-Steam games.
- Automatic export does not appear to run.
- Where to find Playnite logs.

------

## 18. Future Enhancements

These should not be implemented in MVP unless requested later.

### FE-001: Command-Line or URI Trigger

Add a custom `playnite://` URI command to trigger export externally.

Example future behavior:

```text
playnite://library-json-exporter/export
```

This would allow Windows Task Scheduler, PowerShell, or another app to request an export while still using the Playnite plugin as the authority.

### FE-002: Steam AppID Enrichment for Non-Steam Games

Add optional Steam catalog matching for games imported from non-Steam libraries.

Requirements for future version:

- Download/cache Steam app list.
- Match by exact normalized title first.
- Provide confidence scoring.
- Never overwrite exact Playnite Steam metadata.
- Mark fuzzy matches as `probable` or `candidate`, not `exact`.
- Optionally export multiple candidates.
- Allow user to disable external network access.

### FE-003: CSV Export

Add optional CSV output for spreadsheet tools.

### FE-004: Export Only Changed Games

Add incremental export metadata or diff export.

### FE-005: Export on Game Add/Edit/Delete Events

Listen to lower-level database collection changes and export after debounced changes, not only after full library updates.

### FE-006: UI Status Panel

Add a small settings/status panel showing:

- Last export time.
- Last export path.
- Last export result.
- Games exported.
- Steam AppIDs found.

### FE-007: Multiple Export Profiles

Support multiple export targets, such as:

- Full JSON.
- Minimal JSON.
- CSV.
- Filtered Steam-only export.

------

## 19. Suggested Implementation Phases

### Phase 1: Skeleton Plugin

- Create C# GenericPlugin project.
- Add manifest.
- Confirm plugin loads.
- Add logger.
- Add basic settings object.
- Add manual menu item that logs when clicked.

### Phase 2: Core Export

- Read games from Playnite API.
- Map games to DTOs.
- Serialize JSON.
- Write to configured path.
- Add manual success/failure UI.

### Phase 3: Automatic Export

- Hook library-updated lifecycle event.
- Add debouncing/single-flight protection.
- Add startup-missing-file export behavior.
- Add quiet logging.

### Phase 4: Settings UI

- Add settings view.
- Add validation.
- Persist settings.
- Support output path and JSON formatting options.

### Phase 5: Robustness

- Atomic writes.
- Better error handling.
- Edge-case handling for null fields.
- Large-library testing.
- README and example output.

### Phase 6: Polish

- Add “Open export folder.”
- Add “Open settings.”
- Add optional notification settings.
- Package as `.pext` if desired.

------

## 20. Codex Implementation Instructions

When implementing this PRD:

1. Build a standalone Playnite C# Generic Plugin.
2. Do not modify Playnite source code.
3. Do not parse Playnite’s internal database/library files.
4. Use Playnite SDK APIs for all game data access.
5. Keep MVP fully local with no network calls.
6. Make Steam AppID detection conservative.
7. Prefer null Steam AppID over an incorrect Steam AppID.
8. Write JSON atomically.
9. Avoid blocking the Playnite UI.
10. Add useful logging.
11. Include README and example JSON.
12. Keep the code organized into clear components:
    - Plugin entry point.
    - Settings model.
    - Settings view/model.
    - Export service.
    - Game DTO mapper.
    - Steam AppID inference helper.
    - File writer.
13. Include comments only where they clarify non-obvious behavior.
14. Avoid overengineering the MVP.
15. Keep future enhancements out of MVP unless specifically requested.

------

## 21. Open Questions

These can be decided during implementation:

1. Should local install paths be included by default?
2. Should hidden games be included by default? Recommended: yes.
3. Should the default output folder be Documents, the Playnite user data folder, or the plugin data folder?
4. Should the plugin export on every startup or only when the output file is missing? Recommended: only when missing.
5. Should manual export show a dialog or a Playnite notification? Either is acceptable; prefer the least intrusive standard Playnite UI pattern.
6. Should the plugin include genre/category/tag metadata in MVP? Recommended: include if easy and safe; otherwise defer.
7. Should the plugin include platform metadata in MVP? Recommended: include if easy and safe; otherwise defer.

------

## 22. MVP Definition of Done

The MVP is complete when:

1. The plugin loads successfully in Playnite.
2. The user can configure an output JSON path.
3. The user can manually export the library from Playnite’s menu.
4. The plugin automatically exports after Playnite finishes updating the library.
5. The JSON contains all Playnite games.
6. Each game contains at least:
   - Playnite ID.
   - Name.
   - Sorting name.
   - Provider game ID.
   - Plugin ID.
   - Source ID.
   - Source name.
   - Steam AppID if confidently known.
7. Steam games imported by the Steam library plugin receive correct Steam AppIDs when provider IDs are numeric.
8. Non-Steam games are not falsely assigned Steam AppIDs.
9. The JSON file is written atomically.
10. Errors are logged and do not crash Playnite.
11. The repository includes README, build instructions, and example output.