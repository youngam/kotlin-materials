# kotlin-materials

## Links
* [Getting started with android](https://kotlinlang.org/docs/tutorials/kotlin-android.html)
* [“Why you should totally switch to Kotlin”](https://medium.com/@magnus.chatt/why-you-should-totally-switch-to-kotlin-c7bbde9e10d5)
* [Coroutines “A first walk into Kotlin coroutines on Android” @lime_cl](https://android.jlelse.eu/a-first-walk-into-kotlin-coroutines-on-android-fe4a6e25f46a)

## Coroutine

### Unconfined vs confined dispatcher
 
The [Unconfined] coroutine dispatcher starts coroutine in the caller thread, but only until the
first suspension point. After suspension it resumes in the thread that is fully determined by the
suspending function that was invoked. Unconfined dispatcher is appropriate when coroutine does not
consume CPU time nor updates any shared data (like UI) that is confined to a specific thread. 

On the other side, [context][CoroutineScope.context] property that is available inside the block of any coroutine 
via [CoroutineScope] interface, is a reference to a context of this particular coroutine. 
This way, a parent context can be inherited. The default context of [runBlocking], in particular,
is confined to be invoker thread, so inheriting it has the effect of confining execution to
this thread with a predictable FIFO scheduling.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // not confined -- will work with main thread
        println("      'Unconfined': I'm working in thread ${Thread.currentThread().name}")

        withContext(newSingleThreadContext("MyOwnThread")) {delay(500)}

        println("      'Unconfined': After delay in thread ${Thread.currentThread().name}")
    }
    jobs += launch(coroutineContext) { // context of the parent, runBlocking coroutine
        println("'coroutineContext': I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("'coroutineContext': After delay in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-02.kt)

Produces the output: 
 
```text
      'Unconfined': I'm working in thread main
'coroutineContext': I'm working in thread main
      'Unconfined': After delay in thread MyOwnThread
'coroutineContext': After delay in thread main
```
 
So, the coroutine that had inherited `context` of `runBlocking {...}` continues to execute in the `main` thread,
while the unconfined one had resumed in the scheduler thread that [delay] function is using.

### Concurrency and threads

Each individual coroutine, just like a thread, is executed sequentially. It means that the following kind 
of code is perfectly safe inside a coroutine:

```kotlin
launch(CommonPool) { // starts a coroutine
    val m = mutableMapOf<String, String>()
    val v1 = someAsyncTask1().await() // suspends on await
    m["k1"] = v1 // modify map when resumed
    val v2 = someAsyncTask2().await() // suspends on await
    m["k2"] = v2 // modify map when resumed
}
```

You can use all the regular single-threaded mutable structures inside the scope of a particular coroutine.
However, sharing mutable state _between_ coroutines is potentially dangerous. If you use a coroutine builder
that installs a dispatcher to resume all coroutines JS-style in the single event-dispatch thread, 
like the `Swing` interceptor shown in [continuation interceptor](#continuation-interceptor) section,
then you can safely work with all shared
objects that are generally modified from this event-dispatch thread. 
However, if you work in multi-threaded environment or otherwise share mutable state between
coroutines running in different threads, then you have to use thread-safe (concurrent) data structures. 

Coroutines are like threads, albeit they are more lightweight. You can have millions of coroutines running on 
just a few threads. The running coroutine is always executed in some thread. However, a _suspended_ coroutine
does not consume a thread and it is not bound to a thread in any way. The suspending function that resumes this
coroutine decides which thread the coroutine is resumed on by invoking `Continuation.resume` on this thread 
and coroutine's interceptor can override this decision and dispatch the coroutine's execution onto a different thread.


