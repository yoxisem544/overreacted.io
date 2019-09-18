---
title: Thread Safety in iOS
date: '2019-09-17'
spoiler: What exactly is thread safe?
---

大部分在 Swift 中可以 mutate 的東西，都可能不是 thread safe 的。舉一個例子：

假設有一個 Person 的物件，裡面有 name 跟 age 這兩個屬性，且同時有兩個人在不同執行緒中要更改 person 的 name，且在更改的同時有一個人要讀取該 person 的 name 屬性，最後讀取的人會得到什麼值？

>不知道。因為我們不知道這些人誰先完成了讀取或者寫入的動作。

最直接的例子就是資料庫，在現實中的 app 裡，我們可能有很多的執行緒同時在讀取跟寫入，在這個情況下同時讀取寫入就會造成資料庫的 exception 進而讓 app 閃退。一個比較好的作法可能是寫入的時候一個一個寫入，讀取等到所有的寫入完成後才從資料庫讀取最新的資料出來，如此一來讀取時就不會讀取到寫入到一半的資料，或者讀取到舊的資料了。

概念雖然簡單，但具體上應該怎麼做呢？

# 來做一個簡單的 Thread Safe Array 吧

我們先來看一下多個執行緒同時操作同一個 array 會發生什麼事：
```swift
var array: [Int] = []

DispatchQueue.concurrentPerform(iterations: 10, execute: { i in
    DispatchQueue.concurrentPerform(iterations: 10, execute: { j in
        array.append(i*j)
    })
})
```

結果就是閃退：
```
error: Execution was interrupted, reason: EXC_BAD_INSTRUCTION (code=EXC_I386_INVOP, subcode=0x0).
```

---

因為 Swift Array 底層實作的關係，會去操作 pointer（這裡不贅述，有興趣的朋友歡迎看 Swift source code），多執行緒同時操作 array 就會有這樣的問題。

那我們能做的事情就是讓操作 array 的地方統一在一個執行緒中發生，因此我們可以在操作 array 前，先進到一個固定的執行緒後才執行 array 操作。

```swift
var array: [Int] = []
let queue = DispatchQueue(label: "io.some.thread")

DispatchQueue.concurrentPerform(iterations: 10, execute: { i in
    DispatchQueue.concurrentPerform(iterations: 10, execute: { j in
        queue.async {
            array.append(i*j)
        }
    })
})
```

在同一個執行緒中執行 array 操作後就不會有 exception 了。

# Read Write Lock

在上面提到，我們希望所有的讀取發生在寫入之後，如此一來就可以讀取到寫入之後的最新資料。GCD 提供了一些方便的方法讓我們可以做到這樣的效果，就是使用 `DispatchWorkItemFlags` 中的 `.barrier` 可以達到這個效果：

```swift
let queue = DispatchQueue(label: "io.some.thread", attributes: .concurrent)

func write(_ index: Int) {
    queue.async(flags: .barrier, execute: {
        print("performing write \(index)")
        sleep(3)
        print("performing write \(index) completed")
    })
}

func read(_ index: Int) {
    print("sync read task \(index)")
    queue.sync {
        print("inside read sync task \(index)")
        sleep(2)
    }
    print("sync read task \(index) completed")
}

DispatchQueue.concurrentPerform(iterations: 10, execute: { i in
    DispatchQueue.concurrentPerform(iterations: 10, execute: { j in
        Bool.random() ? write((i+1)*(j+1)) : read((i+1)*(j+1))
    })
})
```

可以試著自己跑跑看這個 sample code，可以看到讀取寫入一起發生時，write 會優先執行且 block read。

>>有興趣閱讀更多關於 [Read/Write Lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock) 可以點進連結看

# 完成 Thread Safe Array

Array 常用的操作不外乎就是 append, subscript 等等：
```swift
array.append(newElement)
array[0]
array.first
```

如果想要讓這個 array thread safe 的話，我們必須對他做一層包裝：
```swift
class SafeArray<Element> {
    private let queue = DispatchQueue(label: "io.safe.array.queue", attributes: .concurrent)
    private var array = Array<Element>()
}
```

接著實作可能會用到的一些操作（包含 read/write）：
```swift
// For read
extension SafeArray {
    var first: Element? {
        var result: Element?
        queue.sync { result = array.first }
        return result
    }
}

// For write
extension SafeArray {
    func append(_ newElement: Element) {
        queue.async(flags: .barrier) { self.array.append(newElement) }
    }
}
```

接著就可以放心的直接操作 array 囉：
```swift
let safeArray = SafeArray<Int>()

DispatchQueue.concurrentPerform(iterations: 10, execute: { i in
    DispatchQueue.concurrentPerform(iterations: 10, execute: { j in
        safeArray.append(i*j)
    })
})
```

---

最後把 array 所擁有的方法全部 expose 出來：

# performance measure

measure block：
```swift
extension Array where Element == TimeInterval {
    func averageTime() -> TimeInterval {
        return reduce(0, +) / TimeInterval(count)
    }
}

var normalArrayConsumeTimes: [TimeInterval] = []
var threadSafeArrayConsumeTimes: [TimeInterval] = []
var testArrayA: [Int] = []
var testArrayB = ThreadSafeArray<Int>()

func measure(label: String, _ block: @escaping (() -> Void), complete: (TimeInterval) -> Void) {
    print("measuring \(label)")
    let start = Date().timeIntervalSince1970
    DispatchQueue.global().sync {
        block()
    }
    let end = Date().timeIntervalSince1970
    let time = end - start
    print("measuring \(label) time: \(time) seconds")
    complete(time)
}
```

單純讀取：

```swift
for _ in 1...10 {
    let loopCount = 50000

    measure(label: "normal array", {
        var testArrayA: [Int] = []
        for _ in 1...loopCount {
            _ = testArrayA.last
        }
    }, complete: { time in normalArrayConsumeTimes.append(time) })

    measure(label: "thread safe array", {
        let testArrayB = ThreadSafeArray<Int>()
        for _ in 1...loopCount {
            _ = testArrayB.last
        }
    }, complete: { time in threadSafeArrayConsumeTimes.append(time) })
}

// normal array average time: 0.04547381401062012
// thread safe array average time: 0.09168415069580078
// thread safe array is 50.40166302952637% slower
```

thread safe 讀取效能上降低了 50%。

---

單純寫入：

```swift
for _ in 1...10 {
    let loopCount = 50000

    measure(label: "normal array", {
        var testArrayA: [Int] = []
        for _ in 1...loopCount {
            self.testArrayA.append(1)
        }
    }, complete: { time in normalArrayConsumeTimes.append(time) })

    measure(label: "thread safe array", {
        let testArrayB = ThreadSafeArray<Int>()
        for _ in 1...loopCount {
            self.testArrayB.append(1)
        }
    }, complete: { time in threadSafeArrayConsumeTimes.append(time) })
}

// normal array average time: 0.040544438362121585
// thread safe array average time: 0.3030177116394043
// thread safe array is 86.61977937105864% slower
```

thread safe 讀取效能上降低了 86%。

---

如果同時讀寫的話：

```swift
for _ in 1...10 {
    let loopCount = 50000

    measure(label: "normal array", {
        var testArrayA: [Int] = []
        for _ in 1...loopCount {
            testArrayA.append(1)
            _ = testArrayA.last
        }
    }, complete: { time in normalArrayConsumeTimes.append(time) })

    measure(label: "thread safe array", {
        let testArrayB = ThreadSafeArray<Int>()
        for _ in 1...loopCount {
            testArrayB.append(1)
            _ = testArrayB.last
        }
    }, complete: { time in threadSafeArrayConsumeTimes.append(time) })
}

// normal array average time: 0.05463418960571289
// thread safe array average time: 0.9117634773254395
// thread safe array is 94.00785500138957% slower
```

thread safe 讀寫效能上降低了 94%。

---

雖然效能明顯降低很多，但可以注意到的是每個 iteration 都有 5 萬次，如果單純看一般 array 跟 thread safe array 一次讀寫時間，以下是只讀寫 100 次的時間：

```
normal array average time:
0.00014765262603759765

thread safe array average time:
0.002648663520812988
```

兩者的時間都很小，以手機 60 fps 的更新率來算，1 fps 你有 0.016 秒的運算時間可以使用，相比起來 thread safe 的運算時間小很多。如果怕讀取會卡住等寫入完成的話，可以先到背景讀取，等到取得你要的資料時再回到 main 即可。

>>> thread safe array 一次讀寫操作約耗時 0.000026487 秒，在 0.016 秒中可以執行一樣的操作 600 次。但整體速度也要取決於裝置的 cpu，這裡只是大略測試。
