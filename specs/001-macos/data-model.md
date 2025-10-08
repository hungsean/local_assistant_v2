# Data Model: macOS Battery Monitor

**Feature**: macOS Battery Monitor
**Created**: 2025-10-08
**Storage**: SQLite via GRDB.swift

## Overview

This document defines the data model for storing battery readings, device information, and user preferences locally on macOS. All entities are designed to support future multi-device scenarios while maintaining a clean, normalized structure.

## Entities

### 1. Device

Represents a physical device (currently macOS, future: iOS, Android, Windows, Linux).

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL | Unique device identifier |
| `name` | String | NOT NULL | Device name (e.g., "MacBook Pro") |
| `platform` | String | NOT NULL | Platform type: "macos", "ios", "android", "windows", "linux" |
| `model` | String | NULLABLE | Hardware model (e.g., "MacBookPro18,1") |
| `firstSeen` | DateTime | NOT NULL, DEFAULT CURRENT_TIMESTAMP | First time device was recorded |
| `lastSeen` | DateTime | NOT NULL | Last time device reported battery status |

**Indexes**:
- `idx_device_platform` on `platform`
- `idx_device_last_seen` on `lastSeen`

**Validation Rules**:
- `platform` MUST be one of: "macos", "ios", "android", "windows", "linux"
- `name` MUST be non-empty (1-100 characters)
- `lastSeen` >= `firstSeen`

**State Transitions**:
- Device created when first battery reading is recorded
- `lastSeen` updated each time a new battery reading is stored

---

### 2. BatteryReading

Represents a single battery status snapshot collected every 1 minute.

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | Unique reading ID |
| `deviceId` | UUID | FOREIGN KEY(Device.id), NOT NULL | Reference to device |
| `timestamp` | DateTime | NOT NULL, INDEXED | When reading was taken |
| `percentage` | INTEGER | NOT NULL, CHECK(0-100) | Battery percentage (0-100%) |
| `isCharging` | BOOLEAN | NOT NULL | Whether device is charging |
| `timeRemaining` | INTEGER | NULLABLE | Estimated minutes remaining (NULL if charging or unknown) |
| `capacity` | INTEGER | NULLABLE | Current capacity in mAh (if available) |
| `cycleCount` | INTEGER | NULLABLE | Battery cycle count (if available) |
| `temperature` | DOUBLE | NULLABLE | Battery temperature in Celsius (if available) |
| `health` | STRING | NULLABLE | Health status: "Good", "Fair", "Poor", "Replace Soon", "Replace Now" |

**Indexes**:
- `idx_reading_device_time` on `(deviceId, timestamp DESC)` - for time-range queries
- `idx_reading_timestamp` on `timestamp DESC` - for cleanup operations

**Validation Rules**:
- `percentage` MUST be between 0 and 100 (inclusive)
- `timeRemaining` MUST be >= 0 if not NULL
- `capacity` MUST be > 0 if not NULL
- `cycleCount` MUST be >= 0 if not NULL
- `health` MUST be one of: "Good", "Fair", "Poor", "Replace Soon", "Replace Now", NULL

**Data Retention**:
- Readings older than 30 days are automatically deleted
- Cleanup performed daily at 3:00 AM local time

---

### 3. UserPreferences

Stores user-configurable settings for the application.

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | INTEGER | PRIMARY KEY (always 1) | Singleton record |
| `lowBatteryThresholds` | JSON | NOT NULL | Array of threshold percentages [20, 10] |
| `notificationsEnabled` | BOOLEAN | NOT NULL, DEFAULT TRUE | Whether low battery notifications are enabled |
| `launchAtLogin` | BOOLEAN | NOT NULL, DEFAULT TRUE | Whether app launches at system startup |
| `dataRetentionDays` | INTEGER | NOT NULL, DEFAULT 30 | Number of days to keep battery history |
| `chartDefaultRange` | STRING | NOT NULL, DEFAULT "24h" | Default chart time range: "1h", "6h", "24h", "7d" |
| `updatedAt` | DateTime | NOT NULL | Last time preferences were modified |

**Validation Rules**:
- `lowBatteryThresholds` MUST be valid JSON array of integers between 1-99
- `dataRetentionDays` MUST be between 1 and 365
- `chartDefaultRange` MUST be one of: "1h", "6h", "24h", "7d", "30d"

**Default Values**:
```json
{
  "id": 1,
  "lowBatteryThresholds": [20, 10],
  "notificationsEnabled": true,
  "launchAtLogin": true,
  "dataRetentionDays": 30,
  "chartDefaultRange": "24h",
  "updatedAt": "2025-10-08T00:00:00Z"
}
```

---

## Relationships

```
Device (1) ──< (N) BatteryReading
     │
     └─ deviceId (FK)
```

- One Device can have many BatteryReadings
- BatteryReadings are deleted when Device is deleted (CASCADE)
- UserPreferences is a singleton (no relationships)

---

## Swift Model Definitions

### Device.swift

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

    enum Platform: String, Codable {
        case macos, ios, android, windows, linux
    }

    // GRDB Table definition
    static let databaseTableName = "devices"

    // Define relationships
    static let batteryReadings = hasMany(BatteryReading.self)
}

extension Device {
    static func createTable(_ db: Database) throws {
        try db.create(table: "devices") { t in
            t.column("id", .text).primaryKey().notNull()
            t.column("name", .text).notNull()
            t.column("platform", .text).notNull()
            t.column("model", .text)
            t.column("firstSeen", .datetime).notNull().defaults(sql: "CURRENT_TIMESTAMP")
            t.column("lastSeen", .datetime).notNull()
        }

        try db.create(index: "idx_device_platform", on: "devices", columns: ["platform"])
        try db.create(index: "idx_device_last_seen", on: "devices", columns: ["lastSeen"])
    }
}
```

### BatteryReading.swift

```swift
import Foundation
import GRDB

struct BatteryReading: Codable, FetchableRecord, PersistableRecord {
    var id: Int64?
    var deviceId: UUID
    var timestamp: Date
    var percentage: Int
    var isCharging: Bool
    var timeRemaining: Int?
    var capacity: Int?
    var cycleCount: Int?
    var temperature: Double?
    var health: String?

    enum Health: String, Codable {
        case good = "Good"
        case fair = "Fair"
        case poor = "Poor"
        case replaceSoon = "Replace Soon"
        case replaceNow = "Replace Now"
    }

    static let databaseTableName = "battery_readings"

    // Define relationships
    static let device = belongsTo(Device.self)
}

extension BatteryReading {
    static func createTable(_ db: Database) throws {
        try db.create(table: "battery_readings") { t in
            t.autoIncrementedPrimaryKey("id")
            t.column("deviceId", .text).notNull().references("devices", onDelete: .cascade)
            t.column("timestamp", .datetime).notNull()
            t.column("percentage", .integer).notNull().check { ($0 >= 0) && ($0 <= 100) }
            t.column("isCharging", .boolean).notNull()
            t.column("timeRemaining", .integer)
            t.column("capacity", .integer)
            t.column("cycleCount", .integer)
            t.column("temperature", .double)
            t.column("health", .text)
        }

        try db.create(index: "idx_reading_device_time", on: "battery_readings", columns: ["deviceId", "timestamp"])
        try db.create(index: "idx_reading_timestamp", on: "battery_readings", columns: ["timestamp"])
    }
}
```

### UserPreferences.swift

```swift
import Foundation
import GRDB

struct UserPreferences: Codable, FetchableRecord, PersistableRecord {
    var id: Int
    var lowBatteryThresholds: [Int]
    var notificationsEnabled: Bool
    var launchAtLogin: Bool
    var dataRetentionDays: Int
    var chartDefaultRange: String
    var updatedAt: Date

    enum ChartRange: String, Codable {
        case oneHour = "1h"
        case sixHours = "6h"
        case twentyFourHours = "24h"
        case sevenDays = "7d"
        case thirtyDays = "30d"
    }

    static let databaseTableName = "user_preferences"

    static let defaultPreferences = UserPreferences(
        id: 1,
        lowBatteryThresholds: [20, 10],
        notificationsEnabled: true,
        launchAtLogin: true,
        dataRetentionDays: 30,
        chartDefaultRange: "24h",
        updatedAt: Date()
    )
}

extension UserPreferences {
    static func createTable(_ db: Database) throws {
        try db.create(table: "user_preferences") { t in
            t.column("id", .integer).primaryKey().notNull()
            t.column("lowBatteryThresholds", .text).notNull() // JSON array
            t.column("notificationsEnabled", .boolean).notNull().defaults(to: true)
            t.column("launchAtLogin", .boolean).notNull().defaults(to: true)
            t.column("dataRetentionDays", .integer).notNull().defaults(to: 30)
            t.column("chartDefaultRange", .text).notNull().defaults(to: "24h")
            t.column("updatedAt", .datetime).notNull()
        }

        // Insert default preferences
        try UserPreferences.defaultPreferences.insert(db)
    }
}
```

---

## Database Schema Migration

### Initial Schema (Version 1)

```swift
import GRDB

struct DatabaseMigrator {
    static func migrate(_ db: Database) throws {
        var migrator = DatabaseMigrator()

        migrator.registerMigration("v1") { db in
            try Device.createTable(db)
            try BatteryReading.createTable(db)
            try UserPreferences.createTable(db)
        }

        try migrator.migrate(db)
    }
}
```

---

## Query Patterns

### Common Queries

**1. Get latest battery reading for current device:**

```swift
let latestReading = try BatteryReading
    .filter(Column("deviceId") == currentDeviceId)
    .order(Column("timestamp").desc)
    .limit(1)
    .fetchOne(db)
```

**2. Get battery history for time range:**

```swift
let startTime = Date().addingTimeInterval(-24 * 3600) // Last 24 hours
let readings = try BatteryReading
    .filter(Column("deviceId") == currentDeviceId)
    .filter(Column("timestamp") >= startTime)
    .order(Column("timestamp").asc)
    .fetchAll(db)
```

**3. Delete old readings (cleanup):**

```swift
let cutoffDate = Date().addingTimeInterval(-30 * 24 * 3600) // 30 days ago
try BatteryReading
    .filter(Column("timestamp") < cutoffDate)
    .deleteAll(db)
```

**4. Update user preferences:**

```swift
var prefs = try UserPreferences.fetchOne(db, key: 1)!
prefs.lowBatteryThresholds = [15, 5]
prefs.updatedAt = Date()
try prefs.update(db)
```

---

## Storage Location

**Database File**: `~/Library/Application Support/com.yourcompany.BatteryMonitor/battery_monitor.db`

**File Permissions**: `0600` (owner read/write only)

**Backup**: User's Time Machine backups will include the database file

---

## Data Size Estimates

**Per Reading**: ~100 bytes
**30 days of data**: 43,200 readings × 100 bytes = **~4.2 MB**
**Indexes overhead**: ~20%
**Total estimated size**: **~5 MB**

---

## Performance Considerations

1. **Write Performance**: 1 insert per minute is negligible (~0.017 writes/sec)
2. **Read Performance**: Compound index on `(deviceId, timestamp)` ensures fast time-range queries (<1ms for 24h range)
3. **Cleanup Performance**: Daily cleanup of 1,440 readings (~144ms) scheduled during low-usage time (3:00 AM)
4. **Memory**: GRDB uses connection pooling and prepared statements for efficiency

---

## Future Extensions

When adding support for iOS/Android/Windows/Linux:

1. **No schema changes required** - `Device.platform` already supports all platforms
2. Add new device records with appropriate `platform` value
3. BatteryReading structure remains the same (all fields are optional except core ones)
4. Cross-device queries already supported via `deviceId` filtering
