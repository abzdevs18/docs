# Chat Service Documentation

## Overview
The chat service is a real-time messaging system built with Socket.IO, Redis, and Bull queues. It supports high concurrency with features like message persistence and real-time delivery.

## Authentication
To connect to the WebSocket server, clients must provide authentication credentials in the Socket.IO connection options:

```typescript
const socket = io('ws://localhost:3003/chat/', {
  auth: {
    token: 'Bearer YOUR_JWT_TOKEN', // JWT token with the same format as API requests
    userId: 'YOUR_USER_ID'         // Must match the user ID in the JWT token
  }
});
```

The server will reject connections that:
- Are missing the token or userId
- Have an invalid token format (must start with 'Bearer ')
- Have an invalid or expired JWT token
- Have a userId that doesn't match the one in the JWT token

## Architecture
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │◄────┤  Socket.IO  │◄────┤Redis Adapter│
└─────────────┘     └─────────────┘     └─────────────┘
       ▲                   ▲                   ▲
       │                   │                   │
       │             ┌─────────────┐     ┌─────────────┐
       └─────────────┤Chat Service │◄────┤Message Queue│
                     └─────────────┘     └─────────────┘
                           ▲
                     ┌─────────────┐
                     │   Prisma    │
                     └─────────────┘
```

## Server Configuration
- Default Port: 3003
- WebSocket Path: `/chat/`
- Transport: WebSocket (optimized for performance)
- Authentication: JWT-based

## Currently Implemented Features
1. Real-time messaging
   - Send and receive messages in real-time
   - Message persistence in Redis and database
   - Message editing and deletion
   - Typing indicators

2. Room Management
   - Join and leave chat rooms
   - Direct messaging between users
   - Room-based message broadcasting

3. User Presence
   - Online/Offline/Away status tracking
   - Real-time presence updates
   - Last seen tracking

4. Performance Features
   - WebSocket-only transport
   - Message queuing with Bull
   - Redis caching for messages
   - Connection pooling
   - Horizontal scaling support

5. Security
   - JWT-based authentication
   - Message authorization checks
   - Input validation
   - Secure WebSocket configuration

Note: Some features mentioned in the events section are planned but not yet fully implemented, including:
- Message reactions
- Message threading
- File attachments
- Read receipts
- Room management features
- Favorites management

## Events

### Client to Server Events
```typescript
interface ClientToServerEvents {
  'chat:join': (roomId: string) => void;
  'chat:leave': (roomId: string) => void;
  'chat:send_message': (data: { 
    roomId: string; 
    content: string; 
    type?: MessageType; 
  }) => void;
  'chat:typing': (data: { 
    roomId: string; 
    isTyping: boolean 
  }) => void;
  'chat:presence': (status: 'online' | 'offline' | 'away') => void;
  'chat:update_message': (data: { 
    messageId: string; 
    content: string 
  }) => void;
  'chat:delete_message': (messageId: string) => void;
  'chat:start_direct': (targetUserId: string) => void;
  'chat:toggle_favorite': (data: {
    id: string;
    type: 'ROOM' | 'USER';
  }) => void;
}
```

### Server to Client Events
```typescript
interface ServerToClientEvents {
  'chat:message': (message: ChatMessage) => void;
  'chat:typing_status': (data: { 
    userId: string; 
    roomId: string; 
    isTyping: boolean 
  }) => void;
  'chat:presence_update': (data: { 
    userId: string; 
    status: string 
  }) => void;
  'chat:error': (error: { message: string }) => void;
  'chat:message_update': (data: { 
    roomId: string; 
    message: ChatMessage 
  }) => void;
  'chat:message_delete': (data: { 
    roomId: string; 
    messageId: string 
  }) => void;
  'chat:init_data': (data: {
    rooms: Room[];
    people: Person[];
    favorites: Favorite[];
  }) => void;
  'chat:room_list_update': (rooms: Room[]) => void;
  'chat:people_list_update': (people: Person[]) => void;
  'chat:favorites_update': (favorites: Favorite[]) => void;
}
```

## Usage Examples

### Connecting to the Chat Server
```javascript
const socket = io('ws://localhost:3003/chat/', {
  transports: ['websocket'],
  auth: {
    token: 'Bearer your_jwt_token',
    userId: 'user_id'
  }
});
```

### Joining a Chat Room
```javascript
socket.emit('chat:join', 'room_id');

socket.on('chat:message', (message) => {
  console.log('New message:', message);
});
```

### Sending a Message
```javascript
socket.emit('chat:send_message', {
  roomId: 'room_id',
  content: 'Hello, world!',
  type: 'TEXT'
});
```

### Updating Presence
```javascript
socket.emit('chat:presence', 'online');

socket.on('chat:presence_update', ({ userId, status }) => {
  console.log(`User ${userId} is now ${status}`);
});
```

### Typing Indicators
```javascript
// When user starts typing
socket.emit('chat:typing', {
  roomId: 'room_id',
  isTyping: true
});

// When user stops typing
socket.emit('chat:typing', {
  roomId: 'room_id',
  isTyping: false
});

// Listen for typing updates
socket.on('chat:typing_status', ({ userId, roomId, isTyping }) => {
  console.log(`User ${userId} is ${isTyping ? 'typing' : 'not typing'} in room ${roomId}`);
});
```

### Direct Messaging
To start a direct chat with another user:

```javascript
// Initiate a direct chat with another user
socket.emit('chat:start_direct', 'target_user_id');

// Listen for the room creation/join confirmation
socket.on('chat:message', (message) => {
  if (message.type === 'SYSTEM' && message.content === 'Direct chat started') {
    console.log('Direct chat room created:', message.roomId);
    // Store this roomId for future direct messages with this user
  }
});

// Send a direct message (once you have the roomId)
socket.emit('chat:send_message', {
  roomId: 'direct_room_id', // The roomId received from the previous step
  content: 'Hey, how are you?',
  type: 'TEXT'
});

// Listen for direct messages
socket.on('chat:message', (message) => {
  if (message.roomType === 'DIRECT') {
    console.log('New direct message:', message);
  }
});
```

Note: Direct chat rooms are automatically created when you first initiate a chat with another user. The room ID is generated based on both users' IDs to ensure a unique, consistent room for their conversations.

## Error Handling
The server emits errors through the 'chat:error' event:
```javascript
socket.on('chat:error', (error) => {
  console.error('Chat error:', error.message);
});
```

## Monitoring
The service provides monitoring endpoints:

### Health Check
```
GET /health
Response: { status: 'ok', timestamp: string }
```

### Metrics
```
GET /metrics
Response: {
  connections: number,
  messageQueueJobs: {
    waiting: number,
    active: number,
    completed: number,
    failed: number
  },
  presenceQueueJobs: {
    waiting: number,
    active: number,
    completed: number,
    failed: number
  },
  memoryUsage: {
    heapUsed: number,
    heapTotal: number,
    external: number,
    arrayBuffers: number
  },
  uptime: number
}
```

## Environment Variables
```env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_password
CHAT_SERVER_PORT=3003
FRONTEND_URL=http://localhost:3000
NEXT_PUBLIC_JWT_SECRET=your_jwt_secret
```

## Current Performance Features
1. WebSocket-only transport
2. Message queuing with Bull
3. Redis caching for messages
4. Efficient presence tracking
5. Connection pooling
6. Horizontal scaling with Redis adapter

## Security
1. JWT-based authentication
2. Message authorization checks
3. Input validation
4. Secure WebSocket configuration

## Development
To run the chat server in development mode:
```bash
npm run dev:chat
```

To build and start in production:
```bash
npm run build:chat
npm run start:chat
```

### Listing Rooms, People, and Favorites
To get the lists of rooms, people, and favorites:

```javascript
// Listen for initial lists when connecting
socket.on('chat:init_data', (data) => {
  const {
    rooms,         // List of all rooms the user is in
    people,        // List of all users the user has interacted with
    favorites      // List of favorited rooms/people
  } = data;
});

// Listen for updates to these lists
socket.on('chat:room_list_update', (rooms) => {
  console.log('Updated room list:', rooms);
});

socket.on('chat:people_list_update', (people) => {
  console.log('Updated people list:', people);
});

socket.on('chat:favorites_update', (favorites) => {
  console.log('Updated favorites:', favorites);
});

// Add/Remove from favorites
socket.emit('chat:toggle_favorite', {
  id: 'room_or_user_id',
  type: 'ROOM' // or 'USER' for people
});
```

The data structures for these lists are:

```typescript
interface Room {
  id: string;
  name: string;
  type: 'GROUP' | 'DIRECT';
  lastMessage?: {
    content: string;
    timestamp: string;
    type: 'TEXT' | 'SYSTEM';
  };
  unreadCount: number;
  isEncrypted?: boolean;
  avatar?: string;
}

interface Person {
  id: string;
  name: string;
  status: 'online' | 'offline' | 'away';
  lastSeen?: string;
  avatar?: string;
}

interface Favorite {
  id: string;
  type: 'ROOM' | 'USER';
  name: string;
  avatar?: string;
  lastMessage?: string;
}
``` 

## Communication Methods

The chat service uses two methods of communication:

1. **WebSocket (Socket.IO)** - For real-time features
   - Live messaging
   - Typing indicators
   - Presence updates
   - Real-time notifications
   - Message delivery status

2. **REST API** - For CRUD operations
   - Room management
   - Message history
   - Reactions
   - User management

### REST API Endpoints

All REST endpoints are prefixed with `/api/v2/chat/`

#### Rooms
```
POST /api/v2/chat/rooms
Create a new chat room

Request:
{
    "name": string,
    "participants": string[]  // Array of user IDs
}

Headers:
- Content-Type: application/json
- Authorization: Bearer <token>
- x-tenant-tag: <tenant>  // If using multi-tenancy

Response:
{
    "id": string,
    "name": string,
    "type": "GROUP" | "DIRECT",
    "participants": Array<{
        "user": {
            "id": string,
            "name": string,
            "avatar": string | null
        },
        "role": string,
        "joinedAt": string
    }>
}
```

```
GET /api/v2/chat/rooms
List all rooms for the current user

Headers:
- Authorization: Bearer <token>
- x-tenant-tag: <tenant>  // If using multi-tenancy

Response:
{
    "rooms": Array<{
        "id": string,
        "name": string,
        "type": "GROUP" | "DIRECT",
        "lastMessage": {
            "content": string,
            "timestamp": string,
            "sender": {
                "id": string,
                "name": string
            }
        } | null
    }>
}
```

#### Messages
```
GET /api/v2/chat/messages?roomId=<room_id>
Get message history for a room

Headers:
- Authorization: Bearer <token>
- x-tenant-tag: <tenant>  // If using multi-tenancy

Query Parameters:
- roomId: string (required)
- before: string (timestamp, optional)
- limit: number (optional, default: 50)

Response:
{
    "messages": Array<{
        "id": string,
        "content": string,
        "sender": {
            "id": string,
            "name": string
        },
        "timestamp": string,
        "type": "TEXT" | "SYSTEM" | "FILE",
        "reactions": Array<{
            "emoji": string,
            "users": Array<{
                "id": string,
                "name": string
            }>
        }>
    }>
}
```

#### Reactions
```
POST /api/v2/chat/reactions
Add a reaction to a message

Request:
{
    "messageId": string,
    "emoji": string
}

Headers:
- Content-Type: application/json
- Authorization: Bearer <token>
- x-tenant-tag: <tenant>  // If using multi-tenancy

Response:
{
    "id": string,
    "emoji": string,
    "messageId": string,
    "userId": string
}
```

```
DELETE /api/v2/chat/reactions/:reactionId
Remove a reaction from a message

Headers:
- Authorization: Bearer <token>
- x-tenant-tag: <tenant>  // If using multi-tenancy

Response:
{
    "success": true
}
```

### Error Handling

REST API errors follow this format:
```json
{
    "error": string,
    "details": string | null,
    "code": number
}
```

Common error codes:
- 400: Bad Request (invalid input)
- 401: Unauthorized (missing or invalid token)
- 403: Forbidden (CORS error or insufficient permissions)
- 404: Not Found
- 500: Internal Server Error
``` 
