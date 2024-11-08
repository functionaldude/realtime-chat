# realtime-chat
This is System Design Document with small examples for a realtime chat application.

## Requirements
- Users should be able to log in via username & password
- Users should be able to send and receive messages to another user
- Messages can contain text, a single image, video, or any other file
- The system should be able to fully scale horizontally
- The system should be multiplatform, it serves both web and mobile clients
- The architecture should be modular for future extensibility with new features
- The architecture should track certain metrics for monitoring, alerting and user engagement

## 3rd party technologies
- **MongoDB** for storing user data, session data and messages
- **Cloudflare R2** object storage for storing files
- **Socket IO** for sending and receiving messages
- **Ktor** as authentication and web service

## Code structure
The source code is written in Kotlin to make developing cross-platform easier, it is divided into 4 main modules:

| module         |                                                                         |
|----------------|-------------------------------------------------------------------------|
| common         | This module is pure kotlin, it is included in all other modules         |
| backend-common | Module that holds shared business logic for backend services            |
| auth           | Module that implements the Authentication Service                       |
| messaging      | Module that implements the Messaging Service                            |
| web            | Module that implements the Web Service                                  |
| client-common  | Module that holds shared business logic to communicate with the backend |
| client-native  | Native chat app via Kotlin Multiplatform and Compose Multiplatform      |
| client-web     | Web chat app via KotlinJS and React                                     |

The dependency graph is as follows:
```
└── common
    ├── backend-common
    │   ├── auth
    │   ├── messaging
    │   └── web
    └── client-common
        ├── client-native
        └── client-web   
```

The **common** module, holds various shared "Request" and "Response" classes that are used by all other modules, they are serialized using `kotlinx.serialization`.

## Services

### Authentication Service ([details](AUTHENTICATION_SERVICE.md))
This is a simple API services that allows users to register & log-in. 
It returns a `sessionToken` that can be used for authentication with other services.

### Messaging Service ([details](MESSAGING_SERVICE.md))
This service handles the messaging (sending and receiving) between users. 
It exposes a single Socket IO endpoint that clients can connect to. 

### Web Service
Simple web service that serves the React app HTML & JS files.

### Database Services