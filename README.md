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
