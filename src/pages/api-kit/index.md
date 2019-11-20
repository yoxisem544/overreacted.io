---
title: RESTful API and OAuth 2.0
date: '2019-11-20'
spoiler: How we structure our network layer
---

### Table of Contents

- [Basics](./#basics)
- NetworkClient
- Config
- Structure a Network Request
  - Define Request Type
  - Define Request
  - Decoding
  - Header, Access token
- Retry Request
- Plugins
- OAuth and RESTful together
- RxSwift Submodule

# Basics

## Dependencies

APIKit 使用了以下套件來輔助我們抽象化一些複雜的實作，以達到快速開發的目的：
- [Moya & Moya/RxSwift](https://github.com/Moya/Moya): Help us to structure api code, mocking response data, stubbing network responses.
- [Alamofire](https://github.com/Alamofire/Alamofire): HTTP networking library written in Swift.
- [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON): Help us to tranform response data to handful Swift JSON object. Really useful when you just want to take a look at response json.
- [ObjectMapper](https://github.com/tristanhimmelman/ObjectMapper): Help us to transform json into Swift model. An alternative option of JSON decoding.
- [PromiseKit](https://github.com/mxcl/PromiseKit): Promise for Swift.
- [RxSwift](https://github.com/ReactiveX/RxSwift): Reactive Programming in Swift

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

## Header, Access token

# Retry Request

# Plugins

# OAuth and RESTful together



















ㄦ
