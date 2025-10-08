# Tasks: macOS Battery Monitor

**Feature**: macOS Battery Monitor
**Branch**: `001-macos`
**Generated**: 2025-10-08

This document outlines the implementation tasks for the macOS Battery Monitor application, generated from the design artifacts.

## Phase 1: Project Setup

These tasks focus on initializing the project environment.

- **T001**: Create a new macOS App project in Xcode named `BatteryMonitor` with SwiftUI interface and Swift language.
- **T002**: Add the `GRDB.swift` package dependency to the project.
- **T003**: Create the directory structure as defined in `plan.md` inside the `BatteryMonitor` project.

## Phase 2: Foundational Models and Services

These are the blocking prerequisites that must be completed before any user story can be implemented.

- **T004**: [P] Implement the `Device.swift` data model as defined in `data-model.md`.
- **T005**: [P] Implement the `BatteryReading.swift` data model as defined in `data-model.md`.
- **T006**: [P] Implement the `UserPreferences.swift` data model as defined in `data-model.md`.
- **T007**: [P] Implement the `BatteryStatus.swift` model as defined in `contracts/BatteryService.md`.
- **T008**: Implement the `StorageServiceProtocol` as defined in `contracts/StorageService.md`.
- **T009**: Implement the `StorageService` class with database initialization, migration, and file permission setup (0600) as described in `research.md` and `contracts/StorageService.md`.
- **T010**: Implement the `BatteryServiceProtocol` as defined in `contracts/BatteryService.md`.
- **T011**: Implement the `BatteryService` class to fetch battery data using IOKit as described in `research.md`.
- **T012**: Implement the `NotificationServiceProtocol` as defined in `contracts/NotificationService.md`.
- **T013**: Implement the `NotificationService` class for sending user notifications.
- **T014**: Create unit tests for `StorageService` to verify database creation, writing, and reading.
- **T015**: Create unit tests for `BatteryService` to verify battery data fetching.

## Phase 3: User Story 1 - View Current Device Battery Status (P1)

**Goal**: As a macOS user, I want to quickly see my computer's battery level and status, so I know when to charge it.

- **T016**: [US1] Implement `StatusBarController.swift` to create and manage the menu bar item.
- **T017**: [US1] Implement `MenuBarView.swift` using SwiftUI to display the battery percentage and charging status.
- **T018**: [US1] Implement a popover for the status bar item that hosts the `MenuBarView` to satisfy FR-012.
- **T019**: [US1] Integrate `BatteryService` with `StatusBarController` to update the menu bar view with live battery data every 30 seconds.
- **T020**: [US1] Implement background monitoring in `AppDelegate` to call `BatteryService.startMonitoring` and save a `BatteryReading` to `StorageService` every 1 minute.
- **T021**: [US1] Implement the low battery notification feature. Use `NotificationService` to send a notification when the battery level drops below the thresholds defined in `UserPreferences`.
- **T022**: [US1] Implement the logic to request notification permissions from the user on first launch.
- **T023**: [US1] Add a settings option for the user to enable/disable notifications.
- **T024**: [US1] Add a settings option for the user to customize low battery notification thresholds.
- **T038**: [US1] Enhance `MenuBarView.swift` to also display the estimated remaining battery time, as required by FR-003.

## Phase 4: User Story 2 - Monitor Battery History (P2)

**Goal**: As a macOS user, I want to view my battery usage history over time to understand my usage patterns and battery health.

- **T025**: [US2] Implement `DetailWindowController.swift` to manage the detailed view window.
- **T026**: [US2] Create a detailed statistics view within the detail window, showing information like battery health, cycle count, etc.
- **T027**: [US2] Create `HistoryChartView.swift` using Swift Charts to display battery percentage over time.
- **T028**: [US2] Fetch battery history from `StorageService` and display it in `HistoryChartView`.
- **T029**: [US2] Add controls to `HistoryChartView` to allow users to select different time ranges (1h, 6h, 24h, 7d, 30d).
- **T030**: [US2] Enhance `HistoryChartView` to visually distinguish charging periods.

## Phase 5: User Story 3 - Prepare for Multi-Device Support (P3)

**Goal**: As a developer, I want to ensure the data structure supports future multi-device scenarios for smooth expansion.

- **T031**: [US3] Verify that all battery readings saved to `StorageService` include a `deviceId`.
- **T032**: [US3] Write a test to ensure that `StorageService` can store and retrieve battery readings for multiple device IDs.

## Phase 6: Polish & Cross-Cutting Concerns

- **T033**: Implement the "Launch at Login" feature using `SMAppService` and the legacy equivalent as described in `research.md`.
- **T034**: Create a settings view (`PreferencesView.swift`) to allow users to toggle the "Launch at Login" option.
- **T035**: Implement the data retention policy. Create a scheduled task that runs daily to delete records older than `dataRetentionDays` from `StorageService`.
- **T036**: Implement robust error handling for all services. Display user-friendly error messages for scenarios like "no battery detected" or "permission denied".
- **T037**: Review and improve accessibility for all UI components, ensuring VoiceOver compatibility.
- **T039**: [Polish] Implement performance and stability tests to verify memory usage (SC-003) and 7-day uptime (SC-007).
- **T040**: [Polish] Implement detection for "battery requires service" status and display a clear warning to the user, addressing the edge case from the spec.

## Dependencies

| Task | Depends On |
|------|------------|
| T009 | T008       |
| T011 | T010       |
| T013 | T012       |
| T014 | T009       |
| T015 | T011       |
| T019 | T011, T016, T017, T018 |
| T020 | T009, T011 |
| T021 | T013       |
| T028 | T009, T027 |

## Implementation Strategy

The implementation will follow the phases outlined above. The MVP is defined by completing all tasks in Phase 3 (User Story 1). This will deliver a functional application that meets the core user need. Subsequent phases will add more advanced features and prepare for future expansion.