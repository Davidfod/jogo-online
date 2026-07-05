<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Cyber Parkour Multi-Salas</title>
    
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js"></script>

    <style>
        :root {
            --accent: #00ffcc;
            --bg-chat: rgba(20, 20, 31, 0.75);
        }

        * { margin: 0; padding: 0; box-sizing: border-box; font-family: sans-serif; user-select: none; }

        body, html {
            background: #0e0e14;
            width: 100%;
            height: 100%;
            overflow: hidden;
            position: relative;
        }

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

        /* CONTROLES MOBILE NOS EXTREMOS */
        .controls-mobile {
            position: absolute;
            bottom: 30px;
            left: 0;
            right: 0;
            padding: 0 40px;
            display: flex;
            justify-content: space-between;
            pointer-events: none;
            z-index: 20;
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
            font-size: 2rem;
            font-weight: bold;
            display: flex;
            align-items: center;
            justify-content: center;
            backdrop-filter: blur(4px);
            cursor: pointer;
            touch-action: none;
        }
        .btn-game:active { background: var(--accent); color: #000; border-color: var(--accent); }

        /* HUD DO CHAT */
        .chat-hud {
            position: absolute;
            top: 20px;
            left: 20px;
            width: 300px;
            height: 240px;
            background: var(--bg-chat);
            backdrop-filter: blur(5px);
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
            background: rgba(0, 0, 0, 0.35);
            padding: 6px 10px;
            border-radius: 4px;
            font-size: 0.85rem;
            color: #fff;
            word-break: break-all;
        }
        .chat-msg .name { font-weight: bold; margin-right: 4px; }

        .chat-input-box {
            padding: 8px;
            background: rgba(0,0,0,0.5);
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

        /* LOBBY / MENU INICIAL DE SELEÇÃO */
        #menuModal {
            position: fixed; top:0; left:0; width:100vw; height:100vh;
            background: rgba(10, 10, 15, 0.96); display: flex; align-items: center; justify-content: center;
            z-index: 999;
        }
        .modal-content {
            background: #1f2126; padding: 40px; border-radius: 12px; text-align: center;
            box-shadow: 0 6px 25px rgba(0,0,0,0.8); width: 90%; max-width: 420px;
        }
        .modal-content input {
            padding: 12px; border-radius: 6px; border: none; margin-bottom: 20px; width: 100%; text-align: center; font-size: 1.1rem; background: #383a40; color: white; outline: none;
        }
        
        .select-title { color: #aaa; font-size: 0.85rem; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 8px; text-align: left;}
        
        .level-selector {
            display: flex; gap: 10px; margin-bottom: 25px;
        }
        .level-btn {
            flex: 1; padding: 12px; border: 2px solid #383a40; background: transparent; color: white; font-weight: bold; border-radius: 6px; cursor: pointer; font-size: 0.9rem;
        }
        .level-btn.active {
            border-color: var(--accent); background: rgba(0, 255, 204, 0.1); color: var(--accent);
        }

        .btn-start {
            background: var(--accent); color: black; border: none; padding: 14px; font-weight: bold; font-size: 1.1rem; border-radius: 6px; cursor: pointer; width: 100%; letter-spacing: 0.5px;
        }
    </style>
</head>
<body>

    <div id="menuModal">
        <div class="modal-content">
            <h2 style="color: white; margin-bottom: 5px;">Cyber Parkour Online</h2>
            <p style="color: #6d6f78; font-size: 0.85rem; margin-bottom: 25px;">Escolha seu nível e desafie os outros</p>
            
            <div class="select-title">Seu Apelido</div>
            <input type="text" id="usernameInput" placeholder="Digite seu Nick..." maxlength="12">
            
            <div class="select-title">Escolha a dificuldade (Sala)</div>
            <div class="level-selector">
                <button class="level-btn active" onclick="selectDifficulty('facil')">FÁCIL</button>
                <button class="level-btn" onclick="selectDifficulty('medio')">MÉDIO</button>
                <button class="level-btn" onclick="selectDifficulty('hard')">HARD</button>
            </div>

            <button class="btn-start" onclick="startGame()">CONECTAR AO MUNDO</button>
        </div>
    </div>

    <div class="game-wrapper">
        <canvas id="gameCanvas"></canvas>
        
        <div class="chat-hud">
            <div class="chat-messages" id="chatMessages"></div>
            <div class="chat-input-box">
                <input type="text" id="chatInput" placeholder="Envie uma mensagem..." maxlength="45">
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

        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        const WORLD_WIDTH = 2200;
        const WORLD_HEIGHT = 950;

        let myId = 'p_' + Math.floor(Math.random() * 999999);
        let myName = "";
        let selectedDifficulty = "facil"; 
        let gameStarted = false;

        // Referências dinâmicas do Firebase baseadas na sala escolhida
        let playersRef, chatRef;

        let player = {
            x: 80, y: 800,
            width: 40, height: 75,
            vx: 0, vy: 0,
            color: `hsl(${Math.random() * 360}, 85%, 60%)`,
            isGrounded: false,
            lastMsg: "",
            msgTimer: 0,
            animFrame: 0
        };

        let players = {};
        let platforms = [];
        let finishBlock = { x: 2050, y: 0, width: 60, height: 60 }; // Bloco final de vitória

        // MAPAS DE ACORDO COM A DIFICULDADE
        const maps = {
            facil: [
                { x: 0, y: 880, width: WORLD_WIDTH, height: 70 }, // chão firme grande
                { x: 250, y: 740, width: 220, height: 20 },
                { x: 550, y: 620, width: 220, height: 20 },
                { x: 850, y: 500, width: 220, height: 20 },
                { x: 1150, y: 400, width: 220, height: 20 },
                { x: 1450, y: 300, width: 220, height: 20 },
                { x: 1750, y: 220, width: 300, height: 20 } // Plataforma da vitória fácil
            ],
            medio: [
                { x: 0, y: 880, width: 400, height: 70 }, // chão inicial menor
                { x: 500, y: 760, width: 150, height: 20 },
                { x: 750, y: 650, width: 150, height: 20 },
                { x: 1050, y: 540, width: 140, height: 20 },
                { x: 1300, y: 440, width: 150, height: 20 },
                { x: 1550, y: 330, width: 130, height: 20 },
                { x: 1800, y: 250, width: 120, height: 20 },
                { x: 2000, y: 190, width: 150, height: 20 } // Vitória médio
            ],
            hard: [
                { x: 0, y: 880, width: 200, height: 70 }, // Plataforma minúscula inicial
                { x: 320, y: 770, width: 70, height: 20 }, // Blocos muito pequenos e distantes
                { x: 520, y: 660, width: 70, height: 20 },
                { x: 740, y: 560, width: 60, height: 20 },
                { x: 980, y: 470, width: 60, height: 20 },
                { x: 1220, y: 390, width: 50, height: 20 },
                { x: 1460, y: 320, width: 60, height: 20 },
                { x: 1700, y: 250, width: 60, height: 20 },
                { x: 1920, y: 210, width: 50, height: 20 },
                { x: 2050, y: 160, width: 100, height: 20 } // Vitória difícil
            ]
        };

        let keys = { left: false, right: false, jump: false };

        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        function selectDifficulty(level) {
            document.querySelectorAll('.level-btn').forEach(btn => btn.classList.remove('active'));
            event.currentTarget.classList.add('active');
            selectedDifficulty = level;
        }

        function startGame() {
            const input = document.getElementById('usernameInput');
            myName = input.value.trim() || "User_" + Math.floor(Math.random()*999);
            document.getElementById('menuModal').style.display = 'none';
            
            // Carrega as plataformas e o posicionamento do bloco final baseado no mapa escolhido
            platforms = maps[selectedDifficulty];
            let lastPlat = platforms[platforms.length - 1];
            finishBlock.x = lastPlat.x + (lastPlat.width / 2) - 30;
            finishBlock.y = lastPlat.y - 60;

            // Define caminhos isolados no Firebase para cada sala
            playersRef = database.ref(`rooms/${selectedDifficulty}/players`);
            chatRef = database.ref(`rooms/${selectedDifficulty}/chat`);

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
                // Se for aviso do sistema, pinta de dourado
                if(data.name === "[SISTEMA]") {
                    msgEl.style.color = "#ffcc00";
                    msgEl.style.fontWeight = "bold";
                }
                msgEl.innerHTML = `<span class="name" style="color:${data.color || '#ffcc00'}">${data.name}:</span>${data.text}`;
                chatBox.appendChild(msgEl);
                chatBox.scrollTop = chatBox.scrollHeight;
            });
        }

        // CONTROLES PC (AJUSTADO)
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

        // TOQUE MOBILE REESCRITO (AGORA FUNCIONA SEM TRAVAR)
        function setupMobileControl(id, keyProp) {
            const btn = document.getElementById(id);
            btn.addEventListener('pointerdown', () => { keys[keyProp] = true; });
            btn.addEventListener('pointerup', () => { keys[keyProp] = false; });
            btn.addEventListener('pointerleave', () => { keys[keyProp] = false; });
        }
        setupMobileControl('btnLeft', 'left');
        setupMobileControl('btnRight', 'right');
        setupMobileControl('btnJump', 'jump');

        // ENVIO DE CHAT
        document.getElementById('chatInput').addEventListener('keydown', (e) => {
            if(e.key === 'Enter') {
                const input = e.target;
                const text = input.value.trim();
                if(text !== "") {
                    chatRef.push({ name: myName, text: text, color: player.color });
                    player.lastMsg = text;
                    player.msgTimer = 180; // dura 3 segundos ativo no loop
                    updateFirebase();
                }
                input.value = "";
                input.blur();
            }
        });

        function gameLoop() {
            // Executa movimentos lineares
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

            player.vy += 0.55; // Gravidade realista
            if (keys.jump && player.isGrounded) {
                player.vy = -14.5;
                player.isGrounded = false;
            }

            player.x += player.vx;
            player.y += player.vy;

            // Bloqueio das bordas laterais do mapa
            if (player.x < 0) player.x = 0;
            if (player.x > WORLD_WIDTH - player.width) player.x = WORLD_WIDTH - player.width;

            // Se cair no limbo abaixo do mapa, resgata o player pro início
            if (player.y > WORLD_HEIGHT) {
                player.x = 80;
                player.y = 600;
                player.vy = 0;
            }

            // Colisão fina com plataformas
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

            // CHECAGEM DE VITÓRIA (Tocou no Bloco de Ouro)
            if (player.x + player.width > finishBlock.x &&
                player.x < finishBlock.x + finishBlock.width &&
                player.y + player.height > finishBlock.y &&
                player.y < finishBlock.y + finishBlock.height) {
                
                // Dispara o alerta de campeão no chat global da sala
                chatRef.push({
                    name: "[SISTEMA]",
                    text: ` 🎉 @${myName} ZEROU O PARKOUR DESTA SALA! BRABO DEMAIS!`,
                    color: "#ffcc00"
                });
                
                // Reseta ele de volta pro início
                player.x = 80;
                player.y = 600;
                player.vy = 0;
            }

            // Temporizador do balão de fala
            if(player.msgTimer > 0) {
                player.msgTimer--;
                if(player.msgTimer === 0) {
                    player.lastMsg = "";
                    updateFirebase();
                }
            }

            // Sincroniza em tempo de execução com Firebase se houver mudanças físicas
            if(player.vx !== 0 || player.vy !== 0 || player.msgTimer > 0) {
                updateFirebase();
            }

            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // RENDER DA CÂMERA PANORÂMICA AUTOMÁTICA
            ctx.save();
            let scaleX = canvas.width / WORLD_WIDTH;
            let scaleY = canvas.height / WORLD_HEIGHT;
            let scale = Math.min(scaleX, scaleY) * 1.05; 
            
            let cameraX = (canvas.width - WORLD_WIDTH * scale) / 2;
            let cameraY = (canvas.height - WORLD_HEIGHT * scale) / 2;
            
            ctx.translate(cameraX, cameraY);
            ctx.scale(scale, scale);

            // Desenhar Plataformas da sala ativa
            ctx.fillStyle = "#2c2f33";
            for(let plat of platforms) {
                ctx.fillRect(plat.x, plat.y, plat.width, plat.height);
                ctx.strokeStyle = "#5865F2";
                ctx.lineWidth = 3;
                ctx.strokeRect(plat.x, plat.y, plat.width, plat.height);
            }

            // DESENHAR BLOCO DE VITÓRIA (OURO BRILHANTE)
            ctx.fillStyle = "#ffd700";
            ctx.fillRect(finishBlock.x, finishBlock.y, finishBlock.width, finishBlock.height);
            ctx.strokeStyle = "#ffffff";
            ctx.lineWidth = 2;
            ctx.strokeRect(finishBlock.x, finishBlock.y, finishBlock.width, finishBlock.height);
            // Letra "FIM" em cima do bloco
            ctx.fillStyle = "#000";
            ctx.font = "bold 14px sans-serif";
            ctx.textAlign = "center";
            ctx.fillText("FIM", finishBlock.x + 30, finishBlock.y + 35);

            // Renderizar outros players conectados na mesma sala
            Object.keys(players).forEach(id => {
                if(id === myId) return;
                drawHumanoidCharacter(players[id]);
            });

            // Renderizar seu boneco humanoide
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

            // Pernas com balanço
            ctx.fillStyle = "#16171a";
            ctx.fillRect(px + 6, py + 50, 10, 25 + wave);  
            ctx.fillRect(px + 24, py + 50, 10, 25 - wave); 

            // Corpo
            ctx.fillStyle = c;
            ctx.fillRect(px + 5, py + 20, 30, 32);
            ctx.strokeStyle = "#000";
            ctx.lineWidth = 1.5;
            ctx.strokeRect(px + 5, py + 20, 30, 32);

            // Braços articulados
            ctx.fillStyle = c;
            ctx.fillRect(px - 4, py + 22, 8, 20 - wave * 0.4); 
            ctx.fillRect(px + 36, py + 22, 8, 20 + wave * 0.4); 

            // Cabeça
            ctx.fillStyle = "#ffcc99"; 
            ctx.fillRect(px + 8, py, 24, 20);
            ctx.strokeRect(px + 8, py, 24, 20);

            // Olhos e Cabelo
            ctx.fillStyle = "#222";
            ctx.fillRect(px + 8, py, 24, 5); 
            ctx.fillStyle = "white";
            ctx.fillRect(px + 11, py + 7, 5, 5);
            ctx.fillRect(px + 22, py + 7, 5, 5);
            ctx.fillStyle = "black";
            ctx.fillRect(px + 13, py + 8, 3, 3);
            ctx.fillRect(px + 22, py + 8, 3, 3);

            // Tag fixa do Nome
            ctx.fillStyle = "white";
            ctx.font = "bold 15px sans-serif";
            ctx.textAlign = "center";
            ctx.fillText(p.name, px + 20, py - 14);

            // BALÃO DE FALA ATUALIZADO (Acompanha perfeitamente em cima da cabeça)
            if (p.lastMsg && p.msgTimer > 0) {
                ctx.font = "bold 13px sans-serif";
                let textWidth = ctx.measureText(p.lastMsg).width;
                
                // Retângulo branco do balão
                ctx.fillStyle = "white";
                ctx.beginPath();
                ctx.roundRect(px + 20 - (textWidth/2) - 8, py - 58, textWidth + 16, 26, 6);
                ctx.fill();

                // Triângulo indicador apontando pra baixo
                ctx.beginPath();
                ctx.moveTo(px + 14, py - 32);
                ctx.lineTo(px + 20, py - 26);
                ctx.lineTo(px + 26, py - 32);
                ctx.fill();

                // Texto da mensagem
                ctx.fillStyle = "black";
                ctx.textAlign = "center";
                ctx.fillText(p.lastMsg, px + 20, py - 41);
            }
        }
    </script>
</body>
</html>
