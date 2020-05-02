
[1]: https://github.com/dv8fromtheworld/jda
[2]: https://github.com/kotlin/kotlinx.coroutines
[3]: https://github.com/MinnDevelopment/jda-reactor

[4]: https://github.com/MinnDevelopment/jda-ktx/tree/master/src/main/kotlin/dev/minn/jda/ktx/CoroutineEventManager.kt
[5]: https://github.com/MinnDevelopment/jda-reactor/tree/master/src/main/java/club/minnced/jda/reactor/ReactiveEventManager.java

# jda-ktx

Collection of useful Kotlin extensions for [JDA][1].
Great in combination with [kotlinx-coroutines][2] and [jda-reactor][3].

## Examples

The most useful feature of this library is the [CoroutineEventManager][4] which adds the ability to use
suspending functions in your event handlers.

```kotlin
val jda = JDABuilder.createLight("token")
    .injectKTX() // Sets the coroutine event manager
    .build()

// This can only be used with the CoroutineEventManager
jda.listener<MessageReceivedEvent> {
    val guild = it.guild
    val channel = it.channel
    val message = it.message
    val content = message.contentRaw

    if (content.startsWith("!profile")) {
        // Send typing indicator and wait for it to arrive
        channel.sendTyping().await()
        val user = message.mentionedUsers.firstOrNull() ?: run {
            // Try loading user through prefix loading
            val matches = guild.retrieveMembersByPrefix(content.substringAfter("!profile "), 1).await()
            // Take first result, or null
            matches.firstOrNull()
        }
        
        if (user == null) // unknown user for name
            channel.sendMessageFormat("%s, I cannot find a user for your query!", it.author).queue()
        else // load profile and send it as embed
            channel.sendMessageFormat("%s, here is the user profile:", it.author)
                .embed(profile(user)) // custom profile embed implementation
                .queue()
    }
}
```

### Coroutine Extensions

I've added a few suspending extension functions to various JDA components.
None of these extensions require the `CoroutineEventManager`!

To use `await<Event>` and `awaitMessage` the event manager must support either `EventListener` or `@SubscribeEvent`,
the [`ReactiveEventManager`][5] and [`CoroutineEventManager`][4] both support this.

```kotlin
// Await RestAction result
suspend fun <T> RestAction<T>.await()
// Await Task result (retrieveMembersByPrefix)
suspend fun <T> Task<T>.await()
// Await specific event
suspend fun <T : GenericEvent> JDA.await(filter: (T) -> Boolean = { true })
// Await specific event
suspend fun <T : GenericEvent> ShardManager.await(filter: (T) -> Boolean = { true })
// Await message from specific channel (filter by user and/or filter function)
suspend fun MessageChannel.awaitMessage(author: User? = null, filter: (Message) -> Boolean = { true }): Message
// Coroutine iterators for PaginationAction
suspend fun <T, M: PaginationAction<T, M>> M.produce(scope: CoroutineScope = GlobalScope): ReceiverChannel<T>
// Flow representation for PaginationAction
suspend fun <T, M: PaginationAction<T, M>> M.asFlow(scope: CoroutineScope = GlobalScope): Flow<T>
```