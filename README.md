# glinenetTools

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A free, MIT-licensed Niagara 4 module with 8 utility components for building automation
integration work: decode status words, rank many inputs, map bit masks to enums, hold values
through comms dropouts, and wire dozens of points at once.

Download the module jar and follow the sections below.

---

## Download

| File | Description |
|---|---|
| `dist/glinenetTools-rt-signed.jar` | Signed module, ready to deploy on Niagara 4.14 |

---

## Compatibility

| Item | Value |
|---|---|
| Niagara | 4.14 (`baja` dependency) |
| Java | 8 bytecode |
| Module name | `glinenetTools` |
| Package | `com.glinenet.tools` |

---

## Installation

**1. Copy the jar to both locations**

```
<Niagara>/modules/glinenetTools-rt.jar
<Niagara>/sw/1.0/glinenetTools-rt.jar
```

Both copies are required — the station reads one path, the platform the other.

**2. Confirm the two copies are identical**

```
certutil -hashfile "<path>\modules\glinenetTools-rt.jar" MD5
certutil -hashfile "<path>\sw\1.0\glinenetTools-rt.jar" MD5
```

The two MD5 values must match. If they differ the module refuses to load.

**3. Restart the Niagara service**

```
net stop Niagara
net start Niagara
```

**4. Confirm the module loaded**

- In Workbench, the `glinenetTools` palette folder appears.
- Or over oBIX: `GET /obix/def/glinenetTools:BatchLinker/` returns 200.

The restart briefly interrupts the station — plan a window on production systems.

**Upgrade note:** after replacing the jar and restarting, existing links on created
components may be dropped by the platform. Re-link them after the upgrade.

---

## Components at a glance

| Type | What it does |
|---|---|
| `BatchLinker` | Creates point-to-point links in bulk using source and target ORD queries |
| `BitDecode16` | Splits a 16-bit word into 16 booleans plus byte/hex/bin text |
| `Sort7` | Sorts up to 7 numeric inputs, reports which channel each result came from |
| `EnumBits` | Converts between enum sets and integer bit masks, both directions |
| `NullShedNumeric` | Holds the last valid value on null input, releases after a timeout |
| `NullShedBoolean` | Same, for boolean |
| `NullShedString` | Same, for string |
| `NullShedEnum` | Same, for enum |

---

## 1. BatchLinker — bulk linking

Resolves a source set and a target set from ORD queries, then connects one slot on each
source to one slot on each target. Saves wiring dozens of points by hand in the Wire Sheet.

**Properties**

| Property | Meaning |
|---|---|
| `sourceOrd` | Source query, e.g. `station:|slot:/Demo/SrcGrp` |
| `targetOrd` | Target query, same form |
| `sourceSlot` | Slot on the source component, usually `out` |
| `targetSlot` | Slot on the target component, e.g. `in1` |
| `matchMode` | `name` (pair by component name, default) or `order` (pair by position) |
| `removeExisting` | Unlink existing links before creating new ones |
| `enabled` | Must be `true` |

**Actions:** `linkAll`, `unlinkAll`, `clearLog`

**Result properties:** `status` (e.g. `linked 3 failed 0`), `linkedCount`,
`failedCount`, `detailLog`

**ORD forms that work**

- Single component: `station:|slot:/Demo/GntTest/srcA`
- Folder (recommended): `station:|slot:/Demo/SrcGrp` — expands to all children
- BQL: returns 0 results on tested stations; not recommended

**Example** — three points under `SrcGrp` into same-named points under `DstGrp`:

```json
[{"OP":"mod","PATH":"/Demo/BatchLinker","N":"sourceOrd","V":"station:|slot:/Demo/SrcGrp"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"targetOrd","V":"station:|slot:/Demo/DstGrp"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"sourceSlot","V":"out"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"targetSlot","V":"in1"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"matchMode","V":"name"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"enabled","V":"true"},
 {"OP":"invoke","PATH":"/Demo/BatchLinker","N":"linkAll"}]
```

Returns `linked 3 failed 0`; writing 111/222/333 to the sources yields
111.0/222.0/333.0 on the targets.

**Two things that bite**

- The ORD must be complete. A bare name raises `Missing scheme name`.
- The target slot must exist on that type. `control:NumericWritable` exposes `in1`…`in16`
  and has **no** `in` slot — writing `in` returns `bad target slot 'in'`.
  `NumericPoint` only has `out` and cannot be a target at all.

---

## 2. NullShed family — hold, then release

When the input goes null, do not forward null downstream immediately (that triggers false
alarms). Hold the last valid value for `holdMillis`, and only after the timeout set `out`
to null. `shedCount` accumulates how many times that happened.

Use it to ride out comms dropouts on 4G or unstable networks.

Four types, identical behaviour, different data types:
`NullShedNumeric`, `NullShedBoolean`, `NullShedString`, `NullShedEnum`.

**Properties**

| Property | Meaning |
|---|---|
| `in` | Input — link this from another point rather than writing it |
| `out` | Output, becomes `{null}` after the hold expires |
| `holdMillis` | Hold duration, default 30000 ms; `0` releases immediately |
| `lastChange` | Timestamp of the last valid value received |
| `shedCount` | Number of times the output was released to null |
| `enabled` | When off, passes through without shedding |

**Actions:** `shedNow` (release immediately), `resetCounter`

**Example** — `holdMillis=5000`, write 88.5: `out` reads `88.50`; six seconds later
`out` is `{null}` and `shedCount=1`.

**Note on `NullShedEnum`:** on the tested station, `control:EnumWritable` rejects external
writes to its `StatusEnum` slot (`BDynamicEnum cannot be cast to BStatusEnum`). Supply the
value by link from a driver or parent component rather than writing it directly.

---

## 3. BitDecode16 — split a 16-bit word

Takes one integer and exposes 16 boolean points, the two bytes, and hex/binary text.
Typical use: a Modbus status word read once, then split into individually alarmable points.

**Properties**

| Property | Meaning |
|---|---|
| `in` | Integer to decode |
| `bitOrder` | `lsb0` (default, b0 = least significant) or `msb0` |
| `invert` | Invert every bit |
| `b0`…`b15` | Boolean outputs |
| `lowByte` / `highByte` | Byte values |
| `hexText` / `binText` | Text forms |

**Example** — `in=12345` (`0x3039`), `lsb0`: `lowByte=57`, `highByte=48`,
`hexText=0x3039`, `binText=0011000000111001`, true bits b0/b3/b4/b5/b12/b13.
Switching to `msb0` flips the bit order as expected.

---

## 4. Sort7 — rank 7 inputs

Sorts up to 7 numeric inputs and tells you which channel each result came from — useful for
picking the most-loaded chiller out of seven and knowing which one it was.

**Properties**

| Property | Meaning |
|---|---|
| `in1`…`in7` | Inputs — link these from other points |
| `activeCount` | How many inputs participate (1..7) |
| `descending` | `true` for descending order |
| `holdMillis` | Stability wait; default 30000 — set `0` while testing |
| `out1`…`out7` | Sorted results |
| `outIndex1`…`outIndex7` | Original input number for each result |
| `status` | e.g. `sorted 4 of 4` |

**Example** — inputs 52/12/92/32 ascending gives `12,32,52,92` with indices `2,4,1,3`;
`descending=true` gives `92,52,32,12` with indices `3,1,4,2`.

**Watch the default `holdMillis`** — at 30 s you must wait half a minute after changing an
input before results move. Set it to `0` while testing.

---

## 5. EnumBits — enum set ↔ bit mask

Converts in both directions using bit arithmetic on ordinals 0..31 — no string round-trip.

**Properties**

| Property | Meaning |
|---|---|
| `direction` | `e2n` (enum → mask) or `n2e` (mask → enum) |
| `enumIn` / `numIn` | Input side |
| `numOut` / `enumOut` | Output side |
| `enumInTags` / `enumOutTags` | String form, e.g. `bit0,bit2` — accepts `0,2` too |
| `lastError` | Error text |

**Example** — `n2e`: `numIn=5` → `enumOut` bits 0 and 2. `e2n`: `enumIn` bits 0,1,3 →
`numOut=11`.

**Why the two tag slots exist:** Niagara cannot write `BEnumSet` properties from outside —
REST casts the value to `BEnumRange` and oBIX to `BIEnum`, both of which fail against
`BEnumSet`. The `enumInTags` slot takes plain text (`bit0,bit2` or `0,2`) and parses it,
so any channel can drive the component.

---

## Calling the components over REST API v2

All slots are readable and writable through the station's REST API v2 service.

```
Service  Services/RestApiServiceV2   (enable=true)
Endpoint POST http://<host>:8083/api/batch
Auth     HTTP Basic
Body     [ {"OP":"...","PATH":"...", ...} ]
```

**Operations**

| OP | Use |
|---|---|
| `add` | Create an object: `PATH` (parent) + `T` (type) + `N` (name) |
| `mod` | Set a slot: `PATH` + `N` (slot name) + `V` (value) |
| `invoke` | Run an action: `PATH` + `N` (action name) |
| `read` | Read a slot: `PATH` + `N` |
| `list` | List children: `PATH` |
| `remove` | Delete an object: `PATH` |

**Two rules to remember**

1. `path` is relative to the station — `Demo/Sort7`, not `station:|slot:/Demo/Sort7`.
2. The value field is `V`, not `NAME` (using `NAME` returns `empty String`).

**Example** — configure EnumBits:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '[{"OP":"mod","PATH":"/Demo/EnumBits","N":"enabled","V":true},
       {"OP":"mod","PATH":"/Demo/EnumBits","N":"direction","V":"e2n"},
       {"OP":"mod","PATH":"/Demo/EnumBits","N":"enumInTags","V":"bit0,bit2"}]' \
  "http://USER:PASSWORD@192.0.2.10:8083/api/batch"
```

`numOut` becomes `5` and `enumIn` reads back as `bit0, bit2`.

---

## Reading values over oBIX

```
GET  http://<host>/obix/config/Demo/Sort7/out1/
Auth HTTP Basic
```

**Limitation:** `BStatus*` slots such as `in` and `out` cannot be PUT directly — they must
be fed by a link. Read them, do not write them.

---

## Common pitfalls

| Symptom | Cause |
|---|---|
| `Missing scheme name` | ORD given as a bare name instead of `station:|slot:/...` |
| `bad target slot 'in'` | That type has no `in`; `NumericWritable` uses `in1`…`in16` |
| `empty String` | Used `NAME` instead of `V` in a REST `mod` |
| `BEnumRange cannot be cast to BEnumSet` | Writing a `BEnumSet` slot over REST — use `enumInTags` |
| `BEnumSet cannot be cast to BIEnum` | Same, over oBIX — use `enumInTags` |
| Sort output never moves | `holdMillis` still at its 30 s default |
| Module not visible after copy | Jar present in only one of the two required locations |

---

## License

MIT — see [LICENSE](LICENSE).

Free to use, modify and redistribute, including in commercial projects.
