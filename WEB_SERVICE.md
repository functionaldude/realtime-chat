# Web Service
This is a simple web service that serves the React app HTML & JS files and handles file uploads.
This service is stateless, it can be scaled horizontally without any limitation.

## Endpoints

### WebApp (GET)
This endpoint serves an HTML file that contains the React app.

#### Session Token
When a user loads the React app, an "unauthenticated" anonymous `sessionToken` is created on the backend.

#### CSRF Protection
Additionally, it also contains the meta tag for CSRF protection:
```html
<meta name="csrf-token" content="CSRF_TOKEN">
```

### Get presigned upload URL (POST)
This endpoint returns a presigned URL that can be used to upload a file to the Cloudflare R2 object storage.
The current user's `sessionToken` is used to authenticate the request (via cookie or "session-token" header).

Request:
```kotlin
class GetPresignedUploadUrlRequest(
    val filename: String,
    val channelId: String,
)
```

The server checks if the user is member of the channel, if not, it returns an error.
The filename is sanitized and checked for allowed file types.

Response:

```kotlin
class GetPresignedUploadUrlResponse(
  val uploadUrl: String,
  val downloadUrl: String
)
```