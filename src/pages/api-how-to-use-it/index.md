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

### Creating Models

### Using Promise

### Using Rx

### Multipart Upload

### Error Handling

---

## Advanced Usage

### Retry

### Plugins

### OAuth & Refresh Token


---

## Testing
### Stubbing

### Mock Data
