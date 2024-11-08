# Authentication Service
This is the API service that allows users to register & log-in.

## Endpoints

### Login (POST)
This endpoint allows users to log in. It takes the `username` and `password` as input and returns a `sessionToken`. 
Additionally, it also sets a cookie with the `sessionToken` for future requests, the cookie is `HttpOnly` and `Secure`, and `SameSite=Strict`.

Request:
```kotlin
class LoginRequest(
    val username: String,
    val password: String,
    val appCheckToken: String? // set by the native appCheck API, null if login is coming from the web app
)
```

If the login is coming from the web app, this endpoint also checks the `csrfToken` header to prevent [CSRF attacks](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html#hmac-based-token-pattern).
If the login is coming form the native app, this endpoint verifies if the `appCheckToken` is valid using the [App Attest API](https://firebase.google.com/docs/app-check/custom-resource-backend#other).

Password is hashed and compared with the stored hash.

If all checks pass, the `userId` is set in the `sessionToken`, from then on the session is authenticated.
The session token is returned, and in case of the web client also set as a cookie.
Response:
```kotlin
class LoginResponse(
    val sessionToken: String?, // null if login failed
    val errorMessage: String? // null if login successful, show error message to user
)
```

Failed login attempts are logged and rate-limited to prevent brute-force attacks.

### Register (POST)

This endpoint allows users to register. It takes the `username` and `password` as input.

Request:
```kotlin
class LoginRequest(
    val username: String,
    val password: String,
)
```

Username is checked for uniqueness, password is hashed and stored in the database.

If all checks pass, user is created and saved in the database.
Response:
```kotlin
class RegisterResponse(
    val success: Boolean,
    val errorMessage: String? // null if register successful, show error message to user
)
```

## Password hashing and storage 

Passwords are stored in the [ChatUser](USER_DATABASE.md#user) payload in the database.
They are hashed using HMAC-SHA512 with a unique salt for each user. The HMAC key is fetched from the environment.