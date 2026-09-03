# WLED

Status: WIP

## Purpose

This document proposes which WLED features should be included in the LumenCache JSON protocol.

The basic level translates the existing zone controls:

- `on`
- `bri`
- `rgb`
- `cct`
- `fade`

These controls apply to the whole strip so that SIB-EP-WLED can act like any other zone device.

The protocol may include a small number of additional, commonly used WLED features for animated colour effects. It is not intended to include every WLED API.

## Communication stated in the discussion

- ESP-NOW will not reach SIB-EP-WLED.
- Wi-Fi and RS485 can reach it.
- RS485 is expected to be handled by the STM32.
- A translation layer inside SIB-EP-WLED translates LumenCache commands for WLED.

The exact division of work between the STM32 and ESP32 is not yet defined in this document.

## `on`

LumenCache `set_zone` defines four `on` values:

| LC value | LC behavior | WLED JSON API representation |
|---:|---|---|
| `0` | Off | `{"on":false}` |
| `1` | On at the command/default setpoint | `{"on":true,"bri":<resolved brightness>}` |
| `2` | Toggle | `{"on":"t"}` |
| `3` | Resume the prior level | `{"on":true}` |

WLED does not use the LC integer values directly. Its top-level JSON state parser expects `on` as a Boolean, with the string `"t"` used for toggle.

For LC `on:1`, the translation layer also resolves and supplies the Zone brightness setpoint. For LC `on:3`, WLED restores its remembered prior brightness (`briLast`) when it receives `{"on":true}` while off.

## `bri`

LumenCache represents brightness as a percentage from `0.0` to `100.0`. Its current binary FQM type is `FQM_T_F32`.

WLED's top-level JSON API expects `bri` as an integer from `0` to `255`:

```json
{"bri":128}
```

The translation layer converts the value as follows:

```text
wled_bri = round(lc_bri * 255 / 100)
```

Examples:

| LC `bri` | WLED `bri` |
|---:|---:|
| `0.0` | `0` |
| `50.0` | `128` |
| `100.0` | `255` |

In WLED, global `bri:0` means off. A non-zero `bri` turns the global light state on when `on` is not supplied.

For `set_zone`, if `bri` is omitted, SIB-EP-WLED uses the stored Zone default; if that is also absent, the LumenCache system default is `100.0`.

Note: `pdm_fqm_json.h` still describes `bri` as `int8`, but the current wire-type table in `lc_fqm_types.c` defines it as `FQM_T_F32`.

## `rgb`

LumenCache binary FQM carries `rgb` as `FQM_T_U32` in `0x00RRGGBB` format. The PP converts external forms such as `"#aabbcc"` or `"rgb(170,187,204)"` before sending the command; the device receives a numeric value and does not parse the colour string.

The translation layer extracts the channels as follows:

```text
r = (rgb >> 16) & 0xff
g = (rgb >> 8)  & 0xff
b = rgb         & 0xff
```

WLED does not have a top-level `rgb` state field. It expects colour inside a segment's `col` array. The first `col` entry is the primary colour:

```json
{
  "seg": {
    "col": [[170, 187, 204]]
  }
}
```

Therefore LC `rgb:0x00AABBCC` becomes WLED primary colour `[170,187,204]` on every segment mapped to the whole-strip Zone.

`rgb:0` means black. An absent `rgb` field is different from zero: it supplies no new RGB value. A device without RGB capability omits `rgb` from its state reply.

## `cct`

LumenCache carries `cct` as Kelvin using `FQM_T_I16`:

```json
{"cct":2700}
```

WLED does not have a top-level `cct` state field. It expects `cct` inside a segment object:

```json
{
  "seg": {
    "cct": 2700
  }
}
```

For a valid Kelvin value, no unit conversion is required. The translation layer applies it to every segment mapped to the whole-strip Zone.

WLED also accepts values from `0` to `255`, but those values mean relative warm-to-cold position, not Kelvin. Because LC always defines `cct` in Kelvin, the translation layer must validate it against the supported fixture range before passing it to WLED.

WLED clamps Kelvin input internally to `1900–10091 K` and converts it to its relative `0–255` segment value. SIB-EP-WLED should additionally respect its configured warm and cool limits.

If `cct` is omitted from `set_zone`, the device uses its stored Zone default; if that is also absent, the LumenCache system default is `3500 K`. A device without CCT capability does not apply the field.

## `fade`

LumenCache carries `fade` as transition time in milliseconds using `FQM_T_I16`:

```json
{"fade":500}
```

WLED provides two related top-level JSON fields:

- `transition`: changes the normal transition duration.
- `tt`: overrides the transition duration for the current command only.

LC `fade` describes the current command, so it maps more closely to WLED `tt`.

WLED expresses `tt` in units of `100 ms`:

```text
wled_tt = round(lc_fade_ms / 100)
```

Example:

```text
LC:   {"fade":500}
WLED: {"tt":5}
```

`tt:0` applies the change immediately. When using the native WLED JSON representation, transition times have `100 ms` resolution; the translation layer must define consistent rounding for values that are not exact multiples of `100 ms`.

If `fade` is omitted from `set_zone`, the device uses its stored Zone default; if that is also absent, the LumenCache system default is `500 ms`.

## Advanced WLED Effects

This section will define how LumenCache commands control WLED features beyond the common `on`, `bri`, `rgb`, `cct`, and `fade` fields.

### `seg`: segment

A WLED segment is a logical region of LEDs inside one WLED controller. A segment has its own pixel range and state, including power, brightness, colours, CCT, effect, speed, intensity, and palette.

For example, this addresses segment `0`:

```json
{
  "seg": {
    "id": 0,
    "on": true,
    "bri": 128
  }
}
```

The `seg` field can contain:

- One object with `id`: update that segment.
- One object without `id`: update all active, selected segments.
- An array of objects: update multiple segments.

A segment is not necessarily a physical LED output. WLED creates segments over its logical LED space, and a segment can represent all or part of a strip.

A WLED segment is also not the same as an LC Zone. A segment exists only inside one WLED controller; an LC Zone can contain multiple devices. The SIB-EP-WLED translation layer decides which local segment or segments correspond to the received LC command.

### `col`: segment colour slots

`col` belongs inside a segment object. It is an array containing up to three colour slots:

1. Primary colour.
2. Secondary/background colour.
3. Tertiary/custom colour.

Each colour may be written as an RGB or RGBW channel array:

```json
{
  "seg": {
    "id": 0,
    "col": [
      [255, 0, 0],
      [0, 0, 0],
      [0, 0, 255]
    ]
  }
}
```

In this example, the primary colour is red, the secondary colour is black, and the third colour is blue. Which slots are visibly used depends on the selected effect.

The nested brackets have different meanings:

```text
"col": [ [R, G, B], [R, G, B], [R, G, B] ]
         └ colour 0   └ colour 1   └ colour 2
```

An optional fourth channel represents white: `[R,G,B,W]`.

The basic LC `rgb` field supplies only one RGB value, so its normal mapping updates colour slot `0`, the WLED primary colour.

---

### `fx`: effect

`fx` selects the effect algorithm used by a WLED segment:

```json
{
  "seg": {
    "id": 0,
    "fx": 8
  }
}
```

In this example, segment `0` changes to effect ID `8`. The effect then uses the segment's colours and other effect settings to generate its animation.

`fx:0` is the static/solid effect. Other IDs select animated effects such as blink, breathe, wipe, or rainbow.

The valid IDs depend on the WLED firmware build. This WLED source currently defines `MODE_COUNT` as `220`, giving built-in slots `0–219`; usermods or a different WLED version can change the available list. The receiver must validate an effect ID against its own effect count before applying it.

If `fx` is omitted, the segment keeps its current effect.

LumenCache currently has no `fx` field or `fqm_ji_fx` parameter. Supporting this feature over FQM requires a new protocol field or an agreed WLED command container; that wire format is not yet defined here.

### `sx` and `ix`: effect controls

`sx` and `ix` adjust the current effect on one WLED segment:

- `sx`: effect speed.
- `ix`: effect intensity.

Both are integer values from `0` to `255`:

```json
{
  "seg": {
    "id": 0,
    "fx": 8,
    "sx": 160,
    "ix": 200
  }
}
```

Their exact visual meaning depends on the selected effect. A higher `sx` commonly makes an animation faster, while `ix` may control strength, density, size, or another effect-specific parameter. Some effects may ignore one or both values.

The WLED default for both values is `128`. If `sx` or `ix` is omitted, the segment keeps its current value.

Selecting a new `fx` through the normal JSON API does not necessarily reset `sx` and `ix`. Sending all three fields together gives an explicit, repeatable result.

LumenCache currently has no `sx`, `ix`, speed, or intensity FQM parameters for WLED effects. Their FQM representation is not yet defined here.

### `pal`: palette

`pal` selects the colour palette used by an effect on one WLED segment:

```json
{
  "seg": {
    "id": 0,
    "fx": 8,
    "pal": 6
  }
}
```

A palette supplies a sequence of colours that an effect can use. It is different from `col`: `col` contains up to three explicit segment colours, while `pal` selects a larger predefined or custom colour set.

`pal` is an integer ID. The available IDs depend on the WLED build and may include built-in, custom, and usermod palettes. The receiver must validate the ID against its own palette list.

Palette selection is ignored for segments without RGB capability. If `pal` is omitted, the segment keeps its current palette.

LumenCache currently has no `pal` or palette FQM parameter. Its FQM representation is not yet defined here.

### `c1`, `c2`, `c3`, `o1`, `o2`, and `o3`: effect-specific controls

Some WLED effects expose additional controls whose meanings are defined by that effect:

- `c1` and `c2`: integer sliders from `0` to `255`.
- `c3`: integer slider from `0` to `31`.
- `o1`, `o2`, and `o3`: Boolean options.

Example:

```json
{
  "seg": {
    "id": 0,
    "fx": 8,
    "c1": 120,
    "o1": true
  }
}
```

These keys have no universal visual meaning. One effect may use `c1` for size while another uses it for spacing, and many effects ignore controls they do not define. A client must know the selected effect's metadata before presenting or setting them.

LumenCache currently has no equivalent FQM parameters. Because their meaning changes with `fx`, they are less suitable for the first small WLED protocol subset than `fx`, `sx`, `ix`, and `pal`.

### `rev`, `mi`, and `frz`: simple segment options

WLED also provides simple Boolean options on each segment:

- `rev`: reverse the segment's direction.
- `mi`: mirror the segment.
- `frz`: freeze or resume the segment's animation.

```json
{
  "seg": {
    "id": 0,
    "rev": true,
    "mi": false,
    "frz": false
  }
}
```

These options are local WLED segment behavior and currently have no LC FQM equivalents.

### Complexity boundary

The functions covered so far are direct updates to small scalar fields on an existing segment. The next groups are materially more complex:

- Presets recall a complete stored WLED state.
- Segment geometry creates, resizes, or deletes logical LED regions.
- Individual-pixel data can carry a variable-length colour stream.

These require different protocol and storage decisions, so they are not included in the simple effect-control group above.

## Build and Smoke-Test Status

On 2026-09-03, the first `A_Lumencache` scaffold was successfully built, flashed, booted, and connected to Wi-Fi.

| Item | Verified value |
|---|---|
| PlatformIO environment | `lumencache_s3` |
| WLED version | `17.0.0-devV5` |
| Release configuration | `ESP32-S3_8MB_opi` |
| Architecture / core | ESP32-S3 / `5.5.4.260407` |
| WLED catalogue | 220 effects, 72 palettes |
| Verification endpoint | `/json/info` |
| Usermod result | `"u":{"Lumencache":["loaded"]}` |

This confirms that the usermod was compiled, registered, and initialized. The device reports 16 MB physical Flash and approximately 8 MB PSRAM, while the current build uses the 8 MB Flash configuration; a 16 MB OPI build should be considered for the final hardware profile.
