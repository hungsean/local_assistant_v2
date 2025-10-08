# Service Contract: StorageService

**Purpose**: Manage local storage of battery readings, device information, and user preferences using SQLite/GRDB

**Conformance**: Internal service contract defining Swift protocol and method signatures

---

## Protocol Definition

```swift
protocol StorageServiceProtocol {
    // MARK: - Device Management

    /// Get or create current device record
    /// - Returns: Device record for the current macOS machine
    /// - Throws: StorageServiceError if unable to access database
    func getCurrentDevice() throws -> Device

    // MARK: - Battery Readings

    /// Store a new battery reading
    /// - Parameter reading: Battery reading to store
    /// - Throws: StorageServiceError if unable to save
    func saveBatteryReading(_ reading: BatteryReading) throws

    /// Get battery readings for a time range
    /// - Parameters:
    ///   - deviceId: Device ID (defaults to current device)
    ///   - startDate: Start of time range (inclusive)
    ///   - endDate: End of time range (inclusive)
    /// - Returns: Array of battery readings sorted by timestamp ascending
    /// - Throws: StorageServiceError if query fails
    func getBatteryReadings(
        deviceId: UUID?,
        from startDate: Date,
        to endDate: Date
    ) throws -> [BatteryReading]

    /// Get latest battery reading for device
    /// - Parameter deviceId: Device ID (defaults to current device)
    /// - Returns: Most recent battery reading or nil if none exists
    /// - Throws: StorageServiceError if query fails
    func getLatestReading(deviceId: UUID?) throws -> BatteryReading?

    /// Delete old battery readings based on retention policy
    /// - Parameter olderThan: Delete readings older than this date
    /// - Returns: Number of readings deleted
    /// - Throws: StorageServiceError if deletion fails
    func cleanupOldReadings(olderThan: Date) throws -> Int

    // MARK: - User Preferences

    /// Get user preferences
    /// - Returns: Current user preferences
    /// - Throws: StorageServiceError if unable to load preferences
    func getPreferences() throws -> UserPreferences

    /// Update user preferences
    /// - Parameter preferences: Updated preferences
    /// - Throws: StorageServiceError if unable to save
    func savePreferences(_ preferences: UserPreferences) throws
}
```

---

## Error Handling

### StorageServiceError

```swift
enum StorageServiceError: LocalizedError {
    case databaseNotInitialized
    case migrationFailed(Error)
    case readFailed(query: String, underlyingError: Error)
    case writeFailed(operation: String, underlyingError: Error)
    case deviceNotFound
    case invalidData(reason: String)
    case filePermissionError(path: String)

    var errorDescription: String? {
        switch self {
        case .databaseNotInitialized:
            return "Database is not initialized. Please restart the application."
        case .migrationFailed(let error):
            return "Database migration failed: \(error.localizedDescription)"
        case .readFailed(let query, let error):
            return "Failed to read from database (query: \(query)): \(error.localizedDescription)"
        case .writeFailed(let operation, let error):
            return "Failed to write to database (operation: \(operation)): \(error.localizedDescription)"
        case .deviceNotFound:
            return "Current device record not found in database."
        case .invalidData(let reason):
            return "Invalid data: \(reason)"
        case .filePermissionError(let path):
            return "File permission error at path: \(path)"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .databaseNotInitialized, .migrationFailed:
            return "Try restarting the application. If the problem persists, the database may be corrupted."
        case .readFailed, .writeFailed:
            return "Check that you have sufficient disk space and permissions. Try restarting the app."
        case .deviceNotFound:
            return "The app will automatically create a new device record."
        case .invalidData:
            return "Please file a bug report with the data that caused this error."
        case .filePermissionError:
            return "Ensure the application has read/write permissions in ~/Library/Application Support."
        }
    }
}
```

---

## Behavioral Contracts

### getCurrentDevice()

**Preconditions**:
- Database is initialized

**Postconditions**:
- Returns existing Device record if found
- Creates and returns new Device record if not found
- Updates `lastSeen` timestamp to current time

**Performance**:
- MUST complete within 50ms
- Uses cached device ID after first call

**Example**:

```swift
let device = try storageService.getCurrentDevice()
print("Device: \(device.name), Platform: \(device.platform)")
```

---

### saveBatteryReading(_ reading:)

**Preconditions**:
- `reading.deviceId` exists in database
- `reading.percentage` is 0-100
- `reading.timestamp` is valid

**Postconditions**:
- Reading saved to database
- `reading.id` populated with auto-generated ID
- Device's `lastSeen` updated

**Performance**:
- MUST complete within 100ms
- Uses write-ahead logging (WAL) for concurrency

**Example**:

```swift
var reading = BatteryReading(
    deviceId: device.id,
    timestamp: Date(),
    percentage: 75,
    isCharging: true,
    timeRemaining: nil
)
try storageService.saveBatteryReading(&reading)
print("Saved reading with ID: \(reading.id!)")
```

---

### getBatteryReadings(deviceId:from:to:)

**Preconditions**:
- `startDate` <= `endDate`
- `deviceId` is nil (current device) or valid UUID

**Postconditions**:
- Returns readings within time range, sorted ascending by timestamp
- Returns empty array if no readings found (not an error)

**Performance**:
- MUST complete within 1 second for 24-hour range
- Uses compound index on `(deviceId, timestamp)` for efficiency

**Example**:

```swift
let now = Date()
let yesterday = now.addingTimeInterval(-24 * 3600)
let readings = try storageService.getBatteryReadings(
    deviceId: nil,  // Current device
    from: yesterday,
    to: now
)
print("Found \(readings.count) readings in last 24h")
```

---

### getLatestReading(deviceId:)

**Preconditions**:
- `deviceId` is nil (current device) or valid UUID

**Postconditions**:
- Returns most recent reading for device
- Returns `nil` if no readings exist (not an error)

**Performance**:
- MUST complete within 50ms
- Uses indexed query with LIMIT 1

**Example**:

```swift
if let latest = try storageService.getLatestReading(deviceId: nil) {
    print("Latest: \(latest.percentage)% at \(latest.timestamp)")
} else {
    print("No readings yet")
}
```

---

### cleanupOldReadings(olderThan:)

**Preconditions**:
- `olderThan` is a valid date

**Postconditions**:
- All readings older than `olderThan` are deleted
- Returns count of deleted records

**Performance**:
- SHOULD complete within 500ms for 30 days of data (~43,000 records)
- Uses batched deletion to avoid blocking

**Example**:

```swift
let retentionDays = 30
let cutoffDate = Date().addingTimeInterval(-Double(retentionDays) * 24 * 3600)
let deletedCount = try storageService.cleanupOldReadings(olderThan: cutoffDate)
print("Deleted \(deletedCount) old readings")
```

---

### getPreferences()

**Preconditions**:
- Database initialized

**Postconditions**:
- Returns UserPreferences record (singleton, always exists)

**Performance**:
- MUST complete within 20ms
- SHOULD cache in memory after first read

**Example**:

```swift
let prefs = try storageService.getPreferences()
print("Notifications enabled: \(prefs.notificationsEnabled)")
print("Thresholds: \(prefs.lowBatteryThresholds)")
```

---

### savePreferences(_ preferences:)

**Preconditions**:
- `preferences.id` == 1 (singleton)
- `preferences.lowBatteryThresholds` is valid JSON
- `preferences.dataRetentionDays` is 1-365

**Postconditions**:
- Preferences saved to database
- `preferences.updatedAt` set to current time

**Performance**:
- MUST complete within 50ms

**Example**:

```swift
var prefs = try storageService.getPreferences()
prefs.lowBatteryThresholds = [15, 5]
prefs.notificationsEnabled = false
try storageService.savePreferences(prefs)
```

---

## Database Lifecycle

### Initialization

```swift
class StorageService: StorageServiceProtocol {
    private let dbQueue: DatabaseQueue

    init(databaseURL: URL) throws {
        // Create database file with owner-only permissions (0600)
        let fileManager = FileManager.default
        if !fileManager.fileExists(atPath: databaseURL.path) {
            fileManager.createFile(atPath: databaseURL.path, contents: nil)
            try fileManager.setAttributes(
                [.posixPermissions: 0o600],
                ofItemAtPath: databaseURL.path
            )
        }

        // Open database with WAL mode
        dbQueue = try DatabaseQueue(path: databaseURL.path)
        try dbQueue.write { db in
            db.execute(sql: "PRAGMA journal_mode = WAL")
            db.execute(sql: "PRAGMA foreign_keys = ON")
        }

        // Run migrations
        try migrate()
    }

    private func migrate() throws {
        try dbQueue.write { db in
            try DatabaseMigrator.migrate(db)
        }
    }
}
```

### Database Location

**Path**: `~/Library/Application Support/com.yourcompany.BatteryMonitor/battery_monitor.db`

**Permissions**: `0600` (owner read/write only)

**WAL Files**: `battery_monitor.db-wal`, `battery_monitor.db-shm` (temporary)

---

## Thread Safety

- **All methods are thread-safe**
- GRDB's `DatabaseQueue` serializes all write operations
- Read operations can run concurrently with WAL mode
- No locks required by callers

---

## Testing Contracts

### Unit Tests

```swift
class StorageServiceTests: XCTestCase {
    var storageService: StorageService!
    var tempDatabaseURL: URL!

    override func setUp() {
        super.setUp()
        tempDatabaseURL = FileManager.default.temporaryDirectory
            .appendingPathComponent(UUID().uuidString)
            .appendingPathExtension("db")
        storageService = try! StorageService(databaseURL: tempDatabaseURL)
    }

    override func tearDown() {
        try? FileManager.default.removeItem(at: tempDatabaseURL)
        super.tearDown()
    }

    func testSaveAndRetrieveBatteryReading() throws {
        let device = try storageService.getCurrentDevice()
        var reading = BatteryReading(
            deviceId: device.id,
            timestamp: Date(),
            percentage: 80,
            isCharging: false,
            timeRemaining: 120
        )

        try storageService.saveBatteryReading(&reading)
        XCTAssertNotNil(reading.id)

        let retrieved = try storageService.getLatestReading(deviceId: nil)
        XCTAssertEqual(retrieved?.percentage, 80)
        XCTAssertEqual(retrieved?.isCharging, false)
    }

    func testCleanupOldReadings() throws {
        let device = try storageService.getCurrentDevice()

        // Insert old and new readings
        let oldDate = Date().addingTimeInterval(-40 * 24 * 3600)  // 40 days ago
        let newDate = Date().addingTimeInterval(-10 * 24 * 3600)  // 10 days ago

        try storageService.saveBatteryReading(BatteryReading(
            deviceId: device.id, timestamp: oldDate, percentage: 50, isCharging: false
        ))
        try storageService.saveBatteryReading(BatteryReading(
            deviceId: device.id, timestamp: newDate, percentage: 60, isCharging: false
        ))

        // Cleanup readings older than 30 days
        let cutoff = Date().addingTimeInterval(-30 * 24 * 3600)
        let deletedCount = try storageService.cleanupOldReadings(olderThan: cutoff)

        XCTAssertEqual(deletedCount, 1)  // Only old reading deleted
    }
}
```

---

## Dependencies

- **GRDB.swift**: SQLite wrapper
- **Foundation**: FileManager, Date, UUID

---

## Migration Strategy

### Adding New Fields (Future)

```swift
migrator.registerMigration("v2_add_voltage") { db in
    try db.alter(table: "battery_readings") { t in
        t.add(column: "voltage", .double)
    }
}
```

### Adding New Tables (Future Multi-Platform)

```swift
migrator.registerMigration("v3_add_sync_metadata") { db in
    try db.create(table: "sync_metadata") { t in
        t.autoIncrementedPrimaryKey("id")
        t.column("lastSyncTime", .datetime)
        t.column("syncEnabled", .boolean).defaults(to: false)
    }
}
```
