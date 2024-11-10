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

## Frameworks and technologies used by this system
- **MongoDB** for storing user data, session data and messages
- **Redis** to notify other server instances of changes in a channel
- **Cloudflare R2** object storage for storing files
- **Socket IO** for sending and receiving messages
- **Ktor** as authentication and web service
- **Realm** as local database for native and web clients
- **Sentry** for error reporting
- A reverse proxy that terminates SSL, sets cookies for sticky sessions and handles load balancing between multiple instances of the services

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

### Web Service ([details](WEB_SERVICE.md))
Simple web service that serves the React app HTML & JS files.

### Database Services
There are two logical MongoDB databases in the system:
- [User Database](USER_DATABASE.md) for storing user data
- [Chat Database](CHAT_DATABASE.md) for storing messages and channels

These can be on the same physical 3-node replica set or different ones, depending on the load.
Sharding is also possible to distribute the load.

Redis is only used for pub/sub to notify other server instances 
of [changes in a channel](MESSAGING_SERVICE.md#channel-changes). Sharding is also possible, but most probably necessary.
