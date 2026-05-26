# Lab: Text-Based Network Configuration with `nmtui`

**Series:** linux-ops-mastery — RHCSA Networking
**Subjects covered:** Terminal UI (`nmtui`), editing existing Ethernet connections, setting **manual** IPv4, CIDR addresses, gateway, DNS servers, activating connections, correlating changes with `nmcli` and `ip`, avoiding SSH lockout during edits
**Career arcs covered:** RHCSA (acceptable alternative when `nmcli` syntax slips), RHCE (still automate with Ansible — but know the UI), SRE (serial console rescue), DevOps (minimal rescue images), AI/MLOps (headless-ish GPU nodes with serial BMC)
**Prerequisite:** Conceptual understanding of IPv4 address, prefix, gateway, and DNS from Lab 31
**Time Estimate:** 35 to 50 minutes
**Difficulty arc:** Task 1 foundation · 2–3 navigate `nmtui` edit · 4–5 correlate NM fields · 6 exam-realistic capstone

---

## Objective

Not every exam room brain freeze happens on `nmcli` syntax — sometimes the **fastest safe path** is a guided editor. By the end of this lab you can launch `nmtui`, edit a wired connection's IPv4 configuration to **Manual**, enter address/prefix, gateway, and DNS, activate the change, and verify with `ip` — then revert cleanly.

The capstone is the RHCSA-realistic prompt: *"Using `nmtui`, set interface `eth0`'s connection to manual IPv4 `192.168.60.10/24`, gateway `192.168.60.1`, DNS `8.8.8.8`; activate; show `ip` proof; return to DHCP."*

> **Lab safety note:** Interactive IP edits can **drop SSH** if the new subnet is wrong. Prefer **virtual console** or **hypervisor terminal** while learning. Substitute instructor-provided addresses for your real LAN.

---

## Concept: `nmtui` Is a Front-End to the Same NM Profiles as `nmcli`

`nmtui` (NetworkManager Text User Interface) manipulates the **same connection profiles** on disk that `nmcli con mod` edits. There is no "secret" parallel config — only a menu system that fills in `ipv4.method`, `ipv4.addresses`, and friends for you.

```
   ┌───────────────────────────────────────────────┐
   │  nmtui menus                                 │
   │     Edit a connection                        │
   │       IPv4 CONFIGURATION → Manual            │
   │       Address 192.168.60.10/24               │
   │       Gateway 192.168.60.1                   │
   │       DNS servers 8.8.8.8                    │
   │     Activate a connection                    │
   └───────────────────┬──────────────────────────┘
                       ▼
   ┌───────────────────────────────────────────────┐
   │  NetworkManager persists + applies changes    │
   │       verified with `nmcli` / `ip`            │
   └───────────────────────────────────────────────┘
```

> **Why this matters:** EX200 rewards **correct final state**, not which tool opened the profile. Knowing `nmtui` is an insurance policy against typos under stress.

---

## 📜 Why Text User Interfaces for Networking Persist — The Story

Before ubiquitous graphical desktops, administrators configured PPP dial-up links and static LANs through **menu-driven TUI tools** or flat files. As Linux servers moved into datacenters, the serial console remained the **last-resort** interface when SSH failed, disk was full, or boot landed in emergency mode.

NetworkManager unified disparate scripts behind a single daemon — and shipped **editor UIs** so humans without memorized property keys could still survive. `nmtui` is deliberately simpler than the full GNOME control panel: it fits over **SSH** and **serial** lines, uses curses-style navigation, and maps 1:1 to NM's connection model.

Cloud images and appliance builders still encounter environments where **only** a TUI is comfortable — nested virt serial consoles, IPMI SOL sessions, or air-gapped jump boxes. Automation (`nmcli`, Ansible) won day-to-day operations, but TUIs remain the **human recovery path**.

> **The point of the story:** `nmtui` is not a crutch — it is a **resilience interface**. Master it once; use it rarely; be grateful on the day the network is half-dead.

---

## 👪 The `nmtui` Surface Area — Who Lives There

### Top-level menu entries

| Entry | What you do there |
|---|---|
| **Edit a connection** | Change IPv4/IPv6 methods, addresses, DNS, routes |
| **Activate a connection** | Choose which profile goes live on a carrier |
| **Set system hostname** | Transient/persistent hostname (use sparingly in this lab) |

### IPv4 configuration choices you will use

| UI value | NM property | Meaning |
|---|---|---|
| **Automatic** | `ipv4.method auto` | DHCP (typical client) |
| **Manual** | `ipv4.method manual` | Operator-specified IPv4 |
| **Disabled** | `ipv4.method disabled` | No IPv4 on this profile |

### Post-edit verification commands

| Command | Confirms |
|---|---|
| `nmcli -f ipv4 con show NAME` | Profile fields saved |
| `ip -4 addr show DEV` | Kernel applied address |
| `ip -4 route show default` | Default route present |

> **The point of the family tree:** Menus change layout slightly across minor versions — **verification commands** are the stable contract.

---

## 🔬 The Anatomy of an `nmtui` Edit Session — In One Diagram

```
$ sudo nmtui
  │
  ├─► "Edit a connection"
  │       └─► Select "Wired connection 1" (example)
  │             ├─► IPv4 CONFIGURATION → Manual
  │             ├─► Addresses → Add → 192.168.60.10/24
  │             ├─► Gateway → 192.168.60.1
  │             ├─► DNS servers → 8.8.8.8
  │             └─► OK → Quit
  │
  └─► "Activate a connection"
          └─► Deactivate + Activate same profile (forces apply)

Terminal UI notes:
  Arrow keys move │ Tab switches fields │ Space toggles │ Enter confirms
  Always scroll to "OK" before leaving a form — otherwise edits may discard
```

> **Reading rule:** **Activate a connection** after editing — the exam equivalent of `nmcli con up`.

---

## 📚 `nmtui` Workflow Reference Table

| Goal | UI path | CLI equivalent (for study) |
|---|---|---|
| DHCP client | IPv4 → Automatic | `nmcli con mod CON ipv4.method auto` |
| Static IPv4 | IPv4 → Manual + addresses | `nmcli con mod CON ipv4.method manual ipv4.addresses …` |
| Add gateway | Gateway field | `ipv4.gateway …` |
| Add DNS | DNS servers field | `ipv4.dns …` |
| Apply now | Activate a connection | `nmcli con up CON` |
| Abort safely | Quit without OK | No persistence |

> **Rule one of `nmtui`:** Scroll to **OK** and confirm — half-saved forms are the #1 "it did not stick" bug.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Alternative path if `nmcli` flags blur under pressure — still NM-native. |
| **RHCE candidate** | You will automate with Ansible — but rescue mode may only offer `nmtui`. |
| **SRE / Platform** | Serial console sessions: TUI beats hand-editing ifconfig-era files. |
| **DevOps** | Some minimal containers-host images include `nmtui` for bootstrap networking. |
| **AI / MLOps** | BMC serial fixes on training clusters — no browser, no VS Code — only TUIs. |

---

## 🔧 The 6 Tasks

> Six phases that build **menu navigation** + **CLI verification** discipline.

---

### Task 1 — Launch `nmtui`, identify version, and quit safely

**Purpose:** Confirm the package exists, open the UI, exit without changes.

```bash
sudo rpm -q NetworkManager
sudo nmtui
```

**Human-Readable Breakdown:** Print the installed NetworkManager RPM version (proxy for feature set), then launch `nmtui`. Immediately choose **Quit** from the main menu.

**Reading it left to right:** `nmtui` requires a TTY with sufficient dimensions — widen your terminal if the border draws oddly.

**The story:** First run is about **muscle memory for navigation**, not edits.

**Expected output:**

```text
NetworkManager-1.46.0-1.el9.x86_64
(+ opens full-screen TUI — no stdout until exit)
```

**Switches**

| Token | Meaning |
|---|---|
| `rpm -q` | Query exact NEVRA installed |
| `sudo nmtui` | Root avoids permission surprises editing system profiles |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `nmtui: command not found` | `sudo dnf install -y NetworkManager-tui` |
| Garbled screen | Resize terminal; set `TERM=xterm-256color` |
| SSH drops later | Expected if you misconfigure IP — use console |

---

### Task 2 — Inspect current IPv4 method from CLI before any UI edit

**Purpose:** Capture **before** snapshots so you can diff **after** `nmtui` changes.

```bash
DEV=eth0
CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: -v d="$DEV" '$1==d{print $2; exit}')
echo "CON=$CON"
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns con show "$CON"
ip -4 addr show dev "$DEV"
ip -4 route show default
```

**Human-Readable Breakdown:** Resolve the active connection for `eth0`, print NM IPv4 fields, print kernel address and default route.

**Reading it left to right:** This is the same correlation loop as Lab 33 — `nmtui` will change the first block; `ip` should follow after activation.

**The story:** Screenshots are ephemeral — typed CLI evidence persists in scrollback.

**Expected output:**

```text
CON=Wired connection 1
ipv4.method:                            auto
ipv4.addresses:                        192.168.122.45/24
ipv4.gateway:                           192.168.122.1
ipv4.dns:                               192.168.122.1
    inet 192.168.122.45/24 brd 192.168.122.255 scope global dynamic noprefixroute eth0
default via 192.168.122.1 dev eth0 proto dhcp src 192.168.122.45 metric 100
```

**Switches**

| Token | Meaning |
|---|---|
| `nmcli -f ipv4…` | Filter to IPv4 columns only |
| `ip -4 addr` | Kernel truth for addresses |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `CON` empty | No active NM profile — activate wired connection first |
| Static already | Skip ahead — you can practice toggling DHCP |

---

### Task 3 — Core operation: edit connection in `nmtui` to Manual IPv4 (guided)

**Purpose:** Perform the IPv4 conversion using menus — documented as explicit steps you execute inside the TUI.

```bash
sudo nmtui
```

**Human-Readable Breakdown:** In the TUI: **Edit a connection** → select your wired profile (e.g., `Wired connection 1`) → set **IPv4 CONFIGURATION** to `Manual` → **Addresses** → `Add` → enter `192.168.60.10/24` → **Gateway** `192.168.60.1` → **DNS servers** `8.8.8.8` → **OK** → return to main menu → **Quit**.

**Reading it left to right:** Manual mode requires **all critical fields** your exam task lists — missing gateway yields no default route.

**The story:** Treat `/24` as inseparable from the address — the UI expects CIDR notation in the same token NM uses on disk.

**Expected output:**

```text
(UI session completes without stderr — verification happens in Task 4 CLI output)
```

**Switches**

| Token | Meaning |
|---|---|
| **Manual** | Static operator-defined IPv4 |
| **Addresses Add** | Creates `ipv4.addresses` list entry |
| **OK** | Commits form to NM profile |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Cannot find Addresses | Scroll down — small terminals hide lower fields |
| Gateway field greyed | Pick Manual first — Automatic hides static fields |
| DNS not applied | Remove trailing spaces; separate multiple DNS with spaces |

---

### Task 4 — Activate the connection inside `nmtui` and verify with `nmcli` + `ip`

**Purpose:** Apply the edited profile without opening a second tool — then prove it stuck.

```bash
sudo nmtui
```

**Human-Readable Breakdown:** From main menu choose **Activate a connection** → highlight your wired profile → if it shows as activated, **Deactivate** then **Activate** to force reapply → **Quit**. Then run CLI checks.

```bash
CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: '$1=="eth0"{print $2; exit}')
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns con show "$CON"
ip -4 addr show dev eth0
ip -4 route show default
```

**Human-Readable Breakdown (CLI block):** Re-read NM fields and kernel view — expect `manual`, `192.168.60.10/24`, gateway `192.168.60.1`, DNS `8.8.8.8`, and `proto static` on default route.

**Reading it left to right:** Deactivate/activate is the TUI equivalent of `nmcli con down && nmcli con up` — ensures DHCP leases are dropped when moving to static.

**The story:** Many first attempts "look saved" but never rebind — activation is the apply step.

**Expected output:**

```text
ipv4.method:                            manual
ipv4.addresses:                        192.168.60.10/24
ipv4.gateway:                           192.168.60.1
ipv4.dns:                               8.8.8.8
    inet 192.168.60.10/24 brd 192.168.60.255 scope global noprefixroute eth0
default via 192.168.60.1 dev eth0 proto static metric 100
```

**Switches**

| Token | Meaning |
|---|---|
| **Activate a connection** | TUI-driven `nmcli con up` |
| `proto static` | Route from manual NM config |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ip` still shows DHCP | Activation targeted Wi-Fi profile — re-check selection |
| No DNS resolution | Add secondary DNS or check routing |

---

### Task 5 — Edge case: toggle "Automatically connect" and observe `AUToconnect`

**Purpose:** See how the UI maps to the persistent **autoconnect** property — relevant for laptops vs servers.

```bash
CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: '$1=="eth0"{print $2; exit}')
nmcli -f connection.autoconnect,connection.autoconnect-priority con show "$CON"
```

**Human-Readable Breakdown:** Print autoconnect yes/no and priority — optionally flip **Automatically connect** checkbox in `nmtui` and re-read this output.

**Reading it left to right:** `connection.autoconnect-priority` breaks ties when multiple profiles match one device.

**The story:** Servers want autoconnect **yes** on the primary uplink; lab multi-homed hosts may intentionally disable secondary profiles.

**Expected output:**

```text
connection.autoconnect:                yes
connection.autoconnect-priority:         0
```

**Switches**

| Token | Meaning |
|---|---|
| `connection.autoconnect` | NM should start this profile when link appears |
| `autoconnect-priority` | Higher wins when competing |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Profile will not come up at boot | Enable autoconnect in UI or `nmcli con mod CON connection.autoconnect yes` |

---

### Task 6 — Capstone: repeat static IPv4 via `nmtui`, prove with `getent`, revert to DHCP

**Task statement:** *"Using only `nmtui` for edits/activation, configure `eth0`'s active wired profile with manual IPv4 `192.168.60.10/24`, gateway `192.168.60.1`, DNS `8.8.8.8`. Verify with `ip` and `getent hosts redhat.com`. Then revert the profile to DHCP and confirm renewal."*

**Purpose:** Full exam-style narrative with safe cleanup.

```bash
sudo nmtui
# (UI) Edit → Manual 192.168.60.10/24 gateway 192.168.60.1 DNS 8.8.8.8 → OK
# (UI) Activate → deactivate/activate wired profile → Quit

CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: '$1=="eth0"{print $2; exit}')
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns con show "$CON"
ip -4 addr show dev eth0
ip -4 route show default
getent hosts redhat.com | head -n 2
```

**Human-Readable Breakdown:** Perform capstone edits purely in `nmtui`, then verify NM fields, kernel address/route, and DNS resolution path.

**Layer stack you validated:**

```text
nmtui → writes NM profile → NM applies netlink config → kernel FIB + addresses → libc resolver uses DNS from NM
```

**The story:** This is the **human-friendly** twin of Lab 31's `nmcli` spine — same properties, different UX.

**Expected output:**

```text
ipv4.method:                            manual
ipv4.addresses:                        192.168.60.10/24
ipv4.gateway:                           192.168.60.1
ipv4.dns:                               8.8.8.8
    inet 192.168.60.10/24 brd 192.168.60.255 scope global noprefixroute eth0
default via 192.168.60.1 dev eth0 proto static metric 100
209.132.183.44   redhat.com
209.132.183.45   redhat.com
```

**Cleanup**

```bash
CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: '$1=="eth0"{print $2; exit}')
sudo nmcli con mod "$CON" ipv4.method auto
sudo nmcli con mod "$CON" ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
sudo nmcli con down "$CON" && sudo nmcli con up "$CON"
ip -4 addr show dev eth0
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Locked out mid-capstone | Hypervisor console → fix gateway or DHCP revert |
| DNS stale after revert | `sudo nmcli con up "$CON"` again |
| `nmtui` not installed | `dnf install NetworkManager-tui` |

---

## 🔍 `nmtui` vs `nmcli` Decision Guide

```
Need to change networking on RHEL 9?
  │
  ├── "I know exact NM properties"
  │       └── ✅ `nmcli con mod` (scriptable) — Lab 31
  │
  ├── "I have a console but fuzzy memory"
  │       └── ✅ `nmtui` (guided)
  │
  ├── "Automation / CI"
  │       └── ✅ Ansible `nmcli` module — not `nmtui`
  │
  └── "Verify anything either tool claimed"
          └── ✅ `ip addr` + `ip route` + `nmcli con show`
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Launch `nmtui`, confirm package stack, quit safely
- [ ] 02 Capture pre-change `nmcli` + `ip` baseline for `eth0`
- [ ] 03 Edit wired profile to Manual static IPv4 via menus
- [ ] 04 Deactivate/activate in `nmtui`; verify with `nmcli` + `ip`
- [ ] 05 Inspect `connection.autoconnect` metadata
- [ ] 06 Capstone static + `getent` + DHCP cleanup

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Forgot Activate | `ip` unchanged | Use Activate menu or `nmcli con up` |
| Left form with Quit instead of OK | Lost edits | Re-enter Edit → OK |
| Wrong profile edited | Wi-Fi changed, not Ethernet | Match profile from `nmcli dev status` |
| Static IP on wrong LAN | SSH loss | Console recovery |
| Missing `/24` | Ambiguous mask | Always CIDR in one field |
| Expecting `nmtui` in containers | Not installed by default | Install package or use `nmcli` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Practice **one full** static edit + activate + `ip` verify under a timer — the exam rewards calm hands.

**RHCE candidate**
- State clearly: **TUI is for humans; playbooks are for fleets** — interviewers want that boundary.

**SRE / Platform interview**
- Describe using serial + `nmtui` when **SSHD is down** but link is up — classic war story structure.

**DevOps**
- Golden images: pre-install `NetworkManager-tui` if policy allows — reduces MTTR for edge sites.

**AI / MLOps**
- Document that training VLAN statics were set via NM, not hardcoded in container `run` scripts — auditability win.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 31 — Configure a Static IP Address | CLI-first twin of this lab |
| Lab 32 — Check Network Connectivity | Tests the post-`nmtui` path |
| Lab 33 — Display IP and Routing Info | Reads the kernel view after edits |
| Lab 34 — Inspecting Listening Sockets | Confirms services still bind post-change |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
