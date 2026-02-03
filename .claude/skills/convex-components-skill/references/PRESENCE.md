# Presence Component

Real-time user presence tracking with efficient heartbeat-based updates.

## Installation

```bash
npm install @convex-dev/presence
```

```typescript
// convex/convex.config.ts
import presence from '@convex-dev/presence/convex.config';
app.use(presence);
```

## Backend Setup

```typescript
// convex/presence.ts
import { mutation, query } from './_generated/server';
import { components } from './_generated/api';
import { v } from 'convex/values';
import { Presence } from '@convex-dev/presence';

const presence = new Presence(components.presence);

// Required: Heartbeat to maintain presence
export const heartbeat = mutation({
  args: {
    roomId: v.string(),
    userId: v.string(),
    sessionId: v.string(),
    interval: v.number()
  },
  handler: async (ctx, { roomId, userId, sessionId, interval }) => {
    // Add auth checks here
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Unauthorized');

    return await presence.heartbeat(ctx, roomId, userId, sessionId, interval);
  }
});

// Required: List users in a room
export const list = query({
  args: { roomToken: v.string() },
  handler: async (ctx, { roomToken }) => {
    // Avoid per-user reads so subscriptions share cache
    return await presence.list(ctx, roomToken);
  }
});

// Required: Graceful disconnect (called via sendBeacon)
export const disconnect = mutation({
  args: { sessionToken: v.string() },
  handler: async (ctx, { sessionToken }) => {
    // Can't check auth - called via sendBeacon on tab close
    return await presence.disconnect(ctx, sessionToken);
  }
});
```

## React Client

### Basic Usage with FacePile

```typescript
import { useState } from "react";
import { api } from "../convex/_generated/api";
import usePresence from "@convex-dev/presence/react";
import FacePile from "@convex-dev/presence/facepile";

function ChatRoom({ roomId }: { roomId: string }) {
  const { user } = useUser(); // Your auth hook

  const presenceState = usePresence(
    api.presence,  // Your presence API module
    roomId,        // Room identifier
    user?.name ?? "Anonymous"  // Display name
  );

  return (
    <div>
      <FacePile presenceState={presenceState ?? []} />
      {/* Rest of your UI */}
    </div>
  );
}
```

### Custom Presence UI

```typescript
import usePresence from "@convex-dev/presence/react";

function OnlineUsers({ roomId }: { roomId: string }) {
  const presenceState = usePresence(api.presence, roomId, currentUser.name);

  if (!presenceState) return <div>Loading...</div>;

  return (
    <div className="online-users">
      <span>{presenceState.length} online</span>
      <ul>
        {presenceState.map((user) => (
          <li key={user.sessionId}>
            {user.userId}
            {user.lastOnline && (
              <span className="status">
                Last seen: {new Date(user.lastOnline).toLocaleTimeString()}
              </span>
            )}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## React Native

```bash
npx expo install react-native expo-crypto
```

```typescript
import { usePresence } from '@convex-dev/presence/react-native';

// Same API as web
const presenceState = usePresence(api.presence, roomId, userName);
```

## Additional Server Methods

### Check if User is Online (Any Room)

```typescript
// convex/presence.ts
export const isUserOnline = query({
  args: { userId: v.string() },
  handler: async (ctx, { userId }) => {
    const rooms = await presence.listUser(ctx, userId);
    return rooms.length > 0;
  }
});
```

### List User's Active Rooms

```typescript
export const getUserRooms = query({
  args: { userId: v.string() },
  handler: async (ctx, { userId }) => {
    return await presence.listUser(ctx, userId);
  }
});
```

### Get Room Presence Count

```typescript
export const getRoomCount = query({
  args: { roomToken: v.string() },
  handler: async (ctx, { roomToken }) => {
    const users = await presence.list(ctx, roomToken);
    return users.length;
  }
});
```

## How It Works

```
┌─────────────┐     heartbeat()      ┌─────────────┐
│   Client    │─────────────────────►│   Convex    │
│             │◄─────────────────────│             │
│  usePresence│     list() query     │  Presence   │
│             │     (reactive)       │  Component  │
└─────────────┘                      └─────────────┘
       │                                    │
       │ disconnect() via sendBeacon        │
       └────────────────────────────────────┘
                  (on tab close)
```

1. **Heartbeat**: Client sends periodic heartbeats via mutation
2. **Scheduled cleanup**: Component schedules removal if heartbeat stops
3. **Reactive list**: Query only updates when users join/leave (not on every heartbeat)
4. **Graceful disconnect**: `sendBeacon` ensures cleanup even on tab close

## Use Cases

### Chat Room Presence

```typescript
function ChatRoom({ roomId }: { roomId: string }) {
  const presenceState = usePresence(api.presence, roomId, user.name);

  return (
    <div>
      <header>
        {presenceState?.length ?? 0} users online
      </header>
      <ChatMessages roomId={roomId} />
    </div>
  );
}
```

### Document Collaboration

```typescript
function DocumentEditor({ docId }: { docId: string }) {
  const presenceState = usePresence(api.presence, `doc:${docId}`, user.email);

  const otherEditors = presenceState?.filter(p => p.userId !== user.email);

  return (
    <div>
      {otherEditors?.map(editor => (
        <div key={editor.sessionId} className="collaborator-badge">
          {editor.userId} is editing
        </div>
      ))}
      <Editor docId={docId} />
    </div>
  );
}
```

### "User is typing" Indicator

For typing indicators, consider combining with your own state:

```typescript
// This requires extending the component or custom implementation
// The base presence component tracks online/offline, not custom state
// For typing indicators, you may want to store that in your own table
// and use presence just for "who's in the room"
```

## Performance Notes

- **No polling**: Uses scheduled functions, not periodic queries
- **Efficient updates**: List query only reruns when membership changes
- **Shared cache**: All subscriptions to the same room share one query
- **Graceful degradation**: Works even if heartbeats are delayed
