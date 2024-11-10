# Chat Database

## Payloads

### Channel

```kotlin
class Channel(
  val _id: String, // channelId, random unique ID (most suited for shardKey)
  val members: Set<String>,
  val lastOpened: Date, // Only used for analytics to track user engagement
  val lastMessageSentDate: Date,
) : MongoDocument()
```

Indexes:
- `{userIds: 1, lastMessageSentDate: -1}`: to fetch all channels for a specific user

### Message

```kotlin
class Message(
  val _id: String, // messageId, random unique ID, unique across all channels
  val channelId: String, (most suited for shardKey, so all messages are on the same shard)
  val authorId: String, 
  val sendDate: Date,
  val sortScore: Long, // Currently just the UNIX timestamp when the message was posted
  val deleted: Boolean, // If true, the message will be hidden on the client side
) : MongoDocument() {
  class Content(
    val textContent: String? = null,
  )
}
```

Indexes:
- `{channelId: 1, sortScore: -1}`: to fetch all messages in a channel, sorted by sendDate descending
