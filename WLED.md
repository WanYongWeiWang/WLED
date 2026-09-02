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
