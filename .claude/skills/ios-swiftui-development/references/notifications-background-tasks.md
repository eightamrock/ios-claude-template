# Push Notifications & Background Tasks

## Push Notifications

### Permission Request

```swift
import UserNotifications

func requestNotificationPermission() async -> Bool {
    do {
        let granted = try await UNUserNotificationCenter.current()
            .requestAuthorization(options: [.alert, .badge, .sound])
        if granted {
            await MainActor.run {
                UIApplication.shared.registerForRemoteNotifications()
            }
        }
        return granted
    } catch {
        return false
    }
}
```

### Device Token Handling

```swift
// In App Delegate or @UIApplicationDelegateAdaptor
class AppDelegate: NSObject, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
    ) {
        let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
        // Send token to your server
        Task { try? await PushService.shared.registerToken(token) }
    }

    func application(
        _ application: UIApplication,
        didFailToRegisterForRemoteNotificationsWithError error: Error
    ) {
        print("Push registration failed: \(error.localizedDescription)")
    }
}

// Wire up in App
@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var body: some Scene {
        WindowGroup { ContentView() }
    }
}
```

### Local Notifications

```swift
func scheduleLocalNotification(
    title: String,
    body: String,
    triggerIn seconds: TimeInterval
) async throws {
    let content = UNMutableNotificationContent()
    content.title = title
    content.body = body
    content.sound = .default
    content.badge = 1

    let trigger = UNTimeIntervalNotificationTrigger(
        timeInterval: seconds,
        repeats: false
    )

    let request = UNNotificationRequest(
        identifier: UUID().uuidString,
        content: content,
        trigger: trigger
    )

    try await UNUserNotificationCenter.current().add(request)
}

// Calendar-based trigger
let dateComponents = DateComponents(hour: 9, minute: 0)  // Daily at 9 AM
let trigger = UNCalendarNotificationTrigger(dateMatching: dateComponents, repeats: true)

// Location-based trigger
let region = CLCircularRegion(center: coordinate, radius: 100, identifier: "office")
region.notifyOnEntry = true
let trigger = UNLocationNotificationTrigger(region: region, repeats: false)
```

### Notification Delegate (Foreground & Tap Handling)

```swift
class NotificationDelegate: NSObject, UNUserNotificationCenterDelegate {
    // Show notification while app is in foreground
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification
    ) async -> UNNotificationPresentationOptions {
        [.banner, .badge, .sound]
    }

    // Handle notification tap
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse
    ) async {
        let userInfo = response.notification.request.content.userInfo

        switch response.actionIdentifier {
        case UNNotificationDefaultActionIdentifier:
            // User tapped the notification
            if let itemId = userInfo["itemId"] as? String {
                await MainActor.run {
                    NavigationManager.shared.navigate(to: .item(id: itemId))
                }
            }
        case "MARK_DONE":
            // Custom action
            if let itemId = userInfo["itemId"] as? String {
                try? await ItemService.shared.markDone(id: itemId)
            }
        default:
            break
        }
    }
}

// Register delegate early (in AppDelegate or App init)
UNUserNotificationCenter.current().delegate = notificationDelegate
```

### Rich Notifications (Actions & Categories)

```swift
func registerNotificationCategories() {
    let markDoneAction = UNNotificationAction(
        identifier: "MARK_DONE",
        title: "Mark Done",
        options: []
    )

    let deleteAction = UNNotificationAction(
        identifier: "DELETE",
        title: "Delete",
        options: [.destructive, .authenticationRequired]
    )

    let category = UNNotificationCategory(
        identifier: "ITEM_REMINDER",
        actions: [markDoneAction, deleteAction],
        intentIdentifiers: [],
        options: [.customDismissAction]
    )

    UNUserNotificationCenter.current().setNotificationCategories([category])
}

// Set category on notification content
content.categoryIdentifier = "ITEM_REMINDER"
content.userInfo = ["itemId": item.id.uuidString]
```

### Notification Service Extension

For modifying push content before display (e.g., downloading images):

```swift
// In NotificationServiceExtension target
class NotificationService: UNNotificationServiceExtension {
    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(
        _ request: UNNotificationRequest,
        withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void
    ) {
        self.contentHandler = contentHandler
        bestAttemptContent = request.content.mutableCopy() as? UNMutableNotificationContent

        guard let content = bestAttemptContent,
              let imageURLString = content.userInfo["imageURL"] as? String,
              let imageURL = URL(string: imageURLString) else {
            contentHandler(request.content)
            return
        }

        // Download and attach image
        Task {
            if let (data, _) = try? await URLSession.shared.data(from: imageURL),
               let attachment = try? saveAttachment(data: data, filename: "image.jpg") {
                content.attachments = [attachment]
            }
            contentHandler(content)
        }
    }

    override func serviceExtensionTimeWillExpire() {
        if let content = bestAttemptContent {
            contentHandler?(content)
        }
    }

    private func saveAttachment(data: Data, filename: String) throws -> UNNotificationAttachment {
        let url = FileManager.default.temporaryDirectory.appending(path: filename)
        try data.write(to: url)
        return try UNNotificationAttachment(identifier: filename, url: url)
    }
}
```

### Provisional & Critical Alerts

```swift
// Provisional -- delivers silently to Notification Center (no permission prompt)
try await UNUserNotificationCenter.current()
    .requestAuthorization(options: [.alert, .provisional])

// Critical alerts -- bypass Do Not Disturb (requires Apple entitlement)
try await UNUserNotificationCenter.current()
    .requestAuthorization(options: [.alert, .criticalAlert, .sound])

// Critical alert content
content.sound = UNNotificationSound.defaultCritical
// or custom: UNNotificationSound.criticalSoundNamed("alarm.wav", withAudioVolume: 1.0)
```

## Background Tasks

### BGTaskScheduler Setup

**Info.plist** -- add permitted identifiers:
```xml
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.example.app.refresh</string>
    <string>com.example.app.processing</string>
</array>
```

### Registration

```swift
class AppDelegate: NSObject, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.example.app.refresh",
            using: nil
        ) { task in
            self.handleAppRefresh(task as! BGAppRefreshTask)
        }

        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.example.app.processing",
            using: nil
        ) { task in
            self.handleProcessing(task as! BGProcessingTask)
        }

        return true
    }

    private func handleAppRefresh(_ task: BGAppRefreshTask) {
        scheduleAppRefresh()  // Schedule next refresh

        let refreshTask = Task {
            do {
                try await DataSyncService.shared.syncLatest()
                task.setTaskCompleted(success: true)
            } catch {
                task.setTaskCompleted(success: false)
            }
        }

        task.expirationHandler = {
            refreshTask.cancel()
        }
    }

    private func handleProcessing(_ task: BGProcessingTask) {
        let processingTask = Task {
            do {
                try await DataSyncService.shared.fullSync()
                task.setTaskCompleted(success: true)
            } catch {
                task.setTaskCompleted(success: false)
            }
        }

        task.expirationHandler = {
            processingTask.cancel()
        }
    }
}
```

### Scheduling

```swift
func scheduleAppRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: "com.example.app.refresh")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)  // 15 min minimum
    try? BGTaskScheduler.shared.submit(request)
}

func scheduleProcessing() {
    let request = BGProcessingTaskRequest(identifier: "com.example.app.processing")
    request.requiresNetworkConnectivity = true
    request.requiresExternalPower = false  // true for heavy work
    request.earliestBeginDate = Date(timeIntervalSinceNow: 60 * 60)  // 1 hour
    try? BGTaskScheduler.shared.submit(request)
}

// Schedule when app goes to background
struct MyApp: App {
    @Environment(\.scenePhase) var scenePhase

    var body: some Scene {
        WindowGroup { ContentView() }
            .onChange(of: scenePhase) { _, newPhase in
                if newPhase == .background {
                    scheduleAppRefresh()
                }
            }
    }
}
```

### Background URLSession Downloads/Uploads

```swift
actor BackgroundDownloadManager: NSObject, URLSessionDownloadDelegate {
    static let shared = BackgroundDownloadManager()

    private lazy var session: URLSession = {
        let config = URLSessionConfiguration.background(
            withIdentifier: "com.example.app.background-download"
        )
        config.isDiscretionary = false
        config.sessionSendsLaunchEvents = true
        return URLSession(configuration: config, delegate: self, delegateQueue: nil)
    }()

    func downloadFile(from url: URL) {
        let task = session.downloadTask(with: url)
        task.resume()
    }

    // URLSessionDownloadDelegate
    nonisolated func urlSession(
        _ session: URLSession,
        downloadTask: URLSessionDownloadTask,
        didFinishDownloadingTo location: URL
    ) {
        // Move file from temp location to permanent storage
        let destination = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
            .appending(path: downloadTask.originalRequest?.url?.lastPathComponent ?? "download")
        try? FileManager.default.moveItem(at: location, to: destination)
    }

    nonisolated func urlSession(
        _ session: URLSession,
        task: URLSessionTask,
        didCompleteWithError error: Error?
    ) {
        if let error {
            print("Background download failed: \(error.localizedDescription)")
        }
    }
}
```

### Silent Push (Background Notifications)

Remote notification payload with `content-available`:
```json
{
    "aps": {
        "content-available": 1
    },
    "custom-data": "sync-needed"
}
```

```swift
// In AppDelegate
func application(
    _ application: UIApplication,
    didReceiveRemoteNotification userInfo: [AnyHashable: Any]
) async -> UIBackgroundFetchResult {
    do {
        try await DataSyncService.shared.syncLatest()
        return .newData
    } catch {
        return .failed
    }
}
```

**Required:** Enable "Background Modes > Remote notifications" capability.

### Testing Background Tasks

Via Xcode debugger (pause, then run in LLDB):
```
e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.example.app.refresh"]
```

Or use Xcode > Debug > Simulate Background Fetch.

## Info.plist Keys

```xml
<!-- Push Notifications -->
<key>UIBackgroundModes</key>
<array>
    <string>remote-notification</string>
    <string>fetch</string>
    <string>processing</string>
</array>

<!-- Background Task Identifiers -->
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.example.app.refresh</string>
    <string>com.example.app.processing</string>
</array>
```

## Folder Structure Note

If using a Notification Service Extension:
```
{{PROJECT_NAME}}/
  App/
  Models/
  ...
NotificationServiceExtension/
  NotificationService.swift
  Info.plist
```

Add as a separate target in Xcode (File > New > Target > Notification Service Extension).
