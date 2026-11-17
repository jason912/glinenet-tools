# glinenetTools

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A free, MIT-licensed Niagara 4 module with **8 utility components** for building automation
integration work.

Each component solves one recurring job in BAS engineering: splitting a status word into
alarms, picking the worst of seven chillers, mapping a Modbus bit mask to a Niagara enum,
riding out a 4G dropout without firing a false alarm, or wiring forty points without dragging
forty links by hand.

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

Palette folder `GlineNetTools` — drag any component from it into your station.

---

## The components in one view

![Wire Sheet overview](docs/images/00_wiresheet.png)

*All eight components in one folder. `BatchLinker` in the middle; the two numeric constants
and two numeric writables on the right are the demo set used for the link test.*

| Type | Purpose | Category |
|---|---|---|
| [`BatchLinker`](#1-batchlinker--create-links-in-bulk) | Create point-to-point links in bulk from ORD queries | Efficiency |
| [`BitDecode16`](#2-bitdecode16--split-a-16-bit-word) | Split a 16-bit word into 16 booleans | Decoding |
| [`Sort7`](#3-sort7--rank-seven-inputs) | Sort 7 inputs, report which channel won | Selection |
| [`EnumBits`](#4-enumbits--enum-set--bit-mask) | Convert enum sets ↔ integer bit masks | Protocol mapping |
| [`NullShedNumeric`](#5-nullshed-family--hold-then-release) | Hold last value, release to null after timeout | Comms debounce |
| [`NullShedBoolean`](#5-nullshed-family--hold-then-release) | Same, boolean | Comms debounce |
| [`NullShedString`](#5-nullshed-family--hold-then-release) | Same, string | Comms debounce |
| [`NullShedEnum`](#5-nullshed-family--hold-then-release) | Same, enum | Comms debounce |

---

## 1. BatchLinker — create links in bulk

### What it is for

You have forty source points and forty target points that need connecting one to one.
Dragging forty links in the Wire Sheet is slow and easy to mis-pair. `BatchLinker` takes two
ORD queries, resolves them into two component sets, creates every link, and tells you how
many succeeded — plus the reason for any that failed.

### Screenshot

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
2. Set `sourceOrd` and `targetOrd` to the **folder** ORDs. This form is recommended — it
   expands to every child automatically.
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

- **The ORD must be complete.** A bare name like `SrcA` raises `Missing scheme name`. Write
  the full `station:|slot:/...` form.
- **The target slot must exist on that type.** `control:NumericWritable` exposes `in1`…`in16`
  and has **no** `in` slot — writing `in` returns `bad target slot 'in' on DstA`.
  `control:NumericPoint` only has `out`, so it cannot be a target at all.

---

## 2. BitDecode16 — split a 16-bit word

### What it is for

A Modbus device returns one 16-bit status word where each bit means something different:
bit 0 = running, bit 3 = alarm, bit 10 = filter dirty. Downstream logic needs each bit as its
own boolean point so it can be alarmed, trended and interlocked separately.

`BitDecode16` takes one integer and exposes 16 separate booleans, plus both bytes and the
hex/binary text for diagnostics.

### Screenshot

![BitDecode16 properties](docs/images/01_bitdecode16.png)

### Properties

| Property | Meaning |
|---|---|
| `in` | The integer to decode |
| `bitOrder` | `lsb0` (default, `b0` = least significant bit) or `msb0` (`b0` = most significant) |
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
Switching to `msb0` reverses the bit numbering as expected.

### Pitfalls

- If the vendor documents bit 0 as the **most** significant bit, set `bitOrder=msb0`.
  Getting this wrong mirrors all sixteen alarms.
- `invert` inverts the whole word, not individual bits. Use it only when the device reports
  active-low.

---

## 3. Sort7 — rank seven inputs

### What it is for

Seven chillers report their load. You need the most-loaded one — and you need to know
**which** chiller it is. Sorting the numbers alone loses that information, so `Sort7` also
outputs the original channel number for every result.

The same component works for plant sequencing, "highest demand wins" logic, or finding the
worst sensor in a group.

### Screenshot

![Sort7 properties](docs/images/02_sort7.png)

### Properties

| Property | Meaning |
|---|---|
| `in1` … `in7` | Up to seven inputs — link these, don't write them |
| `activeCount` | How many inputs actually participate (1–7) |
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

- **`holdMillis` defaults to 30 seconds.** This is the single most common "it doesn't work"
  report. Set it to `0` when testing.
- If `activeCount` is larger than the number of wired inputs, the nulls still count and can
  occupy `out1`.

---

## 4. EnumBits — enum set ↔ bit mask

### What it is for

A BACnet or Modbus device reports a set of states as one integer bit mask (value 37 means
bits 0, 2 and 5). Niagara wants to present that as a multi-select enum point. Sometimes you
need the reverse: the operator picks three enum members and you must write back the integer.

`EnumBits` converts both ways using bit arithmetic on ordinals 0–31 — no string round-trip,
so it stays fast and cannot mangle names.

### Screenshot

![EnumBits properties](docs/images/03_enumbits.png)

*`Enum In` reads back as `bit0, bit3, bit7` — the named range is live, not raw integers.*

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
  Use the tag slots instead.
- Ordinals are 0–31 only. Bit 32 and above are not represented.

---

## 5. NullShed family — hold, then release

### What it is for

A 4G-connected device drops out for 30 seconds and the raw point goes null. If that null
reaches your alarm logic immediately you get a false "sensor failed" alarm, an operator phone
call, and a nuisance ticket — every time the modem hiccups.

`NullShedNumeric`, `NullShedBoolean`, `NullShedString` and `NullShedEnum` keep the last valid
value flowing for `holdMillis`, and only release a null once the timeout really expires. They
also count how many times that happened, so you can see the dropout rate.

### Screenshots

![NullShedNumeric](docs/images/04_shednumeric.png)

![NullShedBoolean](docs/images/05_shedboolean.png)

![NullShedString](docs/images/06_shedstring.png)

![NullShedEnum](docs/images/07_shedenum.png)

### Properties — identical across the four types

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

1. Pick the type matching your point: numeric, boolean, string or enum.
2. Link the driver's `out` into `in`.
3. Set `holdMillis` slightly longer than the worst dropout you want to ride out. A 4G device
   that drops for ~30 s calls for 45000–60000 ms.
4. Link `out` onward to your alarm / trend / logic. Everything downstream now sees a steady
   value through short dropouts.
5. Watch `shedCount` over a week — that is your real dropout count, and a good health metric.

### Tested result

`holdMillis = 5000`, value 88.5 written:

- `out` reads `88.50` immediately
- six seconds later `out` is `{null}` and `shedCount = 1`

Both paths were verified: `shedNow` releases instantly, and the timer releases on its own —
the automatic path is the one production depends on.

### Pitfalls

- **`holdMillis=0` disables the hold.** The component becomes a pass-through. Useful for
  testing, wrong for production.
- If `enabled=false`, no shedding happens at all.
- **`NullShedEnum` and writable enum points:** on the tested station, `control:EnumWritable`
  rejects external writes to its `StatusEnum` slot (`BDynamicEnum cannot be cast to
  BStatusEnum`). Feed `NullShedEnum.in` by **link** from a driver or parent component rather
  than writing it directly.

---

## Driving the components from REST

Every slot is readable and writable through the station's REST API v2, which is how you
configure forty similar points in one call instead of forty dialogs.

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
Read them; don't write them.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Missing scheme name` | ORD given as a bare name | Use the full `station:\|slot:/...` form |
| `bad target slot 'in'` | That type has no `in` | `NumericWritable` uses `in1`…`in16`; `NumericPoint` cannot be a target |
| `empty String` | Used `NAME` instead of `V` | Rename the field to `V` |
| `BEnumRange cannot be cast to BEnumSet` | Writing a `BEnumSet` slot over REST | Use `enumInTags` |
| `BEnumSet cannot be cast to BIEnum` | Same, over oBIX | Use `enumInTags` |
| Sort output never moves | `holdMillis` at its 30 s default | Set `holdMillis = 0` while testing |
| Values dropped after upgrade | Platform discards links on jar replace | Re-link the affected points |

---

## License

MIT — see [LICENSE](LICENSE).

Free to use, modify and redistribute, including in commercial projects.
