# Designing modern data access layers in Swift

During the process of building our applications, we are often faced with the need of persisting and querying model objects  in some form of store. The store can be a remote server, a local CoreData database, a set of files, or even a PostgreSQL or MySQL database (if the models are shared between the server and client code). It is also not uncommon to have to manage a combination of 2 or more stores (e.g. saving to a remote server and to CoreData at the same time and retrieving from CoreData when there is no internet connectivity). 

In this article, we'll explore how using Swift features such as [protocols](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html), [generics](https://docs.swift.org/swift-book/LanguageGuide/Generics.html), [enumerations](https://docs.swift.org/swift-book/LanguageGuide/Enumerations.html), and [key paths](https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md), we can build expressive, type-safe and testable data access layers.

## Abstracting the Store

Our first step is to abstract the store in a simple and expressive collection-like API for interacting with persisted objects.

Fundamentally, a store is simply an entity that allows [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations. Whether it is a web service, a PostgreSQL database, or even a set of files on the disk, a store should allow a user to insert, update, delete objects; and to perform queries to search or filter the persisted objects.

We can start by wrapping that definition in a protocol.

```swift
protocol Store {
    associatedtype Object
    
    func insert(_ object: Object) -> Future<Object, Error>
    func update(_ object: Object) -> Future<Object, Error>
    func delete(_ object: Object) -> Future<Object, Error>
    func execute(_ query: ???) -> Future<[Object], Error>
}
```

We defined an associated type `Object` representing the objects the `Store` can handle. We also defined a function for each one of the CRUD operations.

Because CRUD operations can be asynchronous, each function returns a [Future](https://developer.apple.com/documentation/combine/future) object. A `Future` is a [Publisher](https://developer.apple.com/documentation/combine/publisher) that produces a value some time in the future and finishes or fails with an error.

Creating, updating and deleting objects are simple enough operations to express in Swift. The most interesting part will be to design a type-safe and expressive API for querying the stored objects. Let's get to it!

### Querying the Store

We will abstract the specifics of querying a store in an object of type `Query`. Let's start its implementation with an empty `struct`.

```swift
struct Query {
}
```

For our `Query` type to be useful, at a minimum it needs to have capabilities for:

- *filtering* using predicates on one or more of the stored objects properties
- and *sorting* by on one or more of the stored objects properties.

#### Filtering

Filtering is restricting the set of all the objects in the store to a smaller set of objects satisfying a predicate.

Let's create an enumeration to represent predicates.

```swift
enum Predicate {
}
```

And add a reference to the enumeration in our `Query` type.

```swift
struct Query {
    let predicate: Predicate
}
```

One of the most basic predicates we could express is a *comparison*. For example, in a personal movie database app, we could want to retrieve all the movies with a rating equal to PG-13. 

##### Comparison Predicates

We can express comparison predicates with a `case` in the `Predicate` enumeration.

```swift
enum Predicate<T> {
    case comparison(PartialKeyPath<T>, Operator, Primitive)  
}

enum Operator {
    case lessThan
    case lessThanOrEqualTo
    case equalTo
    case greaterThanOrEqualTo
    case greaterThan
}
```

Our comparison predicate has 3 components:

- a `PartialKeyPath<T>` representing the property (on an object of type `T`) to compare. Using a [key path](https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md) guarantees at compile time that the property being used in the comparison, actually exists on the object.
- an `Operator`, an enum value describing the operator to use for the comparison
- and a [Primitive](https://gist.github.com/ftchirou/989f9cc8293ea5a0e72200fae51ae5af) representing the value to compare against the `PartialKeyPath<T>`. Using the `Primitive` protocol ensures that only primitive types or types with primitive raw values (integers, strings, dates etc.) can be used in a comparison predicate.

The use of `PartialKeyPath<T>` forces us to specialize `Predicate` with the object on which the predicate will apply. And because `Predicate` is now generic, `Query` will also have to be specialized with the type of the objects being queried. 

```swift
struct Query<T> {
    let predicate: Predicate<T>
}
```

At this point, we can finally replace the `???` in the signature of the `execute` function in `Store` with the appropriate type.

```swift
protocol Store {
    associatedtype Object
    
    // ...

    func execute(_ query: Query<Object>) -> Future<[Object], Error> 
}
```

So far so good. Assuming we have the following types defined,

```swift
struct Movie {
    let id: Id
    let title: String
    let genre: String
    let budget: Double
    let rating: Rating
    
    typealias Id = String
}

enum Rating: String, Primitive {
    case G
    case PG
    case PG_13 = "PG-13"
    case R
    case NC_17 = "NC-17"
}
```

let's fetch all the movies titled "Pulp Fiction".

```swift
let movieStore: Store
let predicate: Predicate<Movie> = .comparison(\.title, .equalTo, "Pulp Fiction") 
let query = Query(predicate: predicate)
let movies = movieStore.execute(query)
```

This works well enough but the API could certainly be more expressive. Let's fix that!

We start by overloading the operator `==` to return an equality `Predicate`.

```swift
func == <T, U: Equatable & Primitive> (lhs: KeyPath<T, U>, rhs: U) -> Predicate<T> { 
    return .comparison(lhs, .equalTo, rhs)
}
```

And we add an extension function named `filter` on `Store`. That function will accept a `Predicate` as its unique parameter. Its purpose will be to create and execute a `Query` with the provided `Predicate`.

```swift
protocol Store {
    associatedtype Object

    // ...
    
    func execute(_ query: Query<Object>) -> Future<[Object], Error>
}

extension Store {
    func filter(where predicate: Predicate<Object>) -> Future<[Object], Error> {
        let query = Query(predicate: predicate)
        return execute(query)
    }
}
```

We're now ready to rewrite our query fetching all the movies titled "Pulp Fiction".

```swift
let movieStore: Store
let movies = movieStore.filter(where: \.title == "Pulp Fiction")
```

Much better! This version is not only more expressive than the previous one, it is also more type-safe. Indeed, the signature of our equality operator above, `== <T, U: Equatable & Primitive> (lhs: KeyPath<T, U>, rhs: U)`, gives us two guarantees at compile time.

- The value of the property on the left hand side of the operator, and the value on the right hand side have the same type (`U`).
- Both values can be tested for equality.

These compile-time guarantees prevent us from writing code such as: 

```swift
// Does not compile because we can't compare a String (\.title) and an Int (200)
let movies = movieStore.filter(where: \.title == 200)
```

To complete the comparison predicate, let's add overloads for the remaining comparison operators.

```swift
func < <T, U: Comparable & Primitive> (lhs: KeyPath<T, U>, rhs: U) -> Predicate<T> {
    .comparison(lhs, .lessThan, rhs)
}

func <= <T, U: Comparable & Primitive> (lhs: KeyPath<T, U>, rhs: U) -> Predicate<T> {
    .comparison(lhs, .lessThanOrEqualTo, rhs)
}

func > <T, U: Comparable & Primitive> (lhs: KeyPath<T, U>, rhs: U) -> Predicate<T> { 
    .comparison(lhs, .greaterThan, rhs)
}

func >= <T, U: Comparable & Primitive> (lhs: KeyPath<T, U>, rhs: U) -> Predicate<T> {
    .comparison(lhs, .greaterThanOrEqualTo, rhs)
}
```

Et voilà! We built a type-safe and expressive API for filtering based on comparison predicates. Some examples:

```swift
let movieStore: Store

// Movies rated PG-13
movieStore.filter(where: \.rating == .PG_13)

// Movies with a budget of at least $10M
movieStore.filter(where: \.budget >= 10_000_000)

// Heist movies
movieStore.filter(where: \.genre == "Heist")
```


##### AND Predicates

Another type of predicates we can have are AND predicates or logical conjunctions. These predicates will allow us to query for objects matching 2 or more predicates.

We start by updating our `Predicate` enum.

```swift
indirect enum Predicate<T> {
    case comparison(PartialKeyPath<T>, Operator, Primitive)
    case and(Predicate<T>, Predicate<T>)
}
```

We overload the `&&` operator to add some expressivity.

```swift
func && <T> (lhs: Predicate<T>, rhs: Predicate<T>) -> Predicate<T> {
    .and(lhs, rhs)
}
```

And it's ready to use!

```swift
let movieStore: Store

// Movies rated PG-13 with a budget of at least $10 million
movieStore.filter(where: \.rating == .PG_13 && \.budget >= 10_000_000)
```

##### OR Predicates

Similarly to AND predicates, we add OR predicates or logical disjunctions. OR predicates will allow us to query objects matching at least one of 2 or more predicates.

```swift
indirect enum Predicate<T> {
    case comparison(PartialKeyPath<T>, Operator, Primitive)
    case and(Predicate<T>, Predicate<T>)
    case or(Predicate<T>, Predicate<T>)
}

func || <T> (lhs: Predicate<T>, rhs: Predicate<T>) -> Predicate<T> {
    .or(lhs, rhs)
}
```

##### NOT Predicates

We can also add a predicate that negates another predicate.

```swift
indirect enum Predicate<T> {
    // ...
    case not(Predicate<T>)
}

prefix func ! <T> (predicate: Predicate<T>) -> Predicate<T> {
    .not(predicate)
}
```

Usage:

```swift
let movieStore: Store

// Movies with any rating other than PG-13
movieStore.filter(where: !(\.rating == .PG_13))
```

So far, we've built a pretty good and expressive API for filtering objects in a store. Next up to complete our `Query` type: *sorting*.

#### Sorting

To sort a list, we need a property to sort on and the order of the sort (ascending or descending). We start by encapsulating these in a type.

```swift
struct SortCriterion<T> {
    let property: PartialKeyPath<T>
    let order: SortOrder

    enum Order {
        case ascending
        case descending
    }
}
```


Next, we update our `Query` type to add support for sorting criteria while maintaining the expressivity of our API.

```swift
struct Query<T: Store> {
    let store: T
    let predicate: Predicate<T.Object>
    let sortCriteria: [SortCriterion<T.Object>] = []

    func result() -> Future<[T.Object], Error> {
        return store.execute(self)
    }

    func sorted<U: Comparable>(
        by property: KeyPath<T.Object, U>, 
        inOrder order: SortCriterion.Order = .ascending
    ) -> Query<T> {
        return Query(
            store: store,
            predicate: predicate,
            sortCriteria: sortCriteria + .init(
                property: keyPath,
                order: order
            )
        )
    }
}
```


- `Query` is no longer generic on the type of the objects being queried, but on the type of the store in which the query will be executed. 
- In addition to a reference to `Predicate`, `Query` now has a reference to `Store` and an array of `SortCriterion`.
- To keep the API expressive, we add a function `sorted` that creates a copy of the current query, updates the sorting criteria of the copy with the provided arguments and returns that copy. This function also gives us the compile-time guarantee that the property to sort by is actually `Comparable` to values of the same type.
- Again, for expressivity, we add a function `result` that simply calls executes `self` using the referenced `Store`.

To complete our sorting API, we need to change the signature of `execute` in `Store` and to update  `filter` to no longer return a `Future<[Object], Error>`, but a `Query<Self>` that can be executed at a later time.

```swift
protocol Store {
    associatedtype Object

    // ...
    
    func execute(_ query: Query<Self>) -> Future<[Object], Error>
}

extension Store {
    func filter(where predicate: Predicate<Object>) -> Query<Self> {
        return Query(store: store, predicate: predicate)
    }
}
```

All these changes will allow us to express queries such as the ones below.

```swift
let movieStore: Store

// Movies with a budget of at least $10 million sorted by title
movieStore
    .filter(where: \.budget >= 10_000_000)
    .sorted(by: \.title)
    .result()

// G-rated Movies with a budget of at most $5 million sorted by title
// and by budget in descending order
movieStore
    .filter(where: \.rating == .G && \.budget < 5_000_000)
    .sorted(by: \.title)
    .sorted(by: \.budget, inOrder: .descending)
    .result()
```

Our store abstraction is now complete. We have simple functions for creating, updating and deleting objects. And we have an equally simple and expressive API for filtering and sorting the stored objects.

## Up the Ladder of Abstraction

Now that we have a pretty good abstraction of the low-level storage layer, it's time to climb up the ladder of abstraction and explore how we can build expressive, flexible and testable components on top of `Store`.

### Implementing business rules

Let's again assume that we're building a personal movie database app and that we have the following features to implement.

- Users can add and edit movies in the database
- Users can fetch a specific movie using its id
- Users can filter movies based on their rating, their release date or any other movie's property
- All the operations above are made by sending HTTP requests to an API
- All the write operations above (add and edit) should be mirrored in a local database from which movies will be fetched in case an error occurs with the remote store (e.g. when the internet connectivity is not available).

Using our `Store` abstraction, implementing these features should be straightforward.

We start by creating a type `MovieRepository`.

```swift
struct MovieRepository<RemoteStore: Store, LocalStore: Store> where RemoteStore.Object == Movie, LocalStore.Object == Movie {
    let remoteStore: RemoteStore
    let localStore: LocalStore
}
```

`MovieRepository` is specialized with 2 types of `Store`. A `RemoteStore` for performing operations against a REST API and a `LocalStore` for performing operations against a local database.

Next, we implement our features.

##### Add a new movie

To add a new movie, we first need to create it in the remote store and if the operation is successful, we persist it in the local store.

```swift
struct MovieRepository<RemoteStore: Store, LocalStore: Store> where RemoteStore.Object == Movie, LocalStore.Object == Movie {
    let remoteStore: RemoteStore
    let localStore: LocalStore

    func add(_ movie: Movie) -> AnyPublisher<Movie, Error> {
        return remoteStore
            .insert(movie)
            .handleEvents(receiveOutput: { addLocally($0) })
            .eraseToAnyPublisher()
    }

    private func addLocally(_ movie: Movie) {
        localStore.insert(movie)
            .receive(subscriber: Subscribers.Sink(
                receiveCompletion: { _ in },
                receiveValue: { _ in }
            ))
    }
}
```

We simply call `insert(_:Movie)` on the remote store and we use [handleEvents](https://developer.apple.com/documentation/combine/publisher/3204713-handleevents) to intercept the `Movie` emitted and insert it in the local store (by calling the private helper `addLocally(_:Movie)`).

We erase the returned value to the type `AnyPublisher<Movie, Error>` because at the call site, we're not concerned about the exact type of the returned publisher. Also because the initial publisher (`Future<Object, Error>`) can potentially go through many transformations, each one changing its type.

##### Update a movie

Updating a movie follows the exact same pattern as adding a movie.

```swift
struct MovieRepository<RemoteStore: Store, LocalStore: Store> where RemoteStore.Object == Movie, LocalStore.Object == Movie {
    let remoteStore: RemoteStore
    let localStore: LocalStore

    // ...

    func update(_ movie: Movie) -> AnyPublisher<Movie, Error> {
        return remoteStore
            .update(movie)
            .handleEvents(receiveOutput: { updateLocally($0) })
            .eraseToAnyPublisher()
    }

    private func updateLocally(_ movie: Movie) {
        localStore.update(movie)
            .receive(subscriber: Subscribers.Sink(
                receiveCompletion: { _ in },
                receiveValue: { _ in }
            ))
    }
}
```

##### Retrieve a movie by an id

```swift
struct MovieRepository<RemoteStore: Store, LocalStore: Store> where RemoteStore.Object == Movie, LocalStore.Object == Movie {
    let remoteStore: RemoteStore
    let localStore: LocalStore

    // ...

    func movie(withId id: Movie.Id) -> AnyPublisher<Movie?, Error> {
        return remoteStore
            .filter(where: \.id == id)
            .result()
            .catch { _ in 
                self.localStore
                    .filter(where: \.id == id)
                    .result()
            }
            .map { $0.first }
            .eraseToAnyPublisher()
    }
}
```

Retrieving a specific movie by its id is as simple as calling `filter` and passing the equality predicate `\.id == id` in parameter. Because `filter` emits a `[Movie]` (and we're only interested in the first element), we use the [map](https://developer.apple.com/documentation/combine/publisher/3204718-map) operator to get the first element of the array.

We use the [catch](https://developer.apple.com/documentation/combine/publisher/3204690-catch) operator to retrieve the movie from the local store, if an error occurs while retrieving from the remote store.

##### Fetch all movies matching a predicate

```swift
struct MovieRepository<RemoteStore: Store, LocalStore: Store> where RemoteStore.Object == Movie, LocalStore.Object == Movie {
    let remoteStore: RemoteStore
    let localStore: LocalStore

    // ...

    func movies(where predicate: Predicate<Movie>) -> AnyPublisher<[Movie], Error> {
        return remoteStore
            .filter(where: predicate)
            .sorted(by: \.title)
            .result()
            .catch { _ in 
                self.localStore
                    .filter(where: predicate)
                    .result()
            }
            .eraseToAnyPublisher()
    }
}
```

Finally, we add a function to fetch movies matching any predicate. Similarly to fetching a movie by its id, we use the [catch](https://developer.apple.com/documentation/combine/publisher/3204690-catch) operator to perform the query in the local store if any error occurs with the remote store.

We just built a flexible movie repository that can be configured with any store without changing a single line of its implementation. The underlying stores could even potentially be swapped at runtime. The logic of saving and filtering movies is clearly separated from the specifics of how the saving and filtering are done. 🎉

### Testing

The logic of the business rules explored in the previous section is quite peculiar. If this was a real-world application, we would definitely want to write some unit tests to ensure that the rules were properly implemented and to catch any regression in the future.

To unit test `MovieRepository`, we only need to mock the `Store`. This could be done in a couple of lines of code.

```swift
final class MovieStoreMock: Store {
    private(set) var movies: [Movie] = []

    func insert(_ object: Movie) -> Future<Movie, Error> {
        return Future { completion in
            self.movies.append(object)
            completion(.success(object))
        }
    }

    func update(_ object: Movie) -> Future<Movie, Error> {
        return Future { completion in
            if let index = self.movies.firstIndex(where: { $0.id == object.id }) {
                self.movies[index] = object
            }

            completion(.success(object))
        }
    }

    func delete(_ object: Movie) -> Future<Movie, Error> {
        return Future { completion in
            if let index = self.movies.firstIndex(where: { $0.id == object.id }) {
                self.movies.remove(at: index)
            }

            completion(.success(object))
        }
    }

    func execute(_ query: Query<MovieStoreMock>) -> Future<[Movie], Error> {
        return Future { completion in
            let includedMovies = self.movies.filter(query.predicate.isIncluded())
            completion(.success(includedMovies)
        }
    }
}
```

Our mock store uses a simple array of `Movie` as its underlying storage. `insert` adds a new movie to the array, `update` changes a movie at the appropriate index in the array and `delete` removes the provided movie from the array.

`execute` uses an extension function `isIncluded`, on `Predicate`, returning a closure of type `(T) -> Bool` to filter the array.

```swift
extension Predicate {
    fileprivate func isIncluded() -> (T) -> Bool {
        switch self {
        case let .comparison(keyPath, .greaterThan, value):
            return { $0[keyPath: keyPath] > value }

        case let .comparison(keyPath, .greaterThanOrEqualTo, value):
            return{ $0[keyPath: keyPath] >= value }
    
        case let .comparison(keyPath, .equalTo, value):
            return { $0[keyPath: keyPath] == value }

        case let .comparison(keyPath, .lessThanOrEqualTo, value):
            return { $0[keyPath: keyPath] <= value }

        case let .comparison(keyPath, .lessThan, value):
            return  { $0[keyPath: keyPath] < value }

        case let .and(firstPredicate, secondPredicate):
            return { firstPredicate.isIncluded()($0) && secondPredicate.isIncluded()($0) }
    
        case let .or(firstPredicate, secondPredicate):
            return { firstPredicate.isIncluded()($0) || secondPredicate.isIncluded()($0) }

        case let .not(predicate):
            return { predicate.isIncluded()($0) == false }
        }
    }
}
```

`isIncluded` simply transforms the predicate into a closure that takes an object of type `T` in parameter and returns `true` if the object satisfies the predicate.

Using this mock, unit testing `MovieRepository` becomes very straightforward. Below is a test to ensure that adding a movie to the remote store, also adds it to the local store.

```swift
import XCTest

class MovieRepositoryTests: XCTestCase {
    func testAddingToRemoteAlsoAddsLocally() {
        let expectation = self.expectation(description: "...")
        var result: Movie? = nil

        // Given
        let remoteStore = MovieStoreMock()
        let localStore = MovieStoreMock()
        let repoository = MovieRepository(
            remoteStore: remoteStore,
            localStore: localStore
        )
        let movie: Movie = makeMovie(/* ... */)
        
        // When
        _ = repository
            .add(movie)
            .sink(
                receiveCompletion: { _ in expectation.fulfill() },
                receiveValue: { movie in
                    result = movie
                    expectation.fulfill()
                }
            )
            
        wait(for: [expectation], timeout: 5)
        
        // Then
        XCTAssertNotNil(result)
        XCTAssertTrue(remoteStore.movies.contains(where: { $0.id == result?.id })
        XCTAssertTrue(localStore.movies.contains(where: { $0.id == result?.id })
    }
}
```

## Conclusion

Using Swift's powerful type system and features such as key paths, generics, enumerations, or operator overloads, we designed an expressive, flexible and testable data access layer that can be extended with relatively low-effort. For example, we can add support for new types of predicates (`startsWith`, `endsWith`, `between`, `like`, `matches` etc.), add support for query result pagination, build a more robust error handling mechanism, or again build a query optimizer (for more fun stuff).

Using the [plethora of operators](https://developer.apple.com/documentation/combine/publisher#3232891) available on publishers, the functions in `Store` can be composed and/or combined together to express complex business logic without losing the readability and testability of the code.

We did not explore how to implement a practical `Store` but it is not fundamentally different from our `MovieStoreMock` implementation. The interesting part would be the implementation of `execute` where `Predicate` values need to be transformed into store-specific predicates (e.g. `NSPredicate` for CoreData, URL query parameters, or `WHERE` clauses for SQL based databases). Maybe the topic of a future article 😉.

Do you have a comment, a suggestion or a question? Feel free to ping me on Twitter [@ftchirou](https://twitter.com/ftchirou).

Thanks for reading 👋.
