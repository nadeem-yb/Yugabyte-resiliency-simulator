# YugabyteDB Cluster Resiliency Simulator

An interactive, browser-based simulator for exploring **YugabyteDB fault tolerance, quorum behaviour, and tablet group impact** across different replication factors, availability zone configurations, and node failure scenarios.

> **Zero dependencies. No build step. Works offline. Open in any browser.**

---

## Live Demo

Host on GitHub Pages and share via:

```
https://<your-username>.github.io/yugabytedb-resiliency-simulator/
```

Or preview instantly via htmlpreview:

```
https://htmlpreview.github.io/?https://raw.githubusercontent.com/<user>/<repo>/main/yugabytedb-resiliency-simulator.html
```

---

## What It Simulates

| Capability | Details |
|---|---|
| **Replication Factor** | RF=3 and RF=5 |
| **Availability Zones** | 1, 2, or 3 AZs |
| **Nodes per AZ** | 1–9 nodes per AZ (configurable independently) |
| **Tablet groups** | 6, 9, 12, 18, or 24 tablet groups |
| **Node failure** | Click any node to bring it down or restore it |
| **Master re-election** | Same-AZ first → cross-AZ fallback |
| **Tablet quorum** | Per-group quorum status updates in real time |
| **Impact classification** | Healthy / Under-replicated / Leader re-electing / Quorum lost |

---

## Screenshot

> *(Add a screenshot here once hosted. Suggested: `docs/screenshot.png`)*

---

## How to Use

### Option 1 — Open locally

```bash
git clone https://github.com/<your-username>/yugabytedb-resiliency-simulator.git
cd yugabytedb-resiliency-simulator
open yugabytedb-resiliency-simulator.html     # macOS
# or
xdg-open yugabytedb-resiliency-simulator.html # Linux
# or just double-click the file in Explorer/Finder
```

### Option 2 — GitHub Pages

1. Fork or upload this repo
2. Go to **Settings → Pages**
3. Set source to **Branch: main**, folder **/ (root)**
4. Click **Save**
5. Access at `https://<your-username>.github.io/<repo-name>/`

---

## Controls

| Control | What it does |
|---|---|
| **RF toggle** | Switch between Replication Factor 3 and 5 |
| **AZ tabs** | Select 1, 2, or 3 availability zones |
| **AZ dropdowns** | Set nodes per AZ independently (1–9 each) |
| **Tablet groups** | Choose how many tablet groups to simulate (6–24) |
| **Click a node** | Toggle it down/up — watch quorum and tablet groups update instantly |
| **Reset all ↺** | Restore all nodes to healthy state |
| **Filter buttons** | Show all / healthy / under-replicated / re-electing / quorum lost tablet groups |

---

## Key Concepts

### Fault Tolerance Formula

```
FT = ⌊(RF − 1) / 2⌋

RF=3 → FT = 1 node failure tolerated
RF=5 → FT = 2 node failures tolerated
```

### Quorum Requirement

```
Quorum = ⌈RF / 2⌉ replicas must be alive

RF=3 → need 2/3 alive
RF=5 → need 3/5 alive
```

### Tablet Replica Placement

Replicas are placed with **exactly 1 replica per AZ** (when `numAZ ≥ RF`), and **round-robin across nodes within each AZ**:

```
Tablet t, replica slot r:
  AZ   = (r % numAZ) + 1
  Node = (t % nodesInAZ) + 1
```

**Example — 3 AZs × 3 nodes, 12 tablet groups:**

| Tablet group | AZ1 | AZ2 | AZ3 |
|---|---|---|---|
| TG 1 | N1 | N1 | N1 |
| TG 2 | N2 | N2 | N2 |
| TG 3 | N3 | N3 | N3 |
| TG 4 | N1 | N1 | N1 ← wraps |

Losing N1 from each AZ removes 1 replica from TG1, TG4, TG7, TG10 — each still has **2 alive replicas** → quorum maintained ✓

### AZ-Level Fault Tolerance

| AZ count | Survives full AZ loss? | Reason |
|---|---|---|
| **1 AZ** | ❌ Never | Losing the only AZ = total cluster loss |
| **2 AZs** | ❌ Never | Replicas span both AZs — losing either AZ breaks quorum. Both AZs must be alive at all times. |
| **3 AZs** | ✅ Yes (if balanced) | Losing 1 AZ = losing 1/3 replicas per tablet → within FT=1 for RF=3 |

### Tablet Group States

| State | Replicas alive | Reads/Writes | RTO | RPO |
|---|---|---|---|---|
| **Healthy** | RF/RF | ✅ Full | 0 | 0 |
| **Under-replicated** | (QN to RF-1)/RF | ✅ Continue | 0 | 0 |
| **Leader re-electing** | ≥ QN, leader down | ✅ ~3s pause | ~3s | 0 |
| **Quorum lost** | < QN | ❌ Unavailable | Until restore | 0 |

> **RPO is always 0** — YugabyteDB never acknowledges a write until it is committed to a Raft majority. No data is ever lost when a node fails.

### yb-master Placement and Re-election

- Always **RF masters** (3 for RF=3, 5 for RF=5), distributed round-robin across AZs
- **Re-election priority:** same AZ first → cross-AZ fallback → slot lost (if no alive node available)
- Master quorum need: `⌈RF/2⌉` masters alive (2/3 for RF=3, 3/5 for RF=5)

---

## Scenarios to Try

### 1. 3 AZs × 3 nodes — lose 1 node per AZ
**Expected:** All tablet groups remain healthy or under-replicated. Quorum maintained. This is within RF=3 fault tolerance.

### 2. 3 AZs × 3 nodes — lose all 3 nodes in one AZ (full AZ outage)
**Expected:** Tablet groups become under-replicated (1 replica lost per group). Quorum maintained. Masters re-elect within the surviving AZs.

### 3. 2 AZs × 3 nodes — lose all nodes in AZ1
**Expected:** Cluster unavailable. With 2 AZs and RF=3, AZ1 holds 2 replicas per tablet. Losing AZ1 = losing 2 replicas = below quorum.

### 4. RF=5, 3 AZs × 2 nodes — lose 2 nodes across different AZs
**Expected:** Still operational. RF=5 tolerates 2 node failures. All tablet groups maintain quorum with 3+ replicas alive.

### 5. 1 AZ × 9 nodes — lose any 1 node
**Expected:** Under-replicated tablet groups (1 replica lost). Quorum maintained. But no AZ-level protection.

---

## YugabyteDB Reference

| Concept | Value / Formula |
|---|---|
| Replication Factor | User-configured (RF=3 recommended minimum) |
| Fault tolerance | `⌊(RF−1)/2⌋` |
| Quorum | `⌈RF/2⌉` |
| yb-masters | Always = RF, placed across AZs |
| Master quorum | `⌈RF/2⌉` masters must be alive |
| Leader re-election RTO | ~3 seconds |
| RPO on node failure | 0 (Raft commits to majority before ack) |
| Under-replication timeout | ~5 min before re-replication starts |
| Recommended AZ config | 3 AZs for full AZ-level fault tolerance |

---

## File Structure

```
yugabytedb-resiliency-simulator/
├── yugabytedb-resiliency-simulator.html   # The entire simulator — single file
├── README.md                              # This file
└── docs/
    └── screenshot.png                     # Optional: add after hosting
```

---

## Technical Notes

- **Pure HTML/CSS/JavaScript** — no frameworks, no bundler, no npm
- All logic is embedded in a single `<html>` file (~37KB)
- Works from `file://` protocol (no web server needed)
- Responsive down to ~375px viewport width
- Tested in Chrome, Firefox, Safari, Edge

---

## Contributing

Issues and pull requests welcome. Key areas for improvement:

- [ ] RF=7 support
- [ ] Animated leader re-election flow
- [ ] Export failure scenario as JSON
- [ ] Dark mode toggle
- [ ] Per-tablet group drill-down showing which rows are affected

---
---

## Author

Built for YugabyteDB solution engineering — exploring distributed database fault tolerance visually.

> **YugabyteDB** is an open-source, high-performance distributed SQL database built on a document store, with PostgreSQL compatibility.
> Learn more: [yugabyte.com](https://www.yugabyte.com) · [docs.yugabyte.com](https://docs.yugabyte.com)
