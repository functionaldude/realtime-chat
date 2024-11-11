# User Database
This MongoDB database stores user data and session data.

## Payloads
### User

```kotlin
class ChatUser(
  val _id: String, // userId, random unique ID (most suited for shardKey)
  val username: String,
  
  val passwordHash: String, // hashed password using HMAC-SHA512, 
  val salt: String, // salt used for hashing, different salt for each user
  
  val lastLogin: Date,
) : MongoDocument()
```

Indexes:
- `{username: 1}`: to fetch a specific user by username
- `{username: "text"}`: search for a specific user by text search

### SessionToken

```kotlin
class SessionToken(
  val _id: String, // sessionToken, random unique ID
  val userId: String? = null, // userId of the user that owns this session, null if unauthenticated (most suited for shardKey)
  val creationDate: Date,
  val lastUsed: Date,
  val expiryDate: Date,
) : MongoDocument()
```
