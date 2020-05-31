---
featured: true
title: "Kotlin - a pragmatic multiplatform language"
tags: [coding, kotlin, java]
date: 2020-05-31T14:24:10+10:00
lastmod: 2020-05-31T14:24:10+10:00
draft: false
---

In 2019, Google announced Jetbrain's Kotlin as the first-class language for Android development. Since then, Kotlin has evolved to be more than just a language for Android. On kotlinlang.org, it claims to be safe, concise, expressive and cross-platform. To find out how Kotlin can improve my day-to-day workflow and productivity, I rewrote several Java programs and cherry-pick some useful features that I really appreciate.

## Handling Null Safety
NPE is one of the most frequently encountered bugs in JVM world. Since Java 8, `Optional` was
introduced to deal with null reference, but Kotlin introduces a more concise approach. The interface
below has a method that can return an optional result.

```kotlin
interface UserRepository : AutoCloseable {
    ...
    fun get(userId: String): User?
    ...
}
```


Any implementation of the interface must guarantee that `userId` is not null. The syntax `User?` means
that the function can return null or non-nullable object. A sample implementation is:

```kotlin
class InMemoryUserRepository : UserRepository {
    private val userIdToUser: MutableMap<String, User> = mutableMapOf()

    override fun get(userId: String): User? {
        return userIdToUser[userId]
    }
}
```


What the client code looks like in this case? You must handle the possible nullable case otherwise the code
won't compile. You can also use safe call operator `?.` for chaining nested calls and Elvis operator `?:`
to simplify null check statement.

```kotlin
fun onFollow(follower: User, otherUserId: String): FollowStatus {
    return userRepository.get(otherUserId)?.let { userRepository.follow(follower, it) } ?: FollowStatus.INVALID_USER
}
```


More on Null-safety at https://kotlinlang.org/docs/reference/null-safety.html

## Separation of Immutability and Mutability
Mutating objects in runtime is error-prone. Applying Immutability in programming can improve performance, stability
and predictability of software, it is especially important when working with functional / reactive programs. Fortunately, Kotlin 
clearly separates mutable objects and immutable ones using `val`, `var` keywords for variable declarations. This class has both
mutable and immutable properties.

```kotlin
class BusinessRuleEngine(val facts: Facts) {
    var actions: MutableList<Action> = mutableListOf()

    fun addAction(action: Action) {
        actions.add(action)
    }

    fun replaceActions(actions: MutableList<Action>) {
        this.actions = actions
    }

    fun run() {
        actions.forEach { it.execute(facts) }
    }
}
```

In the class above we create a custom rule engine with unmodifiable facts `val facts: Facts`
while declaring a mutable list of actions, allowing the engine to dynamically add new actions or even
replace the whole `actions` list. In my option, it is good practice to keep using `val` for all variables
until you actually need `var` keyword.


## No checked exception handling
Checked exceptions (exceptions that inherit Exception class in Java such as FileNotFoundException) require client code
to handle all possible exceptional cases. Same as C# and Ruby, Kotlin compiler doesn't enforce client code to catch checked exceptions, so that developers are responsible for handling them. This approach has both pros and cons and it depends on situations. Having too many specific checked exceptions leads to code cluttering and impacting developer's productivity (imagine you write Reactive/Stream based codes and have to handle a method which throws multiple exceptions). However, ignoring exception is a bad practice.

## Useful data class
In Java you can use `Lombok` library to annotate a class as data class. Kotlin supports this ootb and 
the compiler automatically creates `equals`, `hashcode`, `toString` and `copy` methods. Data class is 
suitable for implementing Value Object or classes without behavior.

```kotlin
data class SummaryStatistics(
    val sum: Double,
    val max: Double,
    val min: Double,
    val average: Double
)
```


## Built-in Singleton pattern
Instead of reimplementing Singleton pattern, Kotlin provides an easy way
to create a singleton and guarantee it is the only instance available at runtime.
For example, to group global constants and make it available elsewhere.

```kotlin
object DocumentTypes {
    const val INVOICE_TYPE = "invoice"
    const val LETTER_TYPE = "letter"
    const val REPORT_TYPE = "report"
    const val IMAGE_TYPE  = "jpg"
}
```

The object DocumentTypes is lazily initialised at the first access and it is thread-safe.
Its constants can be accessed as `DocumentTypes.INVOICE_TYPE`


## Rich and concise Collections library
There are many flavours of collections for Java such as JDK, Eclipse, Guava and Apache Collections.
Kotlin standard library provides easy to use and a few more features than native Java's Collections.
Collection types such as List, Set, Map are provided with both mutable and read-only versions. Manipulation
is done through various extension functions, for examples:

- Transformation: map, mapKeys, mapValues, zip, unzip, associate, flatten, flatMap, join
- Filtering: filter, filterNot, partition, predicates (any, none, all)
- Grouping: groupBy, fold and reduce, eachCount, aggregate
- Ordering: sorted, sortedBy, reversed, shuffled
- Aggregation: sum, min, max, count, average, fold and reduce

In the example below, we filter transactions from a bank statement and group by month then description.

```kotlin
class BankStatementProcessor(val trans: List<BankTransaction>) {
    fun getTransactionsGroupByMonthAndDesc(): List<BankTransaction> {
        return trans.groupBy { it.date.month }
                    .mapValues { it.value.groupBy { t -> t.description }.values.flatten() }
                    .values.flatten()
    }
}
```

## Coroutines
Generally, asynchronous programming in JVM is done through native threads or third party libraries like RxJava or Project Reactor. Kotlin offers a new way to write concurrent programs: Coroutines. Using a special data structure called Continuations, Coroutines is co-operative multitasking instead of pre-emptive multitasked like native threads. From runtime perspective, the program benefits from leak-free, lock-free and more efficient use of computing resources. With coding style, the coroutine code is sequential (no callback hell), less pervasive, easy to understand and refactor, hence reduce cognitive overhead for coders.

In order to write concurrent code in Kotlin, developers need to identify suspendable functions so the runtime can suspend and resume them in the middle of its executions. Examples of suspendable functions are long-running, I/O intensive ones. This design enables the runtime to schedule the execution of multiple `suspend` functions concurrently. In the example below, the function `getTotalByCountrySlug` is a blocking one.

```kotlin
class CovidStatusClient: CovidStatusApi {
    private val statusUrl = "https://api.covid19api.com/total/country/"

    override fun getTotalByCountrySlug(code: String): CountryCovidStatus? {
        return Klaxon().parseArray<CountryCovidStatus>(URL(statusUrl + code).readText())?.lastOrNull()
    }
}
```

This code below creates an instance of the client and invoke the blocking method asynchronously. Notice the new language elements
`runBlocking {}`, `async() {}` and `await()`

```kotlin
fun main() = runBlocking {
    val format = "%-30s%-20s%-20s%-20s"
    println(String.format(format, "Country", "Confirmed", "Deaths", "Active"))

    val client = CovidStatusClient()
    val countryCodes = listOf<String>("united-states", "russia", "australia", "singapore", "thai-")
    val covidStatuses: List<Deferred<CountryCovidStatus?>> = countryCodes.map { code ->
        async(Dispatchers.IO + SupervisorJob()) {
            client.getTotalByCountrySlug(code)
        }
    }

    for (status in covidStatuses) {
        try {
            val info = status.await()
            println(String.format(format, info?.country, info?.confirmed, info?.deaths, info?.active))
        } catch (ex: Exception) {
            println("Error: ${ex.message?.substring(0..30)}")
        }
    }
}
```

- `runBlocking` block force the main() function to block and wait for asynchronous executions to finished.
- `async()` function executes the method concurrently. To run in parallel, `async` accepts an optional parameter `Dispatchers.IO` as CoroutineContext which basically tells the runtime to execute the code in a separate IO thread pool. Optional parameter `SupervisorJob` has a special role to stop the cancellation of parent coroutine in case child coroutine failed to execute.
- `await()` is called on Deferred object `Deferred<CountryCovidStatus?>` to eventually get the actual result

There are a lot more about Coroutines that is explained at https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html

## Type-safe builders for creating DSL
Type-safe builder is what make Kotlin DSL compatible (DSL - Domain-specific language). Great examples are TornadoFX for UI apps and Ktor framework for writing server-client apps. TornadoFX adopts Kotlin DSL on top of JavaFX, result in very concise and expressive language to build UI layouts.

```kotlin
class AnalyzerView: View() {
    override val root = borderpane {
        top<MenuView>()
        center<TabpaneView>()
    }
}

class MenuView: View() {
    override val root = menubar {
        menu("File") {
            item("New")
            item("Save As")
            item("Quit")
        }
    }
}

class TabpaneView: View() {
    val controller: UIController by inject()
    override val root = tabpane {
        tab("Monthly Statistics") {
            linechart("Monthly Statistics", CategoryAxis(), NumberAxis()) {
                multiseries("Income", "Expense") {
                    controller.getMonthlySummary().forEach {
                        data(it.month, it.income, it.expense)
                    }
                }
            }
        }
        ...
    }
}
```

I only touch the surface here, but there are more in Kotlin that worth discovering. Kotlin ecosystem is expanding
rapidly. Popular frameworks support the language (Spring, Gradle, Spark, Vert.x...) and many have been created from
scratch with Kotlin (Ktor, RxKotlin, MockK...). 

Executable source code at: https://github.com/thanhnamit/rwsd-in-kotlin



