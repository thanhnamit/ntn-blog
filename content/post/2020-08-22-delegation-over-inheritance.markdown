---
featured: true
title: "How to not write inheritance code in Kotlin"
tags:
  [
    testable,
    composition,
    delegation,
    kotlin,
    lambda,
    extension-function
  ]
date: 2020-08-21T14:24:10+10:00
lastmod: 2020-08-21T14:24:10+10:00
draft: false
---

Let'say you write couple of classes and found that there are common functions
can be shared. A natural tendency is to create a parent class and inherit from it.
But with Kotlin you can do better with just interface.

In this example, I have a number of Adapter classes which wrap synchronous JDBC
calls with thread pool

```kotlin
@Component
class ReactiveUserRelationalAdapter(
        @Autowired val repo: UserRepository,
        @Autowired @Qualifier("jdbcScheduler") val jdbcScheduler: Scheduler
): ReactiveUserService {

    override fun findByUserUid(userUid: String): Mono<UserEntity> {
        return Mono.fromSupplier {
            repo.findByUserUid(userUid)
        }.publishOn(jdbcScheduler)
    }

    override fun save(entity: UserEntity): Mono<UserEntity> {
        return Mono.fromSupplier {
            repo.save(entity)
        }.publishOn(jdbcScheduler)
    }
}
```

The common thing here is the part

```kotlin
Mono.fromSupplier {
    // blocking call
}.publishOn(jdbcScheduler)
```

What we can do to remove this is to extract it as a base class with a function
to wrap the blocking call around. But there is a better way. We can define 
an interface and create an extension function. The interface could be named anything
but generally should describe the intention of its functionality, keep it short
and to the point.


```kotlin
interface AsyncJdbc {
    val jdbcScheduler: Scheduler
}
```

Kotlin allows you to declare a variables

Next, you can create an extension function. That says any classes implement AsyncJdbc
interface can invoke this function. Here we pass a lambda in `asyncJdbc` so this function
can invoke arbitary code. You can keep this function in same interface file.

```kotlin
fun <T, R : AsyncJdbc> R.asyncJdbc(supplier: () -> T): Mono<T> {
    return Mono.fromSupplier(supplier).publishOn(jdbcScheduler)
}
```

With the interface and the extension function, we can rewrite all our previous classes
like:

```kotlin
@Component
class ReactiveUserRelationalAdapter(
        @Autowired val repo: UserRepository,
        @Autowired @Qualifier("jdbcScheduler") override val jdbcScheduler: Scheduler
): ReactiveUserService, AsyncJdbc {

    override fun findByUserUid(userUid: String): Mono<UserEntity> {
        return asyncJdbc { repo.findByUserUid(userUid) }
    }

    override fun save(entity: UserEntity): Mono<UserEntity> {
        return asyncJdbc { repo.save(entity) }
    }
}
```

it now removes a fair bit of boilerplate code and heck, I don't use inheritance here.

Have fun!