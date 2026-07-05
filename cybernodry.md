<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Cyber Parkour World - Fullscreen</title>
    
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js"></script>

    <style>
        :root {
            --accent: #00ffcc;
        }

        * { margin: 0; padding: 0; box-sizing: border-box; font-family: sans-serif; user-select: none; }

        body, html {
            background: #14141f;
            width: 100%;
            height: 100%;
            overflow: hidden;
            position: relative;
        }

        /* JOGO OCUPANDO 100% DA TELA */
        .game-wrapper {
            width: 100vw;
            height: 100vh;
            position: relative;
        }

        canvas {
            width: 100%;
            height: 100%;
            display: block;
        }

        /* CONTROLES SEPARADOS NOS EXTREMOS CANTOS DA TELA */
        .controls-mobile {
            position: absolute;
            bottom: 40px;
            left: 0;
            right: 0;
            padding: 0 40px;
            display: flex;
            justify-content: space-between;
            pointer-events: none;
            z-index: 20;
        }

        .d-pad { display: flex; gap: 25px; pointer-events: auto; }
        .action-pad { pointer-events: auto; }
        
        .btn-game {
            width: 90px;
            height: 90px;
            background: rgba(255, 255, 255, 0.15);
            border: 3px solid rgba(255, 255, 255, 0.4);
            border-radius: 50%;
            color: white;
            font-size: 2.2rem;
            font-weight: bold;
            display: flex;
            align-items: center;
            justify-content: center;
            backdrop-filter: blur(5px);
            cursor: pointer;
            touch-action: none;
        }
        .btn-game:active { background: var(--accent); color: #000; border-color: var(--accent); }

        /* CHAT ESTILO HUD DE JOGO (SOBREPOSTO NO CANTO SUPERIOR) */
        .chat-hud {
            position: absolute;
            top: 20px;
            left: 20px;
            width: 320px;
            height: 260px;
            background: rgba(20, 20, 31, 0.6);
            backdrop-filter: blur(4px);
            border-radius: 8px;
            border: 1px solid rgba(255, 255, 255, 0.1);
            display: flex;
            flex-direction: column;
            z-index: 15;
            pointer-events: auto;
        }

        .chat-messages {
            flex: 1;
            padding: 10px;
            overflow-y: auto;
            display: flex;
            flex-direction: column;
            gap: 6px;
        }

        .chat-msg {
            background: rgba(0, 0, 0, 0.3);
            padding: 6px 10px;
            border-radius: 4px;
            font-size: 0.85rem;
            color: #eee;
            word-break: break-all;
        }
        .chat-msg .name { font-weight: bold; margin-right: 4px; }

        .chat-input-box {
            padding: 8px;
            background: rgba(0,0,0,0.4);
            border-radius: 0 0 8px 8px;
        }

        .chat-input-box input {
            width: 100%;
            padding: 8px 12px;
            border: none;
            background: rgba(255,255,255,0.1);
            color: white;
            border-radius: 4px;
            outline: none;
            font-size: 0.9rem;
        }
        .chat-input-box input::placeholder { color: #aaa; }

        /* MODAL DE NOME */
        #nameModal {
            position: fixed; top:0; left:0; width:100vw; height:100vh;
            background: rgba(10, 10, 15, 0.95); display: flex; align-items: center; justify-content: center;
            z-index: 999;
        }
        .modal-content {
            background: #2f3136; padding: 40px; border-radius: 12px; text-align: center;
            box-shadow: 0 4px 20px rgba(0,0,0,0.7); width: 90%; max-width: 400px;
        }
        .modal-content input {
            padding: 12px; border-radius: 6px; border: none; margin-bottom: 20px; width: 100%; text-align: center; font-size: 1.1rem; background: #40444b; color: white; outline: none;
        }
        .modal-content button {
            background: var(--accent); color: black; border: none; padding: 12px 24px; font-weight: bold; font-size: 1rem; border-radius: 6px; cursor: pointer; width: 100%;
        }
    </style>
</head>
<body>

    <div id="nameModal">
        <div class="modal-content">
            <h2 style="color: white; margin-bottom: 10px;">Cyber Parkour World</h2>
            <p style="color: #aaa; font-size: 0.9rem; margin-bottom: 20px;">Entre e jogue com seus amigos</p>
            <input type="text" id="usernameInput" placeholder="Digite seu apelido" maxlength="12">
            <button onclick="startGame()">JOGAR EM TELA CHEIA</button>
        </div>
    </div>

    <div class="game-wrapper">
        <canvas id="gameCanvas"></canvas>
        
        <div class="chat-hud">
            <div class="chat-messages" id="chatMessages"></div>
            <div class="chat-input-box">
                <input type="text" id="chatInput" placeholder="Pressione Enter/Toque para falar..." maxlength="60">
            </div>
        </div>

        <div class="controls-mobile">
            <div class="d-pad">
                <div class="btn-game" id="btnLeft">◀</div>
                <div class="btn-game" id="btnRight">▶</div>
            </div>
            <div class="action-pad">
                <div class="btn-game" id="btnJump">↑</div>
            </div>
        </div>
    </div>

    <script>
        const firebaseConfig = {
            apiKey: "AIzaSyDegW2sEjWvx3Ih7U6vs45Gp8PeXyfJ90w",
            authDomain: "cyber-a41d9.firebaseapp.com",
            projectId: "cyber-a41d9",
            storageBucket: "cyber-a41d9.appspot.com",
            messagingSenderId: "563443982918",
            appId: "1:563443982918:web:61cf4b88f1b4ddd89ee047",
            databaseURL: "https://cyber-a41d9-default-rtdb.firebaseio.com/"
        };
        
        firebase.initializeApp(firebaseConfig);
        const database = firebase.database();
        const playersRef = database.ref('game_players');
        const chatRef = database.ref('game_chat');

        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // Dimensão fixa do mapa de parkour
        const WORLD_WIDTH = 2000;
        const WORLD_HEIGHT = 950;

        let myId = 'p_' + Math.floor(Math.random() * 999999);
        let myName = "";
        let gameStarted = false;

        let player = {
            x: 150, y: 750,
            width: 40, height: 75,
            vx: 0, vy: 0,
            color: `hsl(${Math.random() * 360}, 85%, 60%)`,
            isGrounded: false,
            lastMsg: "",
            msgTimer: 0,
            animFrame: 0
        };

        let players = {};

        // Plataformas distribuídas ao longo de todo o mapa panorâmico
        const platforms = [
            { x: 0, y: 880, width: WORLD_WIDTH, height: 70 }, // Piso base
            { x: 250, y: 730, width: 200, height: 20 },
            { x: 520, y: 600, width: 180, height: 20 },
            { x: 800, y: 480, width: 180, height: 20 },
            { x: 1100, y: 380, width: 220, height: 20 },
            { x: 1400, y: 260, width: 180, height: 20 },
            { x: 1150, y: 150, width: 180, height: 20 },
            { x: 850, y: 160, width: 220, height: 20 },
            { x: 500, y: 220, width: 250, height: 20 }
        ];

        let keys = { left: false, right: false, jump: false };

        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        function startGame() {
            const input = document.getElementById('usernameInput');
            myName = input.value.trim() || "User" + Math.floor(Math.random()*999);
            document.getElementById('nameModal').style.display = 'none';
            gameStarted = true;
            
            updateFirebase();
            initNetwork();
            requestAnimationFrame(gameLoop);
        }

        function updateFirebase() {
            if(!gameStarted) return;
            playersRef.child(myId).set({
                x: player.x,
                y: player.y,
                color: player.color,
                name: myName,
                lastMsg: player.lastMsg,
                msgTimer: player.msgTimer,
                animFrame: player.animFrame
            });
        }

        function initNetwork() {
            playersRef.on('value', (snapshot) => {
                players = snapshot.val() || {};
            });
            playersRef.child(myId).onDisconnect().remove();

            chatRef.limitToLast(20).on('child_added', (snapshot) => {
                const data = snapshot.val();
                const chatBox = document.getElementById('chatMessages');
                const msgEl = document.createElement('div');
                msgEl.classList.add('chat-msg');
                msgEl.innerHTML = `<span class="name" style="color:${data.color}">${data.name}:</span>${data.text}`;
                chatBox.appendChild(msgEl);
                chatBox.scrollTop = chatBox.scrollHeight;
            });
        }

        // TECLADO (PC)
        window.addEventListener('keydown', (e) => {
            if(document.activeElement === document.getElementById('chatInput')) return;
            if(e.key === 'a' || e.key === 'A' || e.key === 'ArrowLeft') keys.left = true;
            if(e.key === 'd' || e.key === 'D' || e.key === 'ArrowRight') keys.right = true;
            if(e.key === 'w' || e.key === 'W' || e.key === ' ' || e.key === 'ArrowUp') keys.jump = true;
        });

        window.addEventListener('keyup', (e) => {
            if(e.key === 'a' || e.key === 'A' || e.key === 'ArrowLeft') keys.left = false;
            if(e.key === 'd' || e.key === 'D' || e.key === 'ArrowRight') keys.right = false;
            if(e.key === 'w' || e.key === 'W' || e.key === ' ' || e.key === 'ArrowUp') keys.jump = false;
        });

        // MECÂNICA DE TOQUE RESTRITA AOS BOTÕES EXTERNOS
        function setupMobileControl(id, keyProp) {
            const btn = document.getElementById(id);
            btn.addEventListener('pointerdown', (e) => { keys[keyProp] = true; });
            btn.addEventListener('pointerup', (e) => { keys[keyProp] = false; });
            btn.addEventListener('pointerleave', (e) => { keys[keyProp] = false; });
        }
        setupMobileControl('btnLeft', 'left');
        setupMobileControl('btnRight', 'right');
        setupMobileControl('btnJump', 'jump');

        document.getElementById('chatInput').addEventListener('keydown', (e) => {
            if(e.key === 'Enter') {
                const input = e.target;
                const text = input.value.trim();
                if(text !== "") {
                    chatRef.push({ name: myName, text: text, color: player.color });
                    player.lastMsg = text;
                    player.msgTimer = 180;
                    updateFirebase();
                }
                input.value = "";
                input.blur();
            }
        });

        function gameLoop() {
            if (keys.left) {
                player.vx = -5.5;
                player.animFrame += 0.25;
            } else if (keys.right) {
                player.vx = 5.5;
                player.animFrame += 0.25;
            } else {
                player.vx = 0;
                player.animFrame = 0;
            }

            player.vy += 0.55;
            if (keys.jump && player.isGrounded) {
                player.vy = -14.5;
                player.isGrounded = false;
            }

            player.x += player.vx;
            player.y += player.vy;

            if (player.x < 0) player.x = 0;
            if (player.x > WORLD_WIDTH - player.width) player.x = WORLD_WIDTH - player.width;

            player.isGrounded = false;
            for (let plat of platforms) {
                if (player.x + player.width > plat.x && 
                    player.x < plat.x + plat.width &&
                    player.y + player.height >= plat.y && 
                    player.y + player.height - player.vy <= plat.y + 12) {
                    
                    player.y = plat.y - player.height;
                    player.vy = 0;
                    player.isGrounded = true;
                }
            }

            if(player.msgTimer > 0) {
                player.msgTimer--;
                if(player.msgTimer === 0) {
                    player.lastMsg = "";
                    updateFirebase();
                }
            }

            if(player.vx !== 0 || player.vy !== 0 || player.msgTimer > 0) {
                updateFirebase();
            }

            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // CÂMERA INTELIGENTE PANORÂMICA (AFASTA E CENTRALIZA TODO O MAPA)
            ctx.save();
            
            // Calcula escala automática baseada no tamanho do navegador para ver o mapa amplo
            let scaleX = canvas.width / WORLD_WIDTH;
            let scaleY = canvas.height / WORLD_HEIGHT;
            let scale = Math.min(scaleX, scaleY) * 1.1; // Ajuste perfeito de distância panorâmica
            
            let cameraX = (canvas.width - WORLD_WIDTH * scale) / 2;
            let cameraY = (canvas.height - WORLD_HEIGHT * scale) / 2;
            
            ctx.translate(cameraX, cameraY);
            ctx.scale(scale, scale);

            // Cenário e Plataformas
            ctx.fillStyle = "#2c2f33";
            for(let plat of platforms) {
                ctx.fillRect(plat.x, plat.y, plat.width, plat.height);
                ctx.strokeStyle = "#5865F2";
                ctx.lineWidth = 3;
                ctx.strokeRect(plat.x, plat.y, plat.width, plat.height);
            }

            // Renderizar outros players
            Object.keys(players).forEach(id => {
                if(id === myId) return;
                drawHumanoidCharacter(players[id]);
            });

            // Renderizar você
            drawHumanoidCharacter({
                x: player.x, y: player.y, 
                color: player.color, name: myName, 
                lastMsg: player.lastMsg, msgTimer: player.msgTimer,
                animFrame: player.animFrame
            });

            ctx.restore();
            requestAnimationFrame(gameLoop);
        }

        function drawHumanoidCharacter(p) {
            let px = p.x;
            let py = p.y;
            let c = p.color;
            let wave = Math.sin(p.animFrame || 0) * 10;

            // Pernas articuladas
            ctx.fillStyle = "#1e1f22";
            ctx.fillRect(px + 6, py + 50, 10, 25 + wave);  
            ctx.fillRect(px + 24, py + 50, 10, 25 - wave); 

            // Tronco
            ctx.fillStyle = c;
            ctx.fillRect(px + 5, py + 20, 30, 32);
            ctx.strokeStyle = "#000";
            ctx.lineWidth = 1.5;
            ctx.strokeRect(px + 5, py + 20, 30, 32);

            // Braços
            ctx.fillStyle = c;
            ctx.fillRect(px - 4, py + 22, 8, 20 - wave * 0.5); 
            ctx.fillRect(px + 36, py + 22, 8, 20 + wave * 0.5); 

            // Cabeça
            ctx.fillStyle = "#ffcc99"; 
            ctx.fillRect(px + 8, py, 24, 20);
            ctx.strokeRect(px + 8, py, 24, 20);

            // Detalhes do Rosto
            ctx.fillStyle = "#333";
            ctx.fillRect(px + 8, py, 24, 5); // Cabelo
            ctx.fillStyle = "white";
            ctx.fillRect(px + 11, py + 7, 5, 5);
            ctx.fillRect(px + 22, py + 7, 5, 5);
            ctx.fillStyle = "black";
            ctx.fillRect(px + 13, py + 8, 3, 3);
            ctx.fillRect(px + 22, py + 8, 3, 3);

            // Tag de Nome
            ctx.fillStyle = "white";
            ctx.font = "bold 15px sans-serif";
            ctx.textAlign = "center";
            ctx.fillText(p.name, px + 20, py - 14);

            // Balão HUD flutuante de Chat
            if (p.lastMsg && p.msgTimer > 0) {
                ctx.font = "13px sans-serif";
                let textWidth = ctx.measureText(p.lastMsg).width;
                
                ctx.fillStyle = "white";
                ctx.beginPath();
                ctx.roundRect(px + 20 - (textWidth/2) - 8, py - 55, textWidth + 16, 26, 6);
                ctx.fill();

                ctx.beginPath();
                ctx.moveTo(px + 14, py - 29);
                ctx.lineTo(px + 20, py - 23);
                ctx.lineTo(px + 26, py - 29);
                ctx.fill();

                ctx.fillStyle = "black";
                ctx.textAlign = "center";
                ctx.fillText(p.lastMsg, px + 20, py - 38);
            }
        }
    </script>
</body>
</html>
