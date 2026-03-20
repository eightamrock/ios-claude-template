# Swift Code Quality Reference

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Types (struct, class, enum, protocol) | UpperCamelCase | `UserProfile`, `NetworkError` |
| Functions, methods | lowerCamelCase, verb phrase | `fetchUser()`, `calculateTotal(for:)` |
| Properties, variables | lowerCamelCase, noun | `isLoading`, `userName`, `itemCount` |
| Enum cases | lowerCamelCase | `.loading`, `.networkError` |
| Protocols (capability) | -able/-ible suffix | `Loadable`, `Sendable` |
| Protocols (role) | Noun | `DataSource`, `Coordinator`, `Repository` |
| Boolean properties | is/has/should prefix | `isValid`, `hasContent`, `shouldRefresh` |
| Factory methods | make- prefix | `makeConfiguration()`, `makeRequest()` |
| Generic type params | Single uppercase or descriptive | `T`, `Element`, `Value` |

## Protocol-Oriented Design

```swift
// Define capability
protocol ItemRepository {
    func fetchAll() async throws -> [Item]
    func save(_ item: Item) async throws
    func delete(id: UUID) async throws
}

// Production implementation
struct RemoteItemRepository: ItemRepository {
    private let client: HTTPClient

    func fetchAll() async throws -> [Item] {
        try await client.get("/items")
    }

    func save(_ item: Item) async throws {
        try await client.post("/items", body: item)
    }

    func delete(id: UUID) async throws {
        try await client.delete("/items/\(id)")
    }
}

// Test mock
struct MockItemRepository: ItemRepository {
    var items: [Item] = []
    var error: Error?

    func fetchAll() async throws -> [Item] {
        if let error { throw error }
        return items
    }

    func save(_ item: Item) async throws {
        if let error { throw error }
    }

    func delete(id: UUID) async throws {
        if let error { throw error }
    }
}
```

## Error Handling

```swift
// Define domain-specific errors
enum NetworkError: LocalizedError {
    case noConnection
    case unauthorized
    case serverError(statusCode: Int)
    case decodingFailed(underlying: Error)

    var errorDescription: String? {
        switch self {
        case .noConnection: "No internet connection"
        case .unauthorized: "Session expired. Please sign in again."
        case .serverError(let code): "Server error (\(code))"
        case .decodingFailed: "Failed to process server response"
        }
    }
}

// Throwing and catching
func fetchUser(id: UUID) async throws -> User {
    guard let url = URL(string: "\(baseURL)/users/\(id)") else {
        throw NetworkError.serverError(statusCode: 400)
    }

    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse else {
        throw NetworkError.serverError(statusCode: 0)
    }

    switch httpResponse.statusCode {
    case 200:
        do {
            return try JSONDecoder().decode(User.self, from: data)
        } catch {
            throw NetworkError.decodingFailed(underlying: error)
        }
    case 401:
        throw NetworkError.unauthorized
    default:
        throw NetworkError.serverError(statusCode: httpResponse.statusCode)
    }
}
```

## Swift Concurrency

### async/await
```swift
// Sequential
let user = try await fetchUser(id: userID)
let posts = try await fetchPosts(for: user)

// Concurrent with async let
async let user = fetchUser(id: userID)
async let preferences = fetchPreferences(id: userID)
let (fetchedUser, fetchedPrefs) = try await (user, preferences)
```

### Actors
```swift
actor ImageCache {
    private var cache: [URL: Data] = [:]

    func image(for url: URL) -> Data? {
        cache[url]
    }

    func store(_ data: Data, for url: URL) {
        cache[url] = data
    }
}
```

### MainActor
```swift
// Apply to entire ViewModel (recommended)
@MainActor @Observable
class ProfileViewModel {
    var user: User?
    var isLoading = false

    func loadProfile() async {
        isLoading = true
        defer { isLoading = false }
        user = try? await service.fetchUser()
    }
}
```

### TaskGroup
```swift
func fetchAllImages(urls: [URL]) async -> [URL: Data] {
    await withTaskGroup(of: (URL, Data?).self) { group in
        for url in urls {
            group.addTask {
                let data = try? await URLSession.shared.data(from: url).0
                return (url, data)
            }
        }

        var results: [URL: Data] = [:]
        for await (url, data) in group {
            if let data { results[url] = data }
        }
        return results
    }
}
```

### Sendable
```swift
// Value types are implicitly Sendable
struct UserDTO: Sendable, Codable {
    let id: UUID
    let name: String
}

// Reference types need explicit conformance
final class AppConfiguration: Sendable {
    let apiBaseURL: URL  // All stored properties must be let + Sendable
    let appVersion: String

    init(apiBaseURL: URL, appVersion: String) {
        self.apiBaseURL = apiBaseURL
        self.appVersion = appVersion
    }
}
```

## Value Types vs Reference Types

| Use | When |
|-----|------|
| `struct` | Data models, DTOs, small state containers, most types |
| `enum` | Fixed set of variants, state machines, errors, namespace |
| `class` | Identity matters, inheritance needed, interop with Obj-C, @Observable ViewModels |
| `actor` | Shared mutable state accessed from multiple isolation domains |

## Access Control

| Level | Use When |
|-------|----------|
| `private` | Implementation detail of this type only |
| `fileprivate` | Shared between types in the same file (e.g., extension helpers) |
| `internal` (default) | Available within the module |
| `package` | Available within the package (SPM) |
| `public` | Exposed from a library/framework |

Default to `private`. Widen access only when needed.

## Optionals

```swift
// guard let -- early exit (preferred for preconditions)
guard let user = currentUser else { return }

// if let -- conditional binding
if let name = user.nickname {
    greet(name)
}

// Chaining
let city = user.address?.city?.uppercased()

// Nil coalescing
let displayName = user.nickname ?? user.fullName ?? "Anonymous"

// map/flatMap for transformations
let avatarURL = user.avatarPath.map { URL(string: $0) } ?? nil
```

## Memory Management

```swift
// Weak references for delegate patterns and closures
class NetworkManager {
    weak var delegate: NetworkDelegate?

    func fetch() {
        Task { [weak self] in
            guard let self else { return }
            let data = try await performRequest()
            delegate?.didReceive(data)
        }
    }
}
```

**Rules:**
- Use `[weak self]` in escaping closures that capture `self` when the closure may outlive `self`
- Use `unowned` only when you can guarantee the reference outlives the closure
- Prefer value types to avoid retain cycles entirely
- With `@Observable` and SwiftUI, retain cycles are rare -- use `[weak self]` only when storing closures long-term
