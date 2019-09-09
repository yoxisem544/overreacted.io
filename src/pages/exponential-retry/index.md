---
title: How to implement exponential retry?
date: '2019-09-09'
spoiler: Ways PromiseKit and RxSwift provided don't always fits our need
---

Sometimes, errors occur in real world application. For example, api sometimes failed due to network issue, server issue or awkward network condition. In such cases, we would like to retry several times before throwing an error out.

# RxSwift's Retry

Let's take a look at RxSwift's retry method

```swift
let disposeBag = DisposeBag()
var count = 1

let sequenceThatErrors = Observable<String>.create { observer in
    observer.onNext("🍎")
    observer.onNext("🍐")
    observer.onNext("🍊")

    if count == 1 {
        observer.onError(TestError.test)
        print("Error encountered")
        count += 1
    }

    observer.onNext("🐶")
    observer.onNext("🐱")
    observer.onNext("🐭")
    observer.onCompleted()

    return Disposables.create()
}

sequenceThatErrors
    .retry()
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

// Output:
// 🍎
// 🍐
// 🍊
// Error encountered
// 🍎
// 🍐
// 🍊
// 🐶
// 🐱
// 🐭
```

>>>> example of these sample codes are from: https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/decision_tree/retry.html

## RxSwiftExt

RxSwift provide us a retry method, with `.retry()`, we are able to retry 1 time before throwing error out. Or `.retry(5)` to retry 5 times before error.

It's simple and clear. But wait! What if I want 2 seconds before each retry? It seems `.retry()` happened immediately after error occured. Unfortunately, RxSwift does not implement this kind of retry for us. We need to do it ourself.

Good news is RxSwift has a great community, someone has done these complex retry already! In a third party project called [RxSwiftExt](https://github.com/RxSwiftCommunity/RxSwiftExt) has a lot handy method to use.

## RepeatBehavior

First, we will need a enum with some different retry rules.
1. `immediate`: Just like what RxSwift gives us.
2. `delayed`: What we want above, retry with some delay.
3. `exponentialDelayed`: a bit different from `delayed`, will extend every retry interval after each error.
4. `customTimerDelayed`: we won't discuss it here.

```swift
import RxSwift

/**
Specifies how observable sequence will be repeated in case of an error
- Immediate: Will be immediatelly repeated specified number of times
- Delayed: Will be repeated after specified delay specified number of times
- ExponentialDelayed: Will be repeated specified number of times.
Delay will be incremented by multiplier after each iteration (multiplier = 0.5 means 50% increment)
- CustomTimerDelayed: Will be repeated specified number of times. Delay will be calculated by custom closure
*/
public enum RepeatBehavior {
	case immediate (maxCount: UInt)
	case delayed (maxCount: UInt, time: Double)
	case exponentialDelayed (maxCount: UInt, initial: Double, multiplier: Double)
	case customTimerDelayed (maxCount: UInt, delayCalculator: (UInt) -> DispatchTimeInterval)
}

extension RepeatBehavior {
	/**
	Extracts maxCount and calculates delay for current RepeatBehavior
	- parameter currentAttempt: Number of current attempt
	- returns: Tuple with maxCount and calculated delay for provided attempt
	*/
	func calculateConditions(_ currentRepetition: UInt) -> (maxCount: UInt, delay: DispatchTimeInterval) {
		switch self {
		case .immediate(let max):
			// if Immediate, return 0.0 as delay
			return (maxCount: max, delay: .never)
		case .delayed(let max, let time):
			// return specified delay
			return (maxCount: max, delay: .milliseconds(Int(time * 1000)))
		case .exponentialDelayed(let max, let initial, let multiplier):
			// if it's first attempt, simply use initial delay, otherwise calculate delay
			let delay = currentRepetition == 1 ? initial : initial * pow(1 + multiplier, Double(currentRepetition - 1))
			return (maxCount: max, delay: .milliseconds(Int(delay * 1000)))
		case .customTimerDelayed(let max, let delayCalculator):
			// calculate delay using provided calculator
			return (maxCount: max, delay: delayCalculator(currentRepetition))
		}
	}
}
```

---

With these rules, we can now implement retry with delay! Let's see the code:

```swift
public typealias RetryPredicate = (Error) -> Bool

extension ObservableType {
	/**
	Repeats the source observable sequence using given behavior in case of an error or until it successfully terminated
	- parameter behavior: Behavior that will be used in case of an error
	- parameter scheduler: Schedular that will be used for delaying subscription after error
	- parameter shouldRetry: Custom optional closure for checking error (if returns true, repeat will be performed)
	- returns: Observable sequence that will be automatically repeat if error occurred
	*/
	public func retry(_ behavior: RepeatBehavior, scheduler: SchedulerType = MainScheduler.instance, shouldRetry: RetryPredicate? = nil) -> Observable<Element> {
		return retry(1, behavior: behavior, scheduler: scheduler, shouldRetry: shouldRetry)
	}

	/**
	Repeats the source observable sequence using given behavior in case of an error or until it successfully terminated
	- parameter currentAttempt: Number of current attempt
	- parameter behavior: Behavior that will be used in case of an error
	- parameter scheduler: Schedular that will be used for delaying subscription after error
	- parameter shouldRetry: Custom optional closure for checking error (if returns true, repeat will be performed)
	- returns: Observable sequence that will be automatically repeat if error occurred
	*/
	internal func retry(_ currentAttempt: UInt, behavior: RepeatBehavior, scheduler: SchedulerType = MainScheduler.instance, shouldRetry: RetryPredicate? = nil)
		-> Observable<Element> {
			guard currentAttempt > 0 else { return Observable.empty() }

			// calculate conditions for bahavior
			let conditions = behavior.calculateConditions(currentAttempt)

			return catchError { error -> Observable<Element> in
				// return error if exceeds maximum amount of retries
				guard conditions.maxCount > currentAttempt else { return Observable.error(error) }

				if let shouldRetry = shouldRetry, !shouldRetry(error) {
					// also return error if predicate says so
					return Observable.error(error)
				}

				guard conditions.delay != .never else {
					// if there is no delay, simply retry
					return self.retry(currentAttempt + 1, behavior: behavior, scheduler: scheduler, shouldRetry: shouldRetry)
				}

				// otherwise retry after specified delay
				return Observable<Void>.just(()).delaySubscription(conditions.delay, scheduler: scheduler).flatMapLatest {
					self.retry(currentAttempt + 1, behavior: behavior, scheduler: scheduler, shouldRetry: shouldRetry)
				}
			}
	}
}
```

With this extension on `ObservableType`, all observables can use our new retry method!

```swift
sequenceThatErrors
    // .retry() we don't need this anymore!
    .retry(RepeatBehavior.delayed(maxCount: 5, time: 2))
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
```

---

# Exponential Delay

We've implement retry with fixed interval, what's next? Are we able to make every retry interval a bit longer if we kept getting an error? Sure, you can! Let's take a look at `exponentialDelayed`. We will first init a `exponentialDelayed behavior`, then take a look at every retry interval of each retry.

```swift
let behavior = RepeatBehavior.exponentialDelayed(maxCount: 5, initial: 1, multiplier: 0.5)

behavior.calculateConditions(1).delay // milliseconds(1000)
behavior.calculateConditions(2).delay // milliseconds(1500)
behavior.calculateConditions(3).delay // milliseconds(2250)
behavior.calculateConditions(4).delay // milliseconds(3375)
behavior.calculateConditions(5).delay // milliseconds(5062)
```

We can see retry interval keeps growing after each retry. This is the difference between `exponentialDelayed` and `delayed` behavior.

---

# PromiseKit common pattern: Attempt

PromiseKit's common pattern: [Retry/Polling](https://github.com/mxcl/PromiseKit/blob/master/Documentation/CommonPatterns.md#retry--polling) provides us a attempt method for promise retry.

```swift
func attempt<T>(maximumRetryCount: Int = 3, delayBeforeRetry: DispatchTimeInterval = .seconds(2), _ body: @escaping () -> Promise<T>) -> Promise<T> {
    var attempts = 0
    func attempt() -> Promise<T> {
        attempts += 1
        return body().recover { error -> Promise<T> in
            guard attempts < maximumRetryCount else { throw error }
            return after(delayBeforeRetry).then(on: nil, attempt)
        }
    }
    return attempt()
}

attempt(maximumRetryCount: 3) {
    flakeyTask(parameters: foo)
}.then {
    //…
}.catch { _ in
    // we attempted three times but still failed
}
```

### How can we add `exponentialDelayed` to PromiseKit?

It's easy! Just replace `maximumRetryCount` and `delayBeforeRetry` with `RepeatBehavior`!

```swift
func attempt<T>(_ behavior: RepeatBehavior, _ body: @escaping () -> Promise<T>) -> Promise<T> {
    var attempts: UInt = 0
    func attempt() -> Promise<T> {
        attempts += 1
        return body().recover({ error -> Promise<T> in
            let (maxCount, delay) = behavior.calculateConditions(attempts)
            guard attempts < maxCount else { throw error }
            return after(delay).then(on: nil, attempt)
        })
    }
    return attempt()
}

let behavior = RepeatBehavior.exponentialDelayed(maxCount: 5, initial: 1, multiplier: 0.5)
attempt(behavior) {
  flakeyTask(parameters: foo)
}.then {
    //…
}.catch { _ in
    // we attempted three times but still failed
}
```
