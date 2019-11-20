---
title: RESTful API and OAuth 2.0
date: '2019-11-20'
spoiler: How we structure our network layer
---

### Table of Contents

- [Basics](./#basics)
- [NetworkClient](./#networkclient)
- [Config](./#config)
- [Structure a Network Request](./#structure-a-network-request)
  - [Define Request Type](./#define-request-type)
  - [Define Request](./#define-request)
  - [Decoding](./#decoding)
- [Retry Request](./#retry-request)
- [Plugins](./#plugins)
  - [Header injection](./#header-injection)
  - [Access Token Injection](./#access-token-injection)
  - [Refresh Token Plugin](./#refresh-token-plugin)
  - [如何使用這些 Plugins](./#如何使用這些-plugins)
- [OAuth and RESTful together](./#oauth-and-restful-together)
- [RxSwift Submodule](./#rxswift-submodule)

# Basics

## Dependencies

APIKit 使用了以下套件來輔助我們抽象化一些複雜的實作，以達到快速開發的目的：
- [Moya & Moya/RxSwift](https://github.com/Moya/Moya): Help us to structure api code, mocking response data, stubbing network responses.
- [Alamofire](https://github.com/Alamofire/Alamofire): HTTP networking library written in Swift.
- [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON): Help us to tranform response data to handful Swift JSON object. Really useful when you just want to take a look at response json.
- [ObjectMapper](https://github.com/tristanhimmelman/ObjectMapper): Help us to transform json into Swift model. An alternative option of JSON decoding.
- [PromiseKit](https://github.com/mxcl/PromiseKit): Promise for Swift.
- [RxSwift](https://github.com/ReactiveX/RxSwift): Reactive Programming in Swift

## Installation

### Cocoapods

```ruby
pod 'APIKit'
pod 'APIKit/RxSwift' # if you prefer to use RxSwift extensions
```

## Requirment

- Xcode 11.x
- Swift 5.x
- Cocoapods >= 1.4.0

# NetworkClient

`NetworkClient` 主要的功能是執行 network call，你可以傳入一個 Request，NetworkClient 會依據 Request 中的網址、參數等去打 api，且回傳 Request 中定義好的 Response Type 回來。

簡單來看一下 `NetworkClient` 內部有什麼東西：

```swift
final public class API {
  public struct `NetworkClient` {
    // MARK: - Property
    internal let requestQueue = DispatchQueue(label: "io.api.network_client.request_queue")
    // MARK: Initialization
    public init(provider: MoyaProvider<MultiTarget>) {
      self.provider = provider
    }
    let provider: MoyaProvider<MultiTarget>

    func handleErrorResponse(_ r: Response) -> API.NetworkClientError { ... }

    public func blockRequestQueue() {
      requestQueue.suspend()
    }

    public func releaseRequestQueue() {
      requestQueue.resume()
    }
  }
}
```

- 每一個 `NetworkClient` 都須傳入一個 `MoyaProvider<MultiTarget>`，這裡的目的是要透過 Moya 來操作網路連線。
- 且可以看到 `NetworkClient` 中有一個 `requestQueue`，目的是為了當需要暫停某個 `NetworkClient` 上所有的連線時可以使用的 thread。

在 APIKit 中，我們提供了一個最基礎的 `NetworkClient`：

```swift
final public class API {
  public static let shared: NetworkClient = {
    let plugins: [PluginType] = [
      NetworkTrafficPlugin(indicators: .start, .done),
    ]
    let provider = MoyaProvider<MultiTarget>(plugins: plugins)
    let client = NetworkClient(provider: provider)
    return client
  }()
}
```

這個 `NetworkClient` 除了 `MoyaProvider<MultiTarget>` 提供的網路連線功能以外，還有偵測連線狀態的功能，當連線開始或結束時，他會在 console 中印出 Request Header, parameters 等資訊。（如果不需要這個功能的話，可以 overload 他，把 plugin 的部分移除即可。）

## 執行網路連線

接著我們來看一下如何使用 `NetworkClient` 來執行網路連線：

```swift
extension API.NetworkClient {
  public func request<Request: TargetType>(_ request: Request) -> Promise<JSON> {
    return perform(request, on: requestQueue)
  }
}

extension API.NetworkClient {
  internal func perform<Request: TargetType>(_ request: Request, on callbackQueue: DispatchQueue) -> Promise<JSON> {
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

Moya 定義的 `TargetType` 為最小可以執行網路連線的單位，裡面定義了如 HTTPMethod, endpoint, base url, parameters, headers 等等可能會用在網路連線上的資訊，所以我們要基於 `TargetType` 來寫我們的 api。不過這裡先不提如何使用 `TargetType` 來寫 api，只要知道當我們傳入任意一個 `TargetType` 到 `NetworkClient` 中我們就能執行網路連線，且針對 `Request` 中定義的 `Response Type` 做 `Decoding` 的動作。而且要注意這些 api 都是跑在 `requestQueue` 之中，方便我們之後統一管理 api call。

且每個 api call 的回傳都是 `Promise`，相信大家對 `PromiseKit` 並不陌生，他能解決掉 callback hell 問題，這裡預設每個 api 都回傳 `Promise`。

對於任意的 `TargetType` 在我們還沒有定義 `Response Type` 以前，我們還不知道要如何轉型成 Swift Object（我們不知道要使用 Decodable 還是其他第三方套件來轉），所以預設使用 `SwiftyJSON` 來轉換成比較方便使用的 `JSON`，相信大家都知道 Swift 中使用 `Dictionary` 其實有諸多的不便。

所以我們在想要執行一個 api call 時只需要這樣做：

```swift
import APIKit

let request = SomeRequest()
API.shared.request(request)
  .done { json in
    // success with returned json
  }
  .catch { e in
    // handle error...
  }
```

## Conclusion of NetworkClient

`NetworkClient` 的職責很簡單：

1. 傳入一個 `Request`，執行網路連線，回傳定義好的 `Response Type`。
2. 暫停/回復 thread 上的 api call。

# Config

很多時候我們會有很多個 server 要連線，可能是 production server 或者是 staging server，且 staging server 可能有好幾台。這時候我們可以這樣定義我們的 server config：

```swift
import APIKit

extension API {
  static var config: Config = .default

  enum Config {
  case `default`, staging, staging_04

    var baseURL: URL {
      switch self {
      case .default: return URL(string: "https://google.com")!
      case .staging: return URL(string: "https://staging.google.com")!
      case .staging_02: return URL(string: "https://staging-02.google.com")!
      }
    }

    var headerAuthSecretKey: String {
      switch self {
      case .default: return "ya"
      case .staging: return "ya-staging"
      case .staging_02: return "ya-staging-04"
      }
    }
  }
}
```

假設我們有一個正式 server 以及兩台測試 server，可以通過過展 API namespace，在 API 底下新增一個 config，且將 server config 寫在 `Config` enum 中，我們要切換 server 時只要更改 `API.config` 即可切換到指定的 server url。

APIKit 不將這些包入 framework 之中是因為管理 config 的方式不只一種，要視情況調整。

# Structure a Network Request

接著來看一下如何定義一個 network call。

## Define Request Type

假設我們的 app 會用到 GitHub 的某些 api，我們可以先定義一些最基礎的型態，定義好之後再基於這個型態建立出各個 api call。前面有提到我們使用了 Moya 幫我們做了的一些抽象層，最小的 network call 是 `TargetType`，`TargetType` 這個 protocol 需要時做一些基本的東西比如 base url, endpoint, HTTP method 等來執行一個 network call，現在我們要基於 `TargetType` 再擴展出一個專屬於 GitHub Request 的 Type。

```swift
import APIKit
import Moya

public protocol GitHubRequestType: TargetType {
  var parameters: [String : Any] { get }
}

extension GitHubRequestType {
  public var baseURL: URL { API.config.baseURL }
  public var headers: [String : String]? { [:] }
  public var sampleData: Data { Data() }
  public var parameters: [String : Any] { [:] }
}
```

由於 `GitHubRequestType` 的基底是 `TargetType`，也因為 `GitHubRequestType` 有固定的 url，所以我們可以透過 extension 的方式給他一個固定的 url，且這個 url 可以透過剛才我們宣告的 API.config 來取得（如果 config 有變化，base url 也會跟著更新）。這裡多宣告了 `parameters: [String : Any]` 的原因是 GitHub api 可能會有很多傳遞參數的情況發生，所以多一個 parameters 來存放可能會傳出的參數（這裡要多什麼 Property 可以根據使用狀況來新增）。

>>> 如果你的服務不只使用了 GitHub，可能用到了比如 Unsplash, Pinterest 等等服務，你可以多定義出 `UnsplashRequestType`, `PinterestRequestType`，就可以支援多個服務囉。

## Define Request

定義好 `GitHubRequestType` 之後我們來看如何定義一個 api call。

```swift
import APIKit

public struct GitHubReqeust {
  public struct User {
    public struct GetProfile: GitHubRequestType {
      var path: String { "/users/\(userID)" }
      var method: Method { .get }
      var task: Task { .requestPlain }

      let userID: String
      init(of userID: String) {
        self.userID = userID
      }
    }
  }
}
```

一個最單純的 GET 只要給他要連線的 path（a.k.a endpoint），告訴他連線的方式為 `.get`，這樣就完成定義一個最簡單的 `Request` 了！

如果要帶上參數的話，就要改 `Task` 中 encoding 方式了，詳情請見 [Moya/Task.swift](https://github.com/Moya/Moya/blob/master/Sources/Moya/Task.swift)。（Multipart 也可以參考）

你可能會好奇為什麼我要包很多層 struct，為什麼不用官方建議的 enum 方式？有以下幾個原因：

1. 使用 enum 定義的話當 api 一多，每個 path, method, parameters 就會很分散，這裡我希望一個 struct 就指定一個 api call（當然使用 enum 也可，單純個人偏好）
2. 使用多層 struct 可以將 namespace 切分出來，不會將所有的 api 集中塞在一個 struct 之中。（當然使用 enum 也可以做到）

到這裡我們要取得 GitHub 上某位使用者的 profile 就可以這樣做：

```swift
import APIKit

API.shared.request(GitHubReqeust.User.GetProfile(of: "some_user_id"))
  .done { json in
    // success with returned json
  }
  .catch { e in
    // handle error...
  }
```

## Decoding

最佳的情況是我們可以固定的將 server 吐給我們的資料轉成 swift object 方便使用，除了可以使用官方提供的 `Decodable` 以外，我發現 `ObjectMapper` 也是不錯的 json decode 工具。所以 `APIKit` 同時支援 `Decodable` 以及 `ObjectMapper`。

假設剛剛的 `GetProfile` 回傳的 json 可以被轉換成 User Object（不管是使用 `Decodable` 還是 `ObjectMapper`）：

```swift
import ObjectMapper

struct User {
  let name: String
  let id: String

  enum CodingKeys: String, CodingKey {
    case name, id
  }
}

extension User: Decodable {
  init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    self.name = try container.decode(String.self, forKey: .name)
    self.id = try container.decode(String.self, forKey: .id)
  }
}

extension User: ImmutableMappable {
  init(map: Map) throws {
    name = try map.value("name")
    id = try map.value("id")
  }
}
```

且告訴該 Request，他是可以被 decode 的型態，APIKit 提供了兩個 decode 用的 protocol：

```swift
import ObjectMapper

public protocol DecodableResponse {
  associatedtype ResponseType: Decodable
}

public protocol MappableResponse {
  associatedtype ResponseType: BaseMappable
}
```

只要將其套上 Request：

```swift
public struct GetProfile: GitHubRequestType, DecodableResponse {
  typealias ResponseType = User
}

// or

public struct GetProfile: GitHubRequestType, MappableResponse {
  typealias ResponseType = User
}
```

就可以在 network call 完成後自動被轉換為該 object 囉。

# Retry Request

如果有些 api 在失敗的時候會需要重試幾次，超過一定次數才會真的失敗的話，APIKit 也提供了一個 protocl：

```swift
public protocol RetryableRquest {
  var retryBehavior: RepeatBehavior { get }
}

public extension RetryableRquest {
  /// Default to general delay with retry count 3 times, each retry with 2 seconds interval.
  var retryBehavior: RepeatBehavior { return .delayed(maxCount: 3, time: 2) }
}

public struct GetProfile: GitHubRequestType, MappableResponse, RetryableRquest {
  // ...
}
```

預設的重試次數為兩次，間隔 3 秒，如果需要間隔與次數的變化，可以 overload 該 property，或者是換成其他 retry 的方式：

```swift
public enum RepeatBehavior {
  case immediate(maxCount: UInt)
  case delayed(maxCount: UInt, time: Double)
  case exponentialDelayed(maxCount: UInt, initial: Double, multiplier: Double)
  case customTimerDelayed(maxCount: UInt, delayCalculator: (UInt) -> DispatchTimeInterval)
}
```

通常比較常使用的是 `delayed` 跟 `exponentialDelayed`，`exponentialDelayed` 為指數避障算法，有興趣者可以自行 google 一下。

# Plugins

會選擇使用 Moya 作為這個框架個基礎是因為我們可以在每個 api call 的前與後做一些手腳，可以看到 [Moya/Plugin](https://github.com/Moya/Moya/blob/master/docs/Plugins.md#plugins) 中提到每個 request 要送出前都可以對其 `URLRequest` 插入一些值，或者在取得 response 時檢查 error code，並且作出處理。

利用這些特性我們可以簡單地做到 inject header 跟 access token 的效果。

## Access Token Injection

參考：[Moya/Plugins/AccessTokenPlugin.swift](https://github.com/Moya/Moya/blob/master/Sources/Moya/Plugins/AccessTokenPlugin.swift)

只要我們需要該 Request 在送出前都加上 access token，我們只要在該 Request 加上 `AccessTokenAuthorizable` 即可：

```swift
public struct GetProfile: GitHubRequestType, MappableResponse, AccessTokenAuthorizable {
  // ...
}
```

## Header injection

同理，如果要加上 Header，也可以參考 [Moya/Plugins/AccessTokenPlugin.swift](https://github.com/Moya/Moya/blob/master/Sources/Moya/Plugins/AccessTokenPlugin.swift)，或者參考 APIKit 中的 `XAuthHeaderInjectingPlugin`。

## Refresh Token Plugin

APIKit 提供了一個 `RefreshTokenPlugin` 來幫助處理 refresh access token 問題，在處理 refresh token ˊ之前，會建議先了解 OAuth 2.0 的 refresh token 具體是在做什麼的。

Refresh token 有一些特性我們要先了解：

1. refresh token 只能使用一次，且一次只能有一個 refresh request 執行（如果一次打兩個以上的 refresh request 出去，就會有問題）
2. access token 有時效性，只要超過時效，就必須使用 refresh token 去換新的 access token（甚至有些 refresh token 也有時效性，但 refresh token 時效要比 access token 還要長）
3. refresh 可能會失效（可能 timeout、可能是對方 server 壞掉，這些 edge case 很罕見，可以斟酌情況處理）
4. api 失敗後要看 server 定義了 401 還是 403 為 unauthorized，取得某些特定 error code 才觸發 refresh
5. 在 refresh 同時，api call 全部暫停（切記 refresh plugin 只會將該 network client 上的所有 api 暫停，如果你有多個 client，並不會全部都暫停）
6. 在 success 後要記得更新 access token 到你存放 token 的地方，不然你會一直用舊的 token 在做驗證
7. 由於某些情況下會觸發很多 401 的 error（比如你打了 3 個 api，全部拿到 401 error code，其中有一個 api 先完成，且觸發 refresh，其他兩個在 10 秒後才帶著 401 error code 回來，這時候還會觸發一次 refresh request，為了處理這個狀況，在 refresh 成功後 60 秒內，我會 100% 相信當前的 token 為有效 token，然後忽略所有的 401 error code）

如果有些特殊的狀況要處理，建議可以自己寫一個客製化的 plugin 來處理。

## 如何使用這些 Plugins

如果要使用 Plugin 的話我們要對 NetworkClient 做一點手腳（以下這個 client 就加入了四個不同效果的 plugin）：

```swift
extension API {
    public static let shared: NetworkClient = {
        let refreshPlugin = RefreshTokenPlugin(
            checkRefreshTokenValidLengthClosure: {
                return true
            },
            triggerRefreshClosure: { response in
                return true
            },
            refreshRequest: SampleReqeust.Auth.RefreshAccessToken(),
            successToRefreshClosure: { json in accessToken += "after refresh" },
            failToRefreshClosure: { error in }
        )
        let xAuthHeaderInjectingPlugin = XAuthHeaderInjectingPlugin(xAuthHeaderClosure: { target in
            return API.config.xAuthToken
        })
        let plugins: [PluginType] = [
            NetworkTrafficPlugin.init(indicators: .start, .done),
            xAuthHeaderInjectingPlugin,
            refreshPlugin,
            AccessTokenProvidingPlugin(tokenClosure: {
                return accessToken
            })
        ]
        let provider = MoyaProvider<MultiTarget>(plugins: plugins)
        let client = NetworkClient(provider: provider)
        refreshPlugin.networkClientRef = client
        return client
    }()
}
```

>>> 我們可以 overload 原有的 `shared` `NetworkClient`，或者再新增另外一個 `NetworkClient`。

# OAuth and RESTful together

搭配上面的 `RefreshTokenPlugin` 跟 `RetryableRquest`，我們可以讓每一個 request 在拿到 401 的同時觸發 refresh request 且 retry 原本的 request，進而達到無縫換 token 的效果。

# RxSwift Submodule

如果你習慣使用 RxSwift，我們也提供了 RxSwift 的擴展，這些擴展支援上述的所有功能包含 decoding, retry。

你可以透過 Cocoapods 安裝：

```ruby
pod 'APIKit/RxSwift'
```
