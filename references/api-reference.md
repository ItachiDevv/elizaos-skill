# REST API & WebSocket Reference

Default base URL: `http://localhost:3000`
Auth header (if `ELIZA_SERVER_AUTH_TOKEN` set): `X-API-KEY: your_token`

## Agent Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/agents` | List all agents |
| POST | `/api/agents` | Create agent (body: character config) |
| GET | `/api/agents/{id}` | Get agent details |
| PATCH | `/api/agents/{id}` | Update agent character/config |
| DELETE | `/api/agents/{id}` | Delete agent |
| POST | `/api/agents/{id}/start` | Start agent runtime |
| POST | `/api/agents/{id}/stop` | Stop agent runtime |
| GET | `/api/agents/{id}/panels` | Get agent UI panels |
| GET | `/api/agents/{id}/worlds` | Get all worlds for agent |
| POST | `/api/agents/{id}/worlds` | Create a world for agent |
| PATCH | `/api/agents/{id}/worlds/{worldId}` | Update world |

## Messaging Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/messaging/submit` | Submit message to central system |
| POST | `/api/messaging/channels/{id}/messages` | Send to channel |
| GET | `/api/messaging/channels/{id}/messages` | Get channel messages |
| GET | `/api/messaging/channels/{id}/info` | Channel info |
| GET | `/api/messaging/channels/{id}/participants` | Channel participants |
| POST | `/api/messaging/channels` | Create channel |
| POST | `/api/messaging/channels/central` | Create central channel |
| PATCH | `/api/messaging/channels/{id}` | Update channel |
| DELETE | `/api/messaging/channels/{id}` | Delete channel |
| POST | `/api/messaging/servers` | Create server |
| GET | `/api/messaging/servers/central` | Get central servers |
| GET | `/api/messaging/servers/{id}/channels` | Server channels |
| GET | `/api/messaging/servers/{id}/agents` | Server agents |
| POST | `/api/messaging/servers/{id}/agents/{agentId}` | Add agent to server |
| DELETE | `/api/messaging/servers/{id}/agents/{agentId}` | Remove agent |
| POST | `/api/messaging/dm/{agentId}/{userId}` | Get/create DM channel |
| POST | `/api/messaging/ingest` | Ingest external messages |
| POST | `/api/messaging/external` | Process external message |

## Sessions API

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/messaging/sessions` | Create session (body: agentId, userId) |
| GET | `/api/messaging/sessions/{id}` | Get session info |
| GET | `/api/messaging/sessions` | List sessions |
| POST | `/api/messaging/sessions/{id}/messages` | Send message |
| GET | `/api/messaging/sessions/{id}/messages` | Get history (pagination: before/after/limit) |
| POST | `/api/messaging/sessions/{id}/renew` | Renew session |
| POST | `/api/messaging/sessions/{id}/heartbeat` | Keep-alive |
| PATCH | `/api/messaging/sessions/{id}/timeout` | Update timeout config |
| DELETE | `/api/messaging/sessions/{id}` | End session |
| GET | `/api/messaging/sessions/health` | Sessions health check |

### Session Config
```typescript
{
  agentId: UUID,
  userId: UUID,
  metadata?: object,
  timeoutConfig?: {
    timeoutMinutes: number,      // 5-1440
    autoRenew: boolean,
    maxDurationMinutes: number,
    warningThresholdMinutes: number,
  }
}
```

## Memory Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/memory/{agentId}/memories` | Get agent memories |
| GET | `/api/memory/{agentId}/rooms/{roomId}` | Get room memories |
| POST | `/api/memory/{agentId}/rooms` | Create room |
| PATCH | `/api/memory/{agentId}/memories/{id}` | Update memory |
| DELETE | `/api/memory/{agentId}/memories` | Delete all memories |
| DELETE | `/api/memory/{agentId}/rooms/{roomId}` | Delete room memories |

## Room Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/rooms/{agentId}` | Get agent rooms |
| GET | `/api/rooms/{agentId}/{roomId}` | Room details |
| POST | `/api/rooms/{agentId}` | Create room |
| PATCH | `/api/rooms/{agentId}/{roomId}` | Update room |
| DELETE | `/api/rooms/{agentId}/{roomId}` | Delete room |

## Audio Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/audio/{agentId}/transcribe` | Transcribe audio file |
| POST | `/api/audio/{agentId}/speech` | Text-to-speech |
| POST | `/api/audio/{agentId}/synthesize` | Synthesize speech |
| POST | `/api/audio/{agentId}/conversation` | Conversation to speech |
| POST | `/api/audio/{agentId}/message` | Process audio message |

## System Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/health` | Basic health |
| GET | `/api/ping` | Ping |
| GET | `/api/status` | System status |
| GET | `/api/version` | Version info |
| GET | `/api/config` | System config |
| GET | `/api/debug` | Debug info |
| GET | `/api/env` | Environment variables |
| PUT | `/api/env` | Update env variables |
| POST | `/api/stop` | Stop server |

## Other

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/jobs` | Create job |
| GET | `/api/jobs/{id}` | Get job status |
| GET | `/api/jobs/health` | Jobs health |
| GET | `/api/logs/{agentId}` | Agent logs |
| GET | `/api/logs/system` | System logs |
| POST | `/api/media/{agentId}/upload` | Upload media |
| POST | `/api/media/channels/{id}/upload` | Upload to channel |
| GET | `/api/runs/{agentId}` | List agent runs |
| GET | `/api/runs/{agentId}/{runId}` | Get specific run |

## WebSocket (Socket.IO)

Connection: `ws://localhost:3000/socket.io/`

### Client Events (emit)
- `join` — Join a room
- `leave` — Leave a room
- `message` — Send message
- `request-world-state` — Request world state

### Server Events (listen)
- `messageBroadcast` — New message in room
- `messageComplete` — Agent finished responding
- `world-state` — World state update
- `logEntry` — Log entry from agent
- `error` — Error notification
- `sessionExpirationWarning` — Session about to expire
- `sessionExpired` — Session expired
- `sessionRenewed` — Session renewed

### Connection Example
```typescript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000', {
  auth: { token: 'your-api-key' },
});

socket.emit('join', { roomId, agentId });

socket.on('messageBroadcast', (data) => {
  console.log(`${data.sender}: ${data.content.text}`);
});

socket.on('messageComplete', (data) => {
  console.log('Agent done:', data.content.text);
});
```
