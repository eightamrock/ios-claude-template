# Networking & API Patterns

## URLSession async/await

```swift
// Simple GET
let (data, response) = try await URLSession.shared.data(from: url)

// Request with configuration
var request = URLRequest(url: url)
request.httpMethod = "POST"
request.setValue("application/json", forHTTPHeaderField: "Content-Type")
request.httpBody = try JSONEncoder().encode(body)

let (data, response) = try await URLSession.shared.data(for: request)
```

## API Client Pattern

```swift
protocol APIClientProtocol: Sendable {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T
}

struct Endpoint {
    let path: String
    let method: HTTPMethod
    var queryItems: [URLQueryItem]?
    var body: Encodable?
    var headers: [String: String]?

    enum HTTPMethod: String {
        case get = "GET"
        case post = "POST"
        case put = "PUT"
        case patch = "PATCH"
        case delete = "DELETE"
    }
}

// Convenience factories
extension Endpoint {
    static func getItems(page: Int = 1) -> Endpoint {
        Endpoint(
            path: "/items",
            method: .get,
            queryItems: [URLQueryItem(name: "page", value: "\(page)")]
        )
    }

    static func createItem(_ item: CreateItemRequest) -> Endpoint {
        Endpoint(path: "/items", method: .post, body: item)
    }

    static func deleteItem(id: UUID) -> Endpoint {
        Endpoint(path: "/items/\(id)", method: .delete)
    }
}
```

### API Client Implementation

```swift
final class APIClient: APIClientProtocol, Sendable {
    private let baseURL: URL
    private let session: URLSession
    private let decoder: JSONDecoder
    private let encoder: JSONEncoder
    private let tokenProvider: TokenProvider?

    init(
        baseURL: URL,
        session: URLSession = .shared,
        tokenProvider: TokenProvider? = nil
    ) {
        self.baseURL = baseURL
        self.session = session
        self.tokenProvider = tokenProvider

        self.decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase
        decoder.dateDecodingStrategy = .iso8601

        self.encoder = JSONEncoder()
        encoder.keyEncodingStrategy = .convertToSnakeCase
        encoder.dateEncodingStrategy = .iso8601
    }

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        var urlComponents = URLComponents(url: baseURL.appending(path: endpoint.path), resolvingAgainstBaseURL: true)!
        urlComponents.queryItems = endpoint.queryItems

        guard let url = urlComponents.url else {
            throw APIError.invalidURL
        }

        var request = URLRequest(url: url)
        request.httpMethod = endpoint.method.rawValue
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("application/json", forHTTPHeaderField: "Accept")

        // Auth token
        if let token = try await tokenProvider?.currentToken() {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        // Custom headers
        endpoint.headers?.forEach { request.setValue($1, forHTTPHeaderField: $0) }

        // Body
        if let body = endpoint.body {
            request.httpBody = try encoder.encode(AnyEncodable(body))
        }

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }

        switch httpResponse.statusCode {
        case 200...299:
            return try decoder.decode(T.self, from: data)
        case 401:
            throw APIError.unauthorized
        case 404:
            throw APIError.notFound
        case 429:
            throw APIError.rateLimited
        case 500...599:
            throw APIError.serverError(statusCode: httpResponse.statusCode)
        default:
            throw APIError.httpError(statusCode: httpResponse.statusCode, data: data)
        }
    }
}

// Type-erased Encodable wrapper
struct AnyEncodable: Encodable {
    private let encode: (Encoder) throws -> Void

    init(_ value: some Encodable) {
        encode = value.encode
    }

    func encode(to encoder: Encoder) throws {
        try encode(encoder)
    }
}
```

## Error Types

```swift
enum APIError: LocalizedError {
    case invalidURL
    case invalidResponse
    case unauthorized
    case notFound
    case rateLimited
    case serverError(statusCode: Int)
    case httpError(statusCode: Int, data: Data)
    case decodingFailed(underlying: Error)

    var errorDescription: String? {
        switch self {
        case .invalidURL: "Invalid request URL"
        case .invalidResponse: "Invalid server response"
        case .unauthorized: "Session expired. Please sign in again."
        case .notFound: "Resource not found"
        case .rateLimited: "Too many requests. Please try again later."
        case .serverError(let code): "Server error (\(code))"
        case .httpError(let code, _): "Request failed (\(code))"
        case .decodingFailed: "Failed to process server response"
        }
    }
}
```

## Token Refresh Pattern

```swift
protocol TokenProvider: Sendable {
    func currentToken() async throws -> String
}

actor AuthTokenProvider: TokenProvider {
    private var accessToken: String?
    private var refreshToken: String?
    private var refreshTask: Task<String, Error>?

    func currentToken() async throws -> String {
        // If already refreshing, await that task
        if let refreshTask {
            return try await refreshTask.value
        }

        if let token = accessToken, !isExpired(token) {
            return token
        }

        return try await refresh()
    }

    private func refresh() async throws -> String {
        let task = Task<String, Error> {
            guard let refreshToken else { throw APIError.unauthorized }

            // Call token refresh endpoint
            let response = try await performTokenRefresh(refreshToken: refreshToken)
            self.accessToken = response.accessToken
            self.refreshToken = response.refreshToken
            return response.accessToken
        }

        refreshTask = task
        defer { refreshTask = nil }

        return try await task.value
    }

    private func isExpired(_ token: String) -> Bool {
        // Decode JWT and check exp claim
        // ...
        false
    }

    private func performTokenRefresh(refreshToken: String) async throws -> TokenResponse {
        // Actual network call to refresh endpoint
        fatalError("Implement token refresh")
    }
}
```

## Multipart Upload

```swift
extension APIClient {
    func upload(
        path: String,
        file: Data,
        filename: String,
        mimeType: String,
        fields: [String: String] = [:]
    ) async throws -> UploadResponse {
        let boundary = UUID().uuidString
        var body = Data()

        // Add form fields
        for (key, value) in fields {
            body.append("--\(boundary)\r\n")
            body.append("Content-Disposition: form-data; name=\"\(key)\"\r\n\r\n")
            body.append("\(value)\r\n")
        }

        // Add file data
        body.append("--\(boundary)\r\n")
        body.append("Content-Disposition: form-data; name=\"file\"; filename=\"\(filename)\"\r\n")
        body.append("Content-Type: \(mimeType)\r\n\r\n")
        body.append(file)
        body.append("\r\n--\(boundary)--\r\n")

        var request = URLRequest(url: baseURL.appending(path: path))
        request.httpMethod = "POST"
        request.setValue("multipart/form-data; boundary=\(boundary)", forHTTPHeaderField: "Content-Type")
        request.httpBody = body

        let (data, _) = try await session.data(for: request)
        return try decoder.decode(UploadResponse.self, from: data)
    }
}

private extension Data {
    mutating func append(_ string: String) {
        append(Data(string.utf8))
    }
}
```

## Pagination

### Cursor-Based

```swift
struct PaginatedResponse<T: Decodable>: Decodable {
    let items: [T]
    let nextCursor: String?
    let hasMore: Bool
}

@MainActor @Observable
class PaginatedListViewModel<T: Decodable & Identifiable> {
    var items: [T] = []
    var isLoading = false
    var hasMore = true
    private var nextCursor: String?
    private let fetchPage: (String?) async throws -> PaginatedResponse<T>

    init(fetchPage: @escaping (String?) async throws -> PaginatedResponse<T>) {
        self.fetchPage = fetchPage
    }

    func loadMore() async {
        guard !isLoading, hasMore else { return }
        isLoading = true
        defer { isLoading = false }

        do {
            let response = try await fetchPage(nextCursor)
            items.append(contentsOf: response.items)
            nextCursor = response.nextCursor
            hasMore = response.hasMore
        } catch {
            // Handle error
        }
    }

    func refresh() async {
        items = []
        nextCursor = nil
        hasMore = true
        await loadMore()
    }
}
```

### Pagination in Views

```swift
struct PaginatedListView: View {
    @Bindable var viewModel: PaginatedListViewModel<Item>

    var body: some View {
        List {
            ForEach(viewModel.items) { item in
                ItemRow(item: item)
                    .onAppear {
                        if item.id == viewModel.items.last?.id {
                            Task { await viewModel.loadMore() }
                        }
                    }
            }

            if viewModel.isLoading {
                ProgressView()
                    .frame(maxWidth: .infinity)
            }
        }
        .refreshable { await viewModel.refresh() }
        .task { await viewModel.loadMore() }
    }
}
```

## Testing with Protocol Mocks

```swift
// Mock API client
struct MockAPIClient: APIClientProtocol {
    var result: Any?
    var error: Error?

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        if let error { throw error }
        guard let result = result as? T else {
            throw APIError.decodingFailed(underlying: NSError(domain: "", code: 0))
        }
        return result
    }
}

// Usage in tests
@Test("fetches items successfully")
func fetchItems() async throws {
    let expected = [Item(id: UUID(), name: "Test")]
    let client = MockAPIClient(result: expected)
    let service = ItemService(client: client)

    let items = try await service.fetchAll()

    #expect(items.count == 1)
    #expect(items.first?.name == "Test")
}
```

### URLProtocol Mock (for integration-level tests)

```swift
class MockURLProtocol: URLProtocol {
    static var requestHandler: ((URLRequest) throws -> (HTTPURLResponse, Data))?

    override class func canInit(with request: URLRequest) -> Bool { true }
    override class func canonicalRequest(for request: URLRequest) -> URLRequest { request }

    override func startLoading() {
        guard let handler = Self.requestHandler else {
            client?.urlProtocolDidFinishLoading(self)
            return
        }

        do {
            let (response, data) = try handler(request)
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } catch {
            client?.urlProtocol(self, didFailWithError: error)
        }
    }

    override func stopLoading() { }
}

// Create test session
func makeTestSession() -> URLSession {
    let config = URLSessionConfiguration.ephemeral
    config.protocolClasses = [MockURLProtocol.self]
    return URLSession(configuration: config)
}
```
