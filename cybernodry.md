<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Cyber Parkour Multiplayer</title>
    
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js"></script>

    <style>
        :root {
            --bg-dark: #1e1e24;
            --bg-chat: #2f3136;
            --accent: #00ffcc;
            --text: #dcddde;
        }

        * { margin: 0; padding: 0; box-sizing: border-box; font-family: sans-serif; user-select: none; }

        body {
            background: var(--bg-dark);
            color: var(--text);
            display: flex;
            height: 100vh;
            width: 100vw;
            overflow: hidden;
        }

        /* ÁREA DO JOGO (CANVAS) */
        .game-container {
            flex: 1;
            position: relative;
            display: flex;
            flex-direction: column;
            background: #111;
        }

        canvas {
            width: 100%;
            height: 100%;
            background: #14141f;
            display: block;
        }

        /* CONTROLES SEPARADOS NOS CANTOS (CELULAR) */
        .controls-mobile {
            position: absolute;
            bottom: 30px;
            left: 0;
            right: 0;
            padding: 0 30px;
            display: flex;
            justify-content: space-between;
            pointer-events: none;
            z-index: 10;
        }

        .d-pad { display: flex; gap: 20px; pointer-events: auto; }
        .action-pad { pointer-events: auto; }
        
        .btn-game {
            width: 85px;
            height: 85px;
            background: rgba(255, 255, 255, 0.2);
            border: 3px solid rgba(255, 255, 255, 0.4);
            border-radius: 50%;
            color: white;
            font-size: 1.8rem;
            font-weight: bold;
            display: flex;
            align-items: center;
            justify-content: center;
            backdrop-filter: blur(4px);
            cursor: pointer;
            touch-action: none;
        }
        .btn-game:active { background: var(--accent); color: #000; border-color: var(--accent); }

        /* ABA LATERAL DO CHAT */
        .chat-sidebar {
            width: 300px;
            background: var(--bg-chat);
            border-left: 2px solid rgba(0,0,0,0.3);
            display: flex;
            flex-direction: column;
            flex-shrink: 0;
        }

        .chat-header {
            padding: 15px;
            background: rgba(0,0,0,0.2);
            font-weight: bold;
            text-align: center;
            border-bottom: 1px solid rgba(0,0,0,0.2);
        }

        .chat-messages {
            flex: 1;
            padding: 10px;
            overflow-y: auto;
            display: flex;
            flex-direction: column;
            gap: 8px;
        }

        .chat-msg {
            background: rgba(0,0,0,0.15);
            padding: 8px 12px;
            border-radius: 6px;
            font-size: 0.95rem;
            word-break: break-all;
        }
        .chat-msg .name { font-weight: bold; }

        .chat-input-box {
            padding: 12px;
            background: rgba(0,0,0,0.2);
        }

        .chat-input-box input {
            width: 100%;
            padding: 12px;
            border: none;
            background: #40444b;
            color: white;
            border-radius: 6px;
            outline: none;
            font-size: 1rem;
        }

        /* NOME DE USUÁRIO */
        #nameModal {
            position: fixed; top:0; left:0; width:100vw; height:100vh;
            background: rgba(0,0,0,0.9); display: flex; align-items: center; justify-content: center;
            z-index: 999;
        }
        .modal-content {
            background: var(--bg-chat); padding: 40px; border-radius: 12px; text-align: center;
            box-shadow: 0 4px 20px rgba(0,0,0,0.7); width: 90%; max-width: 400px;
        }
        .modal-content input {
            padding: 12px; border-radius: 6px; border: none; margin-bottom: 20px; width: 100%; text-align: center; font-size: 1.1rem;
        }
        .modal-content button {
            background: var(--accent); color: black; border: none; padding: 12px 24px; font-weight: bold; font-size: 1rem; border-radius: 6px; cursor: pointer; width: 100%;
        }

        @media (max-width: 768px) {
            body { flex-direction: column; }
            .chat-sidebar { width: 100%; height: 220px; }
            .controls-mobile { bottom: 240px; }
        }
    </style>
</head>
<body>

    <div id="nameModal">
        <div class="modal-content">
            <h3>Cyber Parkour World</h3>
            <br>
            <input type="text" id="usernameInput" placeholder="Seu apelido" maxlength="12">
            <button onclick="startGame()">Entrar no Jogo</button>
        </div>
    </div>

    <div class="game-container">
        <canvas id="gameCanvas"></canvas>
        
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

    <div class="chat-sidebar">
        <div class="chat-header">Chat do Jogo</div>
        <div class="chat-messages" id="chatMessages"></div>
        <div class="chat-input-box">
            <input type="text" id="chatInput" placeholder="Digite aqui e dê Enter..." maxlength="60">
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

        // Tamanho do mundo visível
        const WORLD_WIDTH = 2200;
        const WORLD_HEIGHT = 1000;

        let myId = 'p_' + Math.floor(Math.random() * 999999);
        let myName = "";
        let gameStarted = false;

        // Estrutura física do player
        let player = {
            x: 100, y: 800,
            width: 40, height: 75, // Altura maior para acomodar cabeça e pernas
            vx: 0, vy: 0,
            color: `hsl(${Math.random() * 360}, 85%, 60%)`,
            isGrounded: false,
            lastMsg: "",
            msgTimer: 0,
            animFrame: 0 // Usado para mover as pernas
        };

        let players = {};

        // Configuração das plataformas do Parkour
        const platforms = [
            { x: 0, y: 940, width: WORLD_WIDTH, height: 60 }, // Chão Principal
            { x: 200, y: 800, width: 180, height: 20 },
            { x: 450, y: 680, width: 180, height: 20 },
            { x: 700, y: 550, width: 180, height: 20 },
            { x: 980, y: 440, width: 220, height: 20 },
            { x: 1300, y: 350, width: 180, height: 20 },
            { x: 1050, y: 230, width: 180, height: 20 },
            { x: 750, y: 160, width: 200, height: 20 },
            { x: 400, y: 150, width: 250, height: 20 }
        ];

        let keys = { left: false, right: false, jump: false };

        function resizeCanvas() {
            canvas.width = canvas.parentElement.clientWidth;
            canvas.height = canvas.parentElement.clientHeight;
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

            chatRef.limitToLast(30).on('child_added', (snapshot) => {
                const data = snapshot.val();
                const chatBox = document.getElementById('chatMessages');
                const msgEl = document.createElement('div');
                msgEl.classList.add('chat-msg');
                msgEl.innerHTML = `<span class="name" style="color:${data.color}">${data.name}:</span> ${data.text}`;
                chatBox.appendChild(msgEl);
                chatBox.scrollTop = chatBox.scrollHeight;
            });
        }

        // CONTROLES DE COMPUTADOR
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

        // TOQUE MOBILE
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
            // Física de Andar
            if (keys.left) {
                player.vx = -5;
                player.animFrame += 0.2; // Roda animação de andar
            } else if (keys.right) {
                player.vx = 5;
                player.animFrame += 0.2;
            } else {
                player.vx = 0;
                player.animFrame = 0; // Fica parado
            }

            // Física de Gravidade e Pulo
            player.vy += 0.55;
            if (keys.jump && player.isGrounded) {
                player.vy = -14;
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

            // CÂMERA AFASTADA (DÁ PARA VER MUITO MAIS DO MAPA DE LONGE)
            ctx.save();
            
            // Zoom reduzido para 0.75x para garantir visão ampla dos parkours
            let scale = 0.75;
            let cameraX = -player.x * scale + canvas.width / 2;
            let cameraY = -player.y * scale + canvas.height / 2;
            
            cameraX = Math.min(0, Math.max(canvas.width - WORLD_WIDTH * scale, cameraX));
            cameraY = Math.min(0, Math.max(canvas.height - WORLD_HEIGHT * scale, cameraY));
            
            ctx.translate(cameraX, cameraY);
            ctx.scale(scale, scale);

            // Desenhar plataformas modernas e bonitas
            ctx.fillStyle = "#3a3d42";
            for(let plat of platforms) {
                ctx.fillRect(plat.x, plat.y, plat.width, plat.height);
                ctx.strokeStyle = varColor = "#5865F2";
                ctx.lineWidth = 3;
                ctx.strokeRect(plat.x, plat.y, plat.width, plat.height);
            }

            // Desenhar os outros jogadores
            Object.keys(players).forEach(id => {
                if(id === myId) return;
                drawHumanoidCharacter(players[id]);
            });

            // Desenhar você
            drawHumanoidCharacter({
                x: player.x, y: player.y, 
                color: player.color, name: myName, 
                lastMsg: player.lastMsg, msgTimer: player.msgTimer,
                animFrame: player.animFrame
            });

            ctx.restore();
            requestAnimationFrame(gameLoop);
        }

        // FUNÇÃO QUE DESENHA O BONECO HUMANOIDE COMPLETO (CABEÇA, CORPO, BRAÇOS, PERNAS)
        function drawHumanoidCharacter(p) {
            let px = p.x;
            let py = p.y;
            let c = p.color;

            // Frequência de balanço de pernas usando o frame de animação
            let wave = Math.sin(p.animFrame || 0) * 10;

            // 1. PERNAS (Duas pernas quadradas articuladas)
            ctx.fillStyle = "#111"; // Cor da calça/perna
            ctx.fillRect(px + 6, py + 50, 10, 25 + wave);  // Perna Esquerda
            ctx.fillRect(px + 24, py + 50, 10, 25 - wave); // Perna Direita

            // 2. CORPO (Tronco quadrado principal)
            ctx.fillStyle = c;
            ctx.fillRect(px + 5, py + 20, 30, 32);
            ctx.strokeStyle = "#000";
            ctx.lineWidth = 1.5;
            ctx.strokeRect(px + 5, py + 20, 30, 32);

            // 3. BRAÇOS (Acompanham o movimento das pernas)
            ctx.fillStyle = c;
            ctx.fillRect(px - 4, py + 22, 8, 20 - wave * 0.5); // Braço Esquerdo
            ctx.fillRect(px + 36, py + 22, 8, 20 + wave * 0.5); // Braço Direito

            // 4. CABEÇA (Quadrada por cima do corpo)
            ctx.fillStyle = "#ffcc99"; // Cor de pele padrão
            ctx.fillRect(px + 8, py, 24, 20);
            ctx.strokeRect(px + 8, py, 24, 20);

            // Cabelo / Boné estilizado
            ctx.fillStyle = "#333";
            ctx.fillRect(px + 8, py, 24, 6);

            // Olhos detalhados
            ctx.fillStyle = "white";
            ctx.fillRect(px + 12, py + 7, 5, 5);
            ctx.fillRect(px + 23, py + 7, 5, 5);
            ctx.fillStyle = "black";
            ctx.fillRect(px + 14, py + 8, 3, 3);
            ctx.fillRect(px + 23, py + 8, 3, 3);

            // Nome do Jogador
            ctx.fillStyle = "white";
            ctx.font = "bold 15px sans-serif";
            ctx.textAlign = "center";
            ctx.fillText(p.name, px + 20, py - 15);

            // Balão de Chat flutuante
            if (p.lastMsg && p.msgTimer > 0) {
                ctx.font = "13px sans-serif";
                let textWidth = ctx.measureText(p.lastMsg).width;
                
                ctx.fillStyle = "white";
                ctx.beginPath();
                ctx.roundRect(px + 20 - (textWidth/2) - 8, py - 55, textWidth + 16, 26, 6);
                ctx.fill();

                // Seta do balão
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
