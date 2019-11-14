# Computed and Lazy types in Swift

Swift allows us to define properties that associate values with a particular type. Some properties have interesting behaviors. For instance, computed properties compute their value every time they're accessed and lazy stored properties do not compute their value until the first time they're accessed. Just for fun, let's explore how Swift types can adopt similar behaviors. For example, can we create a "computed" `Date` type whose instances would always return the current date whenever they're accessed?

## Computed Types

Let's reuse the example in the previous paragraph and assume that for some reason, we want instances of `Date` created with the default `Date()` to always return the current date whenever they're evaluated.

```swift
let currentDate = Date()

print(currentDate.timeIntervalSince1970) // prints 1573666964.129724
print(currentDate.timeIntervalSince1970) // prints 1573666964.16371
print(currentDate.timeIntervalSince1970) // prints 1573666964.163834
```

The simplest way to do this in Swift would be to turn `currentDate` into a computed property.

```swift
var currentDate: Date {
    Date()
}
```

Problem solved? Not quite! `currentDate` will always evaluate to the current date until it is passed to a function or copied to another variable. At that point, it will evaluate to whatever value it was before being copied. In most cases, this is what we want but here, we want an instance created with `Date()` to always evaluate to the current date whenever and wherever accessed.

To achieve the desired behavior, we can extract the shape of computed properties into a type. For `currentDate`, that shape is `{ Date() }`. It looks like a closure that takes zero parameter and returns a value of type `Date`. The type of that closure is `() -> Date`. Its type can be generalized to describe all closures that take zero parameter and return a value of type `T`: `() -> T`. Let's encapsulate that closure inside a `struct`.

```swift
struct Computed<T> {
    private let getter: () -> T
    
    init(_ getter: () -> T) {
        self.getter = getter
    }
    
    var value: T {
        getter()
    }
}
```

The initializer of `Computed` takes as parameter a closure used to compute values of type `T`. Those computed values are accessible through the `value` property.

Back to our example, we can use it as follows.

```swift
let currentDate = Computed { Date() }

print(currentDate.value.timeIntervalSince1970) // prints 1573668565.843086
print(currentDate.value.timeIntervalSince1970) // prints 1573668565.854287
print(currentDate.value.timeIntervalSince1970) // prints 1573668565.854409
```

It works as expected. In addition, we can pass around `currentDate` to functions or copy it and its "computed" behavior will remain intact.

However, we have now introduced an indirection on the `Date` type. We need to call `.value` before getting access to the properties of the encapsulated `Date` object. It would be great if we could get rid of that indirection and use a `Computed<Date>` object as if it was a `Date` object.

Fortunately in Swift 5.1, we can do just that by using [key path dynamic member lookup](https://github.com/apple/swift-evolution/blob/master/proposals/0252-keypath-dynamic-member-lookup.md).

```swift
@dynamicMemberLookup
struct Computed<T> {
    private let getter: () -> T
    
    init(_ getter: () -> T) {
        self.getter = getter
    }
    
    var value: T {
        getter()
    }
    
    subscript<U>(dynamicMember keyPath: KeyPath<T, U>) -> U {
        value[keyPath: keyPath]
    }
}
```

Now, we can use the `Computed` type as follows.

```swift
let currentDate = Computed { Date() }

print(currentDate.timeIntervalSince1970) // prints 1573669687.47322
print(currentDate.timeIntervalSince1970) // prints 1573669687.484854
print(currentDate.timeIntervalSince1970) // prints 1573669687.484977
```

Et voilà! We can use any property available in `Date` on `Computed<Date>`. We can go one step further by adding any behavior that `T` has, to `Computed<T>`. For example, below we make `Computed<T>` equatable whenever `T` is equatable.
 
 ```swift
extension Computed: Equatable where T: Equatable {
    static func ==(lhs: Computed<T>, rhs: Computed<T>) -> Bool {
        return lhs.value == rhs.value
    }
}
 ```
 
## Lazy Types

We can use a similar approach to define `Lazy` types. Types whose instances are not evaluated until the first time they are accessed.

```swift
@dynamicMemberLookup
struct Lazy<T> {
    private let getter: () -> T
    private lazy var initialValue = getter()

    init(_ getter: @escaping () -> T) {
        self.getter = getter
    }

    var value: T {
        mutating get {
            initialValue
        }
    }

    subscript<U>(dynamicMember keyPath: KeyPath<T, U>) -> U {
        mutating get {
            value[keyPath: keyPath]
        }
    }

    subscript<U>(dynamicMember keyPath: WritableKeyPath<T, U>) -> U {
        mutating get {
            value[keyPath: keyPath]
        }
        set {
            initialValue[keyPath: keyPath] = newValue
        }
    }
}
```

One of the differences with `Computed<T>` is the `subscript` overload that takes a `WritableKeyPath<T, U>` in parameter. That allows us to mutate properties of an object of type `T` using a `Lazy<T>` object.

We can then use it as follows.

```swift
var connection = Lazy { DatabaseConnection() }

connection.username = "faical"
connection.password = "******"
```


Thanks for reading 👋.

