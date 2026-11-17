# glinenetTools

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A free, MIT-licensed Niagara 4 module with **8 utility components** for building automation
integration work.

Each component solves one recurring job: splitting a status word into alarms, picking the
worst of seven chillers, mapping a Modbus bit mask to a Niagara enum, riding out a 4G dropout
without firing a false alarm, or wiring forty points without dragging forty links by hand.

---

## Download

`dist/glinenetTools-rt-signed.jar`

| Item | Value |
|---|---|
| Niagara | 4.14 (`baja` dependency) |
| Java | 8 bytecode |
| Module name | `glinenetTools` |
| Package | `com.glinenet.tools` |
| Palette folder | `GlineNetTools` |
| Type prefix | `gnt:` |

---

## Where the components live

All eight appear under the **`GlineNetTools`** folder in the Workbench palette.

![Palette overview](docs/images/P0_palette_tree.png)

Drag a component from that folder into a Wire Sheet to create an instance.

---

## The eight components

![Wire Sheet overview](docs/images/00_wiresheet.png)

*All eight placed in one folder. Each section below shows its palette entry, its Wire Sheet
appearance and its property sheet.*

| Type | Purpose | Category |
|---|---|---|
| [`BatchLinker`](#1-batchlinker--create-links-in-bulk) | Create point-to-point links in bulk | Efficiency |
| [`BitDecode16`](#2-bitdecode16--split-a-16-bit-word) | Split a 16-bit word into 16 booleans | Decoding |
| [`Sort7`](#3-sort7--rank-seven-inputs) | Sort 7 inputs, report which channel won | Selection |
| [`EnumBits`](#4-enumbits--enum-set--bit-mask) | Convert enum sets ↔ integer bit masks | Protocol mapping |
| [`NullShedNumeric`](#5-nullshednumeric--hold-then-release) | Hold last value, release after timeout | Comms debounce |
| [`NullShedBoolean`](#6-nullshedboolean--hold-then-release) | Same, boolean | Comms debounce |
| [`NullShedString`](#7-nullshedstring--hold-then-release) | Same, string | Comms debounce |
| [`NullShedEnum`](#8-nullshedenum--hold-then-release) | Same, enum | Comms debounce |

---

## 1. BatchLinker — create links in bulk

### What it is for

You have forty source points and forty target points that need connecting one to one.
Dragging forty links in the Wire Sheet is slow and easy to mis-pair. `BatchLinker` takes two
ORD queries, resolves them into two component sets, creates every link, and tells you how
many succeeded — plus the reason for any that failed.

### Palette

![BatchLinker in palette](docs/images/P_01_BatchLinker_palette.png)

### Wire Sheet

![BatchLinker wire sheet](docs/images/01_BatchLinker_wiresheet.png)

### Property sheet

![BatchLinker properties](docs/images/08_batchlinker.png)

### Properties

| Property | Meaning |
|---|---|
| `sourceOrd` | ORD of the source set, e.g. `station:|slot:/Demo/SrcGrp` |
| `targetOrd` | ORD of the target set, same form |
| `sourceSlot` | Slot on each source component, normally `out` |
| `targetSlot` | Slot on each target component, e.g. `in1` |
| `matchMode` | `name` = pair components with the same name (default); `order` = pair by position |
| `removeExisting` | Unlink existing links before creating new ones |
| `enabled` | Must be `true` |

### Actions

| Action | Effect |
|---|---|
| `linkAll` | Resolve both ORDs and create all links |
| `unlinkAll` | Remove the links it created |
| `clearLog` | Clear `detailLog` |

### Results to read back

| Property | Meaning |
|---|---|
| `status` | e.g. `linked 3 failed 0` |
| `linkedCount` / `failedCount` | Counts |
| `detailLog` | Per-pair detail, including the exact error on a failure |

### How to use it

1. Put your source components in one folder and your targets in another. Name them so the
   pairs match (e.g. `AHU1_SP` → `AHU1_SP`), or use `matchMode=order` if sequence is all
   that matters.
2. Set `sourceOrd` and `targetOrd` to the **folder** ORDs. This form expands to every child
   automatically.
3. Set `sourceSlot=out` and `targetSlot=in1`, or whichever slot your target type exposes.
4. Set `enabled=true`, then invoke `linkAll`.
5. Read `status`. If it reports `failed N`, open `detailLog` — it names the component and the
   reason.

Via REST:

```json
[{"OP":"mod","PATH":"/Demo/BatchLinker","N":"sourceOrd","V":"station:|slot:/Demo/SrcGrp"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"targetOrd","V":"station:|slot:/Demo/DstGrp"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"sourceSlot","V":"out"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"targetSlot","V":"in1"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"matchMode","V":"name"},
 {"OP":"mod","PATH":"/Demo/BatchLinker","N":"enabled","V":"true"},
 {"OP":"invoke","PATH":"/Demo/BatchLinker","N":"linkAll"}]
```

### Tested result

Three sources (`SrcGrp` → n1/n2/n3) into three same-named targets (`DstGrp` → `in1`):

- `status` → `linked 3 failed 0`
- Writing 111 / 222 / 333 to the sources gave `111.0` / `222.0` / `333.0` on the targets

The link is only proven by the target reading the source value — a `linked` count alone does
not prove data flows.

### Three ORD forms, and which to use

| Form | Example | Works? |
|---|---|---|
| Single component | `station:\|slot:/Demo/GntTest/srcA` | Yes |
| **Folder (recommended)** | `station:\|slot:/Demo/SrcGrp` | Yes — expands to all children |
| BQL | `bql:select * from ...` | Returns 0 on tested stations — avoid |

### Pitfalls

- **The ORD must be complete.** A bare name like `SrcA` raises `Missing scheme name`.
- **The target slot must exist on that type.** `control:NumericWritable` exposes `in1`…`in16`
  and has **no** `in` slot. `control:NumericPoint` only has `out` and cannot be a target.

---

## 2. BitDecode16 — split a 16-bit word

### What it is for

A Modbus device returns one 16-bit status word where each bit means something different:
bit 0 = running, bit 3 = alarm, bit 10 = filter dirty. Downstream logic needs each bit as its
own boolean point so it can be alarmed, trended and interlocked separately.

### Palette

![BitDecode16 in palette](docs/images/P_02_BitDecode16_palette.png)

### Wire Sheet

![BitDecode16 wire sheet](docs/images/02_BitDecode16_wiresheet.png)

### Property sheet

![BitDecode16 properties](docs/images/01_bitdecode16.png)

### Properties

| Property | Meaning |
|---|---|
| `in` | The integer to decode |
| `bitOrder` | `lsb0` (default, `b0` = least significant bit) or `msb0` |
| `invert` | Invert every output bit |
| `b0` … `b15` | The 16 boolean outputs |
| `lowByte` / `highByte` | The two bytes as integers |
| `hexText` / `binText` | Text forms, useful for point-sheet checking |

### How to use it

1. Link your Modbus point's `out` to `BitDecode16.in`. Do not write `in` directly — let the
   driver feed it.
2. Leave `bitOrder=lsb0` unless the device documentation numbers its bits the other way.
3. Expand the component in the Wire Sheet. `b0`…`b15` are now individual points you can link
   into alarm extensions, trend extensions or other logic.
4. To verify your wiring, compare `hexText` against the raw Modbus value.

### Tested result

`in = 12345` (`0x3039`), `bitOrder = lsb0`:

| Output | Value |
|---|---|
| `lowByte` | 57 |
| `highByte` | 48 |
| `hexText` | `0x3039` |
| `binText` | `0011000000111001` |
| true bits | b0, b3, b4, b5, b12, b13 |

`0x3039` = `0011 0000 0011 1001`; bits 0, 3, 4, 5, 12, 13 are set — matches exactly.

### Pitfalls

- If the vendor documents bit 0 as the **most** significant bit, set `bitOrder=msb0`.
  Getting this wrong mirrors all sixteen alarms.
- `invert` inverts the whole word, not individual bits.

---

## 3. Sort7 — rank seven inputs

### What it is for

Seven chillers report their load. You need the most-loaded one — and you need to know
**which** chiller it is. Sorting the numbers alone loses that information, so `Sort7` also
outputs the original channel number for every result.

### Palette

![Sort7 in palette](docs/images/P_03_Sort7_palette.png)

### Wire Sheet

![Sort7 wire sheet](docs/images/03_Sort7_wiresheet.png)

### Property sheet

![Sort7 properties](docs/images/02_sort7.png)

### Properties

| Property | Meaning |
|---|---|
| `in1` … `in7` | Up to seven inputs — link these, don't write them |
| `activeCount` | How many inputs participate (1–7) |
| `descending` | `true` = largest first |
| `holdMillis` | Stability wait before results update; **default 30000** |
| `out1` … `out7` | Sorted values |
| `outIndex1` … `outIndex7` | Which input number each result came from |
| `status` | e.g. `sorted 4 of 4` |

### How to use it

1. Link your seven values to `in1`…`in7`.
2. Set `activeCount` to the real number of inputs. Leaving it at 7 while only 4 are wired
   means three nulls participate and the sorted output is misleading.
3. **Set `holdMillis=0` while commissioning.** The 30-second default makes the component look
   broken — you change an input and nothing moves for half a minute.
4. Set `descending` to match what the downstream logic expects.
5. Link `out1` to whatever needs the extreme value, and `outIndex1` to whatever needs the
   device identity.

### Tested result

Inputs 52 / 12 / 92 / 32, `activeCount=4`:

| Setting | `out1..out4` | `outIndex1..outIndex4` |
|---|---|---|
| ascending | 12, 32, 52, 92 | 2, 4, 1, 3 |
| `descending=true` | 92, 52, 32, 12 | 3, 1, 4, 2 |

The smallest value 12 came from `in2`, the largest 92 from `in3` — the index output is what
makes the result actionable.

### Pitfalls

- **`holdMillis` defaults to 30 seconds.** The most common "it doesn't work" report.
- If `activeCount` is larger than the number of wired inputs, nulls count and can occupy
  `out1`.

---

## 4. EnumBits — enum set ↔ bit mask

### What it is for

A BACnet or Modbus device reports a set of states as one integer bit mask (value 37 means
bits 0, 2 and 5). Niagara wants to present that as a multi-select enum point. Sometimes you
need the reverse: the operator picks three enum members and you must write back the integer.

### Palette

![EnumBits in palette](docs/images/P_04_EnumBits_palette.png)

### Wire Sheet

![EnumBits wire sheet](docs/images/04_EnumBits_wiresheet.png)

### Property sheet

![EnumBits properties](docs/images/03_enumbits.png)

*`Enum In` reads back as named bits, not raw integers.*

### Properties

| Property | Meaning |
|---|---|
| `direction` | `e2n` (enum → number) or `n2e` (number → enum) |
| `enumIn` / `numIn` | Input side |
| `enumOut` / `numOut` | Output side |
| `enumInTags` / `enumOutTags` | Text form such as `bit0,bit2`; also accepts `0,2` |
| `lastError` | Error text when a conversion fails |

### How to use it

1. Choose `direction`:
   - Device gives you a mask, Niagara needs an enum → **`n2e`**
   - Operator picks enums, device needs a mask → **`e2n`**
2. Link the input side from the driver.
3. For the enum side, select bits from the dropdown — the component exposes a named range
   `bit0`…`bit31`, so Workbench shows names rather than numbers.
4. Check `lastError` if the output stays empty.

### Tested result

| Direction | Input | Output |
|---|---|---|
| `n2e` | `numIn = 37` | `enumOut` = bits 0, 2, 5 |
| `n2e` | `numIn = 5` | `enumOut` = bits 0, 2 |
| `e2n` | `enumIn` = bits 0, 1, 3 | `numOut = 11` |
| `e2n` | `enumInTags = "bit0,bit2"` | `numOut = 5` |

`37 = 0b100101` → bits 0, 2, 5. Correct.

### Why `enumInTags` / `enumOutTags` exist

Niagara cannot write a `BEnumSet` property from outside. REST casts the incoming value to
`BEnumRange` and oBIX to `BIEnum`; both fail the cast against `BEnumSet`. This is a platform
conversion-chain limit, not a defect in the component.

So the component also exposes plain **string** slots. Write `bit0,bit2` (or `0,2`) and it
parses them. Any channel — REST, oBIX, a program, a link from another point — can drive the
component through these two slots.

### Pitfalls

- Writing the enum property directly over REST returns
  `BEnumRange cannot be cast to BEnumSet`; over oBIX, `BEnumSet cannot be cast to BIEnum`.
- Ordinals are 0–31 only.

---

## 5. NullShedNumeric — hold, then release

### What it is for

A 4G-connected device drops out for 30 seconds and the raw point goes null. If that null
reaches your alarm logic immediately you get a false "sensor failed" alarm and a nuisance
ticket — every time the modem hiccups.

`NullShedNumeric` keeps the last valid value flowing for `holdMillis`, and only releases a
null once the timeout really expires.

### Palette

![NullShedNumeric in palette](docs/images/P_05_NullShedNumeric_palette.png)

### Wire Sheet

![NullShedNumeric wire sheet](docs/images/05_NullShedNumeric_wiresheet.png)

### Property sheet

![NullShedNumeric properties](docs/images/04_shednumeric.png)

### Properties

| Property | Meaning |
|---|---|
| `in` | Input — link from the driver, don't write it |
| `out` | Output; becomes `{null}` once the hold expires |
| `holdMillis` | How long to keep the last value; default 30000 ms. `0` = release immediately |
| `lastChange` | Timestamp of the last valid value received |
| `shedCount` | How many times the output has been released to null |
| `enabled` | When false, passes through without shedding |

### Actions

| Action | Effect |
|---|---|
| `shedNow` | Release to null immediately, ignoring the timer |
| `resetCounter` | Zero `shedCount` |

### How to use it

1. Link the driver's `out` into `in`.
2. Set `holdMillis` slightly longer than the worst dropout you want to ride out. A 4G device
   that drops for ~30 s calls for 45000–60000 ms.
3. Link `out` onward to your alarm / trend / logic.
4. Watch `shedCount` over a week — that is your real dropout count, and a good health metric.

### Tested result

`holdMillis = 5000`, value 88.5 written:

- `out` reads `88.50` immediately
- six seconds later `out` is `{null}` and `shedCount = 1`

### Pitfalls

- **`holdMillis=0` disables the hold** — the component becomes a pass-through.
- If `enabled=false`, no shedding happens at all.

---

## 6. NullShedBoolean — hold, then release

Same behaviour as `NullShedNumeric`, for boolean inputs.

### Palette

![NullShedBoolean in palette](docs/images/P_06_NullShedBoolean_palette.png)

### Wire Sheet

![NullShedBoolean wire sheet](docs/images/06_NullShedBoolean_wiresheet.png)

### Property sheet

![NullShedBoolean properties](docs/images/05_shedboolean.png)

### Properties and actions

Identical to [`NullShedNumeric`](#5-nullshednumeric--hold-then-release): `in`, `out`,
`holdMillis`, `lastChange`, `shedCount`, `enabled`; actions `shedNow`, `resetCounter`.

### How to use it

Link a boolean point's `out` into `in`, set `holdMillis`, and feed `out` onward. A typical
use is a run-status point on a 4G device: the last known state is held through a dropout
instead of flipping to null and alarming.

### Tested result

`holdMillis = 4000`, `true` written: `out` reads `true`, then `{null}` after four seconds,
`shedCount = 1`.

---

## 7. NullShedString — hold, then release

Same behaviour as `NullShedNumeric`, for string inputs.

### Palette

![NullShedString in palette](docs/images/P_07_NullShedString_palette.png)

### Wire Sheet

![NullShedString wire sheet](docs/images/07_NullShedString_wiresheet.png)

### Property sheet

![NullShedString properties](docs/images/06_shedstring.png)

### Properties and actions

Identical to [`NullShedNumeric`](#5-nullshednumeric--hold-then-release).

### How to use it

Use it on text points — device model strings, fault messages, batch codes — where a null
would blank an operator display.

### Tested result

`holdMillis = 4000`, `"RUNNING"` written: `out` reads `RUNNING`, then `{null}` after four
seconds, `shedCount = 1`.

---

## 8. NullShedEnum — hold, then release

Same behaviour as `NullShedNumeric`, for enum inputs.

### Palette

![NullShedEnum in palette](docs/images/P_08_NullShedEnum_palette.png)

### Wire Sheet

![NullShedEnum wire sheet](docs/images/08_NullShedEnum_wiresheet.png)

### Property sheet

![NullShedEnum properties](docs/images/07_shedenum.png)

### Properties and actions

Identical to [`NullShedNumeric`](#5-nullshednumeric--hold-then-release).

### How to use it

Link the enum point's `out` into `in`. **Feed it by link, not by write** — see the pitfall
below.

### Tested result

Logic verified: the hold timer and `shedNow` behave exactly as in the numeric version.

### Pitfalls

- On the tested station, `control:EnumWritable` rejects external writes to its `StatusEnum`
  slot with `BDynamicEnum cannot be cast to BStatusEnum`. Feed `NullShedEnum.in` by **link**
  from a driver or parent component rather than writing it directly.

---

## Driving the components from REST

```
Endpoint POST http://<host>:8083/api/batch
Auth     HTTP Basic
Body     [ {"OP":"...","PATH":"...", ...} ]
```

| OP | Use |
|---|---|
| `add` | Create an object: `PATH` (parent) + `T` (type) + `N` (name) |
| `mod` | Set a slot: `PATH` + `N` (slot name) + `V` (value) |
| `invoke` | Run an action: `PATH` + `N` (action name) |
| `read` | Read a slot: `PATH` + `N` |
| `list` | List children: `PATH` |
| `remove` | Delete an object: `PATH` |

**Two rules that catch everyone**

1. `path` is relative to the station — `Demo/Sort7`, not `station:|slot:/Demo/Sort7`.
2. The value field is `V`, not `NAME`. Using `NAME` returns `empty String`.

**Example — set EnumBits to the reverse direction**

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '[{"OP":"mod","PATH":"/Demo/EnumBits","N":"enabled","V":true},
       {"OP":"mod","PATH":"/Demo/EnumBits","N":"direction","V":"e2n"},
       {"OP":"mod","PATH":"/Demo/EnumBits","N":"enumInTags","V":"bit0,bit2"}]' \
  "http://USER:***@192.0.2.10:8083/api/batch"
```

`numOut` becomes `5`, and `enumIn` reads back as `bit0, bit2`.

---

## Reading values over oBIX

```
GET  http://<host>/obix/config/Demo/Sort7/out1/
Auth HTTP Basic
```

`BStatus*` slots such as `in` and `out` cannot be PUT directly — they must be fed by a link.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Missing scheme name` | ORD given as a bare name | Use the full `station:\|slot:/...` form |
| `bad target slot 'in'` | That type has no `in` | `NumericWritable` uses `in1`…`in16` |
| `empty String` | Used `NAME` instead of `V` | Rename the field to `V` |
| `BEnumRange cannot be cast to BEnumSet` | Writing a `BEnumSet` slot over REST | Use `enumInTags` |
| `BEnumSet cannot be cast to BIEnum` | Same, over oBIX | Use `enumInTags` |
| Sort output never moves | `holdMillis` at its 30 s default | Set `holdMillis = 0` while testing |
| Values dropped after upgrade | Platform discards links on jar replace | Re-link the affected points |

---

## License

MIT — see [LICENSE](LICENSE).

Free to use, modify and redistribute, including in commercial projects.
