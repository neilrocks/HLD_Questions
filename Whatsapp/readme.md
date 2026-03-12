Step 3: Sending One-to-One Message (Online Recipient)
User1 sends: Message via WebSocket {type: 'text', receiver_id: user2, content: 'Hello', client_message_id: UUID}
Chat Service validates: User1 authenticated, receiver exists, content not empty (<10KB text)
Service generates: Server message_id (UUID), timestamp
Service checks: Redis HGET websocket_registry user_{user2} → finds connection_id (user2 is online)
Service delivers: Message to user2's WebSocket connection immediately via WebSocket Gateway
Service persists: Message to Cassandra messages table {message_id, sender_id: user1, receiver_id: user2, content, type: 'text', timestamp, status: 'delivered'}
Service sends: Delivery receipt to user1 (double tick) via WebSocket
User2 sends: Read receipt when message displayed → POST /v1/messages/{message_id}/read → update status='read', send read receipt to user1 (blue tick)

Step 4: Sending Message (Offline Recipient)
User1 sends: Message to user2 who is offline
Service checks: Redis HGET websocket_registry user_{user2} → returns null (offline)
Service writes: Message to Redis Stream XADD stream:user:{user2}:messages * message_id {msg_id} sender_id {user1} content {text} timestamp {ts}
Service persists: To Cassandra (same as online case)
Service sends: Single tick to user1 (sent, not delivered yet)
Service triggers: Push notification via Notification Service → FCM/APNS 'New message from User1: Hello'
When user2 comes online: WebSocket connect → service pulls unread messages from Redis Stream XREAD stream:user:{user2}:messages → delivers via WebSocket → acknowledges and deletes from stream → updates status to 'delivered'

Step 5: Group Messaging
User sends: Message to group {group_id: 'group_123', content: 'Hello group'}
Group Service fetches: Group members from Group DB - SELECT members FROM groups WHERE group_id = 'group_123' → returns [user2, user3, user4, user5]
Service iterates: Through members list, for each member check online status
For online members: Deliver immediately via WebSocket to their respective connections
For offline members: Write to Redis Stream stream:user:{member_id}:messages
Service stores: Single message copy in Cassandra group_messages table {message_id, group_id, sender_id, content, timestamp}
Optimization: Don't create N copies for N members, store once with group_id, retrieve using (group_id, timestamp) index