# Service Contract: BatteryService

**Purpose**: Wrapper around IOKit APIs to read battery status and monitor changes

**Conformance**: This is an internal service contract (not REST/GraphQL), defining Swift protocol and method signatures.

---

## Protocol Definition

```swift
protocol BatteryServiceProtocol {
    /// Get current battery status
    /// - Returns: Current battery status or nil if device has no battery
    /// - Throws: BatteryServiceError if unable to read battery information
    func getCurrentStatus() throws -> BatteryStatus?

    /// Start monitoring battery changes
    /// - Parameter interval: Polling interval in seconds (default: 30)
    /// - Parameter onChange: Callback invoked when battery status changes
    /// - Throws: BatteryServiceError if monitoring cannot be started
    func startMonitoring(interval: TimeInterval, onChange: @escaping (BatteryStatus) -> Void) throws

    /// Stop monitoring battery changes
    func stopMonitoring()

    /// Check if device has a battery
    /// - Returns: true if device supports battery, false otherwise
    func hasBattery() -> Bool
}
```

---

## Data Structures

### BatteryStatus

```swift
struct BatteryStatus {
    /// Battery percentage (0-100)
    let percentage: Int

    /// Whether device is currently charging
    let isCharging: Bool

    /// Estimated time remaining in minutes (nil if charging or unknown)
    let timeRemaining: Int?

    /// Current battery capacity in mAh (nil if unavailable)
    let capacity: Int?

    /// Battery cycle count (nil if unavailable)
    let cycleCount: Int?

    /// Battery temperature in Celsius (nil if unavailable)
    let temperature: Double?

    /// Battery health status
    let health: BatteryHealth

    /// Timestamp when this status was captured
    let timestamp: Date
}

enum BatteryHealth {
    case good
    case fair
    case poor
    case replaceSoon
    case replaceNow
    case unknown
}
```

---

## Error Handling

### BatteryServiceError

```swift
enum BatteryServiceError: LocalizedError {
    case noBatteryDetected
    case permissionDenied
    case ioKitError(code: Int32, message: String)
    case monitoringAlreadyActive
    case unknown(Error)

    var errorDescription: String? {
        switch self {
        case .noBatteryDetected:
            return "No battery detected. This device may not have a battery."
        case .permissionDenied:
            return "Permission denied to access battery information. Please grant necessary permissions."
        case .ioKitError(let code, let message):
            return "IOKit error (\(code)): \(message)"
        case .monitoringAlreadyActive:
            return "Battery monitoring is already active. Stop current monitoring before starting a new session."
        case .unknown(let error):
            return "Unknown error: \(error.localizedDescription)"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .noBatteryDetected:
            return "This application requires a device with a battery. Desktop Macs (Mac mini, Mac Studio, iMac, Mac Pro) are not supported."
        case .permissionDenied:
            return "Check System Preferences > Security & Privacy to ensure this app has necessary permissions."
        case .ioKitError:
            return "Try restarting the application. If the problem persists, please file a bug report."
        case .monitoringAlreadyActive:
            return "Call stopMonitoring() before starting a new monitoring session."
        case .unknown:
            return "Try restarting the application."
        }
    }
}
```

---

## Behavioral Contracts

### getCurrentStatus()

**Preconditions**:
- None

**Postconditions**:
- Returns `nil` if device has no battery (e.g., Mac mini)
- Returns `BatteryStatus` with all available fields populated
- Throws `BatteryServiceError` if unable to read battery info

**Performance**:
- MUST complete within 100ms
- SHOULD cache result for up to 5 seconds to avoid excessive IOKit calls

**Example Usage**:

```swift
let service = BatteryService()

do {
    if let status = try service.getCurrentStatus() {
        print("Battery: \(status.percentage)%, Charging: \(status.isCharging)")
    } else {
        print("No battery detected")
    }
} catch let error as BatteryServiceError {
    print("Error: \(error.errorDescription ?? "")")
    print("Suggestion: \(error.recoverySuggestion ?? "")")
}
```

---

### startMonitoring(interval:onChange:)

**Preconditions**:
- Monitoring is not already active
- `interval` > 0

**Postconditions**:
- Monitoring starts successfully, `onChange` callback invoked on battery changes
- Throws `BatteryServiceError.monitoringAlreadyActive` if already monitoring
- Monitoring continues until `stopMonitoring()` is called

**Performance**:
- Callback invoked on main thread
- Callback execution time SHOULD be <50ms to avoid blocking

**Behavior**:
- Polls battery status at specified interval
- Invokes `onChange` only when status actually changes (percentage, charging state, etc.)
- Automatically handles device sleep/wake cycles

**Example Usage**:

```swift
try service.startMonitoring(interval: 30.0) { status in
    print("Battery changed: \(status.percentage)%")
    // Update UI, store to database, etc.
}
```

---

### stopMonitoring()

**Preconditions**:
- None (safe to call even if monitoring is not active)

**Postconditions**:
- Monitoring stops, no further `onChange` callbacks invoked
- Resources (timers, IOKit connections) released

**Performance**:
- MUST complete within 50ms

---

### hasBattery()

**Preconditions**:
- None

**Postconditions**:
- Returns `true` if device has a battery, `false` otherwise
- Never throws

**Performance**:
- MUST complete within 50ms
- SHOULD cache result (battery presence doesn't change at runtime)

**Example Usage**:

```swift
if !service.hasBattery() {
    showAlert("This app requires a device with a battery")
}
```

---

## Thread Safety

- All methods are thread-safe
- `onChange` callback is always invoked on the main thread
- Internal IOKit operations may use background queue but are synchronized

---

## Dependencies

- **IOKit framework**: `IOPowerSources.h`, `IOPSKeys.h`
- **Foundation framework**: `Date`, `TimeInterval`

---

## Testing Contracts

### Unit Tests

```swift
class BatteryServiceTests: XCTestCase {
    func testGetCurrentStatus_WhenBatteryPresent_ReturnsValidStatus() {
        let service = BatteryService()
        let status = try? service.getCurrentStatus()
        XCTAssertNotNil(status)
        XCTAssertGreaterThanOrEqual(status!.percentage, 0)
        XCTAssertLessThanOrEqual(status!.percentage, 100)
    }

    func testHasBattery_AlwaysReturnsConsistentValue() {
        let service = BatteryService()
        let result1 = service.hasBattery()
        let result2 = service.hasBattery()
        XCTAssertEqual(result1, result2)
    }

    func testStartMonitoring_WhenAlreadyActive_ThrowsError() {
        let service = BatteryService()
        try? service.startMonitoring(interval: 1.0) { _ in }
        XCTAssertThrowsError(try service.startMonitoring(interval: 1.0) { _ in })
    }
}
```

### Integration Tests

```swift
class BatteryServiceIntegrationTests: XCTestCase {
    func testMonitoring_InvokesCallbackOnBatteryChange() {
        let service = BatteryService()
        let expectation = self.expectation(description: "Callback invoked")

        try? service.startMonitoring(interval: 5.0) { status in
            XCTAssertNotNil(status)
            expectation.fulfill()
        }

        wait(for: [expectation], timeout: 10.0)
        service.stopMonitoring()
    }
}
```

---

## Implementation Notes

1. Use `IOPSCopyPowerSourcesInfo()` and `IOPSCopyPowerSourcesList()` for reading battery info
2. Monitor `kIOPSNotificationTypeKey` for power source change notifications
3. Gracefully handle macOS version differences (some fields may not be available on older versions)
4. Log all IOKit errors for debugging but don't crash the app
