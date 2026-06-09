# The Homelab & Media Server Architect — Notion Template Spec
**Protocol:** Zero Drift | **Persona:** The Technical Creator | **Platform:** Notion

---

<template_architecture>

## Overview

Six relational databases. Each is a self-contained module; together they form a complete infrastructure management system. The build sequence below respects Notion's requirement that relation targets exist before the source relation is created.

**Build order:** Port Forwarding Rules → Network Map → Hardware Deployment → Services & Container Registry → Storage Inventory → Maintenance Log

---

### Database 1: Port Forwarding Rules

The atomic unit of network exposure. Created first because Network Map holds a relation to it.

| Property | Type | Options / Notes |
|---|---|---|
| Rule Name | Title | Format convention: `SERVICE:EXTPORT→INTPORT` e.g. `PLEX:32400→32400` |
| External Port | Number | Integer. Port exposed on the router/firewall. |
| Internal Port | Number | Integer. Port the service listens on inside the LAN. |
| Protocol | Select | TCP · UDP · TCP/UDP |
| Active | Checkbox | Unchecked = rule exists but is disabled. |
| Service Description | Text | Human-readable label: "Plex Media Server stream" |
| Last Verified | Date | Last date the rule was confirmed working end-to-end. |
| Network Entry | Relation | → Network Map (back-relation shown on Network Map side). |
| Notes | Text | Firewall appliance notes, NAT hairpin details, etc. |

**Recommended views:**
- **Table (All Rules):** Sorted by External Port ASC. Filter toggle: Active = true.
- **Board (By Protocol):** Group by Protocol. Scan TCP vs UDP exposure at a glance.

---

### Database 2: Network Map

One record per network-attached device identity. A physical machine with two NICs gets two records. A VM and its host each get their own records.

| Property | Type | Options / Notes |
|---|---|---|
| Hostname | Title | Canonical hostname as it appears in DNS/`/etc/hosts`. |
| Static IP | Text | CIDR not included here — lives in Subnet. e.g. `192.168.1.42` |
| MAC Address | Text | Format: `AA:BB:CC:DD:EE:FF`. Primary interface only. |
| Subnet | Select | `192.168.1.0/24` · `10.0.0.0/24` · `172.16.0.0/16` · `Tailscale` · `WireGuard` |
| VLAN ID | Number | Integer. Leave blank for untagged/native VLAN. |
| Interface Type | Select | Ethernet · Wi-Fi · Tailscale · WireGuard · Loopback |
| Gateway | Text | e.g. `192.168.1.1` |
| DNS Servers | Text | Comma-separated. e.g. `192.168.1.53, 1.1.1.1` |
| DHCP Reserved | Checkbox | True = MAC-based DHCP reservation exists in router config. |
| Last ARP Seen | Date | Last confirmed ARP/ping response. Update manually or via script. |
| Port Rules | Relation | → Port Forwarding Rules (back-relation from DB 1). |
| Open Port Count | Rollup | Relation: Port Rules · Property: Rule Name · Aggregation: Count All |
| Notes | Text | Rack position, patch panel port, switch port number. |

**Recommended views:**
- **Table (IP Registry):** Sorted by Static IP ASC. Primary operational view.
- **Board (By VLAN):** Group by VLAN ID. Visualize network segmentation.
- **Table (Exposed Hosts):** Filter: Open Port Count > 0. Security audit view.

---

### Database 3: Hardware Deployment

The core database. One record per deployable compute node — physical machine, VM, LXC container, SBC, or NAS appliance. The one-way relation to Network Map originates here.

| Property | Type | Options / Notes |
|---|---|---|
| Node Name | Title | Descriptive name. e.g. `imac-2011-batocera` · `rpi4-pihole` |
| Node Type | Select | Physical Machine · Virtual Machine · LXC Container · SBC · NAS Appliance · Other |
| Host Machine | Text | For VMs/containers: the physical host running this node. Free-text to avoid circular self-relations. |
| OS / Runtime | Select | Batocera · Ubuntu Server 24.04 · Debian 12 · Raspberry Pi OS · Proxmox VE 8 · TrueNAS SCALE · Windows 11 · macOS · Docker Host · Bare Metal (no OS) · Other |
| Architecture | Select | x86\_64 · ARM64 · ARM32 · RISC-V |
| CPU Model | Text | e.g. `Intel Core i5-2500S` · `Broadcom BCM2711` |
| RAM (GB) | Number | Installed RAM in GB. |
| Primary Storage | Text | e.g. `256GB SSD (SATA)` · `32GB microSD` |
| Role | Multi-select | NAS · Media Server · Emulation · DNS / Ad Blocker · Reverse Proxy · Dev Environment · Game Server · Home Automation · Monitoring · Backup Target |
| OS Lifecycle State | Select | Planning · Installing · Testing · Production · Degraded · Decommissioned |
| Maintenance Mode | Checkbox | Toggle ON to suppress Offline alerts in the Status Flag formula. |
| Last Seen | Date | Last confirmed connectivity. Update manually or via cron → Notion API. |
| Last Boot | Date | Last recorded boot time. |
| Purchase Date | Date | Used for age calculation and warranty tracking. |
| Warranty Expiry | Date | Used by Warranty Status formula. |
| Acquisition Cost | Number | Currency format. For total build cost rollup on the dashboard page. |
| Network Entries | Relation | → Network Map. **One-way: disable the back-relation on the Network Map side.** See implementation note below. |
| Total Exposed Ports | Rollup | Relation: Network Entries · Property: Open Port Count · Aggregation: Sum |
| Status Flag | Formula | See `technical_implementation` section. |
| Days Since Last Seen | Formula | See `technical_implementation` section. |
| Warranty Status | Formula | See `technical_implementation` section. |
| Notes | Text | Build notes, quirks, relevant forum threads. |
| Tags | Multi-select | Legacy · Headless · Always-On · Seasonal · Repurposed |

**One-way relation implementation note:** After creating the `Network Entries` relation from Hardware Deployment → Network Map, open the Network Map database, find the auto-created back-relation property (Notion names it after the source DB), and delete it. This removes the back-reference from Network Map records, making the relation directional from the user's perspective. Hardware Deployment records show their network entries; Network Map records do not show which hardware node claimed them. This prevents the Network Map from becoming cluttered with a relation column that doesn't add operational value in that context.

**Recommended views:**
- **Gallery (Node Cards):** Cover = Node Name. Show: Status Flag, Role, OS Lifecycle State. Grouped by Node Type.
- **Table (Full Inventory):** All properties visible. Sort by OS Lifecycle State.
- **Board (By Lifecycle State):** Group by OS Lifecycle State. Drag nodes through Planning → Production → Decommissioned.
- **Table (Active Nodes):** Filter: Status Flag contains "Active". Production operational view.
- **Table (Port Exposure Audit):** Filter: Total Exposed Ports > 0. Sort: Total Exposed Ports DESC.

---

### Database 4: Services & Container Registry

One record per running service or container. Decoupled from Hardware Deployment so a service can be migrated between hosts by updating a single relation — not by moving records.

| Property | Type | Options / Notes |
|---|---|---|
| Service Name | Title | e.g. `Plex Media Server` · `Pi-hole` · `Nginx Proxy Manager` |
| Container Image | Text | Full image reference. e.g. `lscr.io/linuxserver/plex:latest` |
| Host Node | Relation | → Hardware Deployment. |
| Compose File Path | Text | Absolute path on host. e.g. `/opt/stacks/plex/docker-compose.yml` |
| Exposed Ports | Text | Comma-separated local mappings. e.g. `32400:32400/tcp, 1900:1900/udp` |
| Data Volume Paths | Text | Bind mounts. e.g. `/mnt/media:/media:ro` |
| Auto-Restart | Checkbox | True = `restart: unless-stopped` or equivalent is set. |
| Update Policy | Select | Watchtower (auto) · Manual Pull · Pin Version · Registry Webhook |
| Runtime Status | Select | Running · Stopped · Error · Pending Update · Deprecated |
| Last Updated | Date | Last image pull or config change. |
| Notes | Text | Environment variable notes, known issues, upstream docs URL. |

**Recommended views:**
- **Board (By Runtime Status):** Operational health board. Running / Stopped / Error columns.
- **Table (Pending Updates):** Filter: Update Policy = Manual Pull, sorted by Last Updated ASC.

---

### Database 5: Storage Inventory

One record per physical or logical storage device. Tracks capacity, health, and allocation across the entire homelab.

| Property | Type | Options / Notes |
|---|---|---|
| Drive Label | Title | e.g. `rpi4-sdcard-01` · `nas-hdd-wd4tb-slot2` |
| Host Node | Relation | → Hardware Deployment. |
| Type | Select | HDD · SSD · NVMe · SD Card · USB Flash · eMMC · Optical |
| Interface | Select | SATA · NVMe (M.2) · USB 3.0 · USB-C · PCIe · SD |
| Capacity (TB) | Number | Decimal. e.g. `3.64` for a nominal 4TB drive. |
| Used (TB) | Number | Current usage. Update manually or via script. |
| SMART Status | Select | Healthy · Warning · Reallocated Sectors · Failing · Unknown |
| RAID / Pool | Text | e.g. `ZFS mirror pool-01` · `mdadm RAID 5` · `standalone` |
| Mount Point | Text | e.g. `/mnt/data` · `/media/usb` |
| Filesystem | Select | ext4 · ZFS · Btrfs · exFAT · NTFS · APFS · XFS |
| Purchase Date | Date | For age-based SMART risk assessment. |
| % Used | Formula | See `technical_implementation` section. |
| Notes | Text | S/N, firmware version, RMA history. |

**Recommended views:**
- **Table (All Drives):** Sort by Host Node, then Capacity DESC.
- **Table (Health Alerts):** Filter: SMART Status ≠ Healthy. Maintenance triage view.

---

### Database 6: Maintenance Log

Append-only operational record. One entry per task, incident, or scheduled maintenance event.

| Property | Type | Options / Notes |
|---|---|---|
| Task | Title | Imperative description. e.g. `Renew SSL cert for nginx-pm` |
| Node | Relation | → Hardware Deployment. Multi-relation allowed (cross-node tasks). |
| Type | Select | OS Update · Hardware Swap · Config Change · Incident · Scheduled Maintenance · Certificate Renewal · Drive Replacement |
| Scheduled Date | Date | When the task is planned for. |
| Completed Date | Date | When it was actually done. Used by formula to compute slip. |
| Status | Select | Scheduled · In Progress · Complete · Deferred · Cancelled |
| Notes | Text | Steps taken, commands run, links to relevant docs or forum posts. |

**Recommended views:**
- **Table (Open Tasks):** Filter: Status = Scheduled or In Progress. Sort: Scheduled Date ASC.
- **Board (By Status):** Drag-to-complete kanban view.
- **Table (Incident History):** Filter: Type = Incident. Sort: Completed Date DESC.

---

### Dashboard Page Structure

The template's home page is a Notion page (not a database) that embeds linked views of each database. Build it in this order:

1. **Cluster Health** — Linked view of Hardware Deployment filtered to Active nodes, Gallery layout.
2. **Network Exposure Summary** — Linked view of Network Map filtered to Open Port Count > 0.
3. **Container Health Board** — Linked view of Services & Container Registry, Board by Runtime Status.
4. **Storage at a Glance** — Linked view of Storage Inventory filtered to SMART Status ≠ Healthy (flagged drives only), plus a full table below.
5. **Open Maintenance Tasks** — Linked view of Maintenance Log filtered to Status = Scheduled or In Progress.
6. **Setup Guide** (callout block) — Documents the `Last Seen` update workflow and links to the Notion API automation script template.

</template_architecture>

---

<technical_implementation>

## Notion Formula 2.0 — All Formulas

All formulas use Notion Formula 2.0 syntax. Property names must match exactly (case-sensitive) as defined in the architecture above. Paste each formula into the formula property editor without modification.

---

### Formula 1: Status Flag
**Database:** Hardware Deployment
**Property Type:** Formula → Text output
**Logic:** Maintenance Mode checkbox takes absolute precedence. If Last Seen is null, node is Unknown. Staleness thresholds: > 7 days = Offline, 1–7 days = Stale, ≤ 1 day = Active.

```
if(
  prop("Maintenance Mode"),
  "🔧 Maintenance",
  if(
    empty(prop("Last Seen")),
    "⚪ Unknown",
    if(
      dateBetween(now(), prop("Last Seen"), "days") > 7,
      "🔴 Offline",
      if(
        dateBetween(now(), prop("Last Seen"), "days") > 1,
        "🟡 Stale",
        "🟢 Active"
      )
    )
  )
)
```

**Threshold customization:** Change `7` and `1` to match your monitoring cadence. If you update Last Seen daily via script, tighten to `2` and `0`.

**Filtering:** Because the output is a text string, filter views using `Status Flag` → `contains` → `Active` (not `equals`) to avoid emoji matching issues in Notion's filter engine.

---

### Formula 2: Days Since Last Seen
**Database:** Hardware Deployment
**Property Type:** Formula → Text output
**Logic:** Returns a human-readable staleness string. Empty date returns an em dash to prevent blank cells that confuse sort order.

```
if(
  empty(prop("Last Seen")),
  "—",
  format(dateBetween(now(), prop("Last Seen"), "days")) + "d ago"
)
```

**Example outputs:** `0d ago` · `3d ago` · `14d ago` · `—`

---

### Formula 3: Warranty Status
**Database:** Hardware Deployment
**Property Type:** Formula → Text output
**Logic:** Three states based on days remaining. 90-day warning window catches expiries before they become surprises. Negative `dateBetween` result means the expiry date is in the past.

```
if(
  empty(prop("Warranty Expiry")),
  "—",
  if(
    dateBetween(prop("Warranty Expiry"), now(), "days") < 0,
    "⚠️ Expired",
    if(
      dateBetween(prop("Warranty Expiry"), now(), "days") < 90,
      "⚡ Expiring Soon",
      "✅ Under Warranty"
    )
  )
)
```

**Direction note:** `dateBetween(prop("Warranty Expiry"), now(), "days")` computes `Expiry − Now`. A negative result means the expiry date has passed. A positive result is days remaining. This is the inverse of the Last Seen formula — confirm the argument order before applying.

---

### Formula 4: Storage % Used
**Database:** Storage Inventory
**Property Type:** Formula → Text output
**Logic:** Guards against division by zero on empty or zero-capacity entries. Returns a percentage string for display; does not return a number, so it cannot be aggregated in a rollup. If you need rollup aggregation, change the formula to return a number and drop the `format()` and `"%"`.

```
if(
  empty(prop("Capacity (TB)")),
  "—",
  if(
    prop("Capacity (TB)") == 0,
    "—",
    format(round((prop("Used (TB)") / prop("Capacity (TB)")) * 100)) + "%"
  )
)
```

**Example outputs:** `72%` · `8%` · `100%` · `—`

**Variant for numeric rollup** (returns Number type, not Text):
```
if(
  empty(prop("Capacity (TB)")),
  0,
  if(
    prop("Capacity (TB)") == 0,
    0,
    round((prop("Used (TB)") / prop("Capacity (TB)")) * 100)
  )
)
```

---

### Formula 5: Port Rule Label
**Database:** Port Forwarding Rules
**Property Type:** Formula → Text output
**Logic:** Generates a compact, human-readable label for each rule. Used as display text when this database is embedded or rolled up.

```
format(prop("External Port")) + " → " + format(prop("Internal Port")) + " / " + prop("Protocol")
```

**Example outputs:** `32400 → 32400 / TCP` · `80 → 8080 / TCP/UDP` · `53 → 53 / UDP`

---

## Relation & Rollup Configuration

### One-Way Relation: Hardware Deployment → Network Map

**Step-by-step:**
1. Open Hardware Deployment database → Add property → Relation → select Network Map database.
2. Notion will ask: "Show on Network Map?" — uncheck this, or click through and then open Network Map, find the auto-created property named `Hardware Deployment`, and delete it.
3. Name the relation property `Network Entries` in Hardware Deployment.

**Result:** Hardware Deployment records show their linked Network Map entries. Network Map records show no hardware back-reference. Preserves Network Map as a clean IP registry without hardware-layer noise.

---

### Two-Hop Rollup: Total Exposed Ports on Hardware Deployment

This requires Network Map to already have its own rollup of Port Rules configured.

**Step 1 — Rollup in Network Map:**
- Property name: `Open Port Count`
- Type: Rollup
- Relation: `Port Rules` (→ Port Forwarding Rules)
- Property to rollup: `Rule Name`
- Aggregation: `Count All`

**Step 2 — Rollup in Hardware Deployment:**
- Property name: `Total Exposed Ports`
- Type: Rollup
- Relation: `Network Entries` (→ Network Map)
- Property to rollup: `Open Port Count` (the rollup created in Step 1)
- Aggregation: `Sum`

**Why Sum and not Count:** `Count All` on the Network Entries relation would count the number of linked Network Map records, not the number of port rules. `Sum` of `Open Port Count` correctly aggregates the port count across all network interfaces attached to a given hardware node.

---

### Relation: Services → Hardware Deployment

Standard two-way relation. Leave the back-relation active on Hardware Deployment (name it `Hosted Services`). This gives each hardware record a live count of its container/service workload via a rollup.

**Optional rollup on Hardware Deployment:**
- Property name: `Running Services`
- Relation: `Hosted Services`
- Property: `Runtime Status`
- Aggregation: `Count Values` with filter: `Runtime Status` = `Running`

Note: Notion rollup filters on relations are not yet supported natively in all account tiers. If unavailable, use Count All and note it includes stopped services.

---

## Automation Hook: Updating `Last Seen` via Notion API

The Status Flag formula depends on `Last Seen` being current. Manual updates work for small homelabs; larger builds benefit from automation.

**Recommended approach:** A cron job on any always-on node (e.g., the Raspberry Pi running Pi-hole) runs a shell script every 30 minutes that:
1. Pings or checks each node's IP from the Network Map.
2. On success, calls the Notion API `PATCH /pages/{page_id}` endpoint to update the `Last Seen` date property on the corresponding Hardware Deployment record.

**Notion API property update payload for a Date property:**
```json
{
  "properties": {
    "Last Seen": {
      "date": {
        "start": "2026-06-05T14:30:00-05:00"
      }
    }
  }
}
```

The page ID for each Hardware Deployment record should be stored in a local config file alongside the node's IP address, mapping `ip → notion_page_id`.

</technical_implementation>

---

<marketing_assets>

## SEO Product Title

```
Homelab Management System | Self-Hosted Infrastructure Tracker | Network Map + Container Registry | Notion Template
```

---

## 3 Conversion-Focused Bullet Points

- **Relational node tracking that handles real homelab complexity** — The Hardware Deployment database links physical machines, VMs, LXC containers, and SBCs in a single relational system. Track an old iMac running Batocera, a Raspberry Pi 4 running a Pi-hole stack, and a Proxmox host running eight VMs as distinct records with their own OS lifecycle states (Planning → Production → Decommissioned), architecture flags, and role tags. One linked view surfaces the full picture; individual records hold the detail.

- **A live Network Map with port exposure auditing built in** — Static IP allocations, MAC addresses, VLAN assignments, and DHCP reservation status live in a dedicated Network Map database. A two-hop rollup surfaces each machine's total exposed port count directly on its Hardware Deployment record, so a single filtered view answers "which nodes are exposing ports to the internet" without opening a separate tool. Port Forwarding Rules are their own database: External Port, Internal Port, Protocol, Active toggle, and last-verified date, all queryable and filterable.

- **Computed status flags derived from dates — no manual status updates required** — Four Notion Formula 2.0 formulas are pre-wired into the template. The Status Flag formula reads a Maintenance Mode checkbox and a Last Seen date to output 🟢 Active, 🟡 Stale, 🔴 Offline, or 🔧 Maintenance automatically. Warranty Status flags hardware approaching expiry 90 days out. Storage % Used guards against division-by-zero and returns a display-ready percentage per drive. All formula syntax is documented in the included spec with argument-order notes so you can modify thresholds without debugging from scratch.

---

## 150-Word Product Description

Most homelabs are documented in a mix of sticky notes, stale wiki pages, and memory. This template replaces that with a six-database relational system in Notion, built specifically for the self-hoster who runs mixed hardware — legacy x86 machines, SBCs, VMs, and containers — across multiple network segments.

The Hardware Deployment database tracks OS installations and lifecycle states per node. A one-way relation surfaces each machine's Network Map entries and total exposed ports without cluttering the IP registry with back-references. Port Forwarding Rules are a first-class database: queryable, filterable, and linked directly to the network interface that owns them.

Four pre-built Notion Formula 2.0 expressions handle status flagging, staleness detection, warranty tracking, and storage utilization. The formulas are documented with threshold customization notes and an API payload example for automating Last Seen updates via cron.

No paid Notion plan required for core functionality. Delivered as a Notion template duplication link with a PDF setup guide.

---

## Platform & Prerequisites

- **Platform:** Notion (Free plan compatible — all features work without Notion AI or Plus)
- **Delivery:** PDF guide with a secure template duplication link inside
- **Prerequisites:** Active Notion account (free tier sufficient). Notion API integration optional for Last Seen automation.

</marketing_assets>
