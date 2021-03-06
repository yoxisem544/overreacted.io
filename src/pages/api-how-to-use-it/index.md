---
title: API - How to use it
date: '2019-08-29'
spoiler: User manual of how to wrtie RESTful API 🚕
---

# Basics
#### Dependencies
These third party dependencies were used to help us write structured API code in Swift
- [Moya & Moya/RxSwift](https://github.com/Moya/Moya): Help us to structure api code, mocking response data, stubbing network responses.
- [Alamofire](https://github.com/Alamofire/Alamofire): HTTP networking library written in Swift.
- [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON): Help us to tranform response data to handful Swift JSON object. Really useful when you just want to take a look at response json.
- [ObjectMapper](https://github.com/tristanhimmelman/ObjectMapper): Help us to transform json into Swift model. An alternative option of JSON decoding.
- [PromiseKit](https://github.com/mxcl/PromiseKit): Promise for Swift.
- [RxSwift](https://github.com/ReactiveX/RxSwift): Reactive Programming in Swift

### File Structure
At the very beginning, we have to define a "Request" for our Restful API. Currently, we're working on SCM project, so let's name our request `KKdaySCMRequest`. In this file, we will import `Moya` to help us setup some information that api needed.

We all know url, method, header, parameters are needed to make a api call. `TargetType` is here to help us out! Let's define a protocol called `KKdaySCMRequestType` that conforms to `TargetType`, and give it some default values like `baseURL`. Values like `headers`, `parameters`, `sampleData` will be returning nothing now, we will `overload` these properties later when we start writing our api codes.

Next, define a empty `struct` named "`KKdaySCMRequest`", all our api codes will be structured inside this `struct` later.

```swift
import Moya

public protocol KKdaySCMRequestType : TargetType {
    var parameters: [String : Any] { get }
}

extension KKdaySCMRequestType {
    public var baseURL: URL {
        return URL(string: "https://fake-api.kkday.com/api/v1/")!
    }
    public var headers: [String : String]? { return nil }
    public var sampleData: Data { return Data() }
    public var parameters: [String : Any] { return [:] }
}

public struct KKdaySCMRequest {}
```

### Create a new API request
#### About response
Assume that we are fetching a product list from our server. Server response will be like this:

```json
{
  "metadata": {
    "status": "0000",
    "desc": "some description from server..."
  },
  "data": {
    "products": [
      {
        "name": "Macbook Pro",
        "price": 2499
      },
      {
        "name": "Macbook Air",
        "price": 1799
      },
    ]
  }
}
```
- metadata: message from server, will tell you more about this response. (To see what exactly is going on on response.)
- data: actual json response.

#### How to extend namespace
We hope in `KKdaySCMRequest` namespace, there is another namespace called `Products` for all our product apis structured inside it. What we will do here is to extend `KKdaySCMRequest`, create a new `struct` called `Products`.

Let's try to write api code for request above:

```swift
extension KKdaySCMRequest {
  struct Products {
    static let subpath = "products/"

    struct GetProductList : KKdaySCMRequestType {
      var path: String { return subpath }
      var method: Method { return .get }
      var task: Task { return .requestPlain }
    }
  }
}
```
By conform to `KKdaySCMRequestType` protocol, we will have a default baseURL. We will have to do several things:
1. **path**: endpoint of this api.
2. **method**: HTTP methods like GET/POST/PATCH/PUT/DELETE.
3. **task**: Moya defined some handy methods for us to make a api call, a plain request, request with parameters, or even multipart request can be indicated right here. see more about [Moya/Task.swift](https://github.com/Moya/Moya/blob/master/Sources/Moya/Task.swift).

#### Making an api call
We have defined our new `"Get product list"` request. So, what's next?

`MoyaProvider` is here to do our `"network job"`. When using Moya, we make all API requests through a MoyaProvider instance, passing in a value of our predefined `KKdaySCMRequest` that specifies which endpoint we want to call.

Here, we would like to have another namespace `API` to expose a wrapper and an entry for our API requests.

```swift
final public class API {

  public enum NetworkClientError: Error {
    case clientSideError(statusCode: Int, errorMessage: ServerErrorMessage?)
    case serverSideError(statusCode: Int, errorMessage: ServerErrorMessage?)
    case undefinedError
  }

  /// Wrapper to help us interact with moya provider.
  /// All networking jobs happened here.
  public struct NetworkClient {

    // MARK: - Property
    /// All API request will be executed on this background thread
    internal let requestQueue = DispatchQueue(label: "io.api.network_client.request_queue")

    // MARK: Initialization
    init(provider: MoyaProvider<MultiTarget>) {
      self.provider = provider
    }
    let provider: MoyaProvider<MultiTarget>

    func handleErrorResponse(_ r: Response) -> API.NetworkClientError {
      let message = try? r.map(ServerErrorMessage.self)
      switch r.statusCode {
      case 400...499:
        return API.NetworkClientError.clientSideError(statusCode: r.statusCode, errorMessage: message)
      case 500...599:
        return API.NetworkClientError.serverSideError(statusCode: r.statusCode, errorMessage: message)
      default:
        return API.NetworkClientError.undefinedError
      }
    }

    public func blockRequestQueue() {
      requestQueue.suspend() // you're able to suspend all api calls by suspending request thread
    }

    public func releaseRequestQueue() {
      requestQueue.resume()
    }
  }

  /// Default api client
  public static let shared: NetworkClient = {
    let provider = MoyaProvider<MultiTarget>()
    let client = NetworkClient(provider: provider)
    return client
  }()

  // API singleton
  private init() {}
}
```

We will go deeper to `API.NetworkClientError` and `requestQueue` later, let's first finish our `NetworkClient` now.
`NetworkClient` will be able to handle all requests conformed to `TargetType`. We will encounter a problem here, we do not know how to deal with response data returned from an API call. What we will do here is to use `SwiftyJSON` to just transform response data into SwiftyJSON `JSON object`.

```swift
// MARK: - General Decoding with SwiftyJSON
extension API.NetworkClient {
  func request<Request: TargetType>(_ request: Request) -> Promise<JSON?> {
    return perform(request, on: requestQueue)
  }
}

extension API.NetworkClient {
  internal func perform<Request: TargetType>(_ request: Request, on callbackQueue: DispatchQueue) -> Promise<JSON?> {
    let target = MultiTarget(request)
    return Promise { seal in
      provider.request(target, callbackQueue: callbackQueue, completion: { response in
        switch response {
        case .success(let r):
          do {
            switch r.statusCode {
            case 200...399:
              seal.fulfill(try JSON(data: r.data))
            default:
              seal.reject(self.handleErrorResponse(r))
            }
          } catch let e {
            seal.reject(e)
          }
        case .failure(let e):
          seal.reject(e)
        }
      })
    }
  }
}
```

>>> Note:
Should not use `perform(_:, on:)` method directly, use `request(_:)` instead.

Now, we have `NetworkClient` to do networking job for us, we are able to make our api call now.

```swift
API.shared.request(KKdaySCMRequest.Products.GetProductList())
  .done({ response in
    // success!
  })
  .catch({ e in
    // api call failed, handle error here.
  })
```

---
### Creating Models
We should not use `SwiftyJSON` for every api in our project, cause it not meaningful enough for us human to read whether its a product list response or other response. `ObjectMapper` will be a good option for json decoding here. (You can also use `Decodable` if you wish to.)

Let's recap server response above, we will have metadata and data from server response. We'll define `Metadata` first:

```swift
import ObjectMapper

struct Metadata {
  let status: String
  let desc: String
}

extension Metadata : ImmutableMappable {
  public init(map: Map) throws {
    status = try map.value("status")
    desc = try map.value("desc")
  }
}
```

With `ObjectMapper`, we got a handy mapping method to create a swift object.

Next, server returns a list of products inside `data` field, let's define a `Product` object:

```swift
struct Product {
  let name: String
  let price: Double
}

extension Product : ImmutableMappable {
  public init(map: Map) throws {
    name = try map.value("name")
    price = try map.value("price")
  }
}
```

Then structured these two properties inside a response object:

```swift
struct GetProductListResponse {
  let metadata: Metadata
  let products: [Product]
}

extension GetProductListResponse : ImmutableMappable {
  public init(map: Map) throws {
    metadata = try map.value("metadata")
    products = try map.value("products")
  }
}
```

### Indicates Decoding Response Type

#### Decode using ObjectMapper
In order to let `NetworkClient` know if we've defined a decoding method in our `KKdaySCMRequest`, we will have to define a protocol with an `associatedtype`:

```swift
protocol MappableResponse {
  associatedtype ResponseType: ImmutableMappable
}
```

By pluging `MappableResponse` to any `KKdaySCMRequestType`, tell that api what `ResponseType` is. After Indicating `ResponseType`, we will get object we defined above instead of SwiftyJSON type response.

```swift
extension KKdaySCMRequest {
  struct Products {
    struct GetProductList : KKdaySCMRequestType & MappableResponse {
      typealias ResponseType = GetProductListResponse

      var path: String { return subpath }
      var method: Method { return .get }
      var task: Task { return .requestPlain }
    }
  }
}
```

#### Decode using Decodable
If you prefer to use `Decodable` to decode json, you will need to conform your model to `Decodable` and implement what's required by `Decodable` protocol. Then plug `DecodableResponse` to your API request:

```swift
protocol DecodableResponse {
  associatedtype ResponseType: Decodable
}

extension GetProductListResponse : Decodable {
  // ...
}

extension KKdaySCMRequest {
  struct Products {
    struct GetProductList : KKdaySCMRequestType & DecodableResponse {
      typealias ResponseType = GetProductListResponse
      // ...
    }
  }
}
```

---

(**You can skip this part**)

How does `NetworkClient` know if `KKdaySCMRequestType` contains decoding information indicates in `MappableResponse`? We need to give `NetworkClient` a hand.

>> extend Moya's Response, make it able to map directly to `ImmutableMappable` or `BaseMappable`.

```swift
extension Response {
    func map<T: ImmutableMappable>(_ type: T.Type, context: MapContext? = nil) throws -> T {
        let mapper = Mapper<T>(context: context)
        return try mapper.map(JSONObject: try mapJSON())
    }

    func map<S: Sequence>(_ type: S.Type, context: MapContext? = nil) throws -> [S.Element] where S.Element: ImmutableMappable {
        let mapper = Mapper<S.Element>(context: context)
        return try mapper.mapArray(JSONObject: try mapJSON())
    }
}

extension Response {
    func map<T: BaseMappable>(_ type: T.Type, context: MapContext? = nil) throws -> T {
        let mapper = Mapper<T>(context: context)
        guard let result = mapper.map(JSONObject: try? mapJSON()) else { throw MoyaError.jsonMapping(self) }
        return result
    }

    func map<S: Sequence>(_ type: S.Type, context: MapContext? = nil) throws -> [S.Element] where S.Element: BaseMappable {
        let mapper = Mapper<S.Element>(context: context)
        guard let result = mapper.mapArray(JSONObject: try? mapJSON()) else { throw MoyaError.jsonMapping(self) }
        return result
    }
}
```

>>We will have to tell `NetworkClient` to use another `perform(_:, on:)` if generic `Request` matches `TargetType & MappableResponse` protocols.

```swift
extension API.NetworkClient {
  internal func perform<Request: TargetType & MappableResponse>(_ request: Request, on callbackQueue: DispatchQueue) -> Promise<Request.ResponseType> {
    let target = MultiTarget(request)
    return Promise { seal in
      provider.request(target, callbackQueue: callbackQueue, completion: { response in
        switch response {
        case .success(let r):
          do {
              // check status code if 200~399, 200~299 is success status, 300~399 is for redirect
            switch r.statusCode {
            case 200...399:
              let result = try r.map(Request.ResponseType.self)
              seal.fulfill(result)
            default:
              seal.reject(self.handleErrorResponse(r))
            }
          } catch let e {
            seal.reject(e)
          }
        case .failure(let e):
          seal.reject(e)
        }
      })
    }
  }

  func request<R: TargetType & MappableResponse>(_ request: R) -> Promise<R.ResponseType> {
    return perform(request, on: requestQueue)
  }
}
```

(**You can skip this part**)

---

### Multipart Upload
see: [Moya: Multipart upload](https://www.google.com/search?q=moya+multipart&oq=moya+multipart+&aqs=chrome..69i57.3420j0j4&sourceid=chrome&ie=UTF-8)

---

### Error Handling
We've defined `ServerErrorMessage` above. Here, we will dive into how to handle errors from api. These are errors we might get while api fails:
1. Timeout or no network
2. Decoding error
3. Server error like 400 or 500

```swift
public struct ServerErrorMessage {
    let metadata: Metadata
}

extension ServerErrorMessage : ImmutableMappable {
    public init(map: Map) throws {
        metadata = try map.value("metadata")
    }
}

API.shared.request(KKdaySCMRequest.Products.GetProductList())
  .done({ response in
    // success!
  })
  .catch({ e in
    switch e {
    case let API.clientSideError(statusCode: statusCode, errorMessage: message):
      // status code emit from this error will be bound in 400~499
      // message will be a wrapper of Metadata, more information from server will be wrapped inside it
    case let API.serverSideError(statusCode: statusCode, errorMessage: message):
      // status code emit from this error will be bound in 500~599
    case is API.undefinedError:
      // unknown error
    case is DecodableError, ObjectMapperError....:
      // known decoding errors...
    default:
      // unknown error
    }
  })
```

---

## Advanced Usage

### Retry
Sometimes, user will be in a bad network environment with huge package lost rate or latency. API calls usually failed few times in such condition. In order to make error rate a bit lower, we wish api calls are able to retry few times before it error out. To make api requests have ability to retry, we need another protocol here:

We will need 2 properties here:
1. retryInterval: interval between each retry.
2. retryCount: how many time should we retry before error out.

```swift
public protocol RetryableRquest {
    var retryInterval: DispatchTimeInterval { get }
    var retryCount: Int { get }
}

extension RetryableRquest {
    var retryInterval: DispatchTimeInterval { return .seconds(2) }
    var retryCount: Int { return 3 }
}
```

With `RetryableRquest` protocol, just plug it to API request that is required for retrying:

```swift
extension KKdaySCMRequest {
  struct Products {
    struct GetProductList : KKdaySCMRequestType & DecodableResponse & RetryableRquest {
      typealias ResponseType = GetProductListResponse
      // ...
    }
  }
}
```
---

(**You can skip this part**)

In order to make api retryable, we need to tell `NetworkClient` how to do the job:

```swift

extension API.NetworkClient {
  func request<Request: TargetType & RetryableRquest>(_ retryingRequest: Request) -> Promise<JSON?> {
    return attempt(maximumRetryCount: retryingRequest.retryCount, delayBeforeRetry: retryingRequest.retryInterval, {
      return self.perform(retryingRequest, on: self.requestQueue)
    })
  }

  func request<Request: TargetType & MappableResponse & RetryableRquest>(_ retryingRequest: Request) -> Promise<Request.ResponseType> {
    return attempt(maximumRetryCount: retryingRequest.retryCount, delayBeforeRetry: retryingRequest.retryInterval, {
      return self.perform(retryingRequest, on: self.requestQueue)
    })
  }
}
```

>>>more about [PromiseKit/Attempt](https://github.com/mxcl/PromiseKit/blob/master/Documentation/CommonPatterns.md#retry--polling)

(**You can skip this part**)

---

### Rx Retry
Implemented...

### Plugins
Moya has a powerful tool called `"PluginType"`, allows us to do pre/post process to a request.
see more: [Moya/PluginType](https://github.com/Moya/Moya/blob/master/docs/Plugins.md)

#### Injecting x-auth-token to all KKdaySCMRequests

We will have to add a new protocol `XAuthHeaderInjecting`. By conforming to `XAuthHeaderInjecting` protocol, we can check if request needs to inject `x-auth-token` to its `header field`.

```swift
import Moya

public protocol XAuthHeaderInjecting {}

public class XAuthHeaderInjectingPlugin : PluginType {
  public func prepare(_ request: URLRequest, target: TargetType) -> URLRequest {
    if let multiTarget = target as? MultiTarget, case let MultiTarget.target(actualTarget) = multiTarget {
      if actualTarget is (KKdayRequestType & XAuthHeaderInjecting) {
        // inject x-auth header here
        var request = request // mutabable copy of request
        request.addValue("x-auth-token-will-be-here", forHTTPHeaderField: "x-auth-token")
        return request
      }
    }

    return request
  }
}
```

Then we have to make `KKdaySCMRequestType` conforms to `XAuthHeaderInjecting` protocol.

```swift
public protocol KKdaySCMRequestType : TargetType & XAuthHeaderInjecting {
  // ...
}
```

### OAuth & Refresh Token
Server does not have refresh token now, but refresh plugin is ready.

---

## Testing
Moya provide us some handy ways to do stubbing or injecting mock data for test perpose.
see more: [Moya/Testing](https://github.com/Moya/Moya/blob/master/docs/Testing.md)

Let's take a closer look at `MoyaProvider`:
```swift
public init(endpointClosure: @escaping EndpointClosure = MoyaProvider.defaultEndpointMapping,
            requestClosure: @escaping RequestClosure = MoyaProvider.defaultRequestMapping,
            stubClosure: @escaping StubClosure = MoyaProvider.neverStub,
            callbackQueue: DispatchQueue? = nil,
            manager: Manager = MoyaProvider<Target>.defaultAlamofireManager(),
            plugins: [PluginType] = [],
            trackInflights: Bool = false) { ... }
```

### Mock Data
We have see what `callbackQueue` and `plugins` can do previously. Do you see `endpointClosure` and `stubClosure`? We can inject `mock data` and `fake status code` to `endpointClosure` by providing `Endpoint`.

```swift
func makeMockDataClosure(_ statusCode: Int, _ mockData: Data) -> ((MultiTarget) -> Endpoint) {
  return { (target: MultiTarget) -> Endpoint in
    return Endpoint(
      url: URL(target: target).absoluteString,
      sampleResponseClosure: { .networkResponse(statusCode, mockData) },
      method: target.method,
      task: target.task,
      httpHeaderFields: target.headers
    )
  }
}
```

### Simulates Real World Networking
MoyaProvider can simulates a networking delay when we are testing our api codes. This enables us to test api call with a bit response latency or even timeout condition.

```swift
MoyaProvider.delayedStub(responseTime) // return after a delay
MoyaProvider.immediatelyStub // immediately return
```

### How Do We Write Test Code?
We all know that `NetworkClient` do networking job for us, so we will have provide a fake provider to `NetworkClient`.

```swift
extension API {
  public struct StubbingConstructor {
    private var statusCode: Int = 200
    private var mockData: Data = Data()
    private var responseTime: TimeInterval = 0.3

    public func setSuccess(mockData: Data, statusCode: Int = 200, responseTime: TimeInterval = 0.3) -> NetworkClient {
      return NetworkClient(provider: {
        let mockDataClosure = makeMockDataClosure(statusCode, mockData)
        let stubClosure = getStubClosure(from: responseTime)
        return MoyaProvider<MultiTarget>(endpointClosure: mockDataClosure, stubClosure: stubClosure)
      }())
    }

    public func setFailure(mockData: Data, statusCode: Int = 400, responseTime: TimeInterval = 0.3) -> NetworkClient {
      return NetworkClient(provider: {
        let mockDataClosure = makeMockDataClosure(statusCode, mockData)
        let stubClosure = getStubClosure(from: responseTime)
        return MoyaProvider<MultiTarget>(endpointClosure: mockDataClosure, stubClosure: stubClosure)
      }())
    }

    /// Determine if needs a delayed stubbing from given response time
    private func getStubClosure(from responseTime: TimeInterval) -> ((MultiTarget) -> StubBehavior) {
      return responseTime > 0 ? MoyaProvider.delayedStub(responseTime) : MoyaProvider.immediatelyStub
    }

    private func makeMockDataClosure(_ statusCode: Int, _ mockData: Data) -> ((MultiTarget) -> Endpoint) {
      return { (target: MultiTarget) -> Endpoint in
        return Endpoint(
          url: URL(target: target).absoluteString,
          sampleResponseClosure: { .networkResponse(statusCode, mockData) },
          method: target.method,
          task: target.task,
          httpHeaderFields: target.headers
        )
      }
    }
  }

  /// Starts a stubbing api call
  public class func stubbing() -> StubbingConstructor {
    return StubbingConstructor()
  }
}
```

With `StubbingConstructor`, we can provide mock data, status code and response time to make a fake provider for `NetworkClient`.

```swift
let request = KKdaySCMRequest.Products.GetProductList()
// with setSuccess default value, we will have a 0.3 second delay for api to return, status code set to 200.
API.stubbing().setSuccess(mockData: someData).request(request)
  .done({ response in
    // do some assert...
  })
  .catch({ e in
    // do some assert...
  })
```
