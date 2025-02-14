# Real-Time Chat Backend Documentation

This document describes the backend API and WebSocket configuration for the Real-Time Chat application. It includes information for both REST API endpoints and WebSocket communication so that the frontend team (using Kotlin) can integrate smoothly.

---

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [WebSocket Connection](#websocket-connection)
  - [Endpoint URL](#endpoint-url)
  - [Kotlin Example](#kotlin-example)
- [REST API Endpoints](#rest-api-endpoints)
  - [User Rooms](#1-display-the-groups-or-rooms-the-user-has-joined)
  - [Room Members](#2-list-of-members-in-a-particular-group)
  - [Room Messages (with Pagination)](#3-api-for-displaying-old-messages-of-a-particular-group)
  - [Room List (All Groups)](#4-api-for-listing-all-groups-not-just-groups-joined-by-the-user)
- [Error Codes and Responses](#error-codes-and-responses)
- [Testing Guidelines](#testing-guidelines)
- [Deployment Notes](#deployment-notes)
- [Environment Variables](#environment-variables)
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
  Use the login API (not detailed here) to obtain a token. The token must be included in both REST API requests (in the `Authorization` header) and WebSocket connections (via query parameter).
  
**Example Header for REST requests:**
Authorization: Token <your-auth-token>

**For WebSocket connections, include the token in the URL:**

---

## WebSocket Connection

### Endpoint URL

The WebSocket endpoint follows this pattern:
`ws://csi-backend-wvn0.onrender.com/ws/chat/<room_name>/?token=<your-auth-token>`

- **`<room_name>`:** Use one of the valid room names (e.g., `2nd_year`, `backend_2_3`, etc.).
- **`<your-auth-token>`:** The token obtained after authenticating via the login API.

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
        webSocket.send("""{"message": "Hello, world!", "message_type": "text"}""")
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

## REST API Endpoints
1. **Display the Groups or Rooms the User Has Joined**
  - URL: /api/chat/groups/
  - Method: GET
  - Authentication: Token required
  - Response Example:

```JSON
    [
  {
    "id": 1,
    "name": "2nd_year",
    "description": "Room for second-year students",
    "room_avatar": "https://...",
    "created_at": "2025-02-13T10:00:00Z",
    "updated_at": "2025-02-13T10:00:00Z",
    "members_count": 5
  },
  ...
]
```
2. **List of Members in a Particular Group**
  - URL: /api/chat/groups/<room_name>/members/
  - Method: GET
  - Authentication: Token required
  - Response Example:
``` JSON
[
  {
    "id": 1,
    "username": "anagh",
    "first_name": "Anagh",
    "last_name": "Singh"
  },
  {
    "id": 2,
    "username": "john_doe",
    "first_name": "John",
    "last_name": "Doe"
  }
]
```
## Testing Guidelines

### WebSocket Testing
- **Tools:** Use tools such as Postman or a simple Kotlin client (see the provided Kotlin example) to test WebSocket connectivity.
- **Checklist:**
  - Verify that a WebSocket connection is successfully established when a valid token is provided.
  - Test sending messages over the WebSocket and ensure that messages are correctly received and broadcast to all connected clients.

### REST API Testing
- **Tools:** Use Postman or any API testing tool to verify all endpoints.
- **Checklist:**
  - Ensure each request includes the header:
    ```
    Authorization: Token <your-token>
    ```
  - Test various endpoints for expected behavior, including:
    - Accessing endpoints with valid tokens.
    - Testing edge cases such as accessing a room as a non-member.
    - Sending invalid parameters and verifying that appropriate error messages are returned.






