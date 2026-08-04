# Movecraft-EEOL Usage Guide

This page documents how to run and use this Movecraft fork in practice, including command usage, sign systems, craft type configuration, direct control, and datapack tags.

## Scope

This guide is based on the current codebase behavior in this repository (not generic upstream wiki assumptions).

## Requirements

- Java 17 or newer
- A Paper-compatible server for the supported MC versions in this repo
- Movecraft plugin jar built from this repository

## Build And Deploy

From the repository root:

```bash
./gradlew clean shadowJar --parallel
```

Built jars are produced under `Movecraft/build/libs`.

Install by placing the selected Movecraft jar in your server `plugins/` directory and restarting.

## Repository Layout

- `Movecraft/`: main plugin module
- `api/`: craft type/property API and data model
- `datapack/`: bundled Movecraft datapack resources and tags
- `types/`: craft type definitions (`*.craft`)
- `v1_20_6/`, `v1_21_1/`, `v1_21_4/`, `v1_21_5/`, `v1_21_8/`, `v1_21_10/`, `v1_21_11/`, `v26_1_2/`, `v26_2/`: version compatibility modules

## Startup Behavior

On enable, Movecraft:

1. Loads config and localization.
2. Initializes version compatibility handlers.
3. Ensures Movecraft datapack availability (creates `movecraft-data.zip` in world datapacks when needed).
4. Registers core commands, sign listeners, contacts/status systems, and movement processing.

## Core Configuration (`plugins/Movecraft/config.yml`)

Key settings:

- `PilotTool`: item used for direct control interactions (default `STICK`).
- `RequireCreatePerm`: if true, creating craft/sign controls requires explicit permissions.
- `RequireNamePerm`: enforces permission checks for `Name:` signs.
- `ManOverboardTimeout` and `ManOverboardDistance`: govern `/manoverboard` availability.
- `MaxRemoteSigns`: cap for one remote-sign trigger fanout (`-1` means unlimited).
- `ForbiddenRemoteSigns`: sign strings that remote-sign scans must ignore.
- `FadeWrecksAfter`, `FadeTickCooldown`, `FadePercentageOfWreckPerCycle`: wreck fade behavior.

## Player Commands

### `/pilot <craftType>`

Detects and pilots a craft of the given type at your location.

Requires:

- `movecraft.commands`
- `movecraft.commands.pilot`
- `movecraft.<craftType>.pilot`

### `/release [player|-p|-n|-a]`

- No args: release your current craft.
- `<player>`: force-release that player's craft.
- `-p`: release all player-piloted crafts.
- `-n`: release all unpiloted/non-player crafts.
- `-a`: release all crafts.

Requires base:

- `movecraft.commands`
- `movecraft.commands.release`

Additional for targeting others/flags:

- `movecraft.commands.release.others`

### `/cruise [on|off|north|south|east|west|up|down]`

Toggles or sets cruise direction for your piloted craft.

Notes:

- `/cruise` with no args toggles cruise and infers cardinal direction from yaw.
- `up`/`down` are supported.

Requires:

- `movecraft.commands`
- `movecraft.commands.cruise`
- `movecraft.<craftType>.move`

Craft must allow cruise (`can_cruise`/equivalent property path).

### `/rotate <left|right>`

Rotates your current craft.

Requires:

- `movecraft.commands`
- `movecraft.commands.rotateleft` or `movecraft.commands.rotateright`
- `movecraft.<craftType>.rotate`

### `/manoverboard`

Teleports you back onto your craft under constraints.

Checks include:

- same world
- timeout not exceeded (unless still near craft)
- within configured distance limit
- craft is not disabled or sinking

### `/scuttle [player]`

Sinks a craft.

- No args: self scuttle (if allowed).
- With player arg: target that player's craft (if allowed).

Permissions involved:

- `movecraft.commands.scuttle.self` (self)
- `movecraft.commands.scuttle.others` (targeting)
- `movecraft.<craftType>.scuttle` (type-specific)

### `/contacts [page]`

Shows detected nearby contacts for your currently piloted craft.

- Requires being piloting a craft.
- Uses paginator output.

### `/craftreport [page]`

Lists active crafts with type, pilot, bounds, and mean cruise timing.

Requires:

- `movecraft.commands`
- `movecraft.commands.craftreport`

### `/crafttype [type|list] [page]`

Displays craft type info by reflection-backed pagination.

Common usage:

- `/crafttype list`
- `/crafttype Airship`
- `/crafttype Airship 2`

Requires:

- `movecraft.commands`
- `movecraft.commands.crafttype`
- `movecraft.<craftType>.pilot` for viewing a specific type

### `/craftinfo [player] [page]`

Displays runtime info for a craft (size, bounds, speed, cruise state, etc.).

Common usage:

- `/craftinfo` (your craft)
- `/craftinfo 2` (your craft page 2)
- `/craftinfo <player>`

### `/movecraft [reloadtypes]`

- No args: shows plugin version/author info.
- `reloadtypes`: reloads craft types.

Requires:

- `movecraft.commands.movecraft`
- `movecraft.commands.movecraft.reloadtypes` for reloading

## Sign Systems

Sign text matching is case-insensitive after color stripping.

### Helm

Create sign with line 1:

```text
[helm]
```

It is rewritten to helm art. Interactions:

- Right click: rotate clockwise
- Left click: rotate anticlockwise

Requires `movecraft.<craftType>.rotate`.

### Cruise Toggle

Line 1 must be one of:

```text
Cruise: OFF
Cruise: ON
```

Toggles cruise state and direction from sign facing.

### Ascend / Descend

Line 1:

```text
Ascend: OFF
Descend: OFF
```

Right click toggles ON/OFF and sets cruise up/down.

### Move Sign

```text
Move:
<dx>,<dy>,<dz>
```

Applies bounded static move deltas (clamped by type max static move).

### Relative Move Sign

```text
RMove:
<leftRight>,<dy>,<forwardBack>
```

Converts relative vectors using sign orientation.

### Release Sign

```text
Release
```

Releases your piloted craft.

### Scuttle Sign

```text
Scuttle
```

Scuttles your craft, or nearest craft if you have others-scuttle permission.

### Teleport Sign

```text
Teleport:
<x>,<y>,<z>
<worldName optional>
```

- Coordinates default to sign location if parse fails/missing.
- Requires move permission and type teleport enabled.
- Respects craft teleport cooldown.

### Remote Sign

```text
Remote Sign
<targetText>
```

Triggers matching signs inside the same piloted craft.

Behavior constraints:

- Craft type must allow remote sign use.
- Cannot target another `Remote Sign`.
- Forbidden strings are blocked by config.
- Fanout can be capped by `MaxRemoteSigns`.

### Speed Sign

```text
Speed:
```

Displays live speed/cruise timing/tick cooldown.

Right-click cycles gear if type has `gearShifts > 1`.

### Contacts Sign

```text
Contacts:
```

Displays nearby contacts on lines 2-4 and runs `/contacts` on right-click.

### Status Sign

```text
Status:
```

Displays required-block integrity and fuel range estimates.

### Craft Type Sign (Type Name On Line 1)

If line 1 is a valid craft type name, right-click attempts detection/piloting for that type.

### Subcraft Rotate

```text
Subcraft Rotate
<craftType>
```

Detects subcraft at sign location and rotates it using click direction.

### Name Sign

```text
Name:
<name line(s)>
```

Assigns craft name during detection. Permission enforcement depends on `RequireNamePerm`.

### Pilot Sign

```text
Pilot:
<player optional>
```

If player line is blank on creation, it is auto-filled by creator name.

## Direct Control

Direct control is implemented and active in this fork.

### How It Works

Using the configured `PilotTool` (default `STICK`):

1. Left-click air/block with pilot tool:
- Enter direct control mode (`pilotLocked = true`) when permitted.
- If already in direct control mode, leaves it.

2. Right-click air/block with pilot tool:
- If in direct control mode: vertical movement only.
- Not sneaking: move up.
- Sneaking: move down.
- If not in direct control mode: normal directional translation from yaw/pitch.

### Requirements

- You must be piloting a craft.
- Permission: `movecraft.<craftType>.move`.
- Craft type must allow direct control.
- You must be near the craft hitbox.

### Related Type Flags

- `can_direct_control`
- `gear_shifts_affect_direct_movement`
- `gear_shifts_affect_tick_cooldown`
- `half_when_underwater` (maps to internal half-speed underwater behavior)

## Contacts System

Contacts are computed periodically per world and sorted by range.

Detection depends on target craft size and detection multipliers:

- `detection_multiplier`
- `underwater_detection_multiplier`
- per-world variants

New contacts emit notifications and collision sound cues.

## Craft Types (`types/*.craft`)

### Inheritance Model

This fork uses parented craft definitions via the top-level `parent` field.

Examples:

- `AirShipBase`, `SeaShipBase`, `LandShipBase`: base templates
- Child types (such as `Airship`, `Ship`, `Tank`, `Train`) inherit and override selective fields

### Structure

Craft files are nested config trees under `movecraft:` and optional addon namespaces.

Typical sections:

- `general`
- `movement`
- `constraints`
- `fuel`
- `sinking`

Common addon sections seen in this repo:

- `movecraft-bridges`
- `movecraft-combat`
- `movecraft-eool-additions`

### Recommended Compatibility Notes

For this repository state:

- Prefer `sink_rate_ticks` over legacy sink-speed-style fields.
- Avoid legacy/deprecated keys when modern nested forms already exist.
- Use pass-through/forbidden blocks intentionally for water behavior modeling.
- Validate each type with in-game detect/pilot tests after edits.

## Datapack Tags And `#namespace:tag` Support

Custom block tags such as `#eool:movecraft_craft_blocks/airship` are supported as long as they exist in the loaded datapack registry.

This repo includes EOOL tags under:

- `datapack/src/main/resources/movecraft-data/data/eool/tags/block/`
- `datapack/src/main/resources/movecraft-data/data/eool/tags/block/movecraft_craft_blocks/`

If a referenced tag is missing at runtime, type loading/behavior will fail for those lookups.

## Permissions Quick Reference

Commonly used nodes:

- `movecraft.commands`
- `movecraft.commands.pilot`
- `movecraft.commands.release`
- `movecraft.commands.release.others`
- `movecraft.commands.cruise`
- `movecraft.commands.rotateleft`
- `movecraft.commands.rotateright`
- `movecraft.commands.craftreport`
- `movecraft.commands.crafttype`
- `movecraft.commands.movecraft`
- `movecraft.commands.movecraft.reloadtypes`
- `movecraft.commands.scuttle.self`
- `movecraft.commands.scuttle.others`
- `movecraft.<craftType>.pilot`
- `movecraft.<craftType>.move`
- `movecraft.<craftType>.rotate`
- `movecraft.<craftType>.scuttle`
- `movecraft.<craftType>.create` (when `RequireCreatePerm: true`)
- `movecraft.name.place` (when `RequireNamePerm: true`)
- `movecraft.cruisesign` (when creation perms are required for cruise signs)

## Operational Checklist

1. Build and install plugin jar.
2. Confirm Movecraft datapack is present and enabled.
3. Confirm custom tag namespaces (such as `eool`) exist and load cleanly.
4. Validate craft type parse on startup.
5. In-game test:
- `/pilot <type>`
- movement with pilot tool
- `/cruise` and `/rotate`
- key signs (`Helm`, `Cruise`, `Move`, `Status`, `Contacts`)
6. Reload types with `/movecraft reloadtypes` after type edits.

## Troubleshooting

### "Insufficient Permissions"

Check both command-level and type-level permission nodes. Most movement actions require `movecraft.<craftType>.move`.

### Craft Won't Detect

Check:

- Allowed/forbidden block sets in type.
- Size constraints (`min`/`max`).
- Required block rules.
- Parent inheritance path and overridden fields.

### Custom Tag Reference Fails

Verify datapack file path and namespace spelling. Ensure the server has that datapack loaded and enabled.

### Direct Control Not Entering

Check:

- `PilotTool` matches held item.
- `can_direct_control` is enabled for that craft type.
- Player has `movecraft.<craftType>.move`.
- Player is currently piloting and near the craft.

### Teleport Sign Does Nothing

Check:

- Craft has teleport enabled.
- Player has move permission for the type.
- Teleport cooldown is not active.
- World name on sign is valid (or omitted to use current world).
