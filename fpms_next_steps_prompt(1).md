# FPMS — Continuation Prompt

## Context for Claude

You are continuing work on the **Firewall Policy Management System (FPMS)** for CVS Health Information Security, Network Security Engineering. This is a topology-aware firewall rule builder grounded in the live Panorama estate. The following has been completed in prior sessions.

---

## What has been built

### 1. Data extraction (Python scripts)

`parse_panorama.py` — extracts from four Panorama XML config exports:
- All template stacks with device serials
- Zone definitions per stack (name, interface members, zone protection profile)
- Interface inventory (ethernet, aggregate, subinterfaces, tunnels) with IPs and VLAN tags
- Device group rule/NAT counts

Produces: `full_topology.json`

`build_network_groups.py` (CCD platform equivalent) — processes 728 EDL `.txt` files from the Archive.zip:
- 349 NG (Network Group) objects — location/contextual subnets
- 379 AO (Address Object) objects — functional/application targets
- Each categorized by business unit prefix (CORP, CAREPLUS, OMNICARE, DATACENTER, PARTNER, etc.)

Produces: `edl_catalog.json`

`build_ng_zone_map.py` — cross-references topology with EDL catalog:
- Classifies each zone as TRUST / DMZ / UNTRUST / VPN-PARTNER / CLOUD-DMZ / MGMT
- Assigns NG/AO category constraints per zone (which categories belong where)
- Derives flow pattern per stack (INTERNET-EDGE / DC-DMZ / SIMPLE-EDGE / PARTNER-VPN)
- Sets default profile group and NAT model per zone class

Produces: `ng_zone_map.json`, `rule_builder_data.json`

---

### 2. Panorama estate — ground truth

**Four Panorama instances (PAN-OS 11.2.10):**

| Instance | Role | DGs | Rules |
|---|---|---|---|
| MID-PAN-PROD-01 | Internal DC (Middletown CT) | 13 | ~3,200 |
| WIN-PAN-PROD-01 | Internal DC (Windsor CT) — mirror of MID | 13 | ~3,200 |
| PAZSHDC-SEC-PAN-PROD-01 | Internet edge (Shea AZ) | 10 | 15,629 |
| RRISHDC-SEC-PAN-PROD-01 | Internet edge (Cumberland RI) — mirror of PAZSHDC | 10 | 15,629 |

**Key device groups:**
- `MID-WIN-GLR` — PA-5450/5400F HA pairs, inside/Internet, SIMPLE-EDGE, 924 rules
- `mid-fw-dmz` / `win-fw-dmz` — PA-3400 HA, dmz/inside, DC-DMZ, 426/376 rules
- `mid-fw-clouddmz` / `win-fw-clouddmz` — PA-3400 HA, clouddmz/inside, DC-DMZ, 225/160 rules
- `mid-fw-edmz` / `win-fw-edmz` — PA-5450 HA, inside/outside, SIMPLE-EDGE, 189/179 rules + 1372/1009 NAT
- `DG_AZ_Egress` — PAZSHDC-SEC-WIR, PA-5450 HA, INTERNET-EDGE, 5923 rules + 6462 NAT
- `DG_RI_EXT_APP` / `DG_RI_Egress` — RRIWSDC/RRIWSCP, INTERNET-EDGE, 3036/4162 rules
- `DG_LV_3rd_Party` — LV-3rd_Party-Stack, PARTNER-VPN, named tunnel zones (WEX-Meritain, EBIX-Meritain)

**Critical finding:** 15,448 / 15,629 rules (98%) use `Alert-All-SPG` — alert-only, no blocking. Best-practice profiles exist in shared but are almost never applied.

---

### 3. Four structural flow patterns

Every stack maps to one of these:

**INTERNET-EDGE** (PAZSHDC/RRISHDC stacks): `untrust → DMZ → trust`. DNAT inbound, SNAT egress. AO-DATACENTER-* on DMZ, NG-CORP/DATACENTER on trust. No NG/AO on untrust.

**DC-DMZ** (mid/win-dmz, clouddmz, qadmz): `dmz ↔ inside`. No internet leg. AO-DATACENTER on dmz, NG-DATACENTER on inside. Static 1:1 or SNAT to interface IP.

**SIMPLE-EDGE** (mid/win-glr, edmz, officeext, HOL-PCI): `trust ↔ untrust`. SNAT egress or 1:1 NAT per server (edmz: 1372 rules, one per AO-DATACENTER server). NG-CORP/DATACENTER on trust.

**PARTNER-VPN** (LV-3rd_Party, mid/win-officeext): `VPN-PARTNER ↔ trust`. Named tunnel zones per partner. No NAT. NG-PARTNER-{NAME} maps 1:1 to zone name. Reference model for 3PLZ.

---

### 4. Object model

| Type | Count | Zone class | Source |
|---|---|---|---|
| NG — Network Group | 349 | TRUST (branches/DCs), VPN-PARTNER (VPN/PARTNER) | CCD EDL platform |
| AO — Address Object | 379 | DMZ (DATACENTER/COLO), TRUST (CORP sites) | CCD EDL platform |
| AD groups | 10 | TRUST | Panorama AD config spreadsheet |
| Service objects | 6,670 shared | N/A | Panorama shared |

**Zone constraint rules (ng_zone_map.json):**
- `NG-CORP-*` → TRUST (hard=false)
- `NG-PARTNER-*` → VPN-PARTNER (hard=true — isolation violation if in TRUST)
- `NG-CLOUD/AZURE/GCP` → CLOUD-DMZ (hard=false)
- `AO-DATACENTER-*` → DMZ (hard=false — published server IPs, DMZ leg of hairpin)
- `AO-PARTNER-*` → VPN-PARTNER (hard=true)

---

### 5. Deliverables produced

- `full_topology.json` — complete zone/interface/stack topology from all 4 Panoramas
- `ng_zone_map.json` — zone class constraints, flow patterns, NAT models per stack
- `rule_builder_data.json` — compact payload for the rule builder UI
- `fpms_wireframe.html` — fully functional dark-mode rule builder prototype (topology-aware, live validation, PAN-OS CLI output)
- `fpms_overview.html` — program brief / stakeholder document with findings and roadmap
- AD config spreadsheet analysis — 300+ DC address objects, dynamic tag-based groups
- Zone coherence validation logic — checks NG/AO category against expected zone class, flags hard violations

---

## What still needs to be built

### Priority 1 — Data pipeline automation
The extraction scripts run manually today from Panorama XML exports. Need:
- Scheduled Panorama config pull via REST API: `GET /restapi/v10.1/Panorama/running-config`
- Automated run of extraction scripts (cron / Ansible)
- Output published to a shared location the UI reads from
- Diff-based alerting when topology changes (new stacks, zone renames, new EDL objects)

### Priority 2 — Service / App-ID repository
Currently a gap. Need:
- Applipedia API pull: `GET https://<firewall>/restapi/v10.1/Objects/Applications` — App-ID name, category, default ports, risk level
- IANA port registry CSV cross-reference (downloadable from iana.org)
- Cross-referenced `service_repo.json`: given App-ID → suggest default ports; given port → suggest candidate App-IDs
- Warning when App-ID selected but service port doesn't match `application-default`

### Priority 3 — Security profile template repo
Currently `profile_groups.json` has six entries from the live config. Need:
- Full profile group definitions: AV profile + vulnerability/IPS profile + URL filter profile + file blocking profile + WildFire profile
- Named template sets: `strict-internet`, `internal-trusted`, `dc-to-dc-replication`, `partner-access`, `ad-traffic`
- Profile upgrade campaign tooling: scan rulebase for Alert-All-SPG rules, generate migration plan to appropriate BP_SPG_* profile by zone class
- Each profile's component profile definitions (severity thresholds, blocked categories, file types)

### Priority 4 — FPMS REST API
FastAPI service over the JSON repos. Four endpoints:
- `GET /topology` — live stack/zone/interface data, refreshed from pipeline
- `GET /catalog` — NG/AO EDL objects with zone constraints, App-ID/service repo
- `POST /validate` — zone coherence, shadow rule detection, profile compliance check
- `POST /commit` — ServiceNow CHG creation with rule payload OR direct Panorama API push

### Priority 5 — Commit path
Two modes:
- **ServiceNow**: POST to CHG API with rule payload as structured attachment, link to FPMS rule record
- **Direct Panorama push**: `POST /restapi/v10.1/DeviceGroup/{dg}/PreRulebase/SecurityRules` — requires peer review step to be completed first

### Priority 6 — Profile upgrade campaign
Standalone tool to address the 98% Alert-All-SPG finding:
- Parse all 15,629 PAZSHDC/RRISHDC rules
- For each rule, determine appropriate BP_SPG_* profile based on zone pair (trust→DMZ = BP_SPG_DMZ_Model, trust→untrust = BP_SPG_User_Internet_Model, vpn→trust = Trusted-Apps)
- Generate Panorama set commands to update profile-setting on each rule
- Output as a prioritized migration plan (high-traffic rules first, by hit count if available)

---

## Technical context

- **Panorama REST API base**: `https://<panorama>/restapi/v10.1/`
- **PAN-OS version**: 11.2.10 across all four Panoramas
- **EDL base URL pattern**: `https://edl.cvs.internal/{object-name}.txt`
- **Four Panorama stacks**: MID (Middletown CT), WIN (Windsor CT, mirror of MID), PAZSHDC (Shea AZ), RRISHDC (Cumberland RI, mirror of PAZSHDC)
- **Python extraction scripts**: all in `/tmp/` from prior session — need to be formalized into a proper repo
- **Key JSON schemas**: `full_topology.json`, `ng_zone_map.json`, `rule_builder_data.json`, `edl_catalog.json`

---

## 3PLZ connection

The FPMS work directly supports 3PLZ (Third-Party Landing Zone) POC. The `LV-3rd_Party-Stack` is the reference implementation for 3PLZ's named-tunnel-zone-per-partner model. When 3PLZ POC moves to vendor selection (Phase 3D, end of July 2026), the FPMS rule builder will be used to build the onboarding rules for new partners into the 3PLZ landing zone — following the PARTNER-VPN flow pattern with `NG-PARTNER-{NAME}` mapping 1:1 to a named tunnel zone.

---

## How to continue

Load this prompt at the start of a new session. The assistant should:
1. Have full context on what has been built
2. Be ready to continue on any of the six priority items above
3. Have access to the JSON schemas and prototype HTML if re-uploaded
4. Understand the four-Panorama topology and the zone constraint model

Suggested starting question: *"Let's build out the Applipedia + IANA service repo"* or *"Let's design the FastAPI layer"* or *"Let's build the profile upgrade campaign tool"*
