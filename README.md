# realtime-chat
This is System Design Document with small examples for a realtime chat application.

## Requirements
- Users should be able to log in via username & password
- Users should be able to send and receive messages to another user
- Messages can contain text, a single image, video, or any other file
- The architecture should be modular for future extensibility with new features
- The system should be able to fully scale horizontally
- The system should be multiplatform, it serves both web and mobile clients

## 3rd party technologies
- **MongoDB** for storing user data and messages
- **Cloudflare R2** object storage for storing files
- **Socket IO** for sending and receiving messages
- **Ktor** as authentication and web service

## Services

### Authentication Service ([details](AUTHENTICATION SERVICE.md))
This is a simple API services that allows users to register & log-in. 
It returns a JWT token that can be used for authentication with other services.

### Messaging Service
This service handles the messaging (sending and receiving) between users. 
It exposes a single Socket IO endpoint that clients can connect to. 

### Database Services