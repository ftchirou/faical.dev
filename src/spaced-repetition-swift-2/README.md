# Build a spaced repetition app in Swift: Handling difficulty

In our [first article](../articles/spaced-repetition-swift.html) about spaced repetition, we put together an initial version of the SuperMemo algorithm in a couple of Swift lines. However, our algorithm is far from perfect. In particular, it does not account for how hard it is to learn an item, when calculating the interval of time after which the item should be reviewed. Ideally, difficult items should be reviewed more frequently than the easier ones. Let's explore, in this article, how next versions of SuperMemo handle difficulty when computing repetition intervals.

## Easiness Factor (E-Factor)

[SM-2](http://super-memory.com/english/ol/sm2.htm) introduces the concept of Easiness Factor or [E-Factor](https://help.supermemo.org/wiki/Glossary:E-Factor). The *E-Factor* is a number that represents how difficult it was for a user to learn a specific item. Concretely, the *E-Factor* is a `Double` value between **1.3** (representing the most difficult items to learn) and **2.5** (representing the easiest items to learn). The higher the *E-Factor*, the easier it was for a user to learn the corresponding item.

We can add support for it by adding a new property `easiness` in our `Card` type.

```swift
struct Card {
    let front: String
    let back: String
    var easiness: Double = 2.5
}
```


Every new card created get assigned an *E-Factor* of 2.5. That value decreases whenever the user provides an incorrect response to the card during a repetition. To re-calculate the new value of the *E-Factor* after a repetition, we need a way to assess the quality of the response given by the user. SM-2 defines the quality of response as an integer between **0** (the worst quality) and **5** (the best quality). We can represent it in Swift with an `enum`.

```swift
enum ResponseQuality: Int, Comparable {
    // The user did not answer correctly and could not recall the correct response.
    case blackout = 0

    // The user did not answer correctly but could recall the correct response.
    case incorrectButRecalled = 1

    // The user did not answer correctly but the correct answer seemed easy to recall.
    case incorrectButRecalledWithDifficulty = 2

    // The user recalled the correct response, but with serious difficulty.
    case correctButRecalledWithDifficulty = 3

    // The user recalled the correct response with some hesitation.
    case correctButRecalledWithHesitation = 4

    // The correct response was recalled quickly with no difficulty.
    case perfect = 5
}
```

Using the quality of response, SM-2 re-calculates a card's *E-Factor* with the seemingly arbitrary following formula.

```swift
func computeEasiness(for card: Card, using quality: ResponseQuality) -> Double {
    let maxQuality = ResponseQuality.perfect.rawValue
    let easiness = card.easiness 
        + 0.1 - (maxQuality - quality.rawValue)
        * (0.08 + (maxQuality - quality.rawValue) * 0.02))
        
    return easiness.clamp(min: 1.3, max: 2.5)
}
```

`clamp` is an extension function on `FloatingPoint` that "clamps" a value between a minimum and maximum. In the case of computing *E-Factors*, if the new value is less than 1.3, then we consider  it to be 1.3; and if the new value is greater than 2.5, we consider it to be 2.5.

```swift
extension FloatingPoint {
    func clamp(min: Self, max: Self) -> Self {
        return (self...self)
            .clamped(to: min...max)
            .lowerBound
    }
}
```

## SM-2

With *E-Factor* and quality of response defined above, we can rewrite our initial implementation of SuperMemo to take into account each card's level of difficulty.

We start by changing our `nextInterval` function to take a `Card` instead of a `Deck` in parameter. This is needed because in our next version, each card will have a different repetition interval based on its *E-Factor*.

```swift
func nextInterval(for card: Card) -> TimeInterval {
    func recurse(_ repetitionCount: Int) -> Int {
        switch repetitionCount {
        case 1: return 1
        case 2: return 6
        default: return recurse(repetitionCount - 1) * card.easiness
        }
    }
    
    let repetitions = Current.database.repetitions(of: card)
    return recurse(repetitions.count) * 86400
}
```

This includes adding a function on `Database` to return the total number of repetitions of a specific card a user has done.

```swift
protocol Database {
    /* ... */
    func repetitions(of card: Card) -> [Repetition]
}
```

We also replace the `deck` property in our `Repetition` struct by a property of type `Card`.

```swift
struct Repetition {
    let card: Card
    let date: Date = Date()
}
```

Next, we create a new function to review a `Card`.

```swift
static func review(card: Card) -> (Card, ResponseQuality) {
    // We prompt the user to provide a response to the card.
    _ = Current.input.getString(card.front)

    // We display the correct response to the user after they provide theirs.
    Current.output.printString(card.back)

    // We ask the user to assess the quality of their response.
    let quality: ResponseQuality = Current.input.getResponseQuality()

    // We keep track of the review of this card in the database.
    Current.database.save(repetition: Repetition(card: card)

    // We re-compute the card's E-Factor based on the user's response quality.
    card.easiness = computeEasiness(for: card, using: quality)

    // Finally, we return the card with its E-Factor updated, along with the user's response quality.
    return (card, quality)
}
```

We can now rewrite our `review(:Deck)` function.

```swift
func review(deck: Deck) {
    var remaining = deck
    var updatedCards: [Card] = []

    while remaining.isEmpty == false {
        let cards = remaining.shuffled()
	
        for card in cards {
            let (updatedCard, responseQuality) = review(card: card)

            if responseQuality >= .correctButRecalledWithHesitation {
                remaining.removeAll(where: { $0 == card })
            }

            updatedCards.append(updatedCard)
        }
    }

    let newDecks: [TimeInterval: Deck] = Dictionary(
        grouping: updatedCards, 
        by: { nextInterval(for: $0) }
    )

    for (interval, deck) in newDecks {
        Current.notifier.createNotification(
            withContent: "\(deck.count) cards available to review now!",
            triggeredAfter: interval,
            action: { self.review(deck: deck) }
        )
    }
}
```

Similarly to our previous implementation, this new implementation loops through all the cards in the deck until the user has provided satisfactory responses to all the cards. The differences are:

- the cards have their *E-Factor* updated at the end of the review
- the next repetition interval is re-calculated for each card using its updated *E-Factor*
- cards with the same repetition interval are grouped in the same deck
- for each new deck, a notification for review is scheduled to be triggered at the appropriate time.

## Next steps

We hugely improved our algorithm by introducing *E-Factors* in the equation for computing repetition intervals. However, the formula for computing *E-Factors* seems quite arbitrary. The formula was deducted my means of trial and error by SuperMemo's author [Piotr Woźniak](https://en.wikipedia.org/wiki/Piotr_Woźniak_(researcher)). In newer articles about the topics, we'll explore how subsequent versions of SuperMemo improve that formula using concepts like *the matrix of optimal factors* or *the forgetting rate*.

Thanks for reading 👋.