# Tri-State Booleans

I have recently been thinking about good strategies to deal with the uncertainty involved in decoding JSON payloads, received from a server, into Swift models. In a previous article, I explored how we could safely [decode enumerations](../articles/safely-decode-swift-enums.html) in Swift. In this one, I want to take a look at [Bool](https://developer.apple.com/documentation/swift/bool) values in the same context.

## The Problem

Let's assume that we're building an app where at some point in the user journey, they can pay for some service. Just for the sake of argument, let's assume that to handle payments, our app sends a `POST` request to a (very legacy) server and the server replies with a JSON response containing a boolean describing whether the payment was successful or not, and a message providing more technical context into the result of the operation.

```shell
{
    "isSuccess": true,
    "developerMessage": "OK response received from the banking authority."
}
```

In our app's code, we would decode this response into a type similar to the one below.

```swift
struct PaymentAttemptResult: Decodable {
    let isSuccess: Bool
    let developerMessage: String
}
```

And we could have the following function that displays, depending on the payment attempt result, a `success` state or an `error` state with the possibility to retry.

```swift
func handlePaymentAttemptResult(_ result: PaymentAttemptResult) {
    if result.isSuccess {
        displaySuccessState()
    } else {
        displayErrorState(shouldRetry: true)
    }
}
```

This works great until, for whatever reason, our legacy server starts sending the following response.

```shell
{
    "isSuccess": null
    "developerMessage": ""
}
```

Our app will suddenly start crashing with a `valueNotFound` error `Expected Bool value but found null instead.`. Our immediate fixing attempt would be to make `isSuccess` optional.

```swift
struct PaymentAttemptResult {
    let isSuccess: Bool?
    let developerMessage: String
}
```

Now our app will no longer crash when we receive an unexpected `null` but we can easily introduce a dangerous bug if we're not careful. 

To make our code compile, we would have to update `handlePaymentAttemptResult` and it's very easy to do something like the following.

 ```swift
 func handlePaymentAttemptResult(_ result: PaymentAttemptResult) {
    if result.isSuccess == true {
        displaySuccessState()
    } else {
        displayErrorState(shouldRetry: true)
    }
}
```

Or

```swift
func handlePaymentAttemptResult(_ result: PaymentAttemptResult) {
    if let isSuccess = result.isSuccess, isSuccess {
        displaySuccessState()
    } else {
        displayErrorState(shouldRetry: true)
    }
}
```

The code compiles but we're equating the cases of `isSuccess` being `nil` to the payment attempts being unsuccessful and potentially prompting the user to retry again. However, `nil` here doesn't mean that the payment attempt was unsuccessful. It means *something unexpected happened on the server and we don't know whether the payment attempt was successful or not*. 

The right way to update `handlePaymentAttempt` is to handle the case of `nil` values separately.

```swift
func handlePaymentAttemptResult(_ result: PaymentAttemptResult) {
    if let isSuccess = result.isSuccess {
        if isSuccess {
            displaySuccessState()
        } else {
            displayErrorState(shouldRetry: true)
        }
    } else {
        // Handle the unexpected `nil` here.
    }
}
```

In the urgency of fixing a crash, this type of bugs can easily be introduced in a codebase and may slip through code review. Our best bet to eliminate it is to enforce that unexpected `nil` values are caught and dealt with at compile time.

## The Tri-State Boolean

One solution to this problem could be to use a boolean type that  can have 3 possible values instead of 2: `true`, `false` and `indeterminate`. We can model it using an enumeration.

```swift
enum Boolean {
    case `true`
    case `false`
    case indeterminate
}
```

Let's add a conformance to `Decodable`.

```swift
extension Boolean: Decodable {
    init(from decoder: Decoder) throws {
        do {
            let container = try decoder.singleValueContainer()
            let value = try container.decode(Bool.self)
            self = value ? .true : .false
        } catch {
            self = .indeterminate
        }
    }
}
```

Whenever we have an unexpected value instead of a valid boolean in a JSON payload, the corresponding `Boolean` value will be set to `.indeterminate`.

Next, we change the type of `isSuccess` from `Bool` to `Boolean` and the compiler will force us to handle the `indeterminate` case in `handlePaymentAttemptResult`.

```swift
func handlePaymentAttemptResult(_ result: PaymentAttemptResult) {
    switch result.isSuccess {
    case .true:
        displaySuccessState()
    case .false:
        displayErrorState(shouldRetry: true)
    case .indeterminate:
        // Handle indeterminate cases here.
    }
}
```

Et voilà! We have the compile time guarantee that unexpected boolean values will be handled appropriately.

We can go further and make our `Boolean` type nicer to use by conforming it to `ExpressibleByBooleanLiteral`.

```swift
extension Boolean: ExpressibleByBooleanLiteral {
    init(booleanLiteral value: Bool) {
        self = value ? .true : .false
    }
}
```

We can then write.

```swift
let value: Boolean = true // or false
```

We can also go ahead and overload the boolean operators `&&` and `!!`.

```swift
/// Returns `true` when both sides of the operator
/// are truthy values.
func && (lhs: Boolean, rhs: Boolean) -> Bool {
    switch (lhs, rhs) {
    case (.true, .true):
        return true
    default:
        return false
    }
}

func && (lhs: Boolean, rhs: Bool) -> Bool {
    switch (lhs, rhs) {
    case (.true, true):
        return true
    default:
        return false
    }
}

func && (lhs: Bool, rhs: Boolean) -> Bool {
    return rhs && lhs
}

/// Returns `true` when at least one side
/// of the operator is a truthy value.
func || (lhs: Boolean, rhs: Boolean) -> Bool {
    switch (lhs, rhs) {
    case (.true, _):
        return true
    case (_, .true):
        return true
    default:
        return false
    }
}

func || (lhs: Boolean, rhs: Bool) -> Bool {
    switch (lhs, rhs) {
    case (.true, _):
        return true
    case (_, true):
        return true
    default:
        return false
    }
}

func || (lhs: Bool, rhs: Boolean) -> Bool {
    return rhs || lhs
}
```

Adding a prefix `!` operator should be straightforward but this operator should return a value of type `Boolean` (not `Bool`) because we don't know what the negation of `.indeterminate` is.

```swift
prefix func ! (value: Boolean) -> Boolean {
    switch value {
    case .true:
        return .false
    case .false:
        return .true
    case .indeterminate:
        return .indeterminate
    }
}
```

Similarly, any equality operator we would want to introduce will need to return a value of type `Boolean` because the result of the equality test might be indeterminate.

```swift
/// Returns `.indeterminate` when one side is indeterminate.
/// Otherwise, checks both sides for equality.
func == (lhs: Boolean, rhs: Boolean) -> Boolean {
    switch (lhs, rhs) {
    case (.true, .true):
        return .true
    case (.false, .false):
        return .true
    case (.indeterminate, _):
        return .indeterminate
    case (_, .indeterminate):
        return .indeterminate
    default:
        return .false
    }
}

func == (lhs: Boolean, rhs: Bool) -> Boolean {
    switch (lhs, rhs) {
    case (.true, true):
        return .true
    case (.false, false):
        return .true
    case (.indeterminate, _):
        return .indeterminate
    default:
        return .false
    }
}

func == (lhs: Bool, rhs: Boolean) -> Boolean {
    return rhs == lhs
}
```


## Conclusion

When we decode JSON encoded boolean values to `Bool` values in Swift, in some situations we need to be extra careful and make sure that malformed and `null` boolean values are properly handled. Using a tri-state boolean type gives us the guarantee at compile time that those unexpected values are dealt with. What are your thoughts on this? What techniques do you use to handle unexpected boolean values in your JSON payloads? Feel free to get in touch on Twitter [@ftchirou](https://twitter.com/ftchirou/) to share them 🙂.

Thanks for reading 👋.
