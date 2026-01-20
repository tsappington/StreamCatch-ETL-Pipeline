# StreamCatch: Enterprise Media Capture Platform

**Role:** Lead Software Architect
**Status:** Proprietary / Closed Source
**Platform:** Cross-platform (macOS, Windows 10, Windows 7)

---

## System Overview

StreamCatch is a high-availability media streaming capture platform designed for automated, scheduled recording of internet audio and video streams. The system provides 24/7 unattended operation with sophisticated fault tolerance, enabling broadcast media organizations to reliably capture and archive streaming content at scale. The platform processes dozens of concurrent streams while maintaining sub-second scheduling precision and zero-gap recording integrity.

---

## Tech Stack

| Category | Technologies |
|----------|--------------|
| **Core Language** | Python 3.8–3.12 (dual-compatibility architecture) |
| **Media Processing** | FFmpeg (subprocess orchestration, segment muxing) |
| **Async Runtime** | asyncio, concurrent.futures |
| **Data Layer** | SQLite (async batching), Pandas (schedule parsing) |
| **Configuration** | Pydantic (validation), INI/CSV parsers (hot-reload) |
| **Logging & Observability** | Loguru (structured logging), per-event log isolation |
| **Distribution** | PyInstaller (standalone Windows executables) |
| **Notification** | SMTP (circuit-breaker pattern) |

---

## Key Challenges Solved

### Zero-Downtime Configuration Hot-Reload
Implemented a sophisticated hot-reload system spanning 33 modules (~15K lines) that allows runtime configuration changes without interrupting active recordings. Key innovations include:
- **Recording isolation snapshots** that preserve configuration state for in-flight operations
- **Race condition protection** using tiered response handlers and lifecycle managers
- **Atomic rollback** with schema-versioned snapshots for failed reloads
- **File watcher integration** with debouncing to prevent reload storms

### Gap-Free Continuous Recording
Designed a segment muxer architecture that eliminates recording gaps during long-running 24/7 capture sessions:
- Single FFmpeg process with clock-aligned segment boundaries
- Manifest-based segment tracking with automatic file finalization
- Graceful restart capability that preserves recording continuity
- Duration validation and correction for segment boundaries

### Cross-Platform Async/Sync Abstraction
Engineered a strategy pattern abstraction layer supporting both modern async I/O (Python 3.9+) and legacy synchronous operations (Python 3.8/Windows 7):
- Unified recorder interface with platform-specific strategy implementations
- Transparent stdin/stdout pipe handling across subprocess models
- Graceful degradation for platforms without full asyncio support

### Fault-Tolerant Stream Handling
Built comprehensive resilience mechanisms for unreliable network streams:
- Automatic reconnection with exponential backoff
- Stall detection via file growth monitoring
- Connectivity healing with stream health scoring
- Memory pressure backpressure to prevent OOM conditions

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Configuration Layer                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ INI Parser  │  │ CSV Parser  │  │  Validator  │  │ Hot-Reload │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └─────┬──────┘ │
│         └────────────────┴────────────────┴───────────────┘        │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Scheduling Engine                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │  Schedule   │  │    DST      │  │  Interval   │  │   Event    │ │
│  │   Parser    │  │  Handling   │  │  Generator  │  │  Builder   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          Event Pipeline                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │   Event     │  │  Lifecycle  │  │   State     │  │  Progress  │ │
│  │   Queue     │  │  Manager    │  │  Machine    │  │ Coordinator│ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Recording Layer                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Strategy Pattern                          │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  │  │
│  │  │ Async Strategy │  │ Win7 Strategy  │  │ Segment Muxer  │  │  │
│  │  │   (Modern)     │  │   (Legacy)     │  │  (Continuous)  │  │  │
│  │  └────────────────┘  └────────────────┘  └────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  FFmpeg Process Manager                      │  │
│  │  • Subprocess orchestration    • Stderr monitoring           │  │
│  │  • Graceful termination        • Reconnection handling       │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Persistence & Output                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │  Database   │  │   Event     │  │   File      │  │   Email    │ │
│  │  (SQLite)   │  │   Logs      │  │  Movement   │  │  Alerts    │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Configuration Ingestion**: INI and CSV files are parsed, validated against Pydantic schemas, and loaded into an immutable configuration snapshot.
2. **Schedule Generation**: The scheduling engine processes recording patterns, handles DST transitions, and generates discrete recording events with precise time boundaries.
3. **Event Orchestration**: Events flow through an async pipeline where lifecycle managers coordinate state transitions, handle concurrent recordings, and maintain isolation guarantees.
4. **Recording Execution**: Platform-appropriate strategies spawn FFmpeg subprocesses, manage I/O streams, and monitor recording health in real-time.
5. **Persistence**: Completed recordings are validated, moved to destination storage (with optional pool-based routing), and logged to both per-event files and a central database.

---

## Design Principles

- **Fail-Safe Defaults**: The system prioritizes data preservation—recordings continue even when peripheral systems (email, database) experience failures.
- **Observable Operations**: Every recording event generates an isolated log file capturing the complete lifecycle, enabling post-hoc debugging without log correlation.
- **Graceful Degradation**: Platform limitations (legacy Python, reduced async support) are handled through abstraction layers rather than feature removal.
- **Zero-Trust Streams**: External streams are assumed unreliable; the system implements defensive reconnection, validation, and health monitoring at every integration point.

---

*Note: This repository serves as an architectural overview. The source code is proprietary and not publicly available.*
