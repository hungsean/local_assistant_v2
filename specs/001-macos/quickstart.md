# Quick Start Guide: macOS Battery Monitor

**Target Audience**: Developers implementing this feature
**Prerequisites**: macOS 10.15+, Xcode 14+, Swift 5.7+
**Estimated Setup Time**: 30 minutes

---

## Overview

This guide walks you through setting up and implementing the macOS Battery Monitor application. By the end, you'll have a working menu bar app that displays battery status and stores historical data locally.

---

## Step 1: Project Setup (5 min)

### 1.1 Create Xcode Project

```bash
# Create project directory
cd /path/to/local_assistant_v2
mkdir BatteryMonitor
cd BatteryMonitor

# Open Xcode and create new macOS App project:
# - Product Name: BatteryMonitor
# - Interface: SwiftUI
# - Language: Swift
# - Organization Identifier: com.yourcompany
```

### 1.2 Add Dependencies

Add GRDB.swift using Swift Package Manager:

1. **File** > **Add Packages...**
2. Enter URL: `https://github.com/groue/GRDB.swift`
3. Select version: **6.0.0** or later
4. Add to **BatteryMonitor** target

### 1.3 Project Structure

Create the following folder structure in Xcode:

```
BatteryMonitor/
├── App/
│   └── AppDelegate.swift
├── Models/
│   ├── Device.swift
│   ├── BatteryReading.swift
│   ├── BatteryStatus.swift
│   └── UserPreferences.swift
├── Services/
│   ├── BatteryService.swift
│   ├── StorageService.swift
│   └── NotificationService.swift
├── UI/
│   ├── MenuBar/
│   │   ├── StatusBarController.swift
│   │   └── MenuBarView.swift
│   └── Windows/
│       ├── DetailWindowController.swift
│       ├── HistoryChartView.swift
│       └── PreferencesView.swift
└── Resources/
    └── Assets.xcassets
```

---

## Step 2: Implement Data Models (10 min)

### 2.1 Device.swift

Copy the Swift model definition from `data-model.md`:

```swift
import Foundation
import GRDB

struct Device: Codable, FetchableRecord, PersistableRecord {
    var id: UUID
    var name: String
    var platform: String
    var model: String?
    var firstSeen: Date
    var lastSeen: Date

    static let databaseTableName = "devices"
    static let batteryReadings = hasMany(BatteryReading.self)
}

extension Device {
    static func createTable(_ db: Database) throws {
        // See data-model.md for full implementation
    }
}
```

### 2.2 BatteryReading.swift, BatteryStatus.swift, UserPreferences.swift

Follow the same pattern from `data-model.md` for remaining models.

---

## Step 3: Implement Core Services (15 min)

### 3.1 BatteryService.swift

Refer to `contracts/BatteryService.md` for full implementation contract.

**Key Implementation**:

```swift
import Foundation
import IOKit.ps

class BatteryService: BatteryServiceProtocol {
    func getCurrentStatus() throws -> BatteryStatus? {
        // 1. Get power source info
        guard let info = IOPSCopyPowerSourcesInfo()?.takeRetainedValue(),
              let sources = IOPSCopyPowerSourcesList(info)?.takeRetainedValue() as? [CFTypeRef] else {
            throw BatteryServiceError.ioKitError(code: -1, message: "Cannot access power sources")
        }

        // 2. Find battery
        for source in sources {
            guard let description = IOPSGetPowerSourceDescription(info, source)?.takeUnretainedValue() as? [String: Any] else {
                continue
            }

            // 3. Extract battery data
            let percentage = description[kIOPSCurrentCapacityKey] as? Int ?? 0
            let isCharging = (description[kIOPSIsChargingKey] as? Bool) ?? false
            let timeRemaining = description[kIOPSTimeToEmptyKey] as? Int

            return BatteryStatus(
                percentage: percentage,
                isCharging: isCharging,
                timeRemaining: isCharging ? nil : timeRemaining,
                timestamp: Date()
            )
        }

        return nil // No battery found
    }

    // Implement remaining methods from contract...
}
```

### 3.2 StorageService.swift

Refer to `contracts/StorageService.md` for full implementation.

**Key Implementation**:

```swift
import GRDB

class StorageService: StorageServiceProtocol {
    private let dbQueue: DatabaseQueue

    init(databaseURL: URL) throws {
        // Set up database file with 0600 permissions
        let fileManager = FileManager.default
        if !fileManager.fileExists(atPath: databaseURL.path) {
            fileManager.createFile(atPath: databaseURL.path, contents: nil)
            try fileManager.setAttributes(
                [.posixPermissions: 0o600],
                ofItemAtPath: databaseURL.path
            )
        }

        dbQueue = try DatabaseQueue(path: databaseURL.path)
        try migrate()
    }

    private func migrate() throws {
        try dbQueue.write { db in
            try Device.createTable(db)
            try BatteryReading.createTable(db)
            try UserPreferences.createTable(db)
        }
    }

    // Implement remaining methods from contract...
}
```

### 3.3 NotificationService.swift

Refer to `contracts/NotificationService.md` for full implementation.

---

## Step 4: Build Menu Bar UI (10 min)

### 4.1 StatusBarController.swift

```swift
import Cocoa

class StatusBarController {
    private var statusBar: NSStatusBar
    private var statusItem: NSStatusItem

    init() {
        statusBar = NSStatusBar.system
        statusItem = statusBar.statusItem(withLength: NSStatusItem.variableLength)

        if let button = statusItem.button {
            button.title = "🔋"
            button.action = #selector(showMenu)
            button.target = self
        }
    }

    func updateBatteryStatus(_ status: BatteryStatus) {
        guard let button = statusItem.button else { return }

        let icon = status.isCharging ? "⚡" : "🔋"
        button.title = "\(icon) \(status.percentage)%"
    }

    @objc func showMenu() {
        // Show popover or menu with battery details
    }
}
```

### 4.2 HistoryChartView.swift

```swift
import SwiftUI
import Charts

struct HistoryChartView: View {
    let readings: [BatteryReading]

    var body: some View {
        Chart(readings) { reading in
            LineMark(
                x: .value("Time", reading.timestamp),
                y: .value("Battery %", reading.percentage)
            )
            .foregroundStyle(reading.isCharging ? .green : .blue)
        }
        .chartYScale(domain: 0...100)
        .chartXAxis {
            AxisMarks(values: .stride(by: .hour, count: 6))
        }
        .frame(height: 300)
    }
}
```

---

## Step 5: Wire Everything Together (5 min)

### 5.1 AppDelegate.swift

```swift
import Cocoa

@main
class AppDelegate: NSObject, NSApplicationDelegate {
    var statusBarController: StatusBarController!
    var batteryService: BatteryService!
    var storageService: StorageService!

    func applicationDidFinishLaunching(_ notification: Notification) {
        // Initialize services
        let dbURL = FileManager.default
            .urls(for: .applicationSupportDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("com.yourcompany.BatteryMonitor")
            .appendingPathComponent("battery_monitor.db")

        batteryService = BatteryService()
        storageService = try! StorageService(databaseURL: dbURL)
        statusBarController = StatusBarController()

        // Start monitoring
        startBatteryMonitoring()
    }

    func startBatteryMonitoring() {
        try? batteryService.startMonitoring(interval: 30.0) { [weak self] status in
            guard let self = self else { return }

            // Update UI
            self.statusBarController.updateBatteryStatus(status)

            // Save to database (every minute, not every 30 seconds)
            if shouldSaveReading() {
                let device = try? self.storageService.getCurrentDevice()
                var reading = BatteryReading(
                    deviceId: device!.id,
                    timestamp: Date(),
                    percentage: status.percentage,
                    isCharging: status.isCharging,
                    timeRemaining: status.timeRemaining
                )
                try? self.storageService.saveBatteryReading(&reading)
            }
        }
    }

    private var lastSaveTime: Date?

    func shouldSaveReading() -> Bool {
        let now = Date()
        guard let last = lastSaveTime else {
            lastSaveTime = now
            return true
        }

        if now.timeIntervalSince(last) >= 60 {  // 1 minute
            lastSaveTime = now
            return true
        }
        return false
    }
}
```

---

## Step 6: Test the Application

### 6.1 Build and Run

1. Press **Cmd+R** in Xcode
2. Look for battery icon in menu bar
3. Click icon to see current battery status

### 6.2 Verify Data Storage

Open database file:

```bash
cd ~/Library/Application\ Support/com.yourcompany.BatteryMonitor
sqlite3 battery_monitor.db

# Check tables
.tables

# View readings
SELECT * FROM battery_readings ORDER BY timestamp DESC LIMIT 10;
```

### 6.3 Check File Permissions

```bash
ls -la ~/Library/Application\ Support/com.yourcompany.BatteryMonitor/
# Should show: -rw------- (0600)
```

---

## Troubleshooting

### Issue: "No battery detected"

**Solution**: This app requires a MacBook with a battery. Desktop Macs (Mac mini, iMac, etc.) are not supported.

### Issue: Database file not created

**Solution**:
```bash
# Ensure directory exists
mkdir -p ~/Library/Application\ Support/com.yourcompany.BatteryMonitor

# Check permissions
ls -la ~/Library/Application\ Support/
```

### Issue: Menu bar icon not showing

**Solution**:
- Check that `NSUIElement` is NOT set to `YES` in Info.plist
- Verify `LSBackgroundOnly` is set to `YES` for menu bar-only apps

---

## Next Steps

### P1 (MVP) Checklist

- [x] Read battery status via IOKit
- [x] Display in menu bar
- [x] Store readings locally (SQLite)
- [ ] Handle edge cases (no battery, permission errors)
- [ ] Add low battery notifications
- [ ] Add login at startup functionality

### P2 (History) Checklist

- [ ] Create detail window with history chart
- [ ] Implement time range selector (1h, 6h, 24h, 7d)
- [ ] Add charging period highlighting
- [ ] Test with 24 hours of real data

### P3 (Multi-Device Prep) Checklist

- [x] Data model supports deviceId
- [x] Platform field in Device table
- [ ] Verify no hardcoded assumptions about single device

---

## Development Tips

1. **Use Xcode Previews** for SwiftUI views:
```swift
struct HistoryChartView_Previews: PreviewProvider {
    static var previews: some View {
        HistoryChartView(readings: sampleReadings)
    }
}
```

2. **Test on real hardware**, not simulator (IOKit battery APIs don't work in simulator)

3. **Monitor memory usage** with Instruments (target: <50MB)

4. **Use `os_log`** for debugging:
```swift
import os.log

let logger = Logger(subsystem: "com.yourcompany.BatteryMonitor", category: "battery")
logger.info("Battery status updated: \(percentage)%")
```

5. **Add database logging** in debug builds:
```swift
#if DEBUG
dbQueue.trace { print($0) }
#endif
```

---

## Resources

- **IOKit Documentation**: [Apple Developer - IOKit](https://developer.apple.com/documentation/iokit)
- **GRDB.swift Guide**: [GRDB Documentation](https://github.com/groue/GRDB.swift)
- **Swift Charts**: [Apple Developer - Charts](https://developer.apple.com/documentation/charts)
- **UserNotifications**: [Apple Developer - UserNotifications](https://developer.apple.com/documentation/usernotifications)

---

## Getting Help

If you encounter issues:

1. Check `contracts/` directory for detailed service specifications
2. Review `data-model.md` for database schema
3. Consult `research.md` for technology decisions and alternatives
4. File issues in the project repository with:
   - macOS version
   - Xcode version
   - Steps to reproduce
   - Console logs

---

**Ready to start coding? Begin with Step 1!** 🚀
