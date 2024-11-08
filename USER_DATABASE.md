# User Database
This MongoDB database stores user data and session data.

## Payloads
### User

```kotlin
class ChatUser(
  val userId: String,
  val username: String, // TODO add indexes
  
  val passwordHash: String, // hashed password using HMAC-SHA512, 
  val salt: String, // salt used for hashing, different salt for each user
  
  val lastLogin: Date,
) : MongoDocument()
```
// TODO add indexes