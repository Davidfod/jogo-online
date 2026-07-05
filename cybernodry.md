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

        /* CONTROLES VIRTUAIS (CELULAR) */
        .controls-mobile {
            position: absolute;
            bottom: 20px;
            left: 20px;
            right: 20px;
            display: flex;
            justify-content: space-between;
            pointer-events: none;
        }

        .d-pad { display: flex; gap: 10px; pointer-events: auto; }
        .btn-game {
            width: 65px;
            height: 65px;
            background: rgba(255, 255, 255, 0.15);
            border: 2px solid rgba(255, 255, 255, 0.3);
            border-radius: 50%;
            color: white;
            font-size: 1.5rem;
            font-weight: bold;
            display: flex;
            align-items: center;
            justify-content: center;
            backdrop-filter: blur(4px);
        }
        .btn-game:active { background: var(--accent); color: #000; }
        .action-pad { pointer-events: auto; }

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
            padding: 6px 10px;
            border-radius: 6px;
            font-size: 0.9rem;
            word-break: break-all;
        }
        .chat-msg .name { font-weight: bold; }

        .chat-input-box {
            padding: 10px;
            background: rgba(0,0,0,0.2);
        }

        .chat-input-box input {
            width: 100%;
            padding: 10px;
            border: none;
            background: #40444b;
            color: white;
            border-radius: 4px;
            outline: none;
        }

        /* NOME DE USUÁRIO */
        #nameModal {
            position: fixed; top:0; left:0; width:100vw; height:100vh;
            background: rgba(0,0,0,0.85); display: flex; align-items: center; justify-content: center;
            z-index: 999;
        }
        .modal-content {
            background: var(--bg-chat); padding: 30px; border-radius: 8px; text-align: center;
            box-shadow: 0 4px 15px rgba(0,0,0,0.5);
        }
        .modal-content input {
            padding: 10px; border-radius: 4px; border: none; margin-bottom: 15px; width: 100%; text-align: center;
        }
        .modal-content button {
            background: var(--accent); color: black; border: none; padding: 10px 20px; font-weight: bold; border-radius: 4px; cursor: pointer;
        }

        @media (max-width: 768px) {
            body { flex-direction: column; }
            .chat-sidebar { width: 100%; height: 200px; }
            .controls-mobile { bottom: 220px; }
        }
    </style>
</head>
<body>

    <div id="nameModal">
        <div class="modal-content">
            <h3>Escolha seu Nome para Entrar</h3>
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
                <div class="btn-game" id="btnJump" style="width:75px; height:75px;">Pulo</div>
            </div>
        </div>
    </div>

    <div class="chat-sidebar">
        <div class="chat-header">Chat do Jogo</div>
        <div class="chat-messages" id="chatMessages"></div>
        <div class="chat-input-box">
            <input type="text" id="chatInput" placeholder="Pressione Enter para falar..." maxlength="60">
        </div>
    </div>

    <script>
        // CONFIGURAÇÃO DO SEU FIREBASE
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

        // Configuração lógica do tamanho do mundo do jogo
        const WORLD_WIDTH = 1200;
        const WORLD_HEIGHT = 800;

        let myId = 'p_' + Math.floor(Math.random() * 999999);
        let myName = "";
        let gameStarted = false;

        // O Jogador Local
        let player = {
            x: 100, y: 600,
            width: 30, height: 45,
            vx: 0, vy: 0,
            color: `hsl(${Math.random() * 360}, 80%, 60%)`,
            isGrounded: false,
            lastMsg: "",
            msgTimer: 0
        };

        // Outros jogadores da sala
        let players = {};

        // Definição do Mapa de Parkour (Plataformas)
        const platforms = [
            // Chão Principal
            { x: 0, y: 750, width: WORLD_WIDTH, height: 50 },
            // Plataformas do Parkour
            { x: 150, y: 630, width: 120, height: 15 },
            { x: 340, y: 530, width: 120, height: 15 },
            { x: 550, y: 440, width: 100, height: 15 },
            { x: 750, y: 350, width: 150, height: 15 },
            { x: 520, y: 250, width: 120, height: 15 },
            { x: 300, y: 180, width: 120, height: 15 },
            { x: 100, y: 120, width: 100, height: 15 },
            // Plataforma Secreta Alta
            { x: 950, y: 200, width: 200, height: 15 }
        ];

        // Estado das Teclas / Botões de Movimento
        let keys = { left: false, right: false, jump: false };

        // Ajustar tamanho do canvas na tela dinamicamente
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
            
            // Envia dados iniciais pro Firebase
            updateFirebase();
            
            // Ativa os escutas de rede e os loops
            initNetwork();
            requestAnimationFrame(gameLoop);
        }

        // Envia posição atual do jogador pro Firebase
        function updateFirebase() {
            if(!gameStarted) return;
            playersRef.child(myId).set({
                x: player.x,
                y: player.y,
                color: player.color,
                name: myName,
                lastMsg: player.lastMsg,
                msgTimer: player.msgTimer
            });
        }

        function initNetwork() {
            // Monitora a entrada/movimento de todos no mapa
            playersRef.on('value', (snapshot) => {
                const data = snapshot.val() || {};
                players = data;
            });

            // Remove jogador quando fechar o navegador
            playersRef.child(myId).onDisconnect().remove();

            // Escuta novas mensagens de Chat
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

        // CONTROLES VIA TECLADO (PC)
        window.addEventListener('keydown', (e) => {
            if(document.activeElement === document.getElementById('chatInput')) return; // Bloqueia se estiver digitando no chat
            if(e.key === 'a' || e.key === 'A' || e.key === 'ArrowLeft') keys.left = true;
            if(e.key === 'd' || e.key === 'D' || e.key === 'ArrowRight') keys.right = true;
            if(e.key === 'w' || e.key === 'W' || e.key === ' ' || e.key === 'ArrowUp') keys.jump = true;
        });

        window.addEventListener('keyup', (e) => {
            if(e.key === 'a' || e.key === 'A' || e.key === 'ArrowLeft') keys.left = false;
            if(e.key === 'd' || e.key === 'D' || e.key === 'ArrowRight') keys.right = false;
            if(e.key === 'w' || e.key === 'W' || e.key === ' ' || e.key === 'ArrowUp') keys.jump = false;
        });

        // CONTROLES VIA TOQUE (MOBILE)
        function bindMobileBtn(id, property) {
            const btn = document.getElementById(id);
            btn.addEventListener('touchstart', (e) => { e.preventDefault(); keys[property] = true; });
            btn.addEventListener('touchend', (e) => { e.preventDefault(); keys[property] = false; });
        }
        bindMobileBtn('btnLeft', 'left');
        bindMobileBtn('btnRight', 'right');
        bindMobileBtn('btnJump', 'jump');

        // ENVIO DE CHAT
        document.getElementById('chatInput').addEventListener('keydown', (e) => {
            if(e.key === 'Enter') {
                const input = e.target;
                const text = input.value.trim();
                if(text !== "") {
                    // Envia pro chat global da lateral
                    chatRef.push({
                        name: myName,
                        text: text,
                        color: player.color
                    });
                    
                    // Coloca o balão em cima da cabeça
                    player.lastMsg = text;
                    player.msgTimer = 180; // dura 3 segundos rodando a 60 fps
                    updateFirebase();
                }
                input.value = "";
                input.blur();
            }
        });

        // ENGENHARIA DE FÍSICA E LOOP PRINCIPAL DO JOGO
        function gameLoop() {
            // 1. Movimentação Horizontal
            if (keys.left) player.vx = -4;
            else if (keys.right) player.vx = 4;
            else player.vx = 0;

            // 2. Gravidade e Salto
            player.vy += 0.5; // força da gravidade
            if (keys.jump && player.isGrounded) {
                player.vy = -12; // força do pulo
                player.isGrounded = false;
            }

            // Aplicar movimento projetado
            player.x += player.vx;
            player.y += player.vy;

            // Limites das Paredes do Mundo
            if (player.x < 0) player.x = 0;
            if (player.x > WORLD_WIDTH - player.width) player.x = WORLD_WIDTH - player.width;

            // 3. Checagem de Colisão com as Plataformas do Parkour
            player.isGrounded = false;
            for (let plat of platforms) {
                // Checa se o boneco está caindo exatamente por cima da plataforma
                if (player.x + player.width > plat.x && 
                    player.x < plat.x + plat.width &&
                    player.y + player.height >= plat.y && 
                    player.y + player.height - player.vy <= plat.y + 10) {
                    
                    player.y = plat.y - player.height;
                    player.vy = 0;
                    player.isGrounded = true;
                }
            }

            // Diminuir o tempo do balão de fala local
            if(player.msgTimer > 0) {
                player.msgTimer--;
                if(player.msgTimer === 0) {
                    player.lastMsg = "";
                    updateFirebase();
                }
            }

            // Se mudou de posição ou status, avisa o banco
            if(player.vx !== 0 || player.vy !== 0 || player.msgTimer > 0) {
                updateFirebase();
            }

            // 4. RENDERIZAÇÃO NA TELA (DESENHAR TUDO)
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Efeito de câmera simples para seguir o seu boneco centralizado
            ctx.save();
            let cameraX = -player.x + canvas.width / 2;
            let cameraY = -player.y + canvas.height / 2;
            
            // Previne a câmera de sair do mapa estipulado
            cameraX = Math.min(0, Math.max(canvas.width - WORLD_WIDTH, cameraX));
            cameraY = Math.min(0, Math.max(canvas.height - WORLD_HEIGHT, cameraY));
            ctx.translate(cameraX, cameraY);

            // Desenhar Plataformas Cinzas
            ctx.fillStyle = "#4f545c";
            for(let plat of platforms) {
                ctx.fillRect(plat.x, plat.y, plat.width, plat.height);
                // Borda brilhante na plataforma
                ctx.strokeStyle = "#72767d";
                ctx.strokeRect(plat.x, plat.y, plat.width, plat.height);
            }

            // Desenhar Outros Jogadores Conectados
            Object.keys(players).forEach(id => {
                if(id === myId) return; // desenha você por último
                const p = players[id];
                drawCharacter(p);
            });

            // Desenhar Você mesmo
            drawCharacter({
                x: player.x, y: player.y, 
                color: player.color, name: myName, 
                lastMsg: player.lastMsg, msgTimer: player.msgTimer
            });

            ctx.restore();

            requestAnimationFrame(gameLoop);
        }

        // Função auxiliar para desenhar o boneco e o balão de texto em cima dele
        function drawCharacter(p) {
            // Corpo do Boneco
            ctx.fillStyle = p.color;
            ctx.fillRect(p.x, p.y, 30, 45);

            // Olhinhos para dar carisma
            ctx.fillStyle = "white";
            ctx.fillRect(p.x + 6, p.y + 10, 5, 5);
            ctx.fillRect(p.x + 18, p.y + 10, 5, 5);

            // Nome acima do boneco
            ctx.fillStyle = "white";
            ctx.font = "bold 12px sans-serif";
            ctx.textAlign = "center";
            ctx.fillText(p.name, p.x + 15, p.y - 8);

            // Balão de Mensagem (se houver fala ativa)
            if (p.lastMsg && p.msgTimer > 0) {
                ctx.font = "11px sans-serif";
                let textWidth = ctx.measureText(p.lastMsg).width;
                
                // Caixa do Balão
                ctx.fillStyle = "white";
                ctx.beginPath();
                ctx.roundRect(p.x + 15 - (textWidth/2) - 6, p.y - 42, textWidth + 12, 22, 5);
                ctx.fill();

                // Pequena seta apontando para o boneco
                ctx.beginPath();
                ctx.moveTo(p.x + 11, p.y - 20);
                ctx.lineTo(p.x + 15, p.y - 15);
                ctx.lineTo(p.x + 19, p.y - 20);
                ctx.fill();

                // Texto do balão
                ctx.fillStyle = "black";
                ctx.textAlign = "center";
                ctx.fillText(p.lastMsg, p.x + 15, p.y - 27);
            }
        }
    </script>
</body>
</html>
