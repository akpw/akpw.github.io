---
layout: post
title: "Under the Hood: Refactoring MKTXP for 2.0"
date: 2026-08-28
categories: articles
tags: [MKTXP, MikroTik, RouterOS, Python, Architecture, Refactoring, Prometheus, CLI Tools]
---

It is interesting how software projects take on a life of their own over time.

When [`mktxp`](https://github.com/akpw/mktxp) first started, the premise was refreshingly simple: connect to MikroTik RouterOS devices, harvest telemetry via API, and serve metrics to Prometheus. Over the years, it grew into a reliable workhorse with tons of configuration options and 10K+ daily pulls.

Fast forward to recently, MKTXP took a leap forward by introducing built-in [GitOps configuration management](https://akpw.github.io/articles/2026/08/16/GitOps-for-Mikrotik-RSC.html) through `mktxp rsc` — parsing, sorting, and splitting raw `.rsc` exports into clean, modular version-controlled domains.

Now, with the latest direction towards interactive **CLI network diagnostics** (`mktxp diag` / `mktxp print`) for immediate live visibility into device health, wireless rates, IP connection tracking, and DHCP lease bindings right inside the terminal without requiring a full Prometheus/Grafana stack — imagine asking questions like these directly from the command line:
 
- Am I under attack? Show me live dynamic threat bans like SSH brute-forcers, port scanners, etc. 
- Where is this device connected? Which AP, SSID, interface, and IP? 
- Which clients have poor RSSI (< -75 dBm) or low negotiated link rates? 
- Who sits on the 2G band in that part of the office? And who are the recent joiners? 
- What are those mystery devices on the network with no DHCP hostname and no comment? 
- Who are the top talkers with the highest open socket count? 
- Show me bandwidth hogs in real time, whoever is streaming 4K video or running massive downloads. 
- Which upstream gateways or Netwatch ICMP probes are down? 
 
More on that later, but for now let's see what the architecture looks like and what needed to be updated to get there:

```text
               ┌─────────────────────────────────────────┐
               │                MKTXP 2.0                │
               │  MikroTik Diagnostics, GitOps & Export  │
               └────────────────────┬────────────────────┘
           ┌────────────────────────┼────────────────────────┐
           ▼                        ▼                        ▼
┌─────────────────────┐ ┌──────────────────────┐ ┌─────────────────────┐
│     mktxp export    │ │      mktxp rsc       │ │      mktxp diag     │
│ Prometheus Exporter │ │ GitOps Configuration │ │       Live CLI      │
│ & HTTP Multi-Target │ │ AST Formatter/Split  │ │ Network Diagnostics │
└─────────────────────┘ └──────────────────────┘ └─────────────────────┘
```


---

## Why Refactor?

CLI tools are unbeatable in terms of speed and efficiency, but they inevitably introduce options and flags. While the core export architecture has always been modular and reliable, the surrounding configuration and presentation layers were originally built around a single daemon loop, not an extensible CLI framework.

To support these new capabilities without accumulating technical debt, it was time to step back and reorganize the codebase around clear, decoupled domains.
The original design served its initial purpose well, but now with added GitOps tools and interactive diagnostics, three main areas needed attention:

1. **The Parameter Explosion in CLI Dispatch**:  
   Adding a single flag (like `--low-signal` or `--band`) required threading that parameter through 5 separate files (`options.py` -> `dispatch.py` -> `base_proc.py` -> `*_out.py` -> `output.py`). With more diagnostic domains on the roadmap, this tight coupling would quickly become unmaintainable.

2. **Overloaded Output Handling (`output.py`)**:  
   `BaseOutputProcessor` grew into a 450-line class that combined pure math (rate/duration parsers), string glob matching, table formatting, record augmentation, and the Prometheus HTTP server lifecycle.

3. **Monolithic Configuration (`config.py`)**:  
   An 800-line module handled OS directory discovery, INI parsing, template auto-injection, editor launching, and CLI actions (`show`, `edit`) all in one place.

---

## The MKTXP 2.0 Modular Architecture

After analysis and a few iterations, the repository was reorganised into these focused, single-responsibility packages:

```text
mktxp/
├── cli/                     # CLI Interface & Orchestration Layer
│   ├── config/              # Modular config (os_paths, keys, models, loader, actions)
│   ├── output/              # CLI table schemas (tables.py) & diagnostic formatters
│   ├── options.py           # Streamlined ArgumentParser with dynamic domain discovery
│   └── dispatch.py          # Delegating CLI command router (MKTXPDispatcher)
│
├── diag/                    # Dynamic Diagnostic Domain Engine
│   ├── registry.py          # DiagRegistry for automatic command discovery
│   ├── base.py              # BaseDiagHandler ABC interface
│   ├── wireless.py          # Wireless & CAPsMAN diagnostic domain (-cc, -wc)
│   ├── dhcp.py              # DHCP leases diagnostic domain (-dc)
│   ├── connections.py       # IP connection stats diagnostic domain (-cn)
│   ├── kid_control.py       # Kid Control diagnostic domain (-kc)
│   ├── address_lists.py     # Firewall address lists diagnostic domain (-al)
│   └── netwatch.py          # Netwatch ICMP monitoring diagnostic domain (-nw)
│
├── exporter/                # Dedicated Prometheus Exporter Package
│   ├── app.py               # Server daemon manager & thread lifecycle
│   ├── router.py            # MetricsRouter (/metrics and /probe dispatcher)
│   └── middleware.py        # PrometheusHeadersDeduplicatingMiddleware
│
├── flow/                    # Session Lifecycle & Data Enrichment
│   ├── processor/
│   │   └── enrichment.py    # Record enrichment (DHCP name resolution, MAC mapping)
│   ├── router_connection.py # Robust API & SSH connection wrappers with retry logic
│   └── router_entry.py      # RouterEntry runtime state & credential management
│
├── collector/               # Prometheus metric collector modules
├── datasource/              # RouterOS API query harvesters
├── rsc/                     # GitOps RSC AST Tokenizer, Parser, Formatter & Splitter
└── utils/                   # Shared pure utility modules (zero domain coupling)
    ├── units.py             # Pure unit parsers (bitrates, durations, signals, rates)
    ├── filtering.py         # Glob and multi-criteria wireless matching engine
    └── utils.py             # FSHelper, UniquePartialMatchList, ROS version helpers
```

---

## Key Design Improvements

### 1. Decomposing Presentation and Math
`output.py` was split into four distinct modules:
- **`mktxp.utils.units`**: Pure functions for parsing rates (`18M`, `54 Mbps`), durations (`15m`, `3d`), and signal levels (`-75 dBm`) to standardized integer values.
- **`mktxp.utils.filtering`**: Pure record matching for glob patterns (`-in`, `-ex`) and multi-criteria wireless thresholds.
- **`mktxp.cli.output.tables`**: Namedtuple table schemas and terminal-responsive `Texttable` formatting.
- **`mktxp.flow.processor.enrichment`**: Domain logic for resolving DHCP hostnames/comments and augmenting raw router records.

### 2. Strategy-Based Diagnostic Domain Handlers (`mktxp.diag`)
Instead of hardcoding flags in a central parser, each diagnostic domain (`wireless`, `dhcp`, `connections`, `kid_control`, `address_lists`, `netwatch`) implements `BaseDiagHandler`:

```python
class BaseDiagHandler(ABC):
    @property
    @abstractmethod
    def command_name(self) -> str:
        """Domain name (e.g. 'wireless', 'dhcp')."""
        pass

    @abstractmethod
    def register_arguments(self, parser) -> None:
        """Attach domain-specific CLI flags."""
        pass

    @abstractmethod
    def execute(self, router_entry, args: dict, diag_conf: dict) -> None:
        """Fetch data, apply filters, and render tables."""
        pass
```

`DiagRegistry` discovers handlers dynamically. Adding a new diagnostic feature now requires writing one self-contained handler file and registering it, touching zero shared dispatch code.

### 3. Standalone Exporter Package (`mktxp.exporter`)
The Prometheus daemon was extracted into its own package (`app.py`, `router.py`, `middleware.py`), isolating HTTP server worker threads, `/probe` routing, and Prometheus header deduplication from CLI logic.

---

## Clean Cut: No Legacy Shims

To keep the codebase clean and avoid technical debt, all legacy shims (`config.py`, `output.py`, `base_proc.py`) were deleted once imports were migrated. Every collector, datasource, and test now imports directly from its canonical package (`mktxp.cli.config`, `mktxp.cli.output`, `mktxp.utils.units`, `mktxp.flow.processor.enrichment`).

A smooth migration would never be possible without proper test coverage. With lots of additional tests added for individual areas, here finally comes the satisfying part:
```text
Unit & Integration Tests:
=================== 328 passed in 0.49s ===================
```

---

## What's Next

With the modular foundation in place, MKTXP 2.0 should now be ready for  next steps.
Stay tuned for more in the next post!

---

*For documentation, configuration templates, and dev builds, check out the [MKTXP GitHub Repository](https://github.com/akpw/mktxp).*
