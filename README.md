# Battleship Agora AI

A real-time multiplayer Battleship game powered by Agora RTC with AI voice-controlled agents. Players can attack by voice commands, see each other via video, and spectators can watch live gameplay.

![Battleship Agora](https://img.shields.io/badge/Agora-RTC-blue)
![License](https://img.shields.io/badge/license-MIT-green)

## 🎮 Features

- **Real-time Multiplayer**: Two players compete in classic Battleship with live video/audio
- **AI Voice Control**: Voice-activated AI agents respond to spoken attack commands
  - Say "Attack B4" to fire at coordinates
  - Say "Choose for me" for AI-suggested moves
- **Live Video Feeds**: See your opponent while playing
- **Spectator Mode**: Audience members can join and watch ongoing games
- **Interactive Ship Placement**: Visual ship placement with rotation support
- **Turn-Based Gameplay**: Clear turn indicators with visual feedback
- **Agent Message Log**: Press 'U' to view AI agent conversation history

## 🚀 Quick Start

### Prerequisites

- Modern web browser (Chrome, Firefox, Edge, Safari)
- Agora App ID ([Get one free here](https://console.agora.io/))
- Microphone and camera permissions

### Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/AgoraIO-Community/BattleshipAgora.git
   cd BattleshipAgora
   ```

2. **Configure Agora App ID**
   
   Open `index.html` and replace the App ID on line 196:
   ```javascript
   const AGORA_APP_ID = "YOUR_APP_ID_HERE";
   ```

3. **Serve the application**
   
   Use any static file server:
   ```bash
   # Using Python 3
   python -m http.server 8000
   
   # Using Node.js
   npx serve
   
   # Or just open index.html directly in your browser
   ```

4. **Open in browser**
   ```
   http://localhost:8000
   ```

## 🎯 How to Play

### Starting a Game

**Player A (Host):**
1. Click **"Start (Player A)"**
2. Place your ships on the board
3. Wait for Player B to join

**Player B (Joiner):**
1. Click **"Join (Player B)"**
2. Select an available game from the list
3. Place your ships on the board

### Gameplay

- **Voice Commands**: Say "Attack [COORDINATES]" (e.g., "Attack B4")
- **Click to Attack**: Click enemy board cells directly
- **Turn Indicators**: Green outline = your turn, Red outline = opponent's turn
- **View Agent Log**: Press 'U' to toggle AI message history

### Spectating

1. Click **"Watch"**
2. Choose an ongoing game
3. Watch both players' boards in real-time

## 🏗️ Architecture

This game uses a unique multi-channel architecture:

```
Main Channel (battleship_123456)
├── Player A & Player B video/audio
├── Game state synchronization
└── Spectator feeds

Agent Channel A (battleship_123456_agenta)
└── Player A ↔ AI Agent A

Agent Channel B (battleship_123456_agentb)
└── Player B ↔ AI Agent B
```

### Key Components

- **Data Messaging**: Game moves synchronized via Agora's `sendStreamMessage` API
- **Voice Agents**: Serverless Lambda functions process voice commands
- **Real-time Video**: Live video feeds using Agora Web SDK
- **Turn Management**: Client-side turn validation with server-authoritative agent responses

## 🛠️ Technologies

- **[Agora Web SDK 4.x](https://docs.agora.io/en/video-calling/get-started/get-started-sdk)** - Real-time communication
- **[Agora Conversational AI](https://docs.agora.io/en/voice-ai/)** - Voice-controlled agents
- **Vanilla JavaScript** - No framework dependencies
- **Tailwind CSS** - Styling via CDN
- **AWS Lambda** - Serverless agent backend

## 📝 Configuration

### Agent Endpoints

The game uses several Lambda functions for agent management (lines 198-200 in `index.html`):

```javascript
const LIST_CHANNELS_ENDPOINT = "YOUR_ENDPOINT";
const LIST_CHANNELS_ENDPOINT_TWO_USERS = "YOUR_ENDPOINT";
```

You can deploy your own agent functions or use the provided endpoints for testing.

### Ships Configuration

The game uses standard Battleship ship sizes:
- **Destroyer** (2 cells) × 2
- **Cruiser** (3 cells) × 2
- **Submarine** (3 cells) × 2
- **Battleship** (4 cells) × 2
- **Carrier** (5 cells) × 2

## 📖 Documentation

For a detailed technical guide on how this game works, see [GUIDE.md](./GUIDE.md) which covers:
- Game state synchronization
- Voice agent integration
- Multi-channel architecture
- Message protocols
- Error handling
- Performance optimization

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request. For major changes, please open an issue first to discuss what you would like to change.

### Development

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🙏 Acknowledgments

- Built with [Agora.io](https://www.agora.io) real-time engagement platform
- Part of the [AgoraIO-Community](https://github.com/AgoraIO-Community) projects

## 📧 Support

- 💬 [Agora Developer Slack](https://www.agora.io/en/join-slack/)
- 📚 [Agora Documentation](https://docs.agora.io)
- 🐛 [Report Issues](https://github.com/AgoraIO-Community/BattleshipAgora/issues)

---

Made with ❤️ by the Agora Community