const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const path = require('path');

const app = express();
const server = http.createServer(app);
const io = socketIo(server, {
    cors: {
        origin: "*",
        methods: ["GET", "POST"]
    }
});

// Store active rooms and users
const rooms = new Map();
const users = new Map();

// Serve static files
app.use(express.static(path.join(__dirname, 'public')));

// Generate random gamer tag
function generateGamerTag() {
    const prefixes = ['Pro', 'Elite', 'Shadow', 'Stealth', 'Cyber', 'Neo', 'Alpha', 'Omega'];
    const suffixes = ['Slayer', 'Hunter', 'Warrior', 'Sniper', 'Ghost', 'Phoenix', 'Dragon', 'Wolf'];
    const prefix = prefixes[Math.floor(Math.random() * prefixes.length)];
    const suffix = suffixes[Math.floor(Math.random() * suffixes.length)];
    return `${prefix}${suffix}`;
}

io.on('connection', (socket) => {
    console.log('🟢 New gamer connected:', socket.id);

    // Join gaming room
    socket.on('join-room', (roomId, userData) => {
        const room = roomId || 'global-gaming';
        const gamerTag = userData?.gamerTag || generateGamerTag();
        
        socket.join(room);
        
        // Store user info
        users.set(socket.id, {
            id: socket.id,
            gamerTag: gamerTag,
            room: room,
            isStreaming: false
        });

        // Initialize room if not exists
        if (!rooms.has(room)) {
            rooms.set(room, new Set());
        }
        rooms.get(room).add(socket.id);

        // Notify room
        const roomUsers = Array.from(rooms.get(room)).map(id => users.get(id));
        io.to(room).emit('user-joined', {
            gamerTag: gamerTag,
            users: roomUsers,
            onlineCount: roomUsers.length
        });

        console.log(`🎮 ${gamerTag} joined room: ${room}`);
    });

    // Handle WebRTC signaling
    socket.on('offer', (data) => {
        socket.to(data.target).emit('offer', {
            offer: data.offer,
            sender: socket.id,
            gamerTag: users.get(socket.id)?.gamerTag
        });
    });

    socket.on('answer', (data) => {
        socket.to(data.target).emit('answer', {
            answer: data.answer,
            sender: socket.id
        });
    });

    socket.on('ice-candidate', (data) => {
        socket.to(data.target).emit('ice-candidate', {
            candidate: data.candidate,
            sender: socket.id
        });
    });

    // Handle chat messages
    socket.on('send-message', (data) => {
        const user = users.get(socket.id);
        if (user) {
            io.to(user.room).emit('new-message', {
                gamerTag: user.gamerTag,
                message: data.message,
                timestamp: new Date().toISOString(),
                type: 'chat'
            });
        }
    });

    // Handle game commands
    socket.on('game-command', (data) => {
        const user = users.get(socket.id);
        if (user) {
            io.to(user.room).emit('game-command', {
                gamerTag: user.gamerTag,
                command: data.command,
                target: data.target,
                timestamp: new Date().toISOString()
            });
        }
    });

    // Stream status
    socket.on('stream-started', () => {
        const user = users.get(socket.id);
        if (user) {
            user.isStreaming = true;
            socket.to(user.room).emit('user-streaming', {
                gamerTag: user.gamerTag,
                isStreaming: true
            });
        }
    });

    // Handle disconnect
    socket.on('disconnect', () => {
        const user = users.get(socket.id);
        if (user) {
            const room = rooms.get(user.room);
            if (room) {
                room.delete(socket.id);
                
                // Notify room
                socket.to(user.room).emit('user-left', {
                    gamerTag: user.gamerTag,
                    onlineCount: room.size
                });

                // Clean up empty rooms
                if (room.size === 0) {
                    rooms.delete(user.room);
                }
            }
            
            users.delete(socket.id);
            console.log(`🔴 ${user.gamerTag} disconnected`);
        }
    });

    // Ping to keep connection alive
    socket.on('ping', () => {
        socket.emit('pong', { timestamp: Date.now() });
    });
});

// API endpoints
app.get('/api/stats', (req, res) => {
    res.json({
        totalUsers: users.size,
        totalRooms: rooms.size,
        timestamp: new Date().toISOString()
    });
});

app.get('/api/rooms', (req, res) => {
    const roomData = {};
    rooms.forEach((users, roomId) => {
        roomData[roomId] = {
            userCount: users.size,
            users: Array.from(users).map(id => users.get(id)?.gamerTag).filter(Boolean)
        };
    });
    res.json(roomData);
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`🎮 YO GAMERS server running on port ${PORT}`);
    console.log(`🌐 Access at: http://localhost:${PORT}`);
});
An online video or audio chat room for gamers.
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>YO GAMERS 🎮 - Online</title>
    <script src="/socket.io/socket.io.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #0f0c29, #302b63, #24243e);
            color: #fff;
            min-height: 100vh;
            padding: 15px;
        }
        
        .container {
            max-width: 100%;
            margin: 0 auto;
        }
        
        .header {
            text-align: center;
            margin-bottom: 25px;
            padding: 20px;
            background: linear-gradient(135deg, #ff6b6b, #ffa726);
            border-radius: 20px;
            box-shadow: 0 10px 30px rgba(255, 107, 107, 0.3);
        }
        
        .logo {
            font-size: 2.5em;
            font-weight: bold;
            margin-bottom: 5px;
        }
        
        .tagline {
            font-size: 1.1em;
            opacity: 0.9;
        }
        
        .online-indicator {
            display: inline-block;
            width: 10px;
            height: 10px;
            background: #00ff88;
            border-radius: 50%;
            margin-right: 8px;
            animation: pulse 2s infinite;
        }
        
        @keyframes pulse {
            0% { opacity: 1; }
            50% { opacity: 0.5; }
            100% { opacity: 1; }
        }
        
        .status {
            text-align: center;
            padding: 15px;
            margin: 15px 0;
            border-radius: 15px;
            background: rgba(255,255,255,0.1);
            border: 2px solid #ff6b6b;
        }
        
        .video-grid {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 10px;
            margin-bottom: 20px;
        }
        
        .video-container {
            position: relative;
            border-radius: 15px;
            overflow: hidden;
            background: #000;
            aspect-ratio: 4/3;
        }
        
        video {
            width: 100%;
            height: 100%;
            object-fit: cover;
        }
        
        .video-label {
            position: absolute;
            bottom: 5px;
            left: 5px;
            background: rgba(0,0,0,0.7);
            padding: 5px 10px;
            border-radius: 10px;
            font-size: 0.8em;
        }
        
        .controls {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 10px;
            margin-bottom: 20px;
        }
        
        .btn {
            padding: 15px;
            border: none;
            border-radius: 10px;
            font-size: 16px;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s;
        }
        
        .btn:active {
            transform: scale(0.95);
        }
        
        .btn-primary {
            background: #00ff88;
            color: #000;
        }
        
        .btn-danger {
            background: #ff4757;
            color: white;
        }
        
        .chat-container {
            background: rgba(255,255,255,0.1);
            border-radius: 15px;
            padding: 15px;
            margin-bottom: 15px;
        }
        
        #chat-box {
            height: 200px;
            overflow-y: auto;
            margin-bottom: 15px;
            padding: 10px;
            background: rgba(0,0,0,0.3);
            border-radius: 10px;
        }
        
        .message {
            margin-bottom: 10px;
            padding: 8px 12px;
            border-radius: 10px;
            animation: slideIn 0.3s ease;
        }
        
        @keyframes slideIn {
            from { opacity: 0; transform: translateX(-10px); }
            to { opacity: 1; transform: translateX(0); }
        }
        
        .message.you {
            background: rgba(0,255,136,0.2);
            margin-left: 20px;
        }
        
        .message.friend {
            background: rgba(255,255,255,0.1);
            margin-right: 20px;
        }
        
        .message.system {
            background: rgba(255,193,7,0.2);
            text-align: center;
            font-style: italic;
        }
        
        .input-group {
            display: flex;
            gap: 10px;
        }
        
        #messageInput {
            flex: 1;
            padding: 12px;
            border: none;
            border-radius: 10px;
            background: rgba(255,255,255,0.9);
        }
        
        .users-online {
            background: rgba(255,255,255,0.1);
            padding: 15px;
            border-radius: 15px;
        }
        
        .user-list {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            margin-top: 10px;
        }
        
        .user-tag {
            background: rgba(0,255,136,0.2);
            padding: 5px 10px;
            border-radius: 15px;
            font-size: 0.9em;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <div class="logo">YO GAMERS 🎮</div>
            <div class="tagline">
                <span class="online-indicator"></span>
                ONLINE MULTIPLAYER
            </div>
        </div>
        
        <div class="status" id="status">
            🔴 Connecting to YO GAMERS network...
        </div>
        
        <div class="video-grid">
            <div class="video-container">
                <video id="localVideo" autoplay muted playsinline></video>
                <div class="video-label" id="localLabel">YOU</div>
            </div>
            <div class="video-container">
                <video id="remoteVideo" autoplay playsinline></video>
                <div class="video-label" id="remoteLabel">SQUAD MATE</div>
            </div>
        </div>
        
        <div class="controls">
            <button class="btn btn-primary" onclick="joinRoom()">🎮 JOIN SQUAD</button>
            <button class="btn btn-danger" onclick="leaveRoom()">🛑 LEAVE</button>
            <button class="btn" onclick="toggleAudio()" id="audioBtn">🎤 MUTE</button>
            <button class="btn" onclick="toggleVideo()" id="videoBtn">📹 VIDEO</button>
        </div>
        
        <div class="chat-container">
            <h3>💬 LIVE SQUAD CHAT</h3>
            <div id="chat-box"></div>
            <div class="input-group">
                <input type="text" id="messageInput" placeholder="Chat with your squad...">
                <button class="btn btn-primary" onclick="sendMessage()">SEND</button>
            </div>
        </div>
        
        <div class="users-online">
            <h3>👥 ONLINE GAMERS (<span id="onlineCount">0</span>)</h3>
            <div class="user-list" id="userList">
                <!-- Online users will appear here -->
            </div>
        </div>
    </div>

    <script>
        // Socket.io connection
        const socket = io();
        
        // WebRTC configuration
        const configuration = {
            iceServers: [
                { urls: 'stun:stun.l.google.com:19302' },
                { urls: 'stun:stun1.l.google.com:19302' }
            ]
        };
        
        let localStream = null;
        let peerConnection = null;
        let currentRoom = 'global-gaming';
        let gamerTag = 'Player' + Math.floor(Math.random() * 1000);
        
        // DOM elements
        const localVideo = document.getElementById('localVideo');
        const remoteVideo = document.getElementById('remoteVideo');
        const localLabel = document.getElementById('localLabel');
        const remoteLabel = document.getElementById('remoteLabel');
        const statusDiv = document.getElementById('status');
        const chatBox = document.getElementById('chat-box');
        const messageInput = document.getElementById('messageInput');
        const onlineCount = document.getElementById('onlineCount');
        const userList = document.getElementById('userList');
        const audioBtn = document.getElementById('audioBtn');
        const videoBtn = document.getElementById('videoBtn');
        
        // Socket event handlers
        socket.on('connect', () => {
            updateStatus('🟢 Connected to YO GAMERS!', 'success');
            addSystemMessage('Connected to online gaming network');
        });
        
        socket.on('disconnect', () => {
            updateStatus('🔴 Disconnected from server', 'error');
            addSystemMessage('Lost connection to gaming network');
        });
        
        socket.on('user-joined', (data) => {
            updateOnlineCount(data.onlineCount);
            updateUserList(data.users);
            addSystemMessage(`🎮 ${data.gamerTag} joined the squad!`);
        });
        
        socket.on('user-left', (data) => {
            updateOnlineCount(data.onlineCount);
            addSystemMessage(`👋 ${data.gamerTag} left the game`);
        });
        
        socket.on('new-message', (data) => {
            addMessage(data.gamerTag, data.message, 'friend');
        });
        
        // WebRTC signaling
        socket.on('offer', async (data) => {
            await createPeerConnection();
            await peerConnection.setRemoteDescription(data.offer);
            const answer = await peerConnection.createAnswer();
            await peerConnection.setLocalDescription(answer);
            
            socket.emit('answer', {
                target: data.sender,
                answer: answer
            });
        });
        
        socket.on('answer', async (data) => {
            await peerConnection.setRemoteDescription(data.answer);
        });
        
        socket.on('ice-candidate', async (data) => {
            await peerConnection.addIceCandidate(data.candidate);
        });
        
        // Join gaming room
        async function joinRoom() {
            try {
                // Get user media
                localStream = await navigator.mediaDevices.getUserMedia({
                    video: true,
                    audio: true
                });
                
                localVideo.srcObject = localStream;
                localLabel.textContent = gamerTag;
                
                // Join socket room
                socket.emit('join-room', currentRoom, { gamerTag: gamerTag });
                
                updateStatus('🟢 LIVE - In squad chat!', 'success');
                addSystemMessage('You joined the gaming squad!');
                
            } catch (error) {
                updateStatus('❌ Cannot access camera/microphone', 'error');
                console.error('Media error:', error);
            }
        }
        
        // Leave room
        function leaveRoom() {
            if (localStream) {
                localStream.getTracks().forEach(track => track.stop());
                localStream = null;
            }
            localVideo.srcObject = null;
            remoteVideo.srcObject = null;
            
            updateStatus('🔴 Left the squad', 'error');
            addSystemMessage('You left the gaming squad');
        }
        
        // Create peer connection
        async function createPeerConnection() {
            peerConnection = new RTCPeerConnection(configuration);
            
            // Add local stream
            if (localStream) {
                localStream.getTracks().forEach(track => {
                    peerConnection.addTrack(track, localStream);
                });
            }
            
            // Handle remote stream
            peerConnection.ontrack = (event) => {
                remoteVideo.srcObject = event.streams[0];
            };
            
            // Handle ICE candidates
            peerConnection.onicecandidate = (event) => {
                if (event.candidate) {
                    socket.emit('ice-candidate', {
                        target: socket.id, // In real app, target specific user
                        candidate: event.candidate
                    });
                }
            };
        }
        
        // Toggle audio
        function toggleAudio() {
            if (localStream) {
                const audioTracks = localStream.getAudioTracks();
                audioTracks.forEach(track => {
                    track.enabled = !track.enabled;
                });
                audioBtn.textContent = audioTracks[0].enabled ? '🎤 MUTE' : '🎤 UNMUTE';
            }
        }
        
        // Toggle video
        function toggleVideo() {
            if (localStream) {
                const videoTracks = localStream.getVideoTracks();
                videoTracks.forEach(track => {
                    track.enabled = !track.enabled;
                });
                videoBtn.textContent = videoTracks[0].enabled ? '📹 VIDEO' : '📹 SHOW';
            }
        }
        
        // Send chat message
        function sendMessage() {
            const message = messageInput.value.trim();
            if (message) {
                socket.emit('send-message', { message: message });
                addMessage('YOU', message, 'you');
                messageInput.value = '';
            }
        }
        
        // Add message to chat
        function addMessage(sender, message, type) {
            const messageDiv = document.createElement('div');
            messageDiv.className = `message ${type}`;
            messageDiv.innerHTML = `<strong>${sender}:</strong> ${message}`;
            chatBox.appendChild(messageDiv);
            chatBox.scrollTop = chatBox.scrollHeight;
        }
        
        // Add system message
        function addSystemMessage(message) {
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message system';
            messageDiv.textContent = message;
            chatBox.appendChild(messageDiv);
            chatBox.scrollTop = chatBox.scrollHeight;
        }
        
        // Update status
        function updateStatus(message, type) {
            statusDiv.textContent = message;
            statusDiv.style.background = type === 'success' ? 'rgba(0,255,0,0.2)' : 
                                       type === 'error' ? 'rgba(255,0,0,0.2)' : 
                                       'rgba(255,255,0,0.2)';
        }
        
        // Update online count
        function updateOnlineCount(count) {
            onlineCount.textContent = count;
        }
        
        // Update user list
        function updateUserList(users) {
            userList.innerHTML = '';
            users.forEach(user => {
                const userElement = document.createElement('div');
                userElement.className = 'user-tag';
                userElement.textContent = user.gamerTag;
                userList.appendChild(userElement);
            });
        }
        
        // Enter key to send message
        messageInput.addEventListener('keypress', (e) => {
            if (e.key === 'Enter') sendMessage();
        });
        
        // Initial setup
        addSystemMessage('Welcome to YO GAMERS Online! 🎮');
        addSystemMessage('Click "JOIN SQUAD" to start gaming with others!');
    </script>
</body>
</html>
