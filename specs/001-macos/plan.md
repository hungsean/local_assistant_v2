# Implementation Plan: macOS Battery Monitor

**Branch**: `001-macos` | **Date**: 2025-10-08 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-macos/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Build a macOS menu bar application that monitors battery status in real-time, stores historical data locally, and displays trends via line charts. The MVP (P1) focuses on current battery display in the menu bar with a detailed window. P2 adds historical visualization, and P3 ensures the data model supports future cross-platform expansion (iOS, Android, Windows, Linux).

## Technical Context

**Language/Version**: Swift 5.7+ (included with Xcode 14+)
**Primary Dependencies**:
- AppKit (NSStatusBar for menu bar)
- SwiftUI (for UI components)
- IOKit (IOPowerSources for battery API)
- Swift Charts (for line chart visualization)
- GRDB.swift (SQLite wrapper for local storage)

**Storage**: SQLite via GRDB.swift (~43,200 readings, efficient time-range queries)
**Testing**: XCTest (native Swift testing framework)
**Target Platform**: macOS 10.15 (Catalina) or later
**Project Type**: single (macOS native application)
**Performance Goals**: UI updates <200ms, background memory <50MB, 7-day uptime stability
**Constraints**: Local-only processing (no external APIs), no admin privileges required, 1-minute data collection interval, file permissions 0600
**Scale/Scope**: Single-user local application, 30 days of battery history (~43,200 readings), menu bar + detailed window UI

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Principle I: Local-First & Security

✅ **PASS** - All data processing occurs locally (battery readings, storage, visualization)
✅ **PASS** - No external network calls required
✅ **PASS** - User preferences stored locally using macOS standard mechanisms
✅ **PASS** - No telemetry or analytics
⚠️  **VERIFY** - Ensure local storage uses appropriate file permissions (owner-only access)

### Principle II: User Experience Excellence

✅ **PASS** - Menu bar interface provides quick access (SC-001: <5 seconds)
✅ **PASS** - Clear error messages required (FR-006, FR-009)
✅ **PASS** - Performance targets defined (<200ms UI updates, <50MB memory)
⚠️  **VERIFY** - User workflow testing required before P2/P3 (per constitution)
✅ **PASS** - Quickstart guide will be generated (Phase 1)

### Principle III: Scope Discipline

✅ **PASS** - Exactly 3 user stories (P1: current status, P2: history, P3: multi-device prep)
✅ **PASS** - Single well-defined problem (battery monitoring for macOS)
✅ **PASS** - MVP (P1) clearly identified and independently deliverable
✅ **PASS** - P2 and P3 are optional extensions, not blocking MVP
✅ **PASS** - No feature creep detected in requirements

**Overall Gate Status**: ✅ **PASS** - Proceed to Phase 0 research

**Action Items**:
- Verify file permission strategy during Phase 0 research
- Plan UX testing checkpoints in tasks phase

## Project Structure

### Documentation (this feature)

```
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```
BatteryMonitor/                    # Xcode project root
├── BatteryMonitor/                # Main application target
│   ├── App/
│   │   ├── AppDelegate.swift      # Application lifecycle
│   │   └── Info.plist             # App configuration
│   ├── Models/
│   │   ├── BatteryReading.swift   # Battery data model
│   │   ├── Device.swift           # Device info model
│   │   ├── BatteryStatus.swift    # Current status model
│   │   └── UserPreferences.swift  # Settings model
│   ├── Services/
│   │   ├── BatteryService.swift   # IOKit battery API wrapper
│   │   ├── StorageService.swift   # Local data persistence
│   │   └── NotificationService.swift # Low battery alerts
│   ├── UI/
│   │   ├── MenuBar/
│   │   │   ├── StatusBarController.swift  # Menu bar icon & popover
│   │   │   └── MenuBarView.swift          # Quick status view
│   │   └── Windows/
│   │       ├── DetailWindowController.swift # Main detail window
│   │       ├── HistoryChartView.swift      # Line chart for history
│   │       └── PreferencesView.swift       # Settings panel
│   └── Resources/
│       └── Assets.xcassets/        # Icons and images
├── BatteryMonitorTests/
│   ├── UnitTests/
│   │   ├── BatteryServiceTests.swift
│   │   └── StorageServiceTests.swift
│   └── IntegrationTests/
│       └── DataFlowTests.swift
└── BatteryMonitor.xcodeproj       # Xcode project file
```

**Structure Decision**: Single macOS application using Xcode project structure. Chose native Swift/AppKit approach for menu bar integration and system-level battery API access. The structure separates concerns into Models (data), Services (business logic), and UI (presentation), following standard macOS app patterns.

## Complexity Tracking

*No constitutional violations detected - this section is empty.*
