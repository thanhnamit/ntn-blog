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

When you write a couple of classes and just realise that there are functionalities
that can be shared, a tendency is to create a parent / abstract class or define
an utilitity class. But inheritance and global utilities are often hard to test, 
failed to encapsulate functionality and highly coupled. In practices, using interface 
offers a better design.

In this instance, I have a number of Adapter classes (in Spring) which wrap synchronous JDBC
calls with a thread pool.

```kotlin
@Component
class ReactiveUserRelationalAdapter(
        @Autowired val repo: UserRepository,
        @Autowired @Qualifier("jdbcScheduler") val jdbcScheduler: Scheduler
): ReactiveUserService {

    val logger: Logger = LoggerFactory.getLogger(ReactiveUserRelationalAdapter::class.java)

    override fun findUserById(userId: String): Mono<UserEntity> {
        return Mono.fromSupplier {
            repo.findUserById(userId)
        }.publishOn(jdbcScheduler)
    }

    override fun save(entity: UserEntity): Mono<UserEntity> {
        return Mono.fromSupplier {
            repo.save(entity)
        }.publishOn(jdbcScheduler)
    }
}
```

The common things here are these parts

```kotlin
Mono.fromSupplier {
    // blocking call
}.publishOn(jdbcScheduler)
```

and

```kotlin
val logger: Logger = LoggerFactory.getLogger(ReactiveUserRelationalAdapter::class.java)
```

As mentioned, what we can do to remove the duplicate codes is to extract it into a base class, 
but there is a better way. With Kotlin, we can define an interface and an extension function. 
The interface could be named anything but generally it should describe the intention 
of its functionality, keep it short and to the point.


```kotlin
interface AsyncJdbc {
    val jdbcScheduler: Scheduler
}

fun <T, R : AsyncJdbc> R.asyncJdbc(supplier: () -> T): Mono<T> {
    return Mono.fromSupplier(supplier).publishOn(jdbcScheduler)
}
```

Any class that implements AsyncJdbc interface can invoke the extension function 
`asyncJdbc`. Here we pass a lambda to `asyncJdbc` so the function
can invoke arbitrary code.

The previous adapter now can be written as

```kotlin
@Component
class ReactiveUserRelationalAdapter(
        @Autowired val repo: UserRepository,
        @Autowired @Qualifier("jdbcScheduler") override val jdbcScheduler: Scheduler
): ReactiveUserService, AsyncJdbc {

    override fun findUserById(userId: String): Mono<UserEntity> {
        return asyncJdbc { repo.findUserById(userId) }
    }

    override fun save(entity: UserEntity): Mono<UserEntity> {
        return asyncJdbc { repo.save(entity) }
    }
}
```

It now removes a fair bit of boilerplate code and heck, I don't use inheritance here.

The same thing can be done with the logger

```kotlin
interface Loggable

fun <R: Loggable> R.logger(): Lazy<Logger> {
    return lazy { LoggerFactory.getLogger(this.javaClass.name) }
}
```

Finally, our class looks like

```kotlin
@Component
class ReactiveUserRelationalAdapter(
        @Autowired val repo: UserRepository,
        @Autowired @Qualifier("jdbcScheduler") override val jdbcScheduler: Scheduler
): ReactiveUserService, AsyncJdbc, Loggable {
    val LOG by logger()
    
    override fun findUserById(userId: String): Mono<UserEntity> {
        LOG.debug("find user by id...")
        return asyncJdbc { repo.findUserById(userId) }
    }

    override fun save(entity: UserEntity): Mono<UserEntity> {
        LOG.debug("save user...")
        return asyncJdbc { repo.save(entity) }
    }
}
```

Have fun!