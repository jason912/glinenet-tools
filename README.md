# glinenetTools

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Free, MIT-licensed Niagara 4 module with 8 utility components for building automation integration work.

Built for the repetitive parts of BAS engineering: decoding status words, ranking many inputs,
mapping bit masks to enums, holding values through comms dropouts, and wiring dozens of points at once.

---

## Download

`dist/glinenetTools-rt-signed.jar` — signed, ready to deploy.

---

## Components

| Type | What it does |
|---|---|
| `BatchLinker` | Resolves source/target component sets by ORD and creates point-to-point links in bulk |
| `BitDecode16` | Splits a 16-bit word into 16 boolean points plus low/high byte, hex and binary text |
| `Sort7` | Sorts up to 7 numeric inputs and reports which channel each result came from |
| `EnumBits` | Converts between enum sets and integer bit masks, both directions |
| `NullShedNumeric` | Holds the last valid value when input goes null, then releases after a timeout |
| `NullShedBoolean` | Same for boolean |
| `NullShedString` | Same for string |
| `NullShedEnum` | Same for enum |

All types appear in the `glinenetTools` palette folder in Workbench.

---

## Installation

1. Put `glinenetTools-rt.jar` in **both** locations:
   - `<Niagara>/modules/glinenetTools-rt.jar`
   - `<Niagara>/sw/1.0/glinenetTools-rt.jar`
2. Verify both copies have identical MD5:
   ```
   certutil -hashfile "<path>" MD5
   ```
3. Restart the Niagara service:
   ```
   net stop Niagara
   net start Niagara
   ```
4. Confirm the module loaded: the `glinenetTools` palette folder appears in Workbench,
   or `GET /obix/def/glinenetTools:BatchLinker/` returns 200.

**Upgrade note:** replacing the jar and restarting may drop existing links on created
components; re-link them after the upgrade.

---

## Compatibility

| Item | Value |
|---|---|
| Niagara baseline | 4.14 (`baja` dependency) |
| Java | 8 bytecode |
| Module name | `glinenetTools` |
| Package | `com.glinenet.tools` |

---

## Quick example

Configure `EnumBits` to convert an enum set into a bit mask, over REST API v2:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '[{"OP":"mod","PATH":"/Demo/EnumBits","N":"enabled","V":true},
       {"OP":"mod","PATH":"/Demo/EnumBits","N":"direction","V":"e2n"},
       {"OP":"mod","PATH":"/Demo/EnumBits","N":"enumInTags","V":"bit0,bit2"}]' \
  "http://USER:***@192.0.2.10:8083/api/batch"
```

`numOut` becomes `5`, and `enumIn` reads back as `bit0, bit2`.

---

## Notes and limitations

- **`BEnumSet` properties cannot be written from REST or oBIX.** The platform converts incoming
  values to `BEnumRange` (REST) or `BIEnum` (oBIX) and then fails the cast against `BEnumSet`.
  `EnumBits` therefore exposes `enumInTags` / `enumOutTags` as plain string slots that parse
  `bit0,bit2` or `0,2`, so any channel can drive the component.
- **`BatchLinker` needs full ORDs** — write `station:|slot:/Path/Name`, not a bare name.
- **Target slot names are type-specific.** For example `control:NumericWritable` exposes
  `in1`…`in16` and has no `in` slot.
- Module replacement requires a station restart, which briefly interrupts the station.

---

## License

MIT — see [LICENSE](LICENSE).

Free to use, modify and redistribute, including in commercial projects.
