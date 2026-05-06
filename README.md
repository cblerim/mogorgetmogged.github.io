<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MOG ARENA</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.socket.io/4.7.5/socket.io.min.js"></script>
    <link href="https://fonts.googleapis.com/css2?family=Cinzel:wght@400;700;900&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Cinzel', serif; }
        .scan-line::after {
            content: '';
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background: linear-gradient(transparent, rgba(185, 28, 28, 0.25), transparent);
            animation: scan 3s linear infinite;
        }
        @keyframes scan { 0% { transform: translateY(-100%); } 100% { transform: translateY(400%); } }
    </style>
</head>
<body class="bg-zinc-950 text-zinc-100 min-h-screen">

<div class="max-w-6xl mx-auto p-6">

    <!-- Landing -->
    <div id="landing" class="min-h-screen flex flex-col items-center justify-center text-center">
        <h1 class="text-7xl font-black tracking-tighter mb-3">MOG<span class="text-red-600">ARENA</span></h1>
        <p class="text-red-500 mb-12">Real Multiplayer Blackpill Arena</p>
        
        <div class="bg-zinc-900 p-8 rounded-3xl w-full max-w-md border border-red-900/50">
            <input id="username" placeholder="YOUR HANDLE (e.g. jawletdestroyer)" 
                   class="w-full bg-black border border-zinc-700 rounded-2xl px-6 py-5 mb-6 text-lg outline-none">
            <div class="grid grid-cols-2 gap-4">
                <button onclick="createLobby()" 
                        class="py-6 bg-red-700 hover:bg-red-600 rounded-2xl font-bold text-xl transition">CREATE LOBBY</button>
                <button onclick="joinLobbyPrompt()" 
                        class="py-6 bg-zinc-800 hover:bg-zinc-700 border border-zinc-600 rounded-2xl font-bold text-xl transition">JOIN LOBBY</button>
            </div>
        </div>
    </div>

    <!-- Lobby Screen -->
    <div id="lobbyScreen" class="hidden min-h-screen grid grid-cols-12 gap-6">
        <div class="col-span-3 bg-zinc-900 p-6 rounded-3xl border border-red-900/30">
            <div class="flex justify-between mb-6">
                <span class="text-red-500">LOBBY <span id="roomCode" class="font-mono text-xl"></span></span>
                <button onclick="leaveLobby()" class="text-xs px-4 py-2 border border-red-900 rounded-xl">LEAVE</button>
            </div>
            <div id="playerList" class="space-y-3"></div>
        </div>

        <div class="col-span-6 flex flex-col items-center justify-center">
            <div class="relative scan-line w-96 h-96">
                <video id="video" class="w-96 h-96 object-cover rounded-3xl border-4 border-red-900" autoplay playsinline></video>
            </div>
            <button onclick="startScan()" id="scanBtn"
                    class="mt-10 px-16 py-6 bg-red-700 hover:bg-red-600 text-2xl font-bold rounded-2xl transition">
                START 10S MOG SCAN
            </button>
        </div>

        <div class="col-span-3 bg-zinc-900 p-6 rounded-3xl border border-red-900/30">
            <div id="log" class="font-mono text-sm h-96 overflow-auto"></div>
        </div>
    </div>

    <!-- Results -->
    <div id="resultsScreen" class="hidden min-h-screen p-8">
        <h2 class="text-5xl font-black text-center mb-12">FINAL MOG RANKINGS</h2>
        <div id="leaderboard" class="max-w-5xl mx-auto grid grid-cols-1 md:grid-cols-3 gap-6"></div>
        <div class="text-center mt-12">
            <button onclick="location.reload()" class="px-12 py-6 bg-red-700 hover:bg-red-600 text-2xl font-bold rounded-2xl">NEW BATTLE</button>
        </div>
    </div>
</div>

<script>
    const socket = io('https://YOUR-BACKEND-URL.up.railway.app'); // ← Change this later

    let username = "";
    let currentLobby = "";

    function createLobby() {
        username = document.getElementById("username").value.trim() || "ANONMOG";
        socket.emit("createLobby", username);
    }

    function joinLobbyPrompt() {
        const code = prompt("Enter Lobby Code:").toUpperCase();
        if (!code) return;
        username = document.getElementById("username").value.trim() || "ANONMOG";
        socket.emit("joinLobby", { code, username });
    }

    socket.on("lobbyCreated", (data) => {
        currentLobby = data.code;
        document.getElementById("landing").classList.add("hidden");
        document.getElementById("lobbyScreen").classList.remove("hidden");
        document.getElementById("roomCode").textContent = data.code;
        updatePlayers(data.players);
        startCamera();
    });

    socket.on("playerUpdate", updatePlayers);

    function updatePlayers(players) {
        const div = document.getElementById("playerList");
        div.innerHTML = players.map(p => `
            <div class="bg-zinc-800 p-4 rounded-2xl flex gap-4 items-center border ${p.score ? 'border-green-500' : 'border-zinc-700'}">
                <span class="text-3xl">👤</span>
                <div>
                    <div class="font-medium">${p.name}</div>
                    <div class="text-xs ${p.score ? 'text-green-400' : 'text-zinc-500'}">
                        ${p.score ? p.score + '% MOG' : 'Waiting...'}
                    </div>
                </div>
            </div>
        `).join('');
    }

    async function startCamera() {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true });
        document.getElementById("video").srcObject = stream;
    }

    function startScan() {
        let timeLeft = 10;
        let scores = [];
        const log = document.getElementById("log");
        log.innerHTML = "<div class='text-red-400'>SCAN STARTED...</div>";

        const interval = setInterval(() => {
            timeLeft--;
            const score = Math.floor(45 + Math.random() * 50);
            scores.push(score);

            if (timeLeft <= 0) {
                clearInterval(interval);
                const finalScore = Math.floor(scores.reduce((a,b) => a+b,0) / scores.length);
                socket.emit("submitScore", { code: currentLobby, score: finalScore });
            }
        }, 900);
    }

    socket.on("allSubmitted", (players) => {
        document.getElementById("lobbyScreen").classList.add("hidden");
        document.getElementById("resultsScreen").classList.remove("hidden");
        
        const sorted = players.sort((a,b) => (b.score||0) - (a.score||0));
        const container = document.getElementById("leaderboard");
        container.innerHTML = "";

        sorted.forEach(p => {
            container.innerHTML += `
                <div class="bg-zinc-900 border border-red-900/50 rounded-3xl p-8 text-center">
                    <div class="text-6xl font-black text-red-500">${p.score || 0}</div>
                    <div class="text-sm text-zinc-400">MOG</div>
                    <div class="mt-6 text-2xl">${p.name}</div>
                </div>
            `;
        });
    });

    function leaveLobby() {
        if (confirm("Leave the arena?")) location.reload();
    }
</script>
</body>
</html>
