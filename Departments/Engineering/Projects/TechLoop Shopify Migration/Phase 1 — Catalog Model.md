---
status: done
notion_page_id: 3568c3d3-c03e-8186-9617-f758d69c7385
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-19T15:49:26Z
---

# Phase 1 — Catalog Model

## Context

Resolved catalog model for the WooCommerce → Shopify migration, derived from the Phase 1 review workbook reviewed on 2026-05-04. The workbook is the source of truth for per-row decisions; this document is a consolidated summary for execution in Phase 2 (Catalog Data Migration).

Workbook: https://docs.google.com/spreadsheets/d/1GKHm5K_XiQUeOaUTn59MVeL6HTC0nq8CD3kZUy2X6gI/edit

---

## Categories

Shopify uses a flat collection model. Each surviving WC category becomes a Shopify Collection assigned to a Shopify Standard Product Taxonomy node (which auto-maps to Google taxonomy). 57 WC categories were reviewed: 33 direct, 15 merge, 9 no-match.

### Direct mappings (33)

| WC Category | Shopify Category Name | Shopify Category GID | Key Recommended Attributes |
|---|---|---|---|
| Antivirus & Security Software | Software > Computer Software > Antivirus & Security Software | `gid://shopify/TaxonomyCategory/so-1-1` | additional-security-software-features, license-type, operating-system |
| Cable Management | Electronics > Electronics Accessories > Cable Management | `gid://shopify/TaxonomyCategory/el-7-6` | color, connector-gender, material, usage-type |
| Cables & Connectors | Electronics > Electronics Accessories > Cables | `gid://shopify/TaxonomyCategory/el-7-7` | cable-housing-material, cable-shielding, color, connector-gender, device-interface |
| Computer Accessories | Electronics > Electronics Accessories > Computer Accessories | `gid://shopify/TaxonomyCategory/el-7-8` | color, compatible-device, connector-gender, material, power-source |
| Computer Cases | Electronics > Electronics Accessories > Computer Components > Desktop Computer & Server Cases | `gid://shopify/TaxonomyCategory/el-7-9-8` | chassis-type, color, compatible-motherboard-form-factor, expansion-slots, material, side-panel-style |
| Computer Speakers | Electronics > Audio > Audio Components > Speakers | `gid://shopify/TaxonomyCategory/el-2-2-10` | audio-output-channel, color, connectivity-technology, power-source, speaker-design |
| CPUs / Processors | Electronics > Electronics Accessories > Computer Components > Computer Processors | `gid://shopify/TaxonomyCategory/el-7-9-4` | processor-cores, processor-family, processor-socket, memory-technology, graphics-card-type |
| Desktop Computers | Electronics > Computers > Desktop Computers | `gid://shopify/TaxonomyCategory/el-6-3` | color, graphics-card-type, memory-technology, operating-system, processor-cores, processor-family, storage-drive-type-installed |
| Desktop Memory | Electronics > Electronics Accessories > Memory > RAM | `gid://shopify/TaxonomyCategory/el-7-12-3` | error-correction-type, memory-form-factor, memory-rank, memory-technology, overclock-profile-support |
| Fans, Heatsinks & Cooling | Electronics > Electronics Accessories > Computer Components > Computer System Cooling Parts | `gid://shopify/TaxonomyCategory/el-7-9-7` | color, connector-gender, connector-type, cooling-technology, material, power-source |
| Graphics Cards / GPU | Electronics > Electronics Accessories > Computer Components > I/O Cards & Adapters > Video Cards & Adapters | `gid://shopify/TaxonomyCategory/el-7-9-10-5` | bracket-profile, cooling-technology, device-interface, graphics-card-type, host-interface, video-resolution-supported |
| Hard Drives, SSD & Storage | Electronics > Electronics Accessories > Computer Components > Storage Devices > Hard Drives | `gid://shopify/TaxonomyCategory/el-7-9-14-4` | device-interface, hard-drive-form-factor, hard-drive-installation, storage-drive-interface-type, storage-media-type |
| Headsets & Microphones | Electronics > Audio > Audio Components > Headphones & Headsets | `gid://shopify/TaxonomyCategory/el-2-2-7` | audio-connectivity, color, connectivity-technology, headphone-style, microphone-type, noise-cancellation-technology |
| Keyboards | Electronics > Electronics Accessories > Computer Components > Input Devices > Keyboards | `gid://shopify/TaxonomyCategory/el-7-9-12-8` | color, keyboard-backlight-type, keyboard-format, keyboard-layout, keyboard-switch-type, keyboard-type, keycap-material |
| KVM / AV Switches | Electronics > Electronics Accessories > Computer Components > Input Devices > KVM Switches | `gid://shopify/TaxonomyCategory/el-7-9-12-9` | audio-connectivity, connectivity-technology, device-interface, mounting-type, usb-standard |
| Laptops & Notebooks | Electronics > Computers > Laptops | `gid://shopify/TaxonomyCategory/el-6-6` | color, display-resolution, display-technology, graphics-card-type, keyboard-type, operating-system, processor-cores, processor-family, storage-drive-type-installed |
| Mice | Electronics > Electronics Accessories > Computer Components > Input Devices > Mice & Trackballs | `gid://shopify/TaxonomyCategory/el-7-9-12-11` | battery-type, color, hand-side, mouse-technology, operating-system |
| Monitor & Screen Accessories | Electronics > Video > Video Accessories > Computer Monitor Accessories | `gid://shopify/TaxonomyCategory/el-17-5-2` | color, compatible-device, device-interface, material, vesa-mounting-pattern |
| Monitors | Electronics > Video > Computer Monitors | `gid://shopify/TaxonomyCategory/el-17-1` | adaptive-sync-technology, color, display-resolution, display-technology, monitor-shape, native-aspect-ratio, screen-curvature-rating, vesa-mounting-pattern |
| Motherboards | Electronics > Circuit Boards & Components > Printed Circuit Boards > Computer Circuit Boards > Motherboards | `gid://shopify/TaxonomyCategory/el-3-6-2-3` | bios-type, motherboard-chipset-family, motherboard-form-factor, motherboard-memory-features, multi-gpu-technology, processor-socket, storage-drive-interfaces-supported |
| Network Cables | Electronics > Electronics Accessories > Cables > Network Cables | `gid://shopify/TaxonomyCategory/el-7-7-4` | cable-jacket-rating, cable-shielding, color, connector-gender, ethernet-cable-category, network-cable-interface |
| On-ear Headphones | Electronics > Audio > Audio Components > Headphones & Headsets > Headphones > On-Ear Headphones | `gid://shopify/TaxonomyCategory/el-2-2-7-1-4` | audio-connectivity, color, connectivity-technology, headphone-style, noise-cancellation-technology |
| Power Supplies | Electronics > Electronics Accessories > Computer Components > Computer Power Supplies | `gid://shopify/TaxonomyCategory/el-7-9-3` | 80-plus-efficiency-rating, color, power-supply-form-factor, power-supply-modularity |
| Security Cameras & Surveillance | Cameras & Optics > Cameras > Surveillance Cameras | `gid://shopify/TaxonomyCategory/co-2-5` | camera-features, camera-sensor-type, color, connectivity-technology, surveillance-camera-design, surveillance-camera-mounting-type |
| Solid State Drives (SSD) | Electronics > Electronics Accessories > Computer Components > Storage Devices > Solid State Drives | `gid://shopify/TaxonomyCategory/el-7-9-14-9` | color, device-interface, storage-drive-interface-type |
| Tablets | Electronics > Computers > Tablet Computers | `gid://shopify/TaxonomyCategory/el-6-8` | battery-technology, cellular-capability, color, display-resolution, display-technology, operating-system, processor-family, stylus-support-type |
| Webcams | Cameras & Optics > Cameras > Webcams | `gid://shopify/TaxonomyCategory/co-2-8` | camera-features, camera-hd-type, camera-mounting-type, color, connectivity-technology, webcam-design |
| Wi-Fi Mesh Networking | Electronics > Networking > Bridges & Routers > Mesh WiFi Systems | `gid://shopify/TaxonomyCategory/el-12-1-6` | antenna-type, networking-standards, security-algorithms, wi-fi-band, wi-fi-standard |
| Wired Networking Switches | Electronics > Networking > Hubs & Switches | `gid://shopify/TaxonomyCategory/el-12-3` | color, ethernet-lan-interface-type, mounting-type, network-protocols-supported, poe-standard |
| Wireless Access Points | Electronics > Networking > Bridges & Routers > Wireless Access Points | `gid://shopify/TaxonomyCategory/el-12-1-3` | antenna-type, mesh-networking-technology, networking-standards, poe-standard, wi-fi-band, wi-fi-standard |
| Wireless Networking | Electronics > Networking > Bridges & Routers | `gid://shopify/TaxonomyCategory/el-12-1` | antenna-type, networking-standards, wi-fi-band, wi-fi-standard |
| Wireless Range Extenders | Electronics > Networking > Repeaters & Transceivers > Wi-Fi Range Extenders | `gid://shopify/TaxonomyCategory/el-12-10-3` | color, networking-standards, wi-fi-band, wi-fi-standard |
| Wireless Routers | Electronics > Networking > Bridges & Routers > Wireless Routers | `gid://shopify/TaxonomyCategory/el-12-1-4` | antenna-type, bridge-router-advanced-features, ethernet-lan-interface-type, mimo-technology, networking-standards, security-algorithms, wi-fi-band, wi-fi-standard |

### Merges into attributes (15)

These WC categories collapse into a broader Shopify Collection; the granularity that was formerly expressed by the category name is carried instead by an attribute filter.

| WC Category | Target Shopify Collection | Discriminating Attribute | Expected Value | Notes |
|---|---|---|---|---|
| AMD Socket AM4 | Motherboards (`gid://shopify/TaxonomyCategory/el-3-6-2-3`) | `processor-socket` | AMD AM4 | Generates smart collection "AMD AM4 Motherboards" |
| AMD Socket AM5 | Motherboards (`gid://shopify/TaxonomyCategory/el-3-6-2-3`) | `processor-socket` | AMD AM5 | Generates smart collection "AMD AM5 Motherboards" |
| Intel Socket 1700 (12th Gen) | Motherboards (`gid://shopify/TaxonomyCategory/el-3-6-2-3`) | `processor-socket` | Intel LGA1700 | Generates smart collection "Intel LGA1700 (12th Gen) Motherboards" |
| Intel Socket 1700 (13th Gen) | Motherboards (`gid://shopify/TaxonomyCategory/el-3-6-2-3`) | `processor-socket` | Intel LGA1700 | Generates smart collection "Intel LGA1700 (13th Gen) Motherboards" |
| Intel Socket 1851 (15th Gen) | Motherboards (`gid://shopify/TaxonomyCategory/el-3-6-2-3`) | `processor-socket` | Intel LGA1851 | Generates smart collection "Intel LGA1851 (15th Gen) Motherboards" |
| Gaming Headsets | Headsets (`gid://shopify/TaxonomyCategory/el-2-2-7-2`) | `recommended-use` (custom) | Gaming | Consolidates with "PC Gaming Headsets" WC category |
| PC Gaming Headsets | Headsets (`gid://shopify/TaxonomyCategory/el-2-2-7-2`) | `recommended-use` (custom) | Gaming | Consolidate both WC gaming headset categories into one Shopify collection |
| Gaming Keyboards | Keyboards (`gid://shopify/TaxonomyCategory/el-7-9-12-8`) | `recommended-use` (custom) | Gaming | Generates smart collection "Gaming Keyboards" |
| Gaming Mice | Mice & Trackballs (`gid://shopify/TaxonomyCategory/el-7-9-12-11`) | `recommended-use` (custom) | Gaming | Generates smart collection "Gaming Mice" |
| PC Gaming Mouse Pads | Mouse Pads (`gid://shopify/TaxonomyCategory/el-7-8-8`) | `recommended-use` (custom) | Gaming | Generates smart collection "Gaming Mouse Pads" |
| iPad & Tablet Accessories | Tablet Computers (`gid://shopify/TaxonomyCategory/el-6-8`) | `compatible-device` | iPad | Create separate smart collection for iPad filter |
| Laptop Accessories | Computer Accessories (`gid://shopify/TaxonomyCategory/el-7-8`) | `compatible-device` | Laptop | — |
| Laptop Bags & Cases | Luggage & Bags > Laptop Bags (`gid://shopify/TaxonomyCategory/lb-15`) | bag taxonomy | — | Luggage & Bags branch, not Electronics |
| Laptop Chargers & Adapters | Computer Accessories (`gid://shopify/TaxonomyCategory/el-7-8`) | `compatible-device` | Laptop | No dedicated Laptop Chargers taxonomy node in Shopify |
| Laptop Memory | RAM (`gid://shopify/TaxonomyCategory/el-7-12-3`) | `memory-form-factor` | SO-DIMM | Distinguishes laptop RAM from Desktop Memory (DIMM) |

### No-match — custom collections without Standard Taxonomy assignment (9)

- **Electronics** (WC 44, 58 products) — auto-recommendation was no-match (too broad); user queried this and noted `gid://shopify/TaxonomyCategory/el` exists. **Pending resolution**: assess whether to assign the top-level `el` GID or keep as an unassigned custom collection. See "Decisions and overrides" section below.
- **Motherboard Accessories** (WC 1736, 7 products) — no Shopify equivalent; create custom collection or merge into Computer Components with `compatible-device`=Motherboard.
- **Operating Systems** (WC 438, 1 product) — only 1 product; recommend custom collection or drop.
- **Powerline Networking** (WC 454, 1 product) — no Shopify standard category; custom collection or merge into Networking > Hubs & Switches.
- **Smart Home Automation** (WC 164, 1 product) — minimal stock; recommend custom collection.
- **Smart Lighting** (WC 165, 1 product) — only 1 product; recommend custom collection or drop.
- **Smart Power** (WC 166, 1 product) — no direct Shopify category; custom collection or merge into Smart Home.
- **Wired Networking Accessories** (WC 456, 1 product) — too niche; custom collection or merge into Network Cables / Computer Accessories.
- **Wireless Networking Accessories** (WC 450, 1 product) — too niche; custom collection or merge into Wireless Routers / Computer Accessories.

---

## Smart Collections

Smart collections cover two sources: (1) the merge buckets from Categories — each merge generates a smart collection with an attribute filter rule; and (2) the 1d_tags tab — each tag becomes a smart collection rule.

### From category merges (15)

| Smart Collection Name | Filters On | Filter Rule | Source WC Category |
|---|---|---|---|
| AMD AM4 Motherboards | Motherboards + attribute | `category = el-3-6-2-3 AND metafield.shopify.processor-socket = AM4` | AMD Socket AM4 |
| AMD AM5 Motherboards | Motherboards + attribute | `category = el-3-6-2-3 AND metafield.shopify.processor-socket = AM5` | AMD Socket AM5 |
| Intel LGA1700 (12th Gen) Motherboards | Motherboards + attribute | `category = el-3-6-2-3 AND metafield.shopify.processor-socket = LGA1700` | Intel Socket 1700 (12th Gen) |
| Intel LGA1700 (13th Gen) Motherboards | Motherboards + attribute | `category = el-3-6-2-3 AND metafield.shopify.processor-socket = LGA1700` | Intel Socket 1700 (13th Gen) |
| Intel LGA1851 (15th Gen) Motherboards | Motherboards + attribute | `category = el-3-6-2-3 AND metafield.shopify.processor-socket = LGA1851` | Intel Socket 1851 (15th Gen) |
| Gaming Headsets | Headsets + attribute | `category = el-2-2-7-2 AND metafield.custom.recommended-use = Gaming` | Gaming Headsets + PC Gaming Headsets |
| Gaming Keyboards | Keyboards + attribute | `category = el-7-9-12-8 AND metafield.custom.recommended-use = Gaming` | Gaming Keyboards |
| Gaming Mice | Mice & Trackballs + attribute | `category = el-7-9-12-11 AND metafield.custom.recommended-use = Gaming` | Gaming Mice |
| Gaming Mouse Pads | Mouse Pads + attribute | `category = el-7-8-8 AND metafield.custom.recommended-use = Gaming` | PC Gaming Mouse Pads |
| iPad Accessories | Tablet Computers + attribute | `category = el-6-8 AND metafield.shopify.compatible-device = iPad` | iPad & Tablet Accessories |
| Laptop Accessories | Computer Accessories + attribute | `category = el-7-8 AND metafield.shopify.compatible-device = Laptop` | Laptop Accessories |
| Laptop Bags | Laptop Bags + taxonomy | `category = lb-15` | Laptop Bags & Cases |
| Laptop Chargers | Computer Accessories + attribute | `category = el-7-8 AND metafield.shopify.compatible-device = Laptop AND product_type = Charger` | Laptop Chargers & Adapters |
| Laptop Memory | RAM + attribute | `category = el-7-12-3 AND metafield.shopify.memory-form-factor = SO-DIMM` | Laptop Memory |
| Desktop Memory | RAM + attribute | `category = el-7-12-3 AND metafield.shopify.memory-form-factor = DIMM` | Desktop Memory (disambiguation) |

### From WC tags (16)

| Smart Collection Name | Filters On | Filter Rule | Source WC Tag | Notes |
|---|---|---|---|---|
| DDR4 RAM | Memory > RAM | `category = el-7-12-3 AND metafield.shopify.memory-technology = memory-technology__ddr4` | DDR4 RAM | Canonical match. |
| Gaming Memory | Memory > RAM | tag = gaming-memory | Gaming Memory | Marketing/curated label — no canonical attribute maps to gaming; recommend manual curated collection or tag-based rule. |
| CORSAIR RAM | Memory > RAM | `category = el-7-12-3 AND vendor = "CORSAIR"` | CORSAIR RAM | Vendor filter — uses Shopify vendor field. |
| RGB Memory | Memory > RAM | `category = el-7-12-3 AND metafield.shopify.color = color__multicolor` | RGB Memory | No dedicated RGB lighting attribute in RAM taxonomy; `color: multicolor` is the best proxy — flag for manual review. |
| Crucial RAM | Memory > RAM | `category = el-7-12-3 AND vendor = "Crucial"` | Crucial RAM | Vendor filter. |
| CORSAIR Keyboards | Keyboards | `category = el-7-9-12-8 AND vendor = "CORSAIR"` | CORSAIR Keyboards | Vendor filter. |
| Mechanical Keyboards | Keyboards | `category = el-7-9-12-8 AND metafield.shopify.keyboard-type = keyboard-type__mechanical` | Mechanical Keyboards | Canonical match — maps to `keyboard-type`, not `keyboard-switch-type`. |
| RGB Keyboards | Keyboards | `category = el-7-9-12-8 AND metafield.shopify.keyboard-backlight-type = keyboard-backlight-type__rgb` | RGB Keyboards | Canonical match. Per-key RGB (`keyboard-backlight-type__per-key-rgb`) is more specific if preferred. |
| Black Keyboards | Keyboards | `category = el-7-9-12-8 AND metafield.shopify.color = color__black` | Black Keyboards | Canonical match. |
| RGB Mechanical Keyboards | Keyboards | `category = el-7-9-12-8 AND metafield.shopify.keyboard-type = keyboard-type__mechanical AND metafield.shopify.keyboard-backlight-type = keyboard-backlight-type__rgb` | RGB Mechanical Keyboards | Two-condition AND rule — both attributes canonical. |
| DDR3 RAM | Memory > RAM | `category = el-7-12-3 AND metafield.shopify.memory-technology = memory-technology__ddr3` | DDR3 RAM | Canonical match. |
| Low Profile Keyboards | Keyboards | `category = el-7-9-12-8 AND metafield.shopify.keycap-profile = keycap-profile__low-profile` | Low Profile Keyboards | Partial match — `keycap-profile__low-profile` applies to keycaps, not keyboard form factor. Recommend custom metafield or manual curation. |
| Silver Keyboards | Keyboards | `category = el-7-9-12-8 AND metafield.shopify.color = color__silver` | Silver Keyboards | Canonical match. |
| Gold Keyboards | Keyboards | `category = el-7-9-12-8 AND metafield.shopify.color = color__gold` | Gold Keyboards | Canonical match. |
| White Keyboards | Keyboards | `category = el-7-9-12-8 AND metafield.shopify.color = color__white` | White Keyboards | Canonical match. |
| Blue/other Keyboards | (remaining colour filters) | Per colour value | (remaining colour tags) | — |

---

## Brands

The vendor field on each Shopify product carries the brand name. A Brand metaobject (with a `parent_brand` self-reference field) is created for each brand to enable richer brand pages and filtering.

52 brands in the WC brand taxonomy are mapped to a Shopify vendor + Brand metaobject:

AMD, Antec, AOC Monitors, Astrotek, ASUS, Aten, Belkin, Brateck, Brother (0 products — WC taxonomy entry only), CORSAIR, Cygnett, Deepcool, Edifier, G.Skill, GIGABYTE, Google (0 products — WC taxonomy entry only), HP, Inno3D, Intel, Kingston, Leadtek, Lenovo, Lexar, LG, Logitech, mbeat, Microsoft, MSI, PowerShield (0 products — WC taxonomy entry only), QNAP, RAPOO, Samsung, SanDisk, Seagate (0 products — WC taxonomy entry only), Shuttle (0 products — WC taxonomy entry only), Simplecom, Synology, Targus, Thermaltake, Toshiba (0 products — WC taxonomy entry only), TP-Link, Ubiquiti, USP, Verbatim, ViewSonic, Western Digital, Crucial, Kingston, Logitech, AORUS, Logitech G, Norton, Trend Micro.

**Parent brand relationship:**
- Crucial → `parent_brand` = Micron (Micron is created as a parent Brand metaobject; it is not a WC brand with products and is not assigned as a vendor on any product — it exists solely as the parent metaobject reference).

**4 brands found in active product data but missing from `tl_brand_map`:**

| Brand | WC ID | Product Count | Note | User Decision |
|---|---|---|---|---|
| AORUS | 2079 | 7 | Likely a GIGABYTE sub-brand | Not specified — treat as separate brand (no parent_brand override noted) |
| Logitech G | 2088 | 1 | Logitech gaming sub-brand | Not specified — treat as separate brand |
| Norton | 2040 | 2 | Security software brand | Not specified — add to brand map |
| Trend Micro | 2065 | 1 | Security software brand | Not specified — add to brand map |

These 4 must be added to `tl_brand_map` before Phase 2 product migration.

**6 telephony brands deferred (no active products):**

Fanvil, Grandstream, Jabra, Snom, Teltonika, Yealink — all have 0 products currently. Brand metaobjects to be created on demand when the first product is added; defer creation until then.

---

## Attributes (metafields)

Of the 172 WC attributes reviewed, the resolved migration plan is:

- **15 keep-standard** — migrate as Shopify metafields mapped to canonical Shopify Standard Taxonomy attributes
- **150 keep-custom** — migrate as custom Shopify metafields under the `custom` namespace
- **3 drop** — Brand attribute, Model Number, duplicate Manufacturer's Warranty
- **3 merge** — consolidate into target attributes (memory specs)
- **1 promote-to-series** — Series stays as a `series` metafield

### Keep-standard (15 WC attributes → canonical Shopify Standard Taxonomy handles)

| WC Attribute | WC Slug | Shopify Handle | Shopify Attribute Name |
|---|---|---|---|
| Bluetooth | pa_bluetooth | `bluetooth-functionality` | Bluetooth functionality |
| Chipset | pa_chipset | `motherboard-chipset-family` | Motherboard chipset family |
| Colour | pa_colour | `color` | Color |
| CPU Cores | pa_cpu-cores | `processor-cores` | Processor cores |
| CPU Socket Type | pa_cpu-socket-type | `processor-socket` | Processor socket |
| Interface | pa_interface | `storage-drive-interface-type` | Storage drive interface type |
| Key Switch Type | pa_key-switch-type | `keyboard-switch-type` | Keyboard switch type |
| LAN | pa_lan | `ethernet-lan-interface-type` | Ethernet LAN interface type |
| Memory Type | pa_memory-type | `memory-storage-type` | Memory storage type |
| Operating System | pa_operating-system | `operating-system` | Operating system |
| Operating System Supported | pa_operating-system-supported | `operating-system` | Operating system |
| Resolution | pa_resolution | `display-resolution` | Display resolution |
| Tracking Method | pa_tracking-method | `mouse-technology` | Mouse technology |
| USB | pa_usb | `usb-standard` | USB standard |
| USB 3.1 | pa_usb-3-1 | `usb-standard` | USB standard |

### Drop (3)

| WC Attribute | WC Slug | Reason |
|---|---|---|
| Brand | pa_brand | Dimension duplicate of Shopify vendor field and Brand metaobject |
| Manufacturer's Warranty (duplicate) | pa_manufacturers-warranty | Duplicate of pa_warranty; consolidate to pa_warranty |
| Model Number | pa_model-number | Values are SKUs / model codes, not filterable terms |

### Merge (3)

| WC Attribute | WC Slug | Merge Into | Notes |
|---|---|---|---|
| Memory Channel | pa_memory-channel | Memory Type attribute or product description | Redundant — Dual/Quad/Single channel is implicit in memory kit configuration |
| Memory Components | pa_memory-components | SSD Interface or product description | 3D TLC/QLC NAND is storage technology, not RAM; consolidate to product description |
| Memory Slots (Total) | pa_memory-slots-total | Memory Capacity attribute | Redundant slot count spec — fold into Memory Capacity |

### Promote-to-series (1)

| WC Attribute | WC Slug | Shopify Handle | Notes |
|---|---|---|---|
| Series | pa_series | `series` (custom metafield) | 62 terms; already acts as product-line filter; promoted to a dedicated `series` Shopify metafield |

### Keep-custom (150)

The remaining 150 WC attributes are migrated as custom Shopify metafields under the `custom` namespace. They cover PC-builder specifications (e.g., fan airflow, CAS latency, SSD sequential read/write speeds, RPM, thermal design power), connectivity details (e.g., USB 2.0, USB 3.2, DisplayPort, HDMI count), physical dimensions, and binary/ghost attributes with 0–1 populated terms.

Notable keep-custom attributes with high term counts (good filter facets):
- Memory Speed (`pa_memory-speed`, 31 terms)
- Key Switch Type — WC values (`pa_key-switch-type`, 23 terms — kept-standard at handle level; WC term values need mapping)
- Screen Size (`pa_screen-size`, 22 terms)
- Internal Connectors (`pa_internal-connectors`, 22 terms)
- CPU Socket Type WC terms (`pa_cpu-socket-type`, 15 terms — kept-standard)
- CPU Max Speed (`pa_cpu-max-speed`, 15 terms)
- Max Sequential Write (`pa_max-sequential-write`, 12 terms)
- CPU Speed (`pa_cpu-speed`, 12 terms)
- Cable Length (`pa_cable-length`, 12 terms)
- SSD Capacity (`pa_ssd-capacity`, 13 terms)

For the complete attribute list, see the workbook 1c tab.

---

## Decisions and overrides from the user review

The following rows had non-blank User Decision values that modified or queried the sub-agent recommendation, or contained substantive notes:

| WC Item | Tab | Auto-Recommendation | User Decision / Note | Resolved Action |
|---|---|---|---|---|
| Electronics (WC 44) | 1a_categories | no-match (too broad — top-level taxonomy node) | Queried: "Why? there is an Electronics category — gid://shopify/TaxonomyCategory/el" | Pending: assess whether to assign `gid://shopify/TaxonomyCategory/el` as a direct mapping or retain as an unassigned custom collection. Requires Phase 2 decision before import. |
| Accessories (pa_accessories) | 1c_attributes | keep-custom (8 terms — retain as custom attribute) | "Should this be free text — it's not really a filter" | Note logged: consider converting to a `single_line_text_field` metafield rather than a list/select metafield, so values are free-text rather than constrained terms. Phase 2 schema decision. |
| Aspect Ratio (pa_aspect-ratio) | 1c_attributes | keep-custom (no Shopify match; 2 terms) | "What about Native Aspect Ratio — gid://shopify/TaxonomyAttribute/1958" | Note logged: user proposes mapping to the canonical `native-aspect-ratio` Standard Taxonomy attribute instead of a custom metafield. Upgrade recommendation to keep-standard with handle `native-aspect-ratio`. See workbook for GID. |

**Blank User Decision rows (fallback to sub-agent recommendation):**

| WC Item | Tab | Recommendation Used |
|---|---|---|
| Memory Slots (Total) (pa_memory-slots-total) | 1c_attributes | merge (into Memory Capacity attribute) |
| Series (pa_series) | 1c_attributes | promote-to-series (`series` custom metafield) |

All other rows had User Decision = "Yes", confirming the sub-agent recommendation.

---

## Next steps (Phase 2 inputs)

This document and the workbook feed Phase 2 (Catalog Data Migration). Specifically:

- The category mappings (direct bucket) drive Shopify product `category` assignment during import
- The merge bucket mappings drive attribute value population (e.g., `processor-socket = AM4`) and the corresponding smart collection creation
- The no-match categories each require a manual Shopify custom collection to be created; the "Electronics" row requires a final decision before collection creation
- The brand mappings drive `vendor` field values and Brand metaobject creation; the 4 missing brands (AORUS, Logitech G, Norton, Trend Micro) must be added to `tl_brand_map` first
- The attribute keep-standard mappings drive Shopify metafield definitions under the Shopify Standard Taxonomy namespace
- The attribute keep-custom mappings drive metafield definitions under the `custom` namespace
- The attribute overrides (Aspect Ratio → `native-aspect-ratio`, Accessories → free-text) must be confirmed and finalised before metafield schema deployment
- The smart collection rules (from both category merges and tags) are applied as Shopify smart collections post-product-import
- The 6 deferred telephony brands are not created until first product assignment

---

## Collection dedupe outcome (2026-05-17)

After Phase 1 the seeded Shopify store contained 358 collections — 39 of them with handles that did not match any Woo category, tag, or attribute-archive slug. Working from an orphan audit (sheet, since archived), the following cleanup was executed.

### Deleted (24 collections)

Each deletion is safe because the Woo URL the customer might have been on is still served by the listed keeper (whose handle is a real Woo slug).

| Deleted handle | Kept handle (anchored to Woo) | Notes |
|---|---|---|
| `graphics-cards` (`Graphics Cards & GPUs`, gid 344272666822) | `graphics-cards-gpu` | First sweep — duplicate of WC `graphics-cards-gpu` slug |
| `laptops-notebooks` (gid 344272994502) | `laptops` | First sweep — duplicate of WC `laptops` slug |
| `input-devices-keyboards` (gid 344272732358) | `keyboards` | "Keyboards" with auto-generated `input-devices-` prefix |
| `solid-state-drives` (gid 344272928966) | `solid-state-drives-ssd` | Identical title, missing `-ssd` suffix |
| `powerline-networking` (gid 344273420486) | `powerline-networking-eop` | Identical title; WC slug is `-eop` |
| `gaming-monitors` (gid 344707432646) | `pc-gaming-monitors` | WC slug is `pc-gaming-monitors` |
| `ram` (`RAM & Memory`, gid 344707530950) | `memory` | Manual collection, no rule |
| `pc-cases` (gid 344707629254) | `computer-cases` | PC = Computer synonym |
| `gaming-laptops` (gid 344707727558) | `pc-gaming-laptops` | WC slug is `pc-gaming-laptops` |
| `gaming-mouse-pads` (gid 344271323334) | `pc-gaming-mouse-pads` | WC slug is `pc-gaming-mouse-pads` |
| `laptop-memory-sodimm` (gid 344271454406) | `laptop-memory` | WC slug is `laptop-memory`; SO-DIMM filter to be re-introduced via metafield post-import |
| `graphics-cards` (gid 344707367110) | `graphics-cards-gpu` | Empty stray re-created post first sweep |
| `compatible-with-nintendo-switch` (gid 344389320902) | `nintendo-switch` | — |
| `compatible-with-playstation-4` (gid 344389353670) | `playstation-4` | — |
| `compatible-with-xbox-one` (gid 344389419206) | `xbox-one` | — |
| `processors` (gid 344707399878) | `cpus-processors` | — |
| `gaming-peripherals` (gid 344707465414) | `gaming` | — |
| `headsets` (gid 344707498182) | `gaming-headsets` | — |
| `storage` (gid 344707596486) | `hard-drives-storage` | — |
| `cooling` (gid 344707662022) | `fans-heatsinks-cooling` | — |
| `microphones-camera-accessories` (gid 344281481414) | `microphones` | Anchored-anchored pair — cleaner handle kept |
| `playstation-vr-vr-headsets` (gid 344295997638) | `playstation-vr` | Anchored-anchored pair — cleaner handle kept |
| `frontpage` (`Home page`, gid 343629791430) | `connected-home` | — |
| `brands` (gid 344397807814) | — | — |

### Renamed (5 motherboards orphans)

Phase 1 spec calls for socket-specific motherboards sub-collections. The Woo slugs `amd-socket-am4` / `amd-socket-am5` / `intel-socket-*` belong to the corresponding CPU collections in Shopify, so the motherboards equivalents were given non-conflicting handles. Titles unchanged.

| Old handle | New handle | gid |
|---|---|---|
| `amd-am4-motherboards` | `motherboards-amd-am4` | 344271061190 |
| `amd-am5-motherboards` | `motherboards-amd-am5` | 344271093958 |
| `intel-lga1700-12th-gen-motherboards` | `motherboards-intel-lga1700-12th-gen` | 344271126726 |
| `intel-lga1700-13th-gen-motherboards` | `motherboards-intel-lga1700-13th-gen` | 344271159494 |
| `intel-lga1851-15th-gen-motherboards` | `motherboards-intel-lga1851-15th-gen` | 344271192262 |

### Title fix

- `laptops` (gid 344295440582) — title corrected from HTML-encoded `Laptops &amp; Notebooks` to `Laptops & Notebooks`.

### Retained as Shopify-only (not Woo-anchored)

15 collections kept despite no Woo slug match — Shopify-side curated or filter-tag collections that aren't meant to map back to a Woo URL: `on-sale`, `deals`, `new-arrivals`, `best-sellers`, `streaming-gear`, `gaming-pcs`, `compatible-with-android`, `compatible-with-ios`, `compatible-with-mac`, `compatible-with-windows`, `compatible-with-corsair-icue`, `compatible-with-corsair-link`, `compatible-with-aura-sync`, `compatible-with-armoury-crate`.

### Working files

- Audit tooling: `_scratch/shopify-collection-dedupe/` (audit.js, find-orphans.js, execute-orphans.js, lib/*)
- Final audit sheet (archived state, all rows resolved): https://docs.google.com/spreadsheets/d/1msGtsSoUov4sTXuLBxG3HUhsQooVsjml_v_CW_RuGJ4/edit

### Followups

- Re-run `find-orphans.js` after each catalogue import to surface any new orphans introduced by seeding.
- The Woo→Shopify redirect map needs entries for the 24 deleted handles, pointing each at its keeper.
- Populate `processor-socket` metafield on motherboards so the renamed `motherboards-*` smart collections (currently broad-rule, returning all motherboards) start filtering correctly.
