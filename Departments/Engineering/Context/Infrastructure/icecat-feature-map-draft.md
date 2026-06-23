# Icecat → Shopify schema crosswalk (DRAFT seed for `tl_icecat_feature_map`)

> Phase-3 deliverable from the 2026-06-24 Icecat reconnaissance. Seeds `tl_icecat_feature_map` (keyed by stable global Icecat `Feature.ID`). Build the table from this when Phase 3 starts. **No credentials stored here.**

## Access facts
- **JSON live API** works with the Open Icecat `api_token` header (HTTP 200). Lookup by **Brand + ProductCode** confirmed; **GTIN often empty** in Icecat records → Brand+MPN is the more reliable match key for us (not GTIN as first assumed).
- Feature object (JSON): top-level `Type` (`dropdown|numerical|y_n|multi_dropdown|range|text`), `CategoryFeatureId`, `PresentationValue`, `RawValue`, `Feature.ID`, `Feature.Name.Value`, `Feature.Measure.Signs._` (unit). **Use `RawValue`** (PresentationValue sometimes rescaled, e.g. cooler height shown in cm but RawValue is mm).
- **Reference files (`/export/freexml/refs/` + the bulk index) require the Icecat PORTAL password (HTTP Basic, user `TechLoop`)** — NOT the api/content tokens. The portal password was provided by the owner (held in session; wire into the n8n `Icecat (Open)` credential at build). The JSON API alone was enough to draft this crosswalk.
- **Icecat CategoryIDs:** Memory Modules (RAM) = **911**; Computer Cases = **237**.
- **Error codes:** `9`/HTTP403 = Full-Icecat-only brand; `8`/HTTP404 = not in Icecat.

## Pilot coverage reality (measured)
Open Icecat covers very little of OUR pilot: **RAM 1/50** (Kingston only; Corsair/G.Skill/Crucial = Full-only). Cases: Gigabyte/MSI sponsored; **Antec (68) + Deepcool (47) status UNCONFIRMED — test before relying**; Corsair (48) Full-only; AeroCool absent (404). ⇒ **gemma4 is the primary value engine for the pilot; Icecat supplements covered SKUs.** Full Icecat subscription would unlock Corsair/G.Skill/Crucial/Fractal/NZXT/Thermaltake.

## RAM (Icecat cat 911) → schema
| schema attr | Feature.ID | Icecat name | type | norm notes |
|---|---|---|---|---|
| memory-technology | 427 | Internal memory type | dropdown | DDR4/DDR5… → direct |
| memory-form-factor | 7696 | Memory form factor | dropdown | 288-pin DIMM→DIMM; 260/204-pin SO-DIMM→SO-DIMM |
| memory-rank | 8216 (also 6579 chips org) | Memory ranking | dropdown | 1/2/4 → Single/Dual/Quad |
| error-correction-type | 1628 + 44495 + 38209 | ECC / On-Die ECC / Buffered type | y_n/y_n/multi | compose → ECC / ECC(On-Die) / Registered(RDIMM) / Unbuffered(UDIMM) |
| overclock-profile-support | 21116 + 31816 | Intel XMP / XMP version | y_n/dropdown | XMP only in Open data; AMD EXPO absent |
| memory-capacity (custom GB) | 11381 | Internal memory | numerical (GB) | RawValue direct |
| memory-speed (custom MHz) | 2900 pref, 37536 fallback | Mem clock / data transfer rate | numerical | 37536 in MT/s ≈ marketed MHz (DDR4-3200=3200); 2900 in MHz when present |
| cas-latency (custom int) | 1635 | CAS latency | dropdown | RawValue int |
| voltage (custom V) | 6266 | Memory voltage | multi_dropdown (V) | take primary |
| rgb-lighting (custom bool) | 3709 | Backlight | y_n | Y/N |
| color (custom text) | 1766 | Product colour | multi_dropdown | nullable (absent for many Kingston desktop modules) |
| color-pattern, material, power-source, connector-gender, battery-size, battery-type | — | NO MATCH | — | not carried by Icecat for RAM |

## Cases (Icecat cat 237) → schema
| schema attr | Feature.ID | Icecat name | type | norm notes |
|---|---|---|---|---|
| chassis-type | 771 | Form factor | dropdown | Midi Tower→Mid Tower; Mini-ITX Tower→Mini Tower |
| compatible-motherboard-form-factor | 5605 | Supported mobo form factors | multi | split ","; micro ATX→Micro-ATX; EATX→E-ATX |
| compatible-power-supply-form-factor | 23947 | Supported PSU form factors | multi | PS2→ATX |
| material | 898 | Material | multi | ABS synthetics→ABS Plastic; SECC→Steel |
| side-panel-style | 9645 + 35286 | Side window / Tempered glass | y_n/y_n | N→Solid; Y+glassN→Acrylic; Y+glassY→Tempered Glass |
| expansion-slots | 9650 | # expansion slots | numerical | int |
| max-gpu-length (custom mm) | 9649 | Max GPU length | numerical (mm) | use RawValue (mm) |
| max-cpu-cooler-height (custom mm) | 9648 | Max CPU cooler height | numerical (mm) | use RawValue (mm) |
| fan-count (custom int) | 8133 + 8150 + 8149 | Front/Rear/Top fans installed | dropdown | no total; parse "3x 120mm"→3 and sum |
| tempered-glass (custom bool) | 35286 | Tempered glass panel(s) | y_n | Y/N |
| usb-type-c-ports (custom int) | 22704 | USB-C ports qty | numerical | absent→0/null |
| rgb-lighting (custom bool) | 1861 (+8539 colour) | Illumination | y_n | Y + colour=Multi → true |
| color (custom text) | 1766 | Product colour | multi | take primary |
| color-pattern, lock-type, power-source, connector-gender | — | NO MATCH | — | not carried |
