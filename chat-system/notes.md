# Chat System Design — Learnings

## What This Design Teaches

The chat system introduces:
- WebSocket for real time bidirectional communication
- Multi-server architecture with a message broker
- Redis Pub/Sub for cross-server message routing
- Redis Session for user presence tracking
- Offline handling with Push Notifications
- Message persistence in PostgreSQL

---

## Clarifying Questions to Ask

- One to one only or group chat?
- Online presence (last seen)?
- How long to store messages?
- Text only or media support?
- Read receipts?
- Push notifications for offline users?

---

## Scale

- 50 million total users
- 10 million daily active users
- 40 messages per user per day = 400 million messages per day

---

## Why WebSocket

| Option | Problem |
|---|---|
| HTTP Polling | Client keeps asking — wasteful, not real time |
| Long Polling | Server holds connection until message — better but half duplex |
| WebSocket | Persistent bidirectional connection — perfect for chat |

WebSocket = one connection stays open. Both sides can send anytime.

---

## Core Problem — Multi Server Routing

10 million active users cannot connect to one server. Multiple Chat Servers behind a Load Balancer.

Problem: User A on Server 1, User B on Server 2. How does A's message reach B?

**Solution — Redis Pub/Sub as message broker:**

```
Chat Server 1 → publishes to Redis channel:userB
Redis → delivers to Chat Server 2 (subscribed to channel:userB)
Chat Server 2 → pushes to User B over WebSocket
```

Servers never talk to each other directly. Redis is the middleman.

---

## Redis — Two Responsibilities

**1. Redis Session (presence tracking)**
```
Key:   session:{userId}
Value: chatServerId
TTL:   refreshed while WebSocket connection is alive
```

When User B connects → `session:userB = chatServer2`
When User B disconnects → key expires via TTL

**2. Redis Pub/Sub (message routing)**
- Each Chat Server subscribes to channels for users connected to it
- When message arrives for User B → publish to `channel:userB`
- Chat Server 2 receives it and pushes to User B's WebSocket

---

## Online Message Flow (both users online)

```
1. User A opens app → WebSocket connects to Chat Server 1
2. User B opens app → WebSocket connects to Chat Server 2
3. User A sends "Hi" to User B
4. Load Balancer routes to Chat Server 1 (A's server)
5. Chat Server 1 saves message to PostgreSQL
6. Chat Server 1 checks Redis: session:userB = chatServer2
7. Chat Server 1 publishes to Redis: channel:userB
8. Redis delivers to Chat Server 2 (subscribed to channel:userB)
9. Chat Server 2 pushes message to User B over WebSocket
10. User B sees "Hi" instantly
```

---

## Offline Message Flow

```
1. Chat Server 1 checks Redis: session:userB = empty (offline)
2. Chat Server 1 saves message to PostgreSQL (marked undelivered)
3. Chat Server 1 sends to Push Notification Service
4. APNs (iPhone) / FCM (Android) delivers notification to User B's device
5. User B taps notification → app opens → WebSocket connects
6. App fetches unread messages from PostgreSQL
7. Messages displayed — marked as delivered
```

---

## Architecture

```
Clients (WebSocket)
  ↓
Load Balancer
  ↓
Chat Servers (1, 2, 3, 4...)
  ↕ publish/subscribe/deliver     ↓ save message    ↓ offline user
Redis Pub/Sub + Session        PostgreSQL Primary   Push Notification Service
                                                      (APNs / FCM)
```

---

## Database — PostgreSQL

Stores all messages permanently.

```
messages table:
  id, sender_id, receiver_id, content, created_at, delivered
```

- Messages saved immediately when sent (before delivery to recipient)
- Undelivered messages fetched when user comes back online
- Read receipts updated when message delivered

---

## Key Interview Phrases

- "WebSocket is the right choice here — it gives us a persistent bidirectional connection. HTTP polling would work but wastes resources checking for messages constantly."
- "With multiple Chat Servers we use Redis Pub/Sub as the message broker. Server 1 publishes to a channel for User B, and whichever server User B is connected to has subscribed to that channel."
- "Redis stores two things — the Pub/Sub channels for routing, and the session map of userId to chatServerId for presence tracking."
- "If User B is offline, session:userB is empty in Redis. We save the message to PostgreSQL as undelivered and fire a push notification via APNs or FCM."
- "When User B comes back online, the app fetches unread messages from PostgreSQL on reconnect."

---

## Encryption

- Messages should be encrypted end to end
- Encryption happens on the client before sending
- Server stores encrypted content — cannot read messages
- Decryption happens on recipient's device after receiving
- Keys are exchanged between clients — server never holds private keys

---

## Topics Covered

| Topic | Concept |
|---|---|
| Real time protocol | WebSocket — persistent bidirectional |
| Multi server routing | Redis Pub/Sub — publish to channel, deliver to subscriber |
| User presence | Redis Session — userId → chatServerId with TTL |
| Message storage | PostgreSQL — all messages persisted |
| Offline handling | Push Notification Service (APNs / FCM) |
| Encryption | End to end — client encrypts, server stores encrypted |
