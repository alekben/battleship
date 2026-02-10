# Building a Multiplayer Turn-Based Game with Agora RTC and AI Voice Agents

*A technical guide to synchronizing game state using Agora's data messaging and integrating voice-controlled AI agents*

---

## Why This Matters

When you think about Agora, video conferencing probably comes to mind. But buried in the RTC SDK is a feature that changes how you can build real-time multiplayer games: **data messaging**. Instead of setting up WebSockets, managing reconnections, or dealing with a separate backend for game state, you can piggyback game logic on the same infrastructure handling your audio and video.

This guide walks through building a two-player Battleship game that uses Agora for three things simultaneously:
1. Video/audio communication between players
2. Game state synchronization via data messages
3. Voice-controlled AI agents that respond to player commands

You'll see why this architecture works, where it might break, and how to extend it.

## What We're Building

A browser-based Battleship game where:
- Two players see each other's video feeds while playing
- Players attack by speaking coordinates to an AI agent ("Attack B4")
- Game state (attacks, hits, misses) syncs through Agora's data channel
- Spectators can join and watch live gameplay
- Each player has their own AI agent listening in separate channels

![Game Architecture Diagram - placeholder for image showing player connections, channels, and data flow]

## The Core Problem: Syncing Turn-Based Game State

In a turn-based game, you need every player to agree on:
- Whose turn it is
- What moves were made
- What the board looks like
- When the game ends

The naive approach: poll a server every second. The better approach: server pushes updates via WebSocket. The Agora approach: use the same RTC connection that's already open for video.

### Why Agora's Data Messaging Works Here

Agora's `sendStreamMessage()` API lets you send up to 30KB of data per message at 30 messages per second. For a Battleship move (row, column, result), you need maybe 50 bytes. This is massive overkill, which is perfect—you have headroom.

Here's what the message structure looks like:

```javascript
// Attack message sent when a player makes a move
const msgObj = { 
  type: "attack", 
  row: 3, 
  col: 7 
};
client.sendStreamMessage(JSON.stringify(msgObj));

// Attack result sent back by the defender
const resultMsg = {
  type: "attack_result",
  row: 3,
  col: 7,
  isHit: true,
  isGameOver: false
};
client.sendStreamMessage(JSON.stringify(resultMsg));
```

Every client listens for these messages:

```javascript
client.on("stream-message", handleStreamMessage);

function handleStreamMessage(uid, msgData) {
  const decoder = new TextDecoder();
  const msgStr = decoder.decode(msgData);
  const msg = JSON.parse(msgStr);
  
  switch (msg.type) {
    case "attack":
      handleAttack(msg);
      break;
    case "attack_result":
      handleAttackResult(msg);
      break;
    case "ready":
      handlePlayerReady();
      break;
  }
}
```

This pattern—message types with payloads—gives you a simple RPC-style system. No WebSocket server needed.

## Architecture: Multi-Channel Design

Here's where things get interesting. This game doesn't use one Agora channel. It uses **three channels per game**:

1. **Main game channel** (`battleship_123456`)
   - Both players join as hosts
   - Video/audio/data messages flow here
   - Audience members join as spectators

2. **Agent A channel** (`battleship_123456_agenta`)
   - Player A publishes audio here
   - Agent A joins and listens
   - Agent A responds with voice commands

3. **Agent B channel** (`battleship_123456_agentb`)
   - Player B publishes audio here
   - Agent B joins and listens
   - Agent B responds with voice commands

Why separate channels for agents? **Isolation**. Each agent needs to hear only its player's voice commands, not the opponent's. If both agents were in the main channel, they'd interfere with each other and respond to the wrong player's commands.

### Code: Setting Up Multiple Clients

```javascript
// Main game client for video/audio/data
client = AgoraRTC.createClient({ mode: "live", codec: "vp8", role: "host" });
await client.join(AGORA_APP_ID, currentChannelName, null, "PlayerA");
await client.publish([localAudioTrack, localVideoTrack]);

// Agent client for voice commands
agentClient = AgoraRTC.createClient({ mode: "live", codec: "vp8", role: "host" });
await agentClient.join(AGORA_APP_ID, currentAgentChannelName, null, "PlayerA");
await agentClient.publish(localAgentAudioTrack);
```

Each client is independent. You manage their lifecycles separately, handle events separately, but they share the same user context. This is powerful because it lets you segment communication by purpose.

## The Game Loop: Attack and Response Flow

Let's trace what happens when Player A attacks:

### Step 1: Player Speaks Command

```javascript
// Player says "Attack B4" into their microphone
// Agent hears this via the agent channel
// Agent processes the transcript and extracts coordinates
```

The agent's transcript handler parses the spoken text:

```javascript
function handleAgentStreamMessage(uid, msgData) {
  const messageDataJson = JSON.parse(messageData);
  
  if (messageDataJson.text.includes("Acknowledged")) {
    // Agent confirmed valid coordinates
    const match = messageDataJson.text.match(/([A-J])(1[0]|[1-9])/);
    if (match) {
      const [_, letter, number] = match;
      const row = letter.charCodeAt(0) - 'A'.charCodeAt(0);
      const col = parseInt(number) - 1;
      attackCell(row, col, letter, number);
    }
  }
}
```

### Step 2: Attack Message Sent

```javascript
function attackCell(row, col) {
  if (!isMyTurn || enemyBoardState[row][col] !== 0) {
    showNotification("Invalid move!");
    return;
  }

  const msgObj = { type: "attack", row, col };
  client.sendStreamMessage(JSON.stringify(msgObj));
  
  // Immediately switch turns locally
  isMyTurn = false;
  updateStatus("Waiting for opponent...");
}
```

Notice the optimistic turn switch. Player A assumes the message will arrive, so they disable their controls immediately. This prevents double-moves.

### Step 3: Opponent Receives and Processes

```javascript
function handleAttack(msg) {
  // Check if the attack hit a ship
  const isHit = myShips[msg.row][msg.col];
  myBoardState[msg.row][msg.col] = isHit ? 1 : 2;
  renderMyBoard();
  
  // Check for game over
  const totalHits = myBoardState.flat().filter(cell => cell === 1).length;
  const isGameOver = totalHits === TOTAL_SHIP_CELLS;
  
  // Send result back
  const resultMsg = {
    type: "attack_result",
    row: msg.row,
    col: msg.col,
    isHit,
    isGameOver
  };
  client.sendStreamMessage(JSON.stringify(resultMsg));
  
  if (!isGameOver) {
    isMyTurn = true; // Now it's my turn
    updateStatus("Your turn!");
  }
}
```

### Step 4: Original Attacker Receives Result

```javascript
function handleAttackResult(msg) {
  // Update my view of the enemy's board
  enemyBoardState[msg.row][msg.col] = msg.isHit ? 1 : 2;
  renderEnemyBoard();
  
  if (msg.isGameOver) {
    showGameOver(true); // I won!
    return;
  }
  
  // Result received, but it's opponent's turn now
  isMyTurn = false;
  updateStatus("Waiting for opponent...");
}
```

This four-step flow ensures both clients stay synchronized. The key insight: **the defender is the source of truth**. Player A sends an attack, but Player B determines if it was a hit. Player A must trust Player B's response.

## Handling Race Conditions

What happens if both players click attack at the same moment?

**The Problem**: Without a central server acting as referee, you can't prevent race conditions through locking. Both players might think it's their turn.

**The Solution**: Establish a canonical turn order at game start, then trust the message sequence.

```javascript
function startGame() {
  bothPlayersReady = true;
  // Player A always goes first
  isMyTurn = isPlayerA;
  updateStatus(isMyTurn ? "Your turn!" : "Waiting for opponent...");
}
```

When a player receives an attack while thinking it's their turn, the `handleAttack` function still processes it correctly because it doesn't check `isMyTurn`. It just responds. This makes the game eventually consistent—even if both players briefly think it's their turn, the message sequence will resolve the conflict.

Could both players send attacks simultaneously? Yes. But because each player only updates their *opponent's* board based on results they receive, not results they send, the game stays consistent. Player A's attack updates Player B's board, and vice versa. They never touch the same data structure.

## Voice Agent Integration

The AI agents are the most impressive part. Each player gets a voice assistant that:
- Listens for "Attack [coordinates]" commands
- Validates the coordinates
- Confirms actions with voice feedback
- Can suggest moves when asked "Choose for me"

### Agent Lifecycle

Agents don't start when players join. They start when **both players finish ship placement**:

```javascript
function handlePlayerReady() {
  otherPlayerReady = true;
  
  if (!isPlacingShips) {
    bothPlayersReady = true;
    
    // NOW start the agents
    if (isPlayerA) {
      startAgent("AgentA", currentAgentChannelName, "AgentA", "PlayerA", 
        "You are acting as a commentator for a game of Battleship...",
        "Welcome to Battleship Agora Player A!");
    } else {
      startAgent("AgentB", currentAgentChannelName, "AgentB", "PlayerB",
        "You are acting as a commentator for a game of Battleship...",
        "Welcome to Battleship Agora Player B!");
    }
    
    isMyTurn = isPlayerA;
    updateStatus(isMyTurn ? "Your turn!" : "Waiting for opponent...");
  }
}
```

Why wait? Because you don't want agents listening during ship placement. They might accidentally interpret placement instructions as attack commands.

### Agent Lambda Functions

The agents aren't running in the browser. They're serverless functions:

```javascript
async function startAgent(name, chan, uid, remoteUid, prompt, message) {
  const url = "https://fcnfih4dgp6sysnjl4ywyazkze0gwvhg.lambda-url.us-east-2.on.aws";
  
  const reqBody = {
    agentname: name,
    channel: chan,
    agentuid: uid,
    agentrtmuid: chan,
    rtmflag: true,
    remoteuid: remoteUid,
    prompt: prompt,
    message: message
  };
  
  const resp = await fetch(url, {
    method: "POST",
    headers: {
      "Accept": "application/json",
      "Content-Type": "application/json"
    },
    body: JSON.stringify(reqBody)
  });
  
  const data = await resp.json();
  myAgentsId = data.agent_id; // Save for later cleanup
}
```

The Lambda function spins up an agent that:
1. Joins the specified Agora channel
2. Subscribes to the player's audio
3. Runs speech-to-text on incoming audio
4. Processes commands via GPT-4
5. Generates text-to-speech responses
6. Publishes audio back to the channel

### Agent Message Protocol

Agents send transcripts back via Agora's RTM (Real-Time Messaging) or stream messages. The messages arrive chunked to handle size limits:

```javascript
function handleAgentStreamMessage(uid, msgData) {
  let [messageId, messagePart, messageChunks, messageData] = 
    new TextDecoder().decode(msgData).split("|");
  
  messageData = atob(messageData); // Base64 decode
  
  // Reconstruct chunked messages
  messagesMap.set(messageId, 
    messagesMap.get(messageId) ? 
    messagesMap.get(messageId) + messageData : 
    messageData
  );
  
  // Wait for all chunks
  if (parseInt(messagePart) === parseInt(messageChunks)) {
    const fullMessage = messagesMap.get(messageId);
    messagesMap.delete(messageId);
    processAgentMessage(fullMessage);
  }
}
```

This chunking protocol handles large transcripts. Each chunk contains:
- `messageId`: Unique ID for this message
- `messagePart`: Which chunk (1, 2, 3...)
- `messageChunks`: Total chunks
- `messageData`: Base64-encoded payload

### Parsing Voice Commands

Once you have the full transcript, extract coordinates:

```javascript
if (messageDataJson.text.includes("Acknowledged")) {
  // Agent uses "Acknowledged" prefix for valid commands
  const match = messageDataJson.text.match(/([A-J])(1[0]|[1-9])/);
  if (match) {
    const [_, letter, number] = match;
    const row = letter.charCodeAt(0) - 'A'.charCodeAt(0);
    const col = parseInt(number) - 1;
    attackCell(row, col, letter, number);
  } else {
    showNotification("Invalid coordinates! Try again.");
  }
}
```

The regex `/([A-J])(1[0]|[1-9])/` matches:
- A letter A-J (rows)
- A number 1-10 (columns, with special handling for "10")

This parsing happens client-side, not in the agent. The agent's job is to confirm it heard a valid attack command ("Acknowledged firing on B4"). The client then extracts coordinates and executes the attack.

### Volume Control: Whose Agent Should You Hear?

Both agents are speaking, but you should only hear your own agent clearly:

```javascript
// Player A's agent channel
await agentClient.join(AGORA_APP_ID, currentAgentChannelName, null, "PlayerA");
await agentClient.publish(localAgentAudioTrack);
localAgentAudioTrack.setVolume(100); // Full volume for Player A

// Player B's agent channel
await agentClient.join(AGORA_APP_ID, currentAgentChannelName, null, "PlayerB");
await agentClient.publish(localAgentAudioTrack);
localAgentAudioTrack.setVolume(0); // Muted for Player B during opponent's turn
```

Volume adjusts based on turn:

```javascript
function handleAttack(msg) {
  // Opponent attacked me, now it's my turn
  isMyTurn = true;
  localAgentAudioTrack.setVolume(100); // Unmute my agent
}

function attackCell(row, col) {
  // I'm attacking, switching to opponent's turn
  isMyTurn = false;
  localAgentAudioTrack.setVolume(0); // Mute my agent
}
```

This creates natural turn-taking. Your agent speaks when it's your turn, stays quiet otherwise.

## Audience Mode: Spectator View

The game supports spectators who can join mid-game and watch both boards. This is trickier than it sounds.

### Joining as Audience

```javascript
async function joinAsAudience(chName) {
  // Create main client as audience role
  client = AgoraRTC.createClient({ mode: "live", codec: "vp8", role: "host" });
  await client.join(AGORA_APP_ID, chName, null, audienceId);
  
  // Join both agent channels to hear both agents
  agentClient = AgoraRTC.createClient({ mode: "live", codec: "vp8", role: "audience" });
  await agentClient.join(AGORA_APP_ID, currentAgentAChannelName, null, audienceId);
  
  audienceSpecialClient = AgoraRTC.createClient({ mode: "live", codec: "vp8", role: "audience" });
  await audienceSpecialClient.join(AGORA_APP_ID, currentAgentBChannelName, null, audienceId);
  
  // Send join notification so players can send board state
  const msgObj = { type: "audience-joined" };
  client.sendStreamMessage(JSON.stringify(msgObj));
  
  // Switch to audience role (no publishing)
  await client.setClientRole("audience");
}
```

Audience members need three clients:
1. Main channel client (for game messages and video)
2. Agent A channel client (to hear Agent A)
3. Agent B channel client (to hear Agent B)

They join as "audience" role in agent channels, which means they can subscribe but not publish.

### Board State Synchronization

When an audience member joins, they send an `audience-joined` message. Both players respond by broadcasting their current board state:

```javascript
function handleAudienceJoined() {
  if (client) {
    const boardState = {
      type: "board-state",
      isPlayerA: isPlayerA,
      board: myBoardState,
      ships: myShips
    };
    client.sendStreamMessage(JSON.stringify(boardState));
  }
}
```

The audience receives both board states and renders them:

```javascript
function handleAudienceStreamMessage(uid, msgData) {
  const msg = JSON.parse(msgStr);
  
  if (msg.type === "board-state") {
    if (msg.isPlayerA) {
      myBoardState = msg.board;
      myShips = msg.ships;
      renderMyBoard();
    } else {
      enemyBoardState = msg.board;
      myShips2 = msg.ships;
      renderMyBoard2();
    }
  }
}
```

This works because:
1. Audience members don't modify game state, only render it
2. Players keep audience updated on every move via `board-state` messages
3. Late joiners get caught up by the initial `audience-joined` handshake

## Ship Placement: Pre-Game State

Before the game starts, players place ships on their board. This phase has different rules:

```javascript
function startShipPlacement() {
  isPlacingShips = true;
  currentShipType = 0;
  currentShipOrientation = 'horizontal';
  
  // Enable click handlers on player's own board
  const cells = myBoardEl.querySelectorAll("div");
  cells.forEach((cell, idx) => {
    const row = Math.floor(idx / 10);
    const col = idx % 10;
    cell.onclick = () => placeShip(row, col);
    cell.onmouseover = () => showPlacementPreview(row, col, true);
    cell.onmouseout = () => showPlacementPreview(row, col, false);
  });
}
```

Ships are placed sequentially—you must place all Destroyers before moving to Cruisers, etc. This simplifies UI logic:

```javascript
function placeShip(row, col) {
  if (!canPlaceShip(row, col)) {
    showNotification("Can't place ship here!");
    return;
  }
  
  const ship = SHIPS[currentShipType];
  
  // Mark cells as occupied
  for (let i = 0; i < ship.length; i++) {
    const r = currentShipOrientation === 'horizontal' ? row : row + i;
    const c = currentShipOrientation === 'horizontal' ? col + i : col;
    myShips[r][c] = true;
  }
  
  SHIPS[currentShipType].count--;
  if (SHIPS[currentShipType].count === 0) {
    currentShipType++; // Move to next ship type
  }
  shipsToPlace--;
  
  if (shipsToPlace === 0) {
    finishPlacement();
  }
}
```

### Ready State Synchronization

When a player finishes placement, they send a `ready` message:

```javascript
function finishPlacement() {
  isPlacingShips = false;
  
  const readyMsg = { type: "ready" };
  client.sendStreamMessage(JSON.stringify(readyMsg));
  
  if (otherPlayerReady) {
    startGame(); // Both ready, begin!
  } else {
    updateStatus("Waiting for opponent to finish placing ships...");
  }
}
```

The game doesn't start until both players are ready. This prevents one player from being attacked while still placing ships.

## Error Handling and Edge Cases

### Reconnection

If a player loses connection, Agora's SDK handles most of the heavy lifting:

```javascript
client.on("connection-state-change", (curState, prevState, reason) => {
  console.log(`Connection state: ${prevState} -> ${curState} (${reason})`);
  
  if (curState === "DISCONNECTED") {
    updateStatus("Connection lost. Attempting to reconnect...");
  } else if (curState === "CONNECTED") {
    updateStatus("Reconnected!");
    // Re-sync game state by requesting board updates
    const msgObj = { type: "audience-joined" }; // Reuse audience logic
    client.sendStreamMessage(JSON.stringify(msgObj));
  }
});
```

This isn't implemented in the current code, but it's worth adding for production use.

### Message Delivery Guarantees

Agora's data channel is **unreliable** by default (uses UDP). Messages can be lost or arrive out of order. For a turn-based game, this is mostly fine because:

1. Attacks are confirmed by results—if Player B never receives Player A's attack, Player A won't get a result, and they'll know something went wrong
2. Game state is synchronized frequently (every move)
3. The worst case is one lost move, not complete desynchronization

If you need guaranteed delivery, consider:
- Adding sequence numbers to messages
- Implementing ACK/NACK protocol
- Using Agora RTM (Real-Time Messaging) instead of data messages

### Invalid Moves

The game validates moves on both client and server side (remember, both players act as validators):

```javascript
function attackCell(row, col) {
  if (!isMyTurn) {
    showNotification("Not your turn!");
    agentSpeak(myAgentsId, "Wait your turn!");
    return;
  }
  
  if (enemyBoardState[row][col] !== 0) {
    showNotification("Cell already attacked!");
    agentSpeak(myAgentsId, "That area is already decimated!");
    return;
  }
  
  // Move is valid, proceed
  const msgObj = { type: "attack", row, col };
  client.sendStreamMessage(JSON.stringify(msgObj));
}
```

Even if Player A bypasses these checks (by modifying client code), Player B's `handleAttack` function will process the move correctly based on *their* board state. Player A can't force a hit where there's no ship.

## Performance Considerations

### Message Size Limits

Agora allows up to 30KB per message, but you want to stay well under that:

```javascript
// Good: ~50 bytes
{ type: "attack", row: 3, col: 7 }

// Bad: ~3KB
{ 
  type: "attack", 
  row: 3, 
  col: 7,
  attackerId: "player_a_12345",
  timestamp: "2024-01-15T10:30:00.000Z",
  metadata: { ... },
  fullBoardState: [...] // Don't send unnecessary data
}
```

Send only what's needed. Board state is 100 cells × 1 byte = 100 bytes. Ship positions are similar. You have plenty of headroom.

### Rendering Performance

Updating the board on every message could cause flicker:

```javascript
function renderMyBoard() {
  const cells = myBoardEl.querySelectorAll("div");
  
  cells.forEach((cell, idx) => {
    const row = Math.floor(idx / 10);
    const col = idx % 10;
    const val = myBoardState[row][col];
    const hasShip = myShips[row][col];
    
    // Only update cell className if it changed
    const newClass = `h-7 w-7 border border-gray-600 ${
      val === 0 ? (hasShip ? "bg-yellow-500" : "bg-gray-700") :
      val === 1 ? "bg-red-500" : "bg-blue-500"
    }`;
    
    if (cell.className !== newClass) {
      cell.className = newClass;
    }
  });
}
```

This approach checks if the className changed before updating, preventing unnecessary reflows. For a 10×10 board, this is overkill, but it's a good habit for larger games.

### Agent Response Latency

Voice commands introduce latency:
1. Player speaks (~1-2 seconds)
2. Audio streams to agent (~50-100ms)
3. Speech-to-text processing (~500ms)
4. GPT-4 processes command (~1-2 seconds)
5. Text-to-speech generation (~500ms)
6. Audio streams back (~50-100ms)

Total: **3-5 seconds** from speech to response.

You can improve this by:
- Using faster speech-to-text (Whisper API, Deepgram)
- Streaming TTS (play audio as it generates)
- Optimizing prompts to reduce GPT latency
- Using GPT-4 Turbo or GPT-3.5 instead of GPT-4

But remember: players can click coordinates directly as a fallback. Voice control is a feature, not a requirement.

## Extending This Pattern

This architecture works for any turn-based game:

### Chess
- Moves: `{ type: "move", from: "e2", to: "e4" }`
- State: 64 cells, piece positions
- Agents: "Move pawn to e4" → validates and executes

### Poker
- Moves: `{ type: "bet", amount: 50 }`, `{ type: "fold" }`
- State: cards, pot, player chips
- Agents: "I'll raise 50" → processes bet logic

### Tic-Tac-Toe
- Moves: `{ type: "mark", row: 1, col: 1 }`
- State: 9 cells
- Agents: "Bottom right" → converts to coordinates

The pattern scales to any game where:
1. Moves are small (<1KB)
2. Move frequency is low (<30/sec)
3. Clients can validate moves independently

## What About Cheating?

This architecture trusts clients. Player A could modify their JavaScript to:
- Claim every attack is a hit
- See Player B's ship positions
- Attack multiple times in a row

For a casual game, this is fine. For competitive play, you need a server referee:

```javascript
// Pseudocode for server-validated moves
async function attackCell(row, col) {
  const response = await fetch("/api/validate-move", {
    method: "POST",
    body: JSON.stringify({ gameId, playerId, row, col })
  });
  
  const { valid, result } = await response.json();
  
  if (valid) {
    // Broadcast move via Agora
    client.sendStreamMessage(JSON.stringify({
      type: "attack",
      row,
      col,
      serverSignature: result.signature // Proof server validated this
    }));
  }
}
```

The server acts as source of truth, clients just render the results. Agora still handles real-time communication, but game logic moves server-side.

## Deployment Checklist

Before going live, verify:

1. **Agora App ID**: Replace `a9a4b25e4e8b4a558aa39780d1a84342` with your own
2. **Token authentication**: Current code uses null token (development only)
3. **Lambda endpoints**: Update agent function URLs
4. **CORS headers**: Ensure Lambda functions allow your domain
5. **Rate limiting**: Prevent message spam (though Agora has built-in limits)
6. **Error boundaries**: Add try-catch around all Agora calls
7. **Logging**: Implement analytics for game events
8. **Mobile support**: Test touch controls and mobile browsers

### Token Authentication

For production, generate tokens server-side:

```javascript
// Backend endpoint
app.post("/api/get-token", (req, res) => {
  const { channelName, uid } = req.body;
  const token = RtcTokenBuilder.buildTokenWithUid(
    AGORA_APP_ID,
    AGORA_APP_CERTIFICATE,
    channelName,
    uid,
    RtcRole.PUBLISHER,
    Math.floor(Date.now() / 1000) + 3600 // 1 hour expiry
  );
  res.json({ token });
});

// Frontend
const response = await fetch("/api/get-token", {
  method: "POST",
  body: JSON.stringify({ channelName, uid })
});
const { token } = await response.json();
await client.join(AGORA_APP_ID, channelName, token, uid);
```

Never expose your App Certificate in client code.

## Common Pitfalls

### 1. Forgetting to Unsubscribe

When cleaning up, stop all tracks:

```javascript
async function leaveChannel() {
  if (localAudioTrack) {
    localAudioTrack.close();
    localAudioTrack = null;
  }
  if (localVideoTrack) {
    localVideoTrack.close();
    localVideoTrack = null;
  }
  if (remoteAudioTrack) {
    remoteAudioTrack.stop();
    remoteAudioTrack = null;
  }
  
  await client.leave();
  client = null;
}
```

Forgetting this causes memory leaks and keeps your camera/mic active.

### 2. Not Handling User-Published Events

If you don't subscribe to remote users, you won't see their video:

```javascript
client.on("user-published", async (user, mediaType) => {
  await client.subscribe(user, mediaType);
  
  if (mediaType === "video") {
    user.videoTrack.play("remoteVideo");
  } else if (mediaType === "audio") {
    user.audioTrack.play();
  }
});
```

This is easy to forget when testing with one browser. Always test with two separate browsers or devices.

### 3. Sending Messages Before Joining

```javascript
// Wrong: client not joined yet
client = AgoraRTC.createClient({...});
client.sendStreamMessage("hello"); // Error!

// Right: wait for join
client = AgoraRTC.createClient({...});
await client.join(...);
client.sendStreamMessage("hello"); // Works
```

Always `await client.join()` before sending messages.

### 4. Parsing Messages Without Try-Catch

```javascript
// Dangerous
function handleStreamMessage(uid, msgData) {
  const msg = JSON.parse(new TextDecoder().decode(msgData));
  // What if msgData is corrupted?
}

// Safe
function handleStreamMessage(uid, msgData) {
  try {
    const msgStr = new TextDecoder().decode(msgData);
    const msg = JSON.parse(msgStr);
    // Process msg
  } catch (e) {
    console.error("Invalid message:", e);
  }
}
```

Always validate message structure before using it.

## Conclusion: When to Use This Pattern

Use Agora's data messaging for games when:
- ✅ You already need video/audio communication
- ✅ Moves are infrequent (<10/sec per player)
- ✅ Players are in the same channel anyway
- ✅ You want to minimize backend infrastructure

Don't use it when:
- ❌ You need guaranteed message delivery with ACKs
- ❌ Game state is very large (>10KB per update)
- ❌ You require server-authoritative validation
- ❌ Moves happen faster than network latency allows

This Battleship game hits the sweet spot: real-time communication + simple game state + voice interaction. The same pattern would work for chess, card games, trivia games—anything turn-based where players already want to see each other.

The real win? You build one thing—a video call—and get multiplayer game sync for free.

---

## Additional Resources

- [Agora Web SDK Documentation](https://docs.agora.io/en/video-calling/get-started/get-started-sdk)
- [Data Messaging API Reference](https://api-ref.agora.io/en/video-sdk/web/4.x/interfaces/iagorartcclient.html#sendstreammessage)
- [Agora Token Authentication](https://docs.agora.io/en/video-calling/get-started/authentication-workflow)
- [Building Voice Agents with Agora](https://docs.agora.io/en/voice-ai/)

## Sample Code Repository

Full source code for this Battleship game: [link to your repo]

Questions? Find me on [Twitter/X] or the [Agora Developer Discord].

