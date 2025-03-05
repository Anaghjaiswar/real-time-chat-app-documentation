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
- [Typing](#typing-notification)

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
`ws://csi-backend-wvn0.onrender.com/ws/chat/<room_id>/?token=<your-auth-token>`

- **`<room_id>`:** Use one of the valid room id's (e.g., `5`, `6`, etc.).
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
val roomId = 5
val wsUrl = "ws://csi-backend-wvn0.onrender.com/ws/chat/$roomId/?token=$token"

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

remember few things- 
1. when you click on send message button it should trigger the websocket connect option
2. if successfully connected your message will be sent, and you will receive the following details:-
--- if you send this
```
{
    "action" : "send_message",
    "message": "hello everyone",
    "message_type": "text"
}
```
you will get --

```
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
```
--- if you want to edit the message you need to send 
```
{
    "action": "edit_message",
    "new_content": "hello csi members of 2nd year",
    "message_id": 43
}
```
you will get 
```
{
    "action": "edited",
    "id": 43,
    "new_content": "hello csi members of 2nd year"
}
```

--- if you want to react to message you need to send 
```
{
    "action" : "react_message",
    "message_id" : 43,
    "reaction" : "Like"
}
```
you will get 
```
{
    "action": "reacted",
    "id": 43,
    "reactions": {
        "Like": 1
    }
}
```
--- if you want to mention anyone from the group you have to send 
```
{
    "action": "send_message",
    "message": "hello @achal and @sparsh",
    "message_type": "text", 
    "mentions" : [13, 15] // id's of the respective members whom you have mentioned
}
```
you will get a normal message like this 
```
{
    "message": "hello @achal and @sparsh",
    "sender": {
        "name": "Anagh",
        "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
        "id": 1,
        "role": "member"
    },
    "message_type": "text",
    "id": 62,
    "attachment": null,
    "created_at": "2025-02-16T06:30:39.461351+00:00",
    "room": "2nd_year"
}
 but in database the mentions will be saved so whenever you open that group again and call the api (see_old_messages_of_that_group) you will get the information of mentioned people id's 
```
--- if you want to delete the message 
``` 
{
    "action" : "delete_message",
    "message_id" : 59
}
```
you will get 
```
{
    "action": "deleted",
    "id": 59
}
```



3 - only the person which belongs to that group or room can send message in that room.
4 - only the person who has send the message can edit or delete his message , no one else.
5 - you can mention only those users who have joined that particular group.
6 - this way multiple users can simultaneously talk to each other in any group.
7 - (in real - time only messaging, editing and deleting a message, mentioning anyone in a message is to be done through websockets request).
8 - when you leave that room disconnect from it.

a important postman api will help in fetching old messages of any particular group 
post request to `https://csi-backend-wvn0.onrender.com/api/chat/groups/group_id/messages/`
you can refer to postman for that 
a sample response will be 
```
[
    {
        "id": 62,
        "sender": {
            "id": 1,
            "first_name": "Anagh",
            "last_name": "Jaiswar",
            "role": "member",
            "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "hello @achal and @sparsh",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-16T12:00:39.461351+05:30",
        "updated_at": "2025-02-16T12:00:39.461351+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": [
            13,
            15
        ]
    },
    {
        "id": 61,
        "sender": {
            "id": 1,
            "first_name": "Anagh",
            "last_name": "Jaiswar",
            "role": "member",
            "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "hello @achal and @sparsh",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-16T11:59:40.389105+05:30",
        "updated_at": "2025-02-16T11:59:40.389105+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 60,
        "sender": {
            "id": 15,
            "first_name": "Sparsh",
            "last_name": "Gupta",
            "role": "member",
            "photo": null,
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "hi anagh , hi achal ",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-16T11:52:52.532858+05:30",
        "updated_at": "2025-02-16T11:52:52.532858+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 59,
        "sender": {
            "id": 13,
            "first_name": "Achal",
            "last_name": "Visnoi",
            "role": "member",
            "photo": null,
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "hello anagh , how are you? ",
        "reactions": null,
        "is_deleted": true,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-16T11:52:33.758164+05:30",
        "updated_at": "2025-02-16T11:55:23.556908+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 58,
        "sender": {
            "id": 1,
            "first_name": "Anagh",
            "last_name": "Jaiswar",
            "role": "member",
            "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "hello 2nd year CSI group members",
        "reactions": {
            "Like": 1
        },
        "is_deleted": false,
        "is_edited": true,
        "status": {},
        "created_at": "2025-02-16T11:52:09.400500+05:30",
        "updated_at": "2025-02-16T11:56:52.237555+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 36,
        "sender": {
            "id": 15,
            "first_name": "Sparsh",
            "last_name": "Gupta",
            "role": "member",
            "photo": null,
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "yes anagh, things are working fine now we will integrate this in frontend.",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T17:01:39.980351+05:30",
        "updated_at": "2025-02-14T17:01:39.980370+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 35,
        "sender": {
            "id": 1,
            "first_name": "Anagh",
            "last_name": "Jaiswar",
            "role": "member",
            "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "thanks for verifying it achal",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:58:53.899302+05:30",
        "updated_at": "2025-02-14T16:58:53.899325+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 34,
        "sender": {
            "id": 13,
            "first_name": "Achal",
            "last_name": "Visnoi",
            "role": "member",
            "photo": null,
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "yes anagh it is working fine",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:58:24.369989+05:30",
        "updated_at": "2025-02-14T16:58:24.370006+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 33,
        "sender": {
            "id": 1,
            "first_name": "Anagh",
            "last_name": "Jaiswar",
            "role": "member",
            "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "hello testing chat working",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:57:49.262163+05:30",
        "updated_at": "2025-02-14T16:57:49.262186+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 32,
        "sender": {
            "id": 13,
            "first_name": "Achal",
            "last_name": "Visnoi",
            "role": "member",
            "photo": null,
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "i am also good , ok byy",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:48:37.033472+05:30",
        "updated_at": "2025-02-14T16:48:37.033491+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 31,
        "sender": {
            "id": 1,
            "first_name": "Anagh",
            "last_name": "Jaiswar",
            "role": "member",
            "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "bye mr. achal",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:08:05.300317+05:30",
        "updated_at": "2025-02-14T16:08:05.300336+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 30,
        "sender": {
            "id": 13,
            "first_name": "Achal",
            "last_name": "Visnoi",
            "role": "member",
            "photo": null,
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "i am also good , ok byy ",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:07:43.344634+05:30",
        "updated_at": "2025-02-14T16:07:43.344651+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 29,
        "sender": {
            "id": 1,
            "first_name": "Anagh",
            "last_name": "Jaiswar",
            "role": "member",
            "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "i am good how about you?",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:07:21.227699+05:30",
        "updated_at": "2025-02-14T16:07:21.227716+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 28,
        "sender": {
            "id": 13,
            "first_name": "Achal",
            "last_name": "Visnoi",
            "role": "member",
            "photo": null,
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "hello anagh, how are you ?",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:07:05.573490+05:30",
        "updated_at": "2025-02-14T16:07:05.573513+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    },
    {
        "id": 27,
        "sender": {
            "id": 1,
            "first_name": "Anagh",
            "last_name": "Jaiswar",
            "role": "member",
            "photo": "http://res.cloudinary.com/dcbla9zbl/image/upload/v1739107362/lwfugihthelebahq8pbz.jpg",
            "year": "2nd"
        },
        "attachment": null,
        "message_type": "text",
        "content": "hello 2nd year members",
        "reactions": null,
        "is_deleted": false,
        "is_edited": false,
        "status": {},
        "created_at": "2025-02-14T16:06:46.561550+05:30",
        "updated_at": "2025-02-14T16:06:46.561574+05:30",
        "room": 5,
        "parent_message": null,
        "mentions": []
    }
]
```

# Typing Notification

This document provides the details for handling real-time typing notifications via WebSocket in the chat application. It outlines the JSON messages that the frontend should send when a user is typing or stops typing, as well as the expected JSON response from the backend.

---

## Typing Notification Overview

When a user starts typing in a chat room, the frontend should send a specific JSON payload to notify the backend. Similarly, when the user stops typing, another JSON payload is sent. The backend will then broadcast a corresponding message to all connected clients in the room, allowing the UI to display or hide a typing indicator (e.g., "User X is typing...").

---

## JSON Messages

### 1. Sending a "Typing" Event

**When the user starts typing, send:**

```json
{
  "action": "typing"
}
```
#### Expected Broadcast Response:

All clients in the room will receive:

```json
{
  "action": "typing",
  "sender": {
    "name": "UserFirstName",
    "photo": "http://example.com/photo.jpg",  // May be null if not available
    "id": 1,
    "role": "member"  // or "admin"
  },
  "is_typing": true
}
```
### 2. Sending a "Stop Typing" Event
When the user stops typing (or after a short period of inactivity), send:

```json
{
  "action": "stop_typing"
}
```
#### Expected Broadcast Response:

All clients in the room will receive:
```json
{
  "action": "typing",
  "sender": {
    "name": "UserFirstName",
    "photo": "http://example.com/photo.jpg",
    "id": 1,
    "role": "member"
  },
  "is_typing": false
}
```

## How It Works

### User Starts Typing:

1. The frontend sends the "typing" event.
2. The backend receives this event and broadcasts a JSON message to all clients in the room with `"is_typing": true`.
3. The frontend UI should then display an indicator (e.g., "UserFirstName is typing...").

### User Stops Typing:

1. The frontend sends the "stop_typing" event after detecting inactivity (e.g., after 500ms of no input).
2. The backend broadcasts a JSON message with `"is_typing": false` to all clients.
3. The UI should remove or hide the typing indicator accordingly.

## Additional Notes

### Connection Requirement:

Ensure the WebSocket connection is established before sending any typing events.

### Debouncing:

Implement a debounce mechanism on the frontend to prevent excessive sending of typing events. For example, send a "stop_typing" event after 500ms of inactivity.

### Multiple Users:

When multiple users are typing, the backend will broadcast a separate JSON message for each user. The frontend should be capable of displaying multiple typing indicators (e.g., "User1 and User2 are typing...").

---

## Typing  implementation (for achal specially)

When a user starts typing, the frontend sends a WebSocket message with the `typing` action. The backend then broadcasts to all clients in that room, and they update the UI to show a typing indicator (e.g., "User X is typing..."). The same applies when the user stops typing.

### Client-Side Implementation
--- Sending Typing Events
Triggering the Event:

When the user begins typing, send a JSON payload to notify the server. When they stop typing, send a stop event.

Example Function:
```
import com.google.gson.Gson

val gson = Gson()

fun sendTypingEvent(webSocket: WebSocket, isTyping: Boolean) {
    val event = mapOf("action" to if (isTyping) "typing" else "stop_typing")
    val jsonData = gson.toJson(event)
    webSocket.send(jsonData)
}
```

### Debouncing:
To avoid flooding the server with events, implement debouncing. For example, using Kotlin Coroutines:

```
import kotlinx.coroutines.*

var typingJob: Job? = null

fun onUserTyping(webSocket: WebSocket) {
    typingJob?.cancel()
    // Send the typing event immediately
    sendTypingEvent(webSocket, true)
    // Schedule a stop_typing event after 500ms of inactivity
    typingJob = GlobalScope.launch {
        delay(500)
        sendTypingEvent(webSocket, false)
    }
}
```

Integrate onUserTyping into your text input listener so it triggers whenever the user types.

### Receiving Typing Notifications
Handling the Broadcast:
In your WebSocket listener's onMessage callback, check for the typing action and update the UI accordingly.
```
override fun onMessage(webSocket: WebSocket, text: String) {
    println("Received: $text")
    try {
        val jsonObj = JSONObject(text)
        if (jsonObj.getString("action") == "typing") {
            val isTyping = jsonObj.getBoolean("is_typing")
            val sender = jsonObj.getJSONObject("sender")
            val senderName = sender.getString("name")
            if (isTyping) {
                showTypingIndicator(senderName)
            } else {
                hideTypingIndicator(senderName)
            }
        }
    } catch (e: Exception) {
        e.printStackTrace()
    }
}
```
### UI Updates:
showTypingIndicator(senderName: String): Display a UI element (e.g., a TextView) indicating "senderName is typing...".
hideTypingIndicator(senderName: String): Remove or hide the typing indicator when the user stops typing.
### Handling Multiple Users:
If more than one user is typing, combine their names into a single message like "User1, User2 are typing...".
Testing the Typing Notification

### Verify Debouncing:
Ensure that the "stop_typing" event is sent only after the user has paused typing for the specified delay (e.g., 500ms).
UI Verification:

Confirm that the typing indicator appears when a "typing" event is received and disappears when a "stop_typing" event is received.


This documentation provides all necessary details for integrating the typing notification feature via WebSocket. Follow these guidelines to ensure that typing events are correctly sent from the frontend and appropriately handled in the UI.







