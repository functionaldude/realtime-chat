# Messaging Service
This service handles the messaging (sending and receiving) between users.
It exposes a single Socket IO endpoint that clients can connect to. 
This service is stateful, but it can be scaled horizontally.

## Basic Concept
| entity                              | explanation                                                                                                                                                                     |
|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [ChatUser](USER_DATABASE.md#user)   | Represents a single user                                                                                                                                                        |
| [ChatClient](USER_DATABASE.md#user) | Represents a connected client of a ChatUser, it contains a `ChatUser` and a `SocketIOClient`                                                                                    |
| [Channel](CHAT_DATABASE.md#channel) | Represents a chat channel where two users can message. Currently only two users per channels are supported.                                                                     |
| [Message](CHAT_DATABASE.md#message) | Represents a single message, written by one user, posted into a channel. A message can only belong to one channel and one user. It can also have text or file content, or both. |
| `MESSAGE_LOAD_BATCH_SIZE`           | The number of messages that are loaded on opening a channel, or by scrolling                                                                                                    |

## Socket IO
Socket IO is used for wrapping WebSockets, it allows for real-time, bidirectional and event-based communication between clients and the server.
On the server the Java Socket IO server library is used, on the client the JS and native Socket IO libraries.

There is middleware installed (reverse proxy) in the infrastructure that terminates the SSL connection and forwards the unencrypted 
traffic to the server, it also sets a cookie for sticky sessions. Because of this, the server can be stateful.

The server keeps track of all currently connected clients (=ChatClient), one [ChatUser](USER_DATABASE.md#user) can have multiple connections, for example from a web client and a mobile client.
This is done via two maps:

```kotlin
val connectedClients = ConcurrentHashMap<String, ChatClient>() // Map of all connected clients, key is the clientId
val connectedUsers = ConcurrentHashMap<String, MutableSet<String>>() // Map of all connected clients per user, key is the userId, value contains set of clientId's
```

The `connectedClients` can be used to directly access a client, the `connectedUsers` map is used to find connected 
clients of a specific user to send messages to them in real-time.

## Authentication
The messaging service uses the `sessionToken` from the [Authentication Service](AUTHENTICATION_SERVICE.md) to authenticate users.
It is either passed as a cookie, or the dedicated "session-token" header. The `sessionToken` contains the `userId` of the user.

If the `sessionToken` is valid and authenticated, the server creates a new ChatClient for the user adds it to the map of connected clients.
Every ChatClient has a unique `clientId` that is used to identify the client which is read from it's `SocketIOClient`.

The Client can also send the `oldId` parameter in the `connect` event, this is used to reconnect to an existing ChatClients. 
In this case, the server will update the `SocketIOClient` of the existing ChatClient with the new one, instead of creating a new ChatClient.

## Basic use-case: User1 messages User2
1. User1 searches for User2
2. User1 taps on User2 in the serach results -> channel is created between User1 and User2
3. User1 sends a message to User2 -> message is saved in the channel
4. If User2 is online, User2 receives the message in real-time

## Client Commands
Client commands are sent by the client and answered by the server.

When a `ClientCommand` contains a `channelId`, the server checks if the user is member of the channel, 
if not, it returns an error.

### Get channels
With this command, a user can get all channels they are part of. They are sorted by the last message sent date, 
so the most recent channels are on top.

Request command:
```kotlin
class GetChannelsRequest(): ClientCommand()
```

Response command:
```kotlin
class GetChannelsResponse(
  val channels: List<Channel>,
): ServerCommand() {
  class Channel(
    val channelId: String,
    val username: String, // The other user's username,
    val sortScore: Long, // Sort by this descending on client
  )
}
```

### Search for user
With this command, a user can search for another user by username.
Request command:
```kotlin
class SearchUserRequest(val searchTerm: String): ClientCommand()
```

Each keypress fires a new `SearchUserRequest` command to the server (throttled by a debouncing function), but the server only responds 
when `searchTerm.count() >= 3` to prevent too many results. 

Response command:
```kotlin
class SearchUserResponse(
  val matches: List<ChatUserSearchMatch>,
  val searchTerm: String, // Send search term back to client for easier UI handling -> client ignores all responses that don't equal the search term
): ServerCommand() {
  class ChatUserSearchMatch(
    val userId: String,
    val username: String,
    val online: Boolean
  )
}
```

The server uses the MongoDB text index on the `username` field to search for the user, depending on the load, 
this can be later offloaded to a dedicated ellasticsearch instance.

`SearchUserResponse.matches` are limited to 50 results to save resources.

### Open (or create) channel
With this command, a user can open channel with another user. If the channel doesn't exist yet, it is automatically created. 

Request command:
```kotlin
class SubscribeChannelWithTap(
  val channelId: String? = null, 
  val userId: String? = null,
  val minMessageSortScore: Long? = null, // Sort score of the oldest message in the channel, null if client has no messages from this channel
  val maxMessageSortScore: Long? = null, // Sort score of the youngest message in the channel, null if client has no messages from this channel
): ClientCommand()
```
Note: Either `channelId` or `userId` must be set, but not both.
The client also sends `minMessageSortScore` and `maxMessageSortScore` to the server to determine which messages 
to load (see [Subscription maps](#subscription-maps)) or `null` if client has no messages in local storage for this channel.

The server loads `MESSAGE_LOAD_BATCH_SIZE` messages from the database (sorted descending by `sortScore`).
The server also creates an entry in the [subscription map](#subscription-maps) for the client, so it can receive 
updates for this channel. The loaded messages are used to set the `minMessageSortScore`, the `maxMessageSortScore` 
is `Long.MAX_VALUE` to receive future updates.

Once the messages are loaded, they are sent to the client via an [UpdateMessages](#update-messages) command.

### Scroll upwards in channel
With this command, a user can load more messages from a channel, the client sends it when the user scrolls upwards in the chat view.

Request command:
```kotlin
class SubscribeChannelWithScroll(
  val channelId: String, 
  val minMessageSortScore: Long, // Sort score of the oldest message in the channel
  val maxMessageSortScore: Long, // Sort score of the youngest message in the channel
): ClientCommand()
```

The server loads `MESSAGE_LOAD_BATCH_SIZE` messages from the database (sorted descending by `sortScore`), but with the following search query: 
```json
{
  "sortScore": {
    "$lte": "minMessageSortScore"
  }
}
```
This way we get the next batch of messages that are older than the oldest message in the client's local storage.

The server also updates the `minMessageSortScore` in the [subscription map](#subscription-maps) for the client, 
so it can receive real-time updates those older messages.

Once the messages are loaded, they are sent to the client via an [UpdateMessages](#update-messages) command.

### Send message
With this command, a user can send a new message to a channel.

Request command:
```kotlin
class NewMessage(
  val channelId: String,
  val content: Content,
): ClientCommand() {
  class Content(
    val text: String? = null,
    val fileUrl: String? = null,
  )
}
```

The user can either provide a text content or the `downloadUrl` of a file or both. In case of a file, the client
calls the [get presigned upload URL](WEB_SERVICE.md#get-presigned-upload-url-post) endpoint to upload the 
file and get the `downloadUrl`.

The server will respond with an [UpdateMessages](#update-messages) command to all clients (including the one that 
sent the `NewMessage` command) subscribed to the channel, [more details here](#channel-changes).

### Delete message
With this command, a user can delete an existing message in a channel.

Request command:
```kotlin
class DeleteMessage(
  val messageId: String,
): ClientCommand()
```

The server laods the message and verifies if the user is the author of the message, if yes, the message marked as deleted.

The server will respond with an [UpdateMessages](#update-messages) command to all clients (including the one that
sent the `NewMessage` command) subscribed to the channel, [more details here](#channel-changes).

## Server Commands
These command are sent by the server to the client, they are not always triggered by the client.

### Update Messages
This command instructs a client to "upsert" a message in the local database, and update the UI accordingly.

```kotlin
class UpdateMessages(
  val channelId: String,
  val messages: List<ClientMessage>,
  val deleteExistingMessages: Boolean = false, // If true, delete all existing messages in the channel before inserting the new ones
): ServerCommand() {
  class ClientMessage(
    val messageId: String,
    val author: String, // The author's username
    val owned: Boolean, // True if message was sent by current user
    val date: Date,
    val sortScore: Long, // Sort by this descending on client
    val content: Content? = null, // null if message was deleted, hide it in the UI. When upserting in the db, the content will be deleted (-> set to null)
  ) {
    class Content(
      val text: String? = null,
      val fileUrl: String? = null,
    )
  }
}
```

When transforming a [Message](CHAT_DATABASE.md#message) to a `ClientMessage`, the server checks if the message 
was deleted, if yes the `content` is set to `null`, so the client can hide it in the UI, and delete the content from it's local storage.

## Subscription maps

The server keeps a map of all active subscriptions for each ChatClient, to send [UpdateMessages](#update-messages) 
commands if a new [Message](CHAT_DATABASE.md#message) is posted in a channel, or a Message was deleted or changed.

```kotlin
class ChannelSubscription(
  val channelId: String,
  val minMessageSortScore: Long,
  val maxMessageSortScore: Long, // Long.MAX_VALUE can be used to subscribe to all future messages
)

typealias ClientSubMap = ConcurrentHashMap<ChatClient, ChannelSubscription>
val channelSubscriptions = ConcurrentHashMap<String, ClientSubMap>() // Map of all active subscriptions per client, key is the channelId
```

A `ChannelSubscription` is created when a client opens a channel, the `minMessageSortScore` and `maxMessageSortScore` can 
be "expanded" by scrolling upwards (=sending [SubscribeChannelWithScroll](#scroll-upwards-in-channel) commands).  

## Channel changes
There are currently two types of changes that can happen in a channel. 
1. A new message is posted
2. An existing message is deleted

These changes happen in the [chat database](CHAT_DATABASE.md) first, then clients can be notified, but only if they 
have an active subscription range for the channel which includes the changed message's `sortScore`.

The `channelSubscriptions` is a local map, every server instance has it's own map, the maps are not synchronized, 
so every server instance is responsible for it's own clients.

### Redis PubSub
On start, every server instance subscribes to the Redis channel "channelUpdates". They can then push the 
following update notification:

```kotlin
class ChannelUpdate(
  val channelId: String,
  val messageId: String,
  val messageSortScore: Long,
)
```

When a server instance receives a `ChannelUpdate` message, it checks if the 
channel is in the `channelSubscriptions` map, if yes, then it checks if there are any clients that 
have a `ChannelSubscription` with the [Message](CHAT_DATABASE.md#message)'s `sortScore` in the range.
If yes, the message is loaded and sent via an [UpdateMessages](#update-messages) command to the client.

