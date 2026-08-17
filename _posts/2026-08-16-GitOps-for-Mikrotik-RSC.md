---
layout: post
title: "Wrangling RouterOS Configs: Introducing GitOps for MikroTik with MKTXP"
date: 2026-08-16
categories: articles
tags: [MKTXP, MikroTik, RouterOS, GitOps, CLI Tools, Python]
---

If you have ever tried managing MikroTik RouterOS configurations under version control, you already know the pain: RouterOS `/export` files are intrinsically messy and notoriously hard to handle in Git.

Between arbitrary 80-column line wrapping with trailing backslashes (`\`) breaking words in the middle, unordered dictionaries that jitter and reshuffle on every export, and multi-line scripts crammed inline with escaped `\r\n` characters, standard `git diff` output quickly turns into an unreadable wall of noise.

Over the last few months, while revising my homelab network and router setups across multiple devices, I found myself asking a simple question:

> *What would it take to manage RouterOS configurations cleanly, right alongside all my other dotfiles and infrastructure repos that sit neatly under Git?*

The task looked challenging at first glance, but then thinking some more through the mechanics -- it didn't need to be overly complex. At its core, a RouterOS export is a structured language so instead of diving into hairy regex adventures how about parsing it into an **Abstract Syntax Tree (AST)** and then just passing it through a **Chain of Responsibility** pipeline to normalize formatting, sort unordered sections deterministically, extract scripts into clean sidecars, split the monolithic output into modular component files, and maybe a bit more things like that along the way.

Still does not sound trivial? Well, but let's weigh the odds here -- keep doing tedious manual reviews and diff wrestling across a fleet of routers, or solve the problem once and for all?

So and here we are, after some trials and design iterations... behold: [`mktxp`](https://github.com/akpw/mktxp) extends its reach beyond Prometheus metrics exporter duties and steps onto the new ground of **RouterOS GitOps configuration management**!

---

## How It Works: The Pipeline Architecture

Rather than relying on brittle regex search-and-replace, the new `mktxp rsc` engine treats RouterOS configuration exports as a true formal grammar. Here is the somewhat simplified architecture sketch:

```text
[ Raw export.rsc / Live SSH Stream ]
                │
                ▼
      [ Lexer & AST Parser ]
                │
                ▼
      [ Middleware Pipeline ]
   ├── Determinism Sorter (ordered vs unordered paths)
   ├── Script Extractor (creates clean sidecar .rsc)
   └── Sanitizer (strips transient MAC attributes)
                │
                ▼
[ Handler Chain (Chain of Responsibility) ]
   ├── 01-base Handler       (/interface bridge, ethernet, vlan, ...)
   ├── 02-wifi Handler       (/interface wifi, caps-man, ...)
   ├── 03-system Handler     (/system, /user, /certificate, ...)
   ├── 04-ip Handler         (/ip address, pool, route, dns, ...)
   ├── 05-dhcp-leases        (/ip dhcp-server lease)
   ├── 06-firewall Handler   (/ip firewall, /ipv6 firewall)
   ├── 07-lte Handler        (/interface lte, /tool sms)
   ├── 08-wireguard Handler  (/interface wireguard)
   └── 99-other Handler      (Default fallback catch-all)
                │
                ├──► Split Emitter  ──► [ 01-base.rsc, 02-wifi.rsc, ..., Script.rsc ]
                └──► Format Emitter ──► [ clean_export.rsc ]
```

### Key Engineering Components:

1. **Robust Tokenization & AST Parsing**:  
   Handles multi-command sections, slash paths, keyword parameters, unquoted spaces, escaped quotes, and multiline script code blocks.
2. **Deterministic Sorting for Stable Git Diffs**:  
   Unordered sections (like `/ip firewall address-list`, `/caps-man channel`, or `/interface wifi configuration`) are automatically sorted alphabetically. Strictly ordered sections (such as `/ip firewall filter` rules or `/interface bridge port` bindings) preserve their exact sequence.
3. **Sidecar Script Extraction**:  
   Multi-line scripts stored under `/system script` can be optionally pulled out into standalone, readable `.rsc` files, leaving clean `# Note: <name> exported to <name>.rsc` pointer comments in the main configuration.
4. **Live In-Flight Streaming over SSH**:  
   Fetch configs directly from your live routers with native SSH key authentication (`~/.ssh/id_ed25519`, `ssh-agent`, or `--ssh-key`). The raw export streams directly into RAM, processes in-flight, and emits clean files with zero temporary file residue on disk.
5. **Zero Extra Dependencies**:  
   Implemented entirely with Python standard library capabilities, keeping Homebrew, pip/pipx, and container installations fast and lightweight.

---

### Extensible by Design: Add Custom Handlers with Zero Code

The handler pipeline is completely dynamic. If you want to carve out dedicated files for subsystems like BGP routing, VLAN switches, or custom tunnels, you don't need to write any Python. Just add your handler name to `handler_order` and define its matching slash paths:

```ini
[RSC]
    handler_order = base, wifi, system, ip, dhcp-leases, firewall, lte, wireguard, bgp

    # Claims all BGP / dynamic routing paths into a dedicated 09-bgp.rsc file
    handler_bgp = /routing bgp, /routing bfd, /routing filter, /routing ospf
```

The engine uses **longest-prefix specificity matching** (ensuring `/routing bgp` is claimed by `handler_bgp` rather than general `/routing`), automatically numbers the new file (`09-bgp.rsc`), and gracefully routes any unmapped paths into `99-other.rsc`.

---

## The New Commands in Action

The new functionality is exposed through the `mktxp rsc` subcommands: `format` and `split`.

### 1. Formatting a Single Monolithic Config (`format`)

If you prefer keeping your router configuration in a single `.rsc` file, `format` strips out line-wrap noise, sorts unordered sections deterministically, and formats commands cleanly onto single logical lines:

```bash
# Format from a local file
$ mktxp rsc format -i raw_backup.rsc -o clean_backup.rsc

# Or fetch live directly from a router defined in mktxp.conf over SSH
$ mktxp rsc format -en Gateway-Router -o ./backups/Gateway-clean.rsc
```

Before and after line wrapping:

```diff
- /interface bridge add admin-mac=48:A9:8A:6F:21:4E auto-mac=no comment="LAN Bridge" frame-types=\
- admit-only-vlan-tagged name=bridge_main vlan-filtering=yes
+ /interface bridge add admin-mac=48:A9:8A:6F:21:4E auto-mac=no comment="LAN Bridge" frame-types=admit-only-vlan-tagged name=bridge_main vlan-filtering=yes
```

---

### 2. Splitting into Modular GitOps Directories (`split`)

For modular, maintainable infrastructure repositories, `split` decomposes a monolithic export into structured, numbered component files:

```bash
# Split a local file
$ mktxp rsc split -i MyRouter.rsc --extract-scripts

# Or split live directly from a router over SSH
$ mktxp rsc split -en Gateway-Router --extract-scripts
```

```text
Fetching live RouterOS export from 'Gateway-Router' (192.168.88.1)...
Successfully split RouterOS export into 12 files in: ./exports/Gateway-Router/
  |- 01-base.rsc
  |- 02-wifi.rsc
  |- 03-system.rsc
  |- 04-ip.rsc
  |- 05-dhcp-leases.rsc
  |- 06-firewall.rsc
  |- 07-lte.rsc
  |- 08-wireguard.rsc
  |- Cloud-Backup.rsc
  |- DDNS-Updater.rsc
  |- Failover-Check.rsc
  |- LetsEncrypt.rsc
```

Each component is dedicated to its subsystem:
- **`01-base.rsc`**: Bridges, VLAN interfaces, physical Ethernet ports, interface lists.
- **`02-wifi.rsc`**: WiFi 6/7, CAPsMAN configurations, channels, security profiles.
- **`03-system.rsc`**: Users, services, certificates, logging, watchdog, system settings.
- **`04-ip.rsc`**: IP addressing, pools, DNS, static routes, DHCP server configuration.
- **`05-dhcp-leases.rsc`**: Static DHCP server lease bindings.
- **`06-firewall.rsc`**: Connection tracking, sorted address lists, RAW/Mangle/Filter/NAT chains.
- **`07-lte.rsc`**: LTE interfaces, APNs, SMS settings.
- **`08-wireguard.rsc`**: WireGuard interfaces and peer definitions.
- **`*.rsc` (Sidecars)**: Discrete, syntax-highlighted RouterOS scripts.
- **`99-other.rsc`**: Fallback catch-all for custom peripheral or vendor-specific settings.

---

### 3. Automated Directory Scoping & Conflicts Prevention

Running `split` without passing `-d` automatically scopes the output into an isolated directory named after the router entry (or input file stem):

```bash
$ mktxp rsc split -i CoreSwitch.rsc
# Emits to ./exports/CoreSwitch/

$ mktxp rsc split -en Gateway-Router
# Emits to ./exports/Gateway-Router/
```

This makes batch backup scripts and automated CI/CD jobs clean and collision-free.

---

## Command Line Options Reference

Both commands share a somewhat self-explanatory set of CLI options:

### `mktxp rsc format -h`
```text
$ mktxp rsc format -h
usage: MKTXP rsc format [-h] [-i INPUT]
                        [-en ['Core-Router', 'Office-AP', 'Switch-1']]
                        [-o OUT] [--show-sensitive] [--user USER]
                        [--ssh-key SSH_KEY] [--ssh-port SSH_PORT] [--wrap]
                        [--wrap-col WRAP_COL] [--strip-macs]

Formats raw RouterOS export into a single clean .rsc file

options:
  -h, --help            show this help message and exit
  -i, --input INPUT     Input RouterOS export .rsc file path
  -en, --entry-name ['Core-Router', 'Office-AP', 'Switch-1']
                        Router entry name from mktxp.conf for live export over SSH
  -o, --out OUT         Output file path (defaults to stdout)
  --show-sensitive      Include passwords and sensitive keys in live export
  --user USER           Override SSH username for live export
  --ssh-key SSH_KEY     Path to SSH private key for live export
  --ssh-port SSH_PORT   Override SSH port (default: 22)
  --wrap                Wrap long lines with backslashes
  --wrap-col WRAP_COL   Line wrapping column width (default: 80)
  --strip-macs          Strip dynamic MAC addresses
```

### `mktxp rsc split -h`
```text
$ mktxp rsc split -h
usage: MKTXP rsc split [-h] [-i INPUT]
                       [-en ['Core-Router', 'Office-AP', 'Switch-1']]
                       [-d OUT_DIR] [--show-sensitive] [--user USER]
                       [--ssh-key SSH_KEY] [--ssh-port SSH_PORT]
                       [--no-numbered] [--wrap] [--wrap-col WRAP_COL]
                       [--extract-scripts] [--strip-macs]

Splits raw RouterOS export into modular GitOps directory structure

options:
  -h, --help            show this help message and exit
  -i, --input INPUT     Input RouterOS export .rsc file path
  -en, --entry-name ['Core-Router', 'Office-AP', 'Switch-1']
                        Router entry name from mktxp.conf for live export over SSH
  -d, -o, --out-dir OUT_DIR
                        Output directory to emit .rsc files
  --show-sensitive      Include passwords and sensitive keys in live export
  --user USER           Override SSH username for live export
  --ssh-key SSH_KEY     Path to SSH private key for live export
  --ssh-port SSH_PORT   Override SSH port (default: 22)
  --no-numbered         Disable numeric prefixes on output files
  --wrap                Wrap long lines with backslashes
  --wrap-col WRAP_COL   Line wrapping column width (default: 80)
  --extract-scripts     Extract multi-line scripts into separate .rsc files
  --strip-macs          Strip dynamic MAC addresses
```

---

## Configuration & Customization

The split hierarchy and defaults can be tuned in your `_mktxp.conf` under the `[RSC]` section:

```ini
[RSC]
    base_dir = './exports'                    # Base destination directory
    numbered_files = True                     # Prefix files with 01-, 02-, ...
    wrap_lines = False                        # Single-line mode for clean git diffs
    extract_scripts = False                   # Extract scripts to standalone .rsc sidecars
    strip_mac_addresses = False               # Strip dynamic MACs for hardware swaps
    ssh_port = 22                             # Default SSH port for live exports
    ssh_timeout = 15                          # SSH connection timeout in seconds
    show_sensitive = False                    # Default sensitive export flag (passwords/keys)

    handler_order = base, wifi, system, ip, dhcp-leases, firewall, lte, wireguard

    handler_base = /interface bridge, /interface ethernet, /interface vlan, /interface list
    handler_wifi = /caps-man, /interface wifi, /interface wireless
    handler_system = /system, /user, /certificate, /zerotier, /snmp, /ip ssh
    handler_dhcp-leases = /ip dhcp-server lease
    handler_ip = /ip address, /ip pool, /ip dhcp-server, /ip dns, /ip route
    handler_firewall = /ip firewall, /ipv6 firewall
    handler_lte = /interface lte, /tool sms
    handler_wireguard = /interface wireguard
```

### Extensible by Design: Add Custom Handlers with Zero Code

The handler pipeline is completely dynamic. If you want to carve out dedicated files for subsystems like BGP routing, VLAN switches, or custom tunnels, you don't need to write any Python. Just add your handler name to `handler_order` and define its matching slash paths:

```ini
[RSC]
    handler_order = base, wifi, system, ip, dhcp-leases, firewall, lte, wireguard, bgp

    # Claims all BGP / dynamic routing paths into a dedicated 09-bgp.rsc file
    handler_bgp = /routing bgp, /routing bfd, /routing filter, /routing ospf
```

The engine uses **longest-prefix specificity matching** (ensuring `/routing bgp` is claimed by `handler_bgp` rather than general `/routing`), automatically numbers the new file (`09-bgp.rsc`), and gracefully routes any unmapped paths into `99-other.rsc`.

---

## Installation & Getting Started

`mktxp` can be installed or run via your tool of choice:

- **With [Homebrew](https://brew.sh)** (macOS / Linux):
  ```bash
  $ brew install mktxp
  ```
- **With `pipx`** (Isolated CLI environment):
  ```bash
  $ pipx install mktxp
  ```
- **With `pip` / PyPI**:
  ```bash
  $ pip install --upgrade mktxp
  ```
- **With Docker**:
  ```bash
  $ docker pull ghcr.io/akpw/mktxp:latest
  ```

For full documentation, configuration templates, and metrics exporter guides, check out the [MKTXP GitHub Repository](https://github.com/akpw/mktxp).

---

## Wrapping Up

With `mktxp rsc`, version-controlling MikroTik routers shifts from an annoying chore to a single, automated step. Diffs become clean and meaningful, scripts are easily readable as standalone files, and entire router configurations can be stored safely in Git alongside the rest of your infrastructure.

Give it a spin with `mktxp rsc split -h` or `mktxp rsc format -h` and happy GitOps-ing!
