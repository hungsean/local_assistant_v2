# Service Contract: NotificationService

**Purpose**: Manage low battery notifications using macOS UserNotifications framework

**Conformance**: Internal service contract defining Swift protocol and method signatures

---

## Protocol Definition

```swift
protocol NotificationServiceProtocol {
    /// Request permission to send notifications
    /// - Returns: true if permission granted, false otherwise
    func requestPermission() async -> Bool

    /// Send low battery notification
    /// - Parameters:
    ///   - percentage: Current battery percentage
    ///   - threshold: Threshold that triggered the notification
    /// - Throws: NotificationServiceError if unable to send
    func sendLowBatteryNotification(percentage: Int, threshold: Int) throws

    /// Clear all delivered notifications
    func clearNotifications()

    /// Check if notifications are enabled in system preferences
    /// - Returns: true if user has granted notification permission
    func areNotificationsEnabled() async -> Bool
}
```

---

## Data Structures

### Notification Content

Low battery notifications include:
- **Title**: "Low Battery"
- **Body**: "Battery is at {percentage}% (threshold: {threshold}%)"
- **Sound**: System default sound
- **Category**: "LOW_BATTERY"
- **User Info**: `["percentage": Int, "threshold": Int]`

---

## Error Handling

### NotificationServiceError

```swift
enum NotificationServiceError: LocalizedError {
    case permissionDenied
    case notificationFailed(Error)
    case invalidParameters(reason: String)

    var errorDescription: String? {
        switch self {
        case .permissionDenied:
            return "Notification permission denied by user."
        case .notificationFailed(let error):
            return "Failed to send notification: \(error.localizedDescription)"
        case .invalidParameters(let reason):
            return "Invalid notification parameters: \(reason)"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .permissionDenied:
            return "Go to System Preferences > Notifications > Battery Monitor to enable notifications."
        case .notificationFailed:
            return "Try restarting the application."
        case .invalidParameters:
            return "Please report this bug with the parameters that caused the error."
        }
    }
}
```

---

## Behavioral Contracts

### requestPermission()

**Preconditions**:
- None

**Postconditions**:
- Returns `true` if user grants permission
- Returns `false` if user denies permission or permission already denied
- System shows permission dialog if not previously answered

**Performance**:
- Completes immediately if permission already granted/denied
- May take several seconds if waiting for user response

**Example**:

```swift
let service = NotificationService()
let granted = await service.requestPermission()
if granted {
    print("Notifications enabled")
} else {
    print("Notifications denied")
}
```

---

### sendLowBatteryNotification(percentage:threshold:)

**Preconditions**:
- User preferences have notifications enabled
- `percentage` is 0-100
- `threshold` is 1-99
- `percentage` <= `threshold`

**Postconditions**:
- Notification delivered to system notification center
- Duplicate notifications suppressed (max 1 per threshold per session)
- Throws if permission denied or delivery fails

**Performance**:
- MUST complete within 200ms
- Non-blocking (notification delivery is asynchronous)

**Deduplication Logic**:
- Only send ONE notification per threshold per app session
- Reset on app restart
- Example: If threshold is 20%, only notify once when crossing 20%, not on every reading at 19%, 18%, etc.

**Example**:

```swift
let prefs = try storageService.getPreferences()
if prefs.notificationsEnabled {
    let currentPercentage = 18
    let threshold = 20

    if currentPercentage <= threshold {
        try notificationService.sendLowBatteryNotification(
            percentage: currentPercentage,
            threshold: threshold
        )
    }
}
```

---

### clearNotifications()

**Preconditions**:
- None (safe to call anytime)

**Postconditions**:
- All delivered Battery Monitor notifications removed from notification center

**Performance**:
- MUST complete within 100ms

**Example**:

```swift
// Clear notifications when user acknowledges or battery starts charging
if batteryStatus.isCharging {
    notificationService.clearNotifications()
}
```

---

### areNotificationsEnabled()

**Preconditions**:
- None

**Postconditions**:
- Returns current notification permission status
- Never throws

**Performance**:
- MUST complete within 50ms

**Example**:

```swift
let enabled = await notificationService.areNotificationsEnabled()
if !enabled {
    showAlert("Enable notifications in System Preferences to receive low battery alerts")
}
```

---

## Implementation Details

### Notification Deduplication

To avoid spamming users, the service maintains in-memory state:

```swift
class NotificationService: NotificationServiceProtocol {
    private var notifiedThresholds: Set<Int> = []

    func sendLowBatteryNotification(percentage: Int, threshold: Int) throws {
        // Skip if already notified for this threshold
        guard !notifiedThresholds.contains(threshold) else {
            return
        }

        // Send notification...
        // ...

        // Mark threshold as notified
        notifiedThresholds.insert(threshold)
    }

    func resetNotificationState() {
        // Call when battery starts charging or user dismisses
        notifiedThresholds.removeAll()
    }
}
```

---

## Notification Categories

### LOW_BATTERY Category

Defines actions users can take from notification:

```swift
let category = UNNotificationCategory(
    identifier: "LOW_BATTERY",
    actions: [
        UNNotificationAction(
            identifier: "SHOW_DETAILS",
            title: "Show Details",
            options: .foreground
        ),
        UNNotificationAction(
            identifier: "DISMISS",
            title: "Dismiss",
            options: .destructive
        )
    ],
    intentIdentifiers: [],
    options: []
)
```

**Action Handling**:
- **SHOW_DETAILS**: Opens main app window to show battery history
- **DISMISS**: Clears notification and resets threshold state

---

## User Preferences Integration

The service respects user preferences:

```swift
func shouldSendNotification(
    currentPercentage: Int,
    preferences: UserPreferences
) -> (send: Bool, threshold: Int?) {
    // Check if notifications globally enabled
    guard preferences.notificationsEnabled else {
        return (false, nil)
    }

    // Find applicable threshold
    for threshold in preferences.lowBatteryThresholds.sorted(by: >) {
        if currentPercentage <= threshold && !notifiedThresholds.contains(threshold) {
            return (true, threshold)
        }
    }

    return (false, nil)
}
```

---

## Thread Safety

- All methods are thread-safe
- Notification delivery happens on background queue
- Completion handlers invoked on main thread
- `notifiedThresholds` access is synchronized

---

## Testing Contracts

### Unit Tests

```swift
class NotificationServiceTests: XCTestCase {
    var service: NotificationService!

    override func setUp() {
        super.setUp()
        service = NotificationService()
    }

    func testSendLowBatteryNotification_ValidParameters_Succeeds() throws {
        // Assume permission granted
        try service.sendLowBatteryNotification(percentage: 15, threshold: 20)
        // Notification should be sent (verify via notification center mock)
    }

    func testSendLowBatteryNotification_DuplicateThreshold_IgnoresSecond() throws {
        try service.sendLowBatteryNotification(percentage: 15, threshold: 20)
        try service.sendLowBatteryNotification(percentage: 14, threshold: 20)
        // Second call should be no-op
    }

    func testSendLowBatteryNotification_InvalidPercentage_Throws() {
        XCTAssertThrowsError(
            try service.sendLowBatteryNotification(percentage: 101, threshold: 20)
        )
    }
}
```

### Integration Tests

```swift
class NotificationServiceIntegrationTests: XCTestCase {
    func testRequestPermission_ShowsSystemDialog() async {
        let service = NotificationService()
        let granted = await service.requestPermission()
        // Manual verification: Check that system dialog appeared
        XCTAssertNotNil(granted)
    }

    func testNotificationDelivery_AppearsInNotificationCenter() async throws {
        let service = NotificationService()
        _ = await service.requestPermission()

        try service.sendLowBatteryNotification(percentage: 18, threshold: 20)

        // Wait for notification delivery
        try await Task.sleep(nanoseconds: 1_000_000_000)

        // Manual verification: Check notification center
    }
}
```

---

## Dependencies

- **UserNotifications framework**: `UNUserNotificationCenter`, `UNNotificationRequest`
- **Foundation**: `Set`, async/await

---

## Privacy Considerations

- Notifications contain only battery percentage (no sensitive data)
- User can disable notifications globally in System Preferences
- App respects user's notification preferences

---

## Accessibility

- Notification text is clear and concise
- Supports VoiceOver (macOS reads notification aloud)
- Action buttons have descriptive labels
