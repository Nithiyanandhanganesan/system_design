# WebSockets - Complete Guide

## What are WebSockets?

WebSockets provide **full-duplex communication** between client and server over a single TCP connection. Unlike HTTP's request-response model, WebSockets allow both client and server to send messages at any time.

### HTTP vs WebSockets

```
HTTP (Request-Response):
Client  ──Request──>  Server
Client  <──Response── Server
(Connection closes)

WebSocket (Persistent Connection):
Client  <──────────> Server
       (Bidirectional, real-time)
```

## WebSocket Handshake

### 1. Initial HTTP Request
```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

### 2. Server Response
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### 3. WebSocket Connection Established
Now both client and server can send messages anytime.

## Basic WebSocket Implementation

### Client-Side (JavaScript)
```javascript
// Create WebSocket connection
const socket = new WebSocket('ws://localhost:8080');

// Connection opened
socket.addEventListener('open', (event) => {
    console.log('Connected to WebSocket server');
    socket.send('Hello Server!');
});

// Listen for messages
socket.addEventListener('message', (event) => {
    console.log('Message from server:', event.data);
    
    // Handle different message types
    const message = JSON.parse(event.data);
    switch (message.type) {
        case 'chat':
            displayChatMessage(message.data);
            break;
        case 'notification':
            showNotification(message.data);
            break;
        case 'user_joined':
            updateUserList(message.data);
            break;
    }
});

// Handle connection errors
socket.addEventListener('error', (error) => {
    console.error('WebSocket error:', error);
});

// Handle connection close
socket.addEventListener('close', (event) => {
    console.log('WebSocket connection closed:', event.code, event.reason);
    
    // Attempt to reconnect
    setTimeout(() => {
        console.log('Attempting to reconnect...');
        connectWebSocket();
    }, 3000);
});

// Send different types of messages
function sendChatMessage(message) {
    socket.send(JSON.stringify({
        type: 'chat',
        data: {
            message: message,
            timestamp: new Date().toISOString(),
            userId: getCurrentUserId()
        }
    }));
}

function joinRoom(roomId) {
    socket.send(JSON.stringify({
        type: 'join_room',
        data: { roomId: roomId }
    }));
}
```

### Server-Side (Node.js with ws library)
```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

// Store active connections
const clients = new Map();
const rooms = new Map();

wss.on('connection', (ws, request) => {
    console.log('New client connected');
    
    // Generate unique client ID
    const clientId = generateClientId();
    clients.set(clientId, {
        socket: ws,
        userId: null,
        rooms: new Set()
    });
    
    // Handle incoming messages
    ws.on('message', (data) => {
        try {
            const message = JSON.parse(data);
            handleMessage(clientId, message);
        } catch (error) {
            console.error('Invalid message format:', error);
        }
    });
    
    // Handle client disconnect
    ws.on('close', () => {
        console.log('Client disconnected');
        const client = clients.get(clientId);
        
        if (client) {
            // Remove from all rooms
            client.rooms.forEach(roomId => {
                leaveRoom(clientId, roomId);
            });
            
            clients.delete(clientId);
        }
    });
    
    // Handle connection errors
    ws.on('error', (error) => {
        console.error('WebSocket error:', error);
    });
});

function handleMessage(clientId, message) {
    const client = clients.get(clientId);
    
    switch (message.type) {
        case 'authenticate':
            client.userId = message.data.userId;
            sendToClient(clientId, {
                type: 'authenticated',
                data: { success: true }
            });
            break;
            
        case 'join_room':
            joinRoom(clientId, message.data.roomId);
            break;
            
        case 'leave_room':
            leaveRoom(clientId, message.data.roomId);
            break;
            
        case 'chat':
            broadcastToRoom(message.data.roomId, {
                type: 'chat',
                data: {
                    message: message.data.message,
                    userId: client.userId,
                    timestamp: new Date().toISOString()
                }
            }, clientId);
            break;
            
        default:
            console.log('Unknown message type:', message.type);
    }
}

function joinRoom(clientId, roomId) {
    const client = clients.get(clientId);
    
    if (!rooms.has(roomId)) {
        rooms.set(roomId, new Set());
    }
    
    rooms.get(roomId).add(clientId);
    client.rooms.add(roomId);
    
    // Notify others in the room
    broadcastToRoom(roomId, {
        type: 'user_joined',
        data: {
            userId: client.userId,
            roomId: roomId
        }
    }, clientId);
}

function leaveRoom(clientId, roomId) {
    const client = clients.get(clientId);
    
    if (rooms.has(roomId)) {
        rooms.get(roomId).delete(clientId);
        client.rooms.delete(roomId);
        
        // Remove room if empty
        if (rooms.get(roomId).size === 0) {
            rooms.delete(roomId);
        } else {
            // Notify others
            broadcastToRoom(roomId, {
                type: 'user_left',
                data: {
                    userId: client.userId,
                    roomId: roomId
                }
            });
        }
    }
}

function sendToClient(clientId, message) {
    const client = clients.get(clientId);
    if (client && client.socket.readyState === WebSocket.OPEN) {
        client.socket.send(JSON.stringify(message));
    }
}

function broadcastToRoom(roomId, message, excludeClientId = null) {
    if (!rooms.has(roomId)) return;
    
    rooms.get(roomId).forEach(clientId => {
        if (clientId !== excludeClientId) {
            sendToClient(clientId, message);
        }
    });
}

function broadcast(message, excludeClientId = null) {
    clients.forEach((client, clientId) => {
        if (clientId !== excludeClientId && client.socket.readyState === WebSocket.OPEN) {
            client.socket.send(JSON.stringify(message));
        }
    });
}
```

## Advanced WebSocket Patterns

### 1. **Heartbeat/Ping-Pong**
```javascript
// Server-side heartbeat
const HEARTBEAT_INTERVAL = 30000; // 30 seconds

wss.on('connection', (ws) => {
    ws.isAlive = true;
    
    ws.on('pong', () => {
        ws.isAlive = true;
    });
});

// Check for dead connections
const heartbeatInterval = setInterval(() => {
    wss.clients.forEach((ws) => {
        if (!ws.isAlive) {
            ws.terminate();
            return;
        }
        
        ws.isAlive = false;
        ws.ping();
    });
}, HEARTBEAT_INTERVAL);

// Client-side heartbeat
let pingInterval;

socket.addEventListener('open', () => {
    pingInterval = setInterval(() => {
        if (socket.readyState === WebSocket.OPEN) {
            socket.ping();
        }
    }, 30000);
});

socket.addEventListener('close', () => {
    clearInterval(pingInterval);
});
```

### 2. **Automatic Reconnection**
```javascript
class ReconnectingWebSocket {
    constructor(url, options = {}) {
        this.url = url;
        this.options = {
            maxReconnectAttempts: 5,
            reconnectInterval: 1000,
            maxReconnectInterval: 30000,
            reconnectDecay: 1.5,
            ...options
        };
        
        this.reconnectAttempts = 0;
        this.readyState = WebSocket.CONNECTING;
        this.connect();
    }
    
    connect() {
        this.socket = new WebSocket(this.url);
        
        this.socket.addEventListener('open', (event) => {
            console.log('WebSocket connected');
            this.reconnectAttempts = 0;
            this.readyState = WebSocket.OPEN;
            this.onopen?.(event);
        });
        
        this.socket.addEventListener('message', (event) => {
            this.onmessage?.(event);
        });
        
        this.socket.addEventListener('close', (event) => {
            this.readyState = WebSocket.CLOSED;
            this.onclose?.(event);
            
            if (this.reconnectAttempts < this.options.maxReconnectAttempts) {
                this.scheduleReconnect();
            }
        });
        
        this.socket.addEventListener('error', (error) => {
            this.onerror?.(error);
        });
    }
    
    scheduleReconnect() {
        const timeout = this.options.reconnectInterval * 
                       Math.pow(this.options.reconnectDecay, this.reconnectAttempts);
        
        const actualTimeout = Math.min(timeout, this.options.maxReconnectInterval);
        
        console.log(`Reconnecting in ${actualTimeout}ms (attempt ${this.reconnectAttempts + 1})`);
        
        setTimeout(() => {
            this.reconnectAttempts++;
            this.connect();
        }, actualTimeout);
    }
    
    send(data) {
        if (this.readyState === WebSocket.OPEN) {
            this.socket.send(data);
        } else {
            console.warn('WebSocket is not open. Message not sent:', data);
        }
    }
    
    close() {
        this.socket.close();
    }
}

// Usage
const ws = new ReconnectingWebSocket('ws://localhost:8080');
ws.onmessage = (event) => console.log('Message:', event.data);
```

### 3. **Message Queue for Offline Support**
```javascript
class QueuedWebSocket extends ReconnectingWebSocket {
    constructor(url, options = {}) {
        super(url, options);
        this.messageQueue = [];
        this.maxQueueSize = options.maxQueueSize || 100;
    }
    
    send(data) {
        if (this.readyState === WebSocket.OPEN) {
            // Send queued messages first
            while (this.messageQueue.length > 0) {
                const queuedMessage = this.messageQueue.shift();
                super.send(queuedMessage);
            }
            
            // Send current message
            super.send(data);
        } else {
            // Queue message for later
            this.queueMessage(data);
        }
    }
    
    queueMessage(data) {
        if (this.messageQueue.length >= this.maxQueueSize) {
            // Remove oldest message
            this.messageQueue.shift();
        }
        
        this.messageQueue.push(data);
        console.log(`Message queued. Queue size: ${this.messageQueue.length}`);
    }
}
```

## Real-World Use Cases

### 1. **Chat Application**
```javascript
// Chat client
class ChatClient {
    constructor(userId, userName) {
        this.userId = userId;
        this.userName = userName;
        this.socket = new ReconnectingWebSocket('ws://localhost:8080/chat');
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        this.socket.onopen = () => {
            this.authenticate();
        };
        
        this.socket.onmessage = (event) => {
            const message = JSON.parse(event.data);
            this.handleMessage(message);
        };
    }
    
    authenticate() {
        this.socket.send(JSON.stringify({
            type: 'authenticate',
            data: {
                userId: this.userId,
                userName: this.userName,
                token: localStorage.getItem('authToken')
            }
        }));
    }
    
    joinRoom(roomId) {
        this.socket.send(JSON.stringify({
            type: 'join_room',
            data: { roomId }
        }));
    }
    
    sendMessage(roomId, message) {
        this.socket.send(JSON.stringify({
            type: 'message',
            data: {
                roomId,
                message,
                timestamp: Date.now()
            }
        }));
    }
    
    handleMessage(message) {
        switch (message.type) {
            case 'message':
                this.displayMessage(message.data);
                break;
            case 'user_joined':
                this.showUserJoined(message.data);
                break;
            case 'user_left':
                this.showUserLeft(message.data);
                break;
            case 'typing':
                this.showTypingIndicator(message.data);
                break;
        }
    }
}
```

### 2. **Live Trading Dashboard**
```javascript
// Trading client
class TradingDashboard {
    constructor() {
        this.socket = new WebSocket('ws://trading-api.com/stream');
        this.subscriptions = new Set();
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        this.socket.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.handleMarketData(data);
        };
    }
    
    subscribeToSymbol(symbol) {
        if (!this.subscriptions.has(symbol)) {
            this.socket.send(JSON.stringify({
                action: 'subscribe',
                symbol: symbol
            }));
            this.subscriptions.add(symbol);
        }
    }
    
    unsubscribeFromSymbol(symbol) {
        if (this.subscriptions.has(symbol)) {
            this.socket.send(JSON.stringify({
                action: 'unsubscribe',
                symbol: symbol
            }));
            this.subscriptions.delete(symbol);
        }
    }
    
    handleMarketData(data) {
        switch (data.type) {
            case 'price_update':
                this.updatePrice(data.symbol, data.price);
                break;
            case 'trade':
                this.addTradeToFeed(data);
                break;
            case 'order_book':
                this.updateOrderBook(data.symbol, data.book);
                break;
        }
    }
    
    updatePrice(symbol, price) {
        const element = document.getElementById(`price-${symbol}`);
        if (element) {
            element.textContent = price.toFixed(2);
            element.classList.add('updated');
            setTimeout(() => element.classList.remove('updated'), 1000);
        }
    }
}
```

### 3. **Real-time Collaboration (Google Docs style)**
```javascript
// Collaborative editor
class CollaborativeEditor {
    constructor(documentId, userId) {
        this.documentId = documentId;
        this.userId = userId;
        this.socket = new WebSocket('ws://collab-server.com/docs');
        this.operationalTransform = new OperationalTransform();
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        this.socket.onopen = () => {
            this.joinDocument();
        };
        
        this.socket.onmessage = (event) => {
            const message = JSON.parse(event.data);
            this.handleOperation(message);
        };
    }
    
    joinDocument() {
        this.socket.send(JSON.stringify({
            type: 'join_document',
            data: {
                documentId: this.documentId,
                userId: this.userId
            }
        }));
    }
    
    sendOperation(operation) {
        this.socket.send(JSON.stringify({
            type: 'operation',
            data: {
                documentId: this.documentId,
                userId: this.userId,
                operation: operation,
                revision: this.getRevision()
            }
        }));
    }
    
    handleOperation(message) {
        switch (message.type) {
            case 'operation':
                this.applyRemoteOperation(message.data);
                break;
            case 'cursor_position':
                this.updateCursor(message.data);
                break;
            case 'user_joined':
                this.showUserJoined(message.data);
                break;
        }
    }
    
    applyRemoteOperation(data) {
        if (data.userId !== this.userId) {
            const transformedOp = this.operationalTransform.transform(
                data.operation,
                this.getPendingOperations()
            );
            this.applyOperation(transformedOp);
        }
    }
}
```

## WebSocket Security

### 1. **Authentication**
```javascript
// Server-side authentication
wss.on('connection', (ws, request) => {
    let authenticated = false;
    
    // Set authentication timeout
    const authTimeout = setTimeout(() => {
        if (!authenticated) {
            ws.close(1008, 'Authentication timeout');
        }
    }, 10000);
    
    ws.on('message', (data) => {
        const message = JSON.parse(data);
        
        if (message.type === 'authenticate' && !authenticated) {
            const token = message.data.token;
            
            if (validateToken(token)) {
                authenticated = true;
                clearTimeout(authTimeout);
                ws.userId = extractUserIdFromToken(token);
                
                ws.send(JSON.stringify({
                    type: 'authenticated',
                    data: { success: true }
                }));
            } else {
                ws.close(1008, 'Invalid authentication');
            }
        } else if (!authenticated) {
            ws.close(1008, 'Not authenticated');
        } else {
            // Handle authenticated messages
            handleAuthenticatedMessage(ws, message);
        }
    });
});
```

### 2. **Rate Limiting**
```javascript
const rateLimiter = new Map();

function checkRateLimit(clientId, limit = 10, window = 60000) {
    const now = Date.now();
    const clientData = rateLimiter.get(clientId) || { count: 0, resetTime: now + window };
    
    if (now > clientData.resetTime) {
        clientData.count = 1;
        clientData.resetTime = now + window;
    } else {
        clientData.count++;
    }
    
    rateLimiter.set(clientId, clientData);
    
    return clientData.count <= limit;
}

ws.on('message', (data) => {
    const clientId = ws.clientId;
    
    if (!checkRateLimit(clientId)) {
        ws.send(JSON.stringify({
            type: 'error',
            data: { message: 'Rate limit exceeded' }
        }));
        return;
    }
    
    // Process message
});
```

### 3. **Input Validation**
```javascript
const messageSchemas = {
    chat: {
        type: 'object',
        properties: {
            message: { type: 'string', maxLength: 1000 },
            roomId: { type: 'string', pattern: '^[a-zA-Z0-9-_]+$' }
        },
        required: ['message', 'roomId']
    }
};

function validateMessage(type, data) {
    const schema = messageSchemas[type];
    if (!schema) return false;
    
    // Use a validation library like Ajv
    return validate(schema, data);
}

ws.on('message', (rawData) => {
    try {
        const message = JSON.parse(rawData);
        
        if (!validateMessage(message.type, message.data)) {
            ws.send(JSON.stringify({
                type: 'error',
                data: { message: 'Invalid message format' }
            }));
            return;
        }
        
        handleMessage(message);
    } catch (error) {
        ws.close(1003, 'Invalid message format');
    }
});
```

## Performance Optimization

### 1. **Connection Pooling**
```javascript
// Server-side connection management
class ConnectionManager {
    constructor() {
        this.connections = new Map();
        this.rooms = new Map();
        this.maxConnectionsPerIP = 10;
    }
    
    addConnection(ws, clientIP) {
        // Check IP limits
        const ipConnections = this.getConnectionsByIP(clientIP);
        if (ipConnections.length >= this.maxConnectionsPerIP) {
            ws.close(1008, 'Too many connections from this IP');
            return false;
        }
        
        const connectionId = generateId();
        this.connections.set(connectionId, {
            socket: ws,
            ip: clientIP,
            joinedAt: Date.now(),
            lastActivity: Date.now()
        });
        
        return connectionId;
    }
    
    removeConnection(connectionId) {
        const connection = this.connections.get(connectionId);
        if (connection) {
            // Remove from all rooms
            this.rooms.forEach((room, roomId) => {
                if (room.has(connectionId)) {
                    room.delete(connectionId);
                }
            });
            
            this.connections.delete(connectionId);
        }
    }
    
    getConnectionsByIP(ip) {
        return Array.from(this.connections.values())
                   .filter(conn => conn.ip === ip);
    }
}
```

### 2. **Message Compression**
```javascript
// Enable per-message compression
const wss = new WebSocket.Server({
    port: 8080,
    perMessageDeflate: {
        // Compression settings
        threshold: 1024,        // Only compress messages > 1KB
        concurrencyLimit: 10,   // Limit concurrent compressions
        memLevel: 7,           // Memory usage vs compression ratio
    }
});
```

### 3. **Binary Messages for Performance**
```javascript
// Send binary data instead of JSON for high-frequency updates
function sendBinaryPriceUpdate(ws, symbol, price, volume) {
    const buffer = Buffer.allocUnsafe(12); // 4 + 4 + 4 bytes
    
    buffer.writeUInt32BE(getSymbolId(symbol), 0);  // Symbol ID (4 bytes)
    buffer.writeFloatBE(price, 4);                 // Price (4 bytes)
    buffer.writeUInt32BE(volume, 8);               // Volume (4 bytes)
    
    ws.send(buffer);
}

// Client-side binary parsing
socket.addEventListener('message', (event) => {
    if (event.data instanceof ArrayBuffer) {
        const view = new DataView(event.data);
        const symbolId = view.getUint32(0);
        const price = view.getFloat32(4);
        const volume = view.getUint32(8);
        
        updatePrice(getSymbolName(symbolId), price, volume);
    } else {
        // Handle text messages
        const message = JSON.parse(event.data);
        handleTextMessage(message);
    }
});
```

## WebSocket vs Alternatives

### WebSocket vs Server-Sent Events (SSE)
```javascript
// SSE - Server to client only
const eventSource = new EventSource('/events');
eventSource.onmessage = (event) => {
    console.log('Server sent:', event.data);
};

// WebSocket - Bidirectional
const socket = new WebSocket('ws://localhost:8080');
socket.send('Client message');  // ✅ Can send to server
socket.onmessage = (event) => {
    console.log('Server sent:', event.data);
};
```

### WebSocket vs Polling
```javascript
// Polling - Inefficient
setInterval(async () => {
    const response = await fetch('/api/messages');
    const messages = await response.json();
    updateMessages(messages);
}, 1000); // Check every second

// WebSocket - Efficient
socket.onmessage = (event) => {
    const message = JSON.parse(event.data);
    updateMessages([message]); // Real-time updates
};
```

## Summary

**WebSockets are ideal for:**
✅ Real-time chat applications
✅ Live trading/gaming platforms  
✅ Collaborative editing
✅ Live notifications
✅ Real-time dashboards

**Key Benefits:**
- Full-duplex communication
- Low latency
- Persistent connection
- Real-time data exchange

**Considerations:**
- More complex than REST APIs
- Connection management overhead
- Security considerations
- Scaling challenges with many connections
