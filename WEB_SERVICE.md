# Web Service

This is a simple web service that serves the React app HTML & JS files.

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

