# Real-Time Chat Backend Documentation

This document describes the backend API and WebSocket configuration for the Real-Time Chat application. It includes information for  WebSocket communication so that the frontend team (using Kotlin) can integrate smoothly.

---

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [WebSocket Connection](#websocket-connection)
  - [Endpoint URL](#endpoint-url)
  - [Kotlin Example](#kotlin-example)
- [Testing Guidelines](#testing-guidelines)
- [Additional Information](#additional-information)

---

## Overview

The backend is built using Django, Django REST Framework, and Django Channels for real-time communication. We use DRF token-based authentication for REST and WebSocket connections. For the channel layer in production, we use Redis (or for development/in a small scale, the In-Memory Channel Layer).

**Production URL:**  
`csi-backend-wvn0.onrender.com`

---

## Authentication

- **Method:** Token-based authentication via Django REST Framework.
- **Obtaining a Token:**  
  Use the login API (not detailed here) to obtain a token. The token must be included in both REST API requests (in the `Authorization` header) and WebSocket connections (via query params).
  

**For WebSocket connections, include the token in the URL:**

---

## WebSocket Connection

### Endpoint URL

The WebSocket endpoint follows this pattern:
`ws://csi-backend-wvn0.onrender.com/ws/chat/<room_name>/?token=<your-auth-token>`

- **`<room_name>`:** Use one of the valid room names (e.g., `2nd_year`, `backend_2_3`, etc.).
- **`<your-auth-token>`:** The token obtained after authenticating via the login API.
- also after <room_name> in url , then you can put token through params in key value pair key-- `token` and value will be `your auth token` 

### Kotlin Example

Here’s a sample Kotlin snippet to connect to the WebSocket:

```kotlin
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.WebSocket
import okhttp3.WebSocketListener
import okio.ByteString

val client = OkHttpClient()

val token = "934dc28f497f315b24966dbc5c72a9bded24d0e7"
val roomName = "2nd_year"
val wsUrl = "ws://csi-backend-wvn0.onrender.com/ws/chat/$roomName/?token=$token"

val request = Request.Builder()
    .url(wsUrl)
    .build()

val webSocketListener = object : WebSocketListener() {
    override fun onOpen(webSocket: WebSocket, response: okhttp3.Response) {
        println("Connected to WebSocket")
        // Example: Send a message when connected
        webSocket.send("""{"message": "Hello, world!", "message_type": "text"}""")  // you don't need to send message_type as by default we assume it to be text type
    }

    override fun onMessage(webSocket: WebSocket, text: String) {
        println("Received message: $text")
    }

    override fun onMessage(webSocket: WebSocket, bytes: ByteString) {
        println("Received bytes: ${bytes.hex()}")
    }

    override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
        println("Closing WebSocket: $code / $reason")
        webSocket.close(1000, null)
    }

    override fun onFailure(webSocket: WebSocket, t: Throwable, response: okhttp3.Response?) {
        println("WebSocket error: ${t.message}")
    }
}

val webSocket = client.newWebSocket(request, webSocketListener)
// Shutdown the client when done:
client.dispatcher.executorService.shutdown()
```
Note: This example uses OkHttp. Adjust as needed if you use a different WebSocket library.


## Testing Guidelines

### WebSocket Testing
- **Tools:** Use tools such as Postman or a simple Kotlin client (see the provided Kotlin example) to test WebSocket connectivity.
- **Checklist:**
  - Verify that a WebSocket connection is successfully established when a valid token is provided.
  - Test sending messages over the WebSocket and ensure that messages are correctly received and broadcast to all connected clients.

### Additional Information
```
remember few things- 
1. when you click on send message button it should trigger the websocket connect option
2. if successfully connected your message will be sent, and you will receive the following details:-
if you send this
{
    "message": "hello everyone"
}

you will get --

{
    "message": "hello everyone", // the message 
    "sender": {
        "name": "Anagh", //first name
        "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg", //  photo by default null
        "id": 1, // id of the sender 
        "role": "member" // role of the sender admin or member
    }, // sender details
    "message_type": "text", // message type
    "id": 43, // id of message
    "attachment": null, // attachment sending will be done through HTTPS
    "created_at": "2025-02-15T17:07:32.256110+00:00", // timestamps of the message 
    "room": "2nd_year" // the message belongs to which room
}
3 - only the person which belongs to that group or room can send message in that room
4 - this way multiple users can simultaneously talk to each other in any group.
5 - other important info (in real - time only messaging is possible editing and deleting a message is to be done through http request refer to postman documentation for that).
6 - when you leave that room disconnect from it.

```
