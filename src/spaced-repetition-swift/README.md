# Build a spaced repetition app in Swift and SwiftUI: First Steps

A few weeks ago, I started taking evening classes to learn the German language 🇩🇪 and so far I've been loving it. I have always loved learning new languages, acquiring new vocabulary and idioms. Some ten years ago, I learnt English organically by consuming materials in English (technical books, articles, and talks, movies with subtitles etc.) and letting my brain build an intuition of the language over a period of many months and years. Today, however, due to the fast paced nature of my German classes and my various professional and social obligations, I don't have as much of a time to dedicate to learning German organically. So I decided to turn to a proven and effective learning method called [Spaced Repetition](https://en.wikipedia.org/wiki/Spaced_repetition). In addition, I thought this would be the perfect opportunity to learn [SwiftUI](https://developer.apple.com/xcode/swiftui/) by building a spaced repetition app to aid in my learning process.

## What is spaced repetition?

[Spaced repetition](https://en.wikipedia.org/wiki/Spaced_repetition) is a learning method based on the fact that learning and retention are more effective when study sessions are spaced out over time: the pyschological [spacing effect](https://en.wikipedia.org/wiki/Spacing_effect).

Concretely, spaced repetition is a learning technique in which knowledge about a material is broken down into a set of meaningful pairs of questions and answers. For each pair, the question is written  on the front of a card and the answer is written at the back. The resulting cards are commonly called [flashcards](https://en.wikipedia.org/wiki/Flashcard). To review the materials, one simply has to go through every flashcard and try to answer the question on the front. Each card is reviewed repeatedly after a certain amount of time `t` has elapsed. For each card, `t` is computed through a combination of time (when was the last time I reviewed this card?) and the difficulty of the card (how many times did I get this card wrong?). 

This is a huge over-simplification, we will refine our definition and understanding of spaced repetition throughout our implementations in Swift.

## A basic spaced repetition program in Swift

One of the most efficient spaced repetition algorithms is called  [SuperMemo](https://help.supermemo.org/wiki/SuperMemo_Algorithm), created by [Piotr Woźniak](https://en.wikipedia.org/wiki/Piotr_Woźniak_(researcher)). SuperMemo is the basis of many spaced repetition softwares such as [Anki](https://apps.ankiweb.net), [Memrise](https://www.memrise.com), or ... [SuperMemo](https://www.supermemo.com/en).

In this article, we will start our journey of building a spaced repetition app by implementing the very first version of SuperMemo, [SM-0](http://super-memory.com/english/ol/beginning.htm#Algorithm). In subsequent articles, we will implement the various improvements introduced by newer versions of SuperMemo up to [SM-18](https://supermemo.guru/wiki/Algorithm_SM-18) (the latest version of SuperMemo at the time of writing this article).

## SM-0

The first version of SuperMemo is simple. Flashcards need to be grouped into decks of 20-40 cards with each deck being reviewed after an interval of time (in days) `I` computed with the following formula.

```shell
I(1) = 1
I(2) = 7
I(3) = 16
I(4) = 35
I(i) = I(i - 1) * 2, for i > 4 and I(i) being the interval to use after reviewing the deck i times.
```

This means that if I start learning a deck of German words today, I will review the deck tomorrow and a third time after 7 days. A fourth time after 16 days, a fifth time after 35 days, so on and so forth.

We start our implementation of *SM-0* in Swift by defining a type representing a flashcard.

```swift
struct Card: Equatable {
    let front: String
    let back: String
}
```

Then, we define a type to represent a deck. A `typealias` should be enough for now.

```swift
typealias Deck = [Card]
```

We will also need a type to model a repetition. A repetition is just a deck and the time at which that deck was reviewed.

```swift
struct Repetition {
    let deck: Deck
    let date: Date
}
```

Before moving forward, let's define some objects our program will need to perform its tasks. A `Database` to persist the repetitions the user performs, a `Notifier` to send notifications whenever it's time to repeat a specific deck, an `Input` to get the user inputs and an `Output` to present some information to the user.

```swift
protocol Database {
    func save(repetition: Repetition) -> Bool
    func repetitions(of deck: Deck) -> [Repetition]
}

protocol Notifier {
    func createNotification(
        withContent content: String, 
        triggeredAfter interval: TimeInterval, 
        action: @escaping () -> Void
    )
}

protocol Input {
    func getString(_ prompt: String) -> String
}

protocol Output {
    func printString(_ string: String)
}
```

Let's group these dependencies and make them available globally in a variable named [Current](https://www.pointfree.co/blog/posts/21-how-to-control-the-world).

```swift
struct Dependencies {
    let database: Database
    let notifier: Notifier
    let input: Input
    let output: Output
}

var Current = Dependencies(
    database: SomeDatabaseImplementation(),
    notifier: SomeNotifierImplementatioon(),
    input: SomeInputImplementation(),
    output: SomeOutputImplementation()
)
```

We can then proceed to implement the bulk of *SM-0*, the function to compute the interval of time after which a specific deck should be reviewed.

```swift
func nextInterval(for deck: Deck) -> TimeInterval {
    func recurse(_ repetitionCount: Int) -> Int {
        switch repetitionCount {
        case 1: return 1
        case 2: return 7
        case 3: return 16
        case 4: return 35
        default: return recurse(repetitionCount - 1) * 2
        }
    }
  
    let repetitions = Current.database.repetitions(of: deck)
    return recurse(repetitions.count) * 86400 // There are 86400 seconds in a day.
}
```

The next step would be to implement a function `review` allowing the user to review a specific deck.

```swift
func review(deck: Deck) {
    var remaining = deck
	
    while remaining.isEmpty == false {
        let cards = remaining.shuffled()
	    
        for card in cards {
            let userAnswer = Current.input.getString(card.front)
            if userAnswer == card.answer {
                remaining.removeAll(where: { $0 == card })
            }

            Current.output.printString(card.back)
        }
    }

    Current.database.save(repetition: Repetition(
        deck: deck,
        date: Date()
    )

    Current.notifer.createNotification(
        withContent: "\(deck.count) cards available to review now!",
        triggeredAfter: nextInterval(for: deck),
        action: { self.review(deck: deck) }
    )
}
```

This function does 3 things.

- First, the function loops through all the cards in the deck in random order. For each card, the user is prompted for its answer. The looping continues until the user has provided the correct answers to all the cards.
- Then, the function adds a record of the review in the database  by saving a `Repetition` value.
- And finally, the function creates a notification to be triggered at some appropriate time in the future. The user will be prompted to review the same deck after interacting with the notification.

As a final touch, we can group these functions into a top-level type `SuperMemo` ...


```swift
enum SuperMemo {
    static func review(deck: Deck) {
        var remaining = deck
	
        while remaining.isEmpty == false {
            let cards = remaining.shuffled()
	    
            for card in cards {
                let userAnswer = Current.input.getString(card.front)
                if userAnswer == card.answer {
                    remaining.removeAll(where: { $0 == card })
                }

                Current.output.printString(card.back)
            }
        }

        Current.database.save(repetition: Repetition(
            deck: deck,
            date: Date()
        )

        Current.notifer.createNotification(
            withContent: "\(deck.count) cards available to review now!",
            triggeredAfter: nextInterval(for: deck),
            action: { self.review(deck: deck) }
        )
    }

    private static func nextInterval(for deck: Deck) -> TimeInterval {
        func recurse(_ repetitionCount: Int) -> Int {
            switch repetitionCount {
            case 1: return 1
            case 2: return 7
            case 3: return 16
            case 4: return 35
            default: return recurse(repetitionCount - 1) * 2
            }
        }
  
        let repetitions = Current.database.repetitions(of: deck)
        return recurse(repetitions.count) * 86400 // There are 86400 seconds in a day.
    } 
}
```

Et voilà! We have the core of *SM-0* up and running. We can use it as follows.

```swift
SuperMemo.review(deck: germanIrregularVerbs)
```

Assuming I reviewed the German irregular verbs once already, this function call will aid me in reviewing them a second time and will create a third review notification that will be presented to me in 7 days.

## Next steps

You may have already noticed one of the major shortcomings of SM-0: *all the cards in a deck are reviewed together and at the same time*. In practice, some cards will be more difficult to master than others. For example, in my learning of German, I struggle with verbs such as **sein** while I find others such as **kommen** easy. It makes sense that I review the verbs I struggle with more frequently and the verbs I find easy less frequently.

To address that shortcoming, the version of SuperMemo (after SM-0) [SM-2](http://super-memory.com/english/ol/sm2.htm) introduces a new variable called the [Easiness Factor (E-Factor)](https://help.supermemo.org/wiki/Glossary:E-Factor). With *SM-2*, cards with the same *E-Factor* are grouped and reviewed together. In a next article, we will study how to compute the *E-Factor* and how it improves our basic SM-0 implementation. And in subsequent articles, we will look into building a beautiful and compelling user interface in Swift UI for SuperMemo.

Thanks for reading 👋.