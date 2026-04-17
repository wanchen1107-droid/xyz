<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>小恐龍：進化之路</title>
    <style>
        /* --- 遊戲基礎畫布 --- */
        body { margin: 0; padding: 0; display: flex; justify-content: center; align-items: center; height: 100vh; background-color: #e0f7fa; font-family: 'Courier New', Courier, monospace; overflow: hidden;}
        #game-container { position: relative; width: 800px; height: 250px; background-color: white; border-bottom: 2px solid #535353; overflow: hidden; border-radius: 8px; box-shadow: 0 10px 20px rgba(0,0,0,0.1); }
        
        /* --- 恐龍樣式 (更換為你提供的圖片) --- */
        #dino { 
            position: absolute; 
            bottom: 0; 
            left: 50px; 
            width: 50px; 
            height: 50px; 
            /* 這裡使用你提供的小恐龍圖床網址 */
            background-image: url('https://files.catbox.moe/g2is8d.png'); 
            background-size: contain;
            background-repeat: no-repeat;
            z-index: 10; 
            transition: filter 0.3s; 
        }
        
        /* 進化濾鏡特效 */
        #dino.evo-gold { filter: sepia(1) saturate(5) hue-rotate(10deg) drop-shadow(0 0 5px #FFD700); }
        #dino.evo-flame { filter: saturate(2) hue-rotate(-10deg) drop-shadow(0 0 8px #ff4757); animation: flamePulse 0.5s infinite alternate; }
        
        @keyframes flamePulse { from { transform: scale(1); } to { transform: scale(1.08); } }

        /* 火焰粒子特效 */
        .flame-particle { position: absolute; width: 6px; height: 6px; background: #ffa502; border-radius: 50%; z-index: 9; animation: flameFly 0.6s linear forwards; pointer-events: none; }
        @keyframes flameFly { 0% { opacity: 1; transform: translate(0,0); } 100% { opacity: 0; transform: translate(-30px, -20px); } }

        /* --- 障礙物與背景 --- */
        .obstacle { position: absolute; bottom: 0; z-index: 5; background-color: #535353; border-radius: 4px; }
        .bird { bottom: 65px; border-radius: 50% 50% 0 0; }
        .cloud { position: absolute; border-radius: 50px; z-index: 1; opacity: 0.8; animation: cloudDrift linear infinite; }
        
        @keyframes cloudDrift { from { left: 850px; } to { left: -150px; } }

        /* --- 使用者介面 (UI) --- */
        #ui-layer { position: absolute; top: 10px; width: 100%; display: flex; justify-content: flex-end; padding: 0 20px; box-sizing: border-box; z-index: 15; color: #535353; font-weight: bold; font-size: 18px;}
        #message { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); text-align: center; display: none; z-index: 20; background: rgba(255,255,255,0.95); padding: 30px; border-radius: 15px; border: 3px solid #535353; box-shadow: 0 0 20px rgba(0,0,0,0.2);}
        #evo-status { position: absolute; top: 10px; left: 20px; color: #535353; z-index: 15; font-weight: bold;}
    </style>
</head>
<body>

    <div id="game-container">
        <div id="evo-status">狀態: 幼年體</div>
        <div id="ui-layer">
            <div id="score">SCORE: 00000</div>
        </div>
        <div id="dino"></div>
        <div id="message">
            <h1 style="color: #ff4757; margin-top: 0;">GAME OVER</h1>
            <p id="final-score" style="font-size: 20px; font-weight: bold;"></p>
            <p>按下 <span style="background: #eee; padding: 2px 8px; border-radius: 4px;">空白鍵</span> 重新挑戰</p>
        </div>
    </div>

<script>
    const container = document.getElementById('game-container');
    const dinoElement = document.getElementById('dino');
    const scoreElement = document.getElementById('score');
    const evoStatusElement = document.getElementById('evo-status');
    const messageElement = document.getElementById('message');
    const finalScoreElement = document.getElementById('final-score');

    // 遊戲核心變數
    let isGameRunning = true;
    let currentEvo = 'normal';
    let score = 0;
    let gameSpeed = 6;
    let baseGravity = 0.8;
    
    let dino = { x: 50, y: 0, vy: 0, width: 40, height: 45, isJumping: false };
    let obstacles = [];
    let cloudTimer = 0;
    let obstacleTimer = 0;
    let flameEffectTimer = 0;

    const cloudColors = ['#ffffff', '#ffedff', '#e3f2fd', '#e8f5e9', '#fff3e0'];

    // 監聽按鍵
    document.addEventListener('keydown', (e) => {
        if (e.code === 'Space' || e.code === 'ArrowUp') {
            if (!isGameRunning) {
                resetGame();
            } else if (!dino.isJumping) { 
                dino.vy = -15; // 跳躍力
                dino.isJumping = true; 
            }
        }
    });

    function gameLoop() {
        if (!isGameRunning) return;

        // 1. 更新恐龍狀態
        updateDino();
        // 2. 更新障礙物
        updateObstacles();
        // 3. 更新背景裝飾
        updateClouds();
        // 4. 進化檢測
        handleEvolution();
        // 5. 碰撞檢測 (現在火焰狀態也會死)
        checkCollisions();
        
        // 分數與速度遞增 (每幀增加 0.15)
        score += 0.15;
        gameSpeed = 6 + (score / 150); // 難度曲線平滑增加
        
        scoreElement.innerText = 'SCORE: ' + Math.floor(score).toString().padStart(5, '0');

        requestAnimationFrame(gameLoop);
    }

    function updateDino() {
        dino.vy += baseGravity;
        dino.y -= dino.vy;
        if (dino.y <= 0) { 
            dino.y = 0; 
            dino.vy = 0; 
            dino.isJumping = false; 
        }
        dinoElement.style.bottom = dino.y + 'px';

        // 火焰粒子產生
        if (currentEvo === 'flame' && ++flameEffectTimer > 4) {
            createFlameParticle();
            flameEffectTimer = 0;
        }
    }

    function handleEvolution() {
        let s = Math.floor(score);
        if (s >= 1000 && currentEvo !== 'flame') {
            currentEvo = 'flame';
            dinoElement.className = 'evo-flame';
            evoStatusElement.innerText = '狀態: 火焰進化';
            evoStatusElement.style.color = '#ff4757';
        } else if (s >= 500 && s < 1000 && currentEvo !== 'gold') {
            currentEvo = 'gold';
            dinoElement.className = 'evo-gold';
            evoStatusElement.innerText = '狀態: 金色覺醒';
            evoStatusElement.style.color = '#FFB900';
        }
    }

    function createFlameParticle() {
        const p = document.createElement('div');
        p.className = 'flame-particle';
        p.style.left = (dino.x + 5) + 'px';
        p.style.bottom = (dino.y + 10 + Math.random() * 20) + 'px';
        container.appendChild(p);
        setTimeout(() => { if(p.parentNode) container.removeChild(p); }, 600);
    }

    function createObstacle() {
        let rand = Math.random();
        let config = { type: 'cactus', width: 25, height: 45, y: 0 };

        if (rand > 0.8) {
            config = { type: 'bird', width: 35, height: 25, y: 65 }; // 鳥
        } else if (rand > 0.6) {
            config = { type: 'cactus', width: 40, height: 60, y: 0 }; // 大仙人掌
        }

        const el = document.createElement('div');
        el.className = `obstacle ${config.type === 'bird' ? 'bird' : ''}`;
        el.style.left = '800px';
        el.style.bottom = config.y + 'px';
        el.style.width = config.width + 'px';
        el.style.height = config.height + 'px';
        container.appendChild(el);

        obstacles.push({ element: el, x: 800, y: config.y, width: config.width, height: config.height });
    }

    function updateObstacles() {
        for (let i = obstacles.length - 1; i >= 0; i--) {
            obstacles[i].x -= gameSpeed;
            obstacles[i].element.style.left = obstacles[i].x + 'px';
            if (obstacles[i].x + obstacles[i].width < 0) {
                container.removeChild(obstacles[i].element);
                obstacles.splice(i, 1);
            }
        }
        if (++obstacleTimer > Math.max(45, 110 - gameSpeed * 2)) {
            createObstacle();
            obstacleTimer = 0;
        }
    }

    function updateClouds() {
        if (++cloudTimer > 100) {
            const el = document.createElement('div');
            el.className = 'cloud';
            const baseWidth = 60 + Math.random() * 40;
            el.style.width = baseWidth + 'px';
            el.style.height = (baseWidth * 0.5) + 'px';
            el.style.background = cloudColors[Math.floor(Math.random() * cloudColors.length)];
            el.style.top = (30 + Math.random() * 70) + 'px';
            const duration = 20 + Math.random() * 10;
            el.style.animationDuration = duration + 's';
            container.appendChild(el);
            setTimeout(() => { if(el.parentNode) container.removeChild(el); }, duration * 1000);
            cloudTimer = 0;
        }
    }

    function checkCollisions() {
        // 碰撞框稍微縮小（Padding），增加遊戲容錯感
        let dRect = { 
            left: dino.x + 12, 
            right: dino.x + dino.width - 12, 
            top: dino.y + dino.height - 8, 
            bottom: dino.y 
        };

        for (let o of obstacles) {
            let oRect = { 
                left: o.x + 5, 
                right: o.x + o.width - 5, 
                top: o.y + o.height, 
                bottom: o.y 
            };

            if (dRect.right > oRect.left && dRect.left < oRect.right && dRect.top > oRect.bottom && dRect.bottom < oRect.top) {
                gameOver();
            }
        }
    }

    function gameOver() { 
        isGameRunning = false; 
        finalScoreElement.innerText = "最終得分: " + Math.floor(score);
        messageElement.style.display = 'block'; 
    }

    function resetGame() {
        isGameRunning = true; 
        currentEvo = 'normal'; 
        score = 0; 
        gameSpeed = 6; 
        dino.y = 0; 
        dino.vy = 0; 
        obstacleTimer = 0;
        messageElement.style.display = 'none';
        dinoElement.className = '';
        evoStatusElement.innerText = '狀態: 幼年體';
        evoStatusElement.style.color = '#535353';
        
        obstacles.forEach(o => container.removeChild(o.element));
        obstacles = [];
        
        gameLoop();
    }

    // 啟動遊戲
    gameLoop();
</script>
</body>
</html>
