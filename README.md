<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Void Warden: Rich Maps</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body { margin: 0; overflow: hidden; background: #050508; color: white; font-family: sans-serif; }
        canvas { display: block; background: radial-gradient(circle, #1a1a2e 0%, #050508 100%); }
        #ui { position: absolute; top: 20px; left: 20px; right: 20px; pointer-events: none; }
        .bar-container { width: 100%; height: 12px; background: #222; border: 2px solid #444; border-radius: 6px; overflow: hidden; margin-bottom: 8px; }
        #boss-hp { width: 100%; height: 100%; background: linear-gradient(90deg, #ff0055, #aa0033); transition: width 0.3s; }
        #player-hp { width: 100%; height: 100%; background: linear-gradient(90deg, #00ff88, #00aa66); transition: width 0.3s; }
        .label { font-size: 14px; font-weight: bold; text-transform: uppercase; margin-bottom: 4px; display: flex; justify-content: space-between; }
        #msg-overlay { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); visibility: hidden; pointer-events: auto; z-index: 100; }
        .btn { padding: 12px 24px; background: #ff0055; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; margin-top: 20px; }
        .controls-hint { position: absolute; bottom: 20px; left: 20px; color: #666; font-size: 12px; }
        #audio-prompt { position: absolute; bottom: 60px; left: 20px; background: rgba(0,0,0,0.7); padding: 8px 12px; border-radius: 4px; border-left: 4px solid #00ff88; font-size: 12px; transition: opacity 0.5s; }
        .map-name { position: absolute; top: 100px; left: 50%; transform: translateX(-50%); font-size: 32px; font-weight: 900; opacity: 0; transition: opacity 1.5s; color: rgba(255,255,255,0.1); pointer-events: none; text-transform: uppercase; letter-spacing: 8px; text-align: center; }
    </style>
</head>
<body>

<div id="ui">
    <div>
        <div class="label"><span>虛空守望者 - 阿爾法</span><span id="boss-phase-text">PHASE 1</span></div>
        <div class="bar-container"><div id="boss-hp"></div></div>
    </div>
    <div style="width: 250px; margin-top: 20px;">
        <div class="label"><span>英雄 (Hero)</span></div>
        <div class="bar-container"><div id="player-hp"></div></div>
    </div>
</div>

<div id="map-display" class="map-name">地圖載入中...</div>
<div id="audio-prompt">點擊畫面任何地方以啟動音樂</div>

<div id="msg-overlay">
    <h1 id="msg-title" class="text-6xl font-black mb-4">遊戲結束</h1>
    <p id="msg-desc" class="text-xl">阿爾法守住了虛空...</p>
    <button class="btn" onclick="resetGame()">再次挑戰（隨機新地圖）</button>
</div>

<div class="controls-hint">
    [A/D] 移動 | [W] 跳躍 | [J/左鍵] 近戰 | [I/右鍵] 遠射 | [K/Shift] 衝刺
</div>

<canvas id="gameCanvas"></canvas>

<script>
/**
 * 背景音樂系統
 */
const bgm = new Audio();
bgm.src = encodeURIComponent("輕快的一天 (3).mp3");
bgm.loop = true;
bgm.volume = 0.4;
let audioStarted = false;

function startAudio() {
    if (audioStarted) return;
    bgm.play().then(() => {
        audioStarted = true;
        document.getElementById('audio-prompt').style.opacity = '0';
    }).catch(() => {});
}
window.addEventListener('mousedown', startAudio);
window.addEventListener('keydown', startAudio);

/**
 * 地圖庫定義：增加特殊物件 (跳躍台、陷阱、裝飾)
 */
const mapTemplates = [
    {
        name: "高聳神殿 (The Tall Temple)",
        color: "#2d1b4e",
        decor: "columns",
        elements: [
            { x: 0.1, y: 0.7, w: 0.15, h: 20 },
            { x: 0.75, y: 0.7, w: 0.15, h: 20 },
            { x: 0.45, y: 0.75, w: 0.1, h: 15, type: 'jumpPad' }, // 跳躍台
            { x: 0.35, y: 0.5, w: 0.3, h: 20 },
            { x: 0.2, y: 0.3, w: 0.1, h: 20 },
            { x: 0.7, y: 0.3, w: 0.1, h: 20 }
        ]
    },
    {
        name: "尖刺荒野 (Thorny Wastes)",
        color: "#1b2d2d",
        decor: "sparks",
        elements: [
            { x: 0.2, y: 0.65, w: 0.15, h: 15, moving: true, range: 80 },
            { x: 0.65, y: 0.65, w: 0.15, h: 15, moving: true, range: -80 },
            { x: 0.4, y: 0.45, w: 0.2, h: 15 },
            { x: 0.25, y: 0.9, w: 0.1, h: 20, type: 'trap' }, // 尖刺
            { x: 0.65, y: 0.9, w: 0.1, h: 20, type: 'trap' }, // 尖刺
            { x: 0.05, y: 0.4, w: 0.1, h: 15 },
            { x: 0.85, y: 0.4, w: 0.1, h: 15 }
        ]
    },
    {
        name: "虛空監獄 (The Abyss Cage)",
        color: "#2a0a0a",
        decor: "chains",
        elements: [
            { x: 0.05, y: 0.75, w: 0.2, h: 15 },
            { x: 0.75, y: 0.75, w: 0.2, h: 15 },
            { x: 0.3, y: 0.85, w: 0.4, h: 15, type: 'trap' }, // 底部大陷阱
            { x: 0.4, y: 0.6, w: 0.2, h: 15, moving: true, range: 120 },
            { x: 0.45, y: 0.3, w: 0.1, h: 15, type: 'jumpPad' }
        ]
    }
];

/**
 * 遊戲引擎
 */
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
let width, height;
let gameActive = true;
let currentMap = null;
let platforms = [];
let particles = [];
let bgParticles = [];
let screenShake = 0;

const player = {
    x: 100, y: 0, w: 30, h: 45, vx: 0, vy: 0, hp: 100, maxHp: 100,
    grounded: false, jumpCount: 0, dashCooldown: 0, isDashing: 0,
    attackCooldown: 0, isAttacking: 0, shootCooldown: 0, facing: 1, invul: 0,
    ghosts: [], projectiles: []
};

const boss = {
    x: 0, y: 0, w: 80, h: 120, vx: 0, vy: 0, hp: 1000, maxHp: 1000,
    phase: 1, state: 'IDLE', stateTimer: 0, attackType: '', facing: -1, bullets: []
};

const keys = {};

function initMap() {
    const template = mapTemplates[Math.floor(Math.random() * mapTemplates.length)];
    currentMap = template;
    
    const display = document.getElementById('map-display');
    display.innerText = template.name;
    display.style.opacity = '1';
    setTimeout(() => { display.style.opacity = '0'; }, 3000);

    // 基礎地板
    platforms = [{ x: 0, y: height - 100, w: width, h: 100, type: 'floor' }];
    
    template.elements.forEach(el => {
        platforms.push({
            x: el.x * width,
            y: el.y * height,
            baseX: el.x * width,
            w: el.w * width,
            h: el.h,
            type: el.type || 'platform',
            moving: el.moving || false,
            range: el.range || 0,
            timer: Math.random() * 10
        });
    });

    // 初始化背景粒子
    bgParticles = [];
    for(let i=0; i<50; i++) {
        bgParticles.push({
            x: Math.random() * width,
            y: Math.random() * height,
            size: Math.random() * 3,
            speed: Math.random() * 0.5 + 0.2
        });
    }
}

function resize() {
    width = window.innerWidth;
    height = window.innerHeight;
    canvas.width = width;
    canvas.height = height;
    if (currentMap) {
        platforms[0].y = height - 100;
        platforms[0].w = width;
    }
}

function update() {
    if (!gameActive) return;

    // 背景粒子
    bgParticles.forEach(p => {
        p.y -= p.speed;
        if (p.y < 0) p.y = height;
    });

    // 平台邏輯
    platforms.forEach(p => {
        if (p.moving) {
            p.timer += 0.02;
            p.x = p.baseX + Math.sin(p.timer) * p.range;
        }
    });

    handlePlayerLogic();
    handleBossAI();
    handleCollisions();

    draw();
    requestAnimationFrame(update);
}

function handlePlayerLogic() {
    if (player.invul > 0) player.invul--;
    if (player.dashCooldown > 0) player.dashCooldown--;
    if (player.attackCooldown > 0) player.attackCooldown--;
    if (player.shootCooldown > 0) player.shootCooldown--;
    if (player.isAttacking > 0) player.isAttacking--;
    
    if (player.isDashing > 0) {
        player.isDashing--;
        player.ghosts.push({x: player.x, y: player.y, life: 0.5});
    } else {
        if (keys['a'] || keys['arrowleft']) { player.vx = -7; player.facing = -1; }
        else if (keys['d'] || keys['arrowright']) { player.vx = 7; player.facing = 1; }
        else { player.vx *= 0.8; }
    }

    player.vy += 0.6; // Gravity
    player.x += player.vx;
    player.y += player.vy;

    if (player.x < 0) player.x = 0;
    if (player.x + player.w > width) player.x = width - player.w;

    player.projectiles.forEach((p, i) => {
        p.x += p.vx;
        if (p.x > boss.x && p.x < boss.x + boss.w && p.y > boss.y && p.y < boss.y + boss.h) {
            damageBoss(15);
            player.projectiles.splice(i, 1);
        }
        if (p.x < 0 || p.x > width) player.projectiles.splice(i, 1);
    });

    player.ghosts.forEach((g, i) => {
        g.life -= 0.05;
        if (g.life <= 0) player.ghosts.splice(i, 1);
    });
}

function handleCollisions() {
    player.grounded = false;
    platforms.forEach(p => {
        // 核心碰撞偵測
        if (player.x + player.w > p.x && player.x < p.x + p.w &&
            player.y + player.h > p.y && player.y + player.h < p.y + p.h + Math.max(5, player.vy)) {
            
            if (p.type === 'trap') {
                damagePlayer(1); // 尖刺持續傷害
                return;
            }

            if (p.type === 'jumpPad') {
                player.vy = -22;
                player.jumpCount = 1;
                screenShake = 5;
                return;
            }

            // 普通平台著地
            if (player.vy >= 0) {
                player.y = p.y - player.h;
                player.vy = 0;
                player.grounded = true;
                player.jumpCount = 0;
            }
        }
    });
}

function handleBossAI() {
    boss.stateTimer--;
    const dx = (player.x + player.w/2) - (boss.x + boss.w/2);
    boss.facing = dx > 0 ? 1 : -1;

    if (boss.state === 'IDLE') {
        boss.vx = dx > 0 ? 2.5 : -2.5;
        if (boss.stateTimer <= 0) {
            boss.state = 'PREP';
            boss.stateTimer = 35;
            boss.attackType = Math.random() > 0.4 ? 'SLAM' : 'BURST';
        }
    } else if (boss.state === 'PREP') {
        boss.vx *= 0.5;
        if (boss.stateTimer <= 0) {
            boss.state = 'ATTACK';
            if (boss.attackType === 'BURST') {
                for(let i=0; i<10; i++){
                    const a = (Math.PI*2/10)*i;
                    boss.bullets.push({x: boss.x+boss.w/2, y: boss.y+boss.h/2, vx: Math.cos(a)*6, vy: Math.sin(a)*6});
                }
                boss.stateTimer = 25;
            } else { boss.stateTimer = 10; }
        }
    } else if (boss.state === 'ATTACK') {
        if (boss.attackType === 'SLAM' && boss.stateTimer === 1) {
            screenShake = 12;
            if (player.grounded && Math.abs(dx) < 350) damagePlayer(25);
        }
        if (boss.stateTimer <= 0) { boss.state = 'IDLE'; boss.stateTimer = 50; }
    }

    boss.x += boss.vx;
    boss.y = (height - 100) - boss.h;

    boss.bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if (player.invul === 0 && Math.abs(b.x - (player.x+player.w/2)) < 25 && Math.abs(b.y - (player.y+player.h/2)) < 25) {
            damagePlayer(15);
            boss.bullets.splice(i, 1);
        }
        if (b.x < -50 || b.x > width+50 || b.y < -50 || b.y > height+50) boss.bullets.splice(i, 1);
    });
}

function damagePlayer(amt) {
    if (player.invul > 0 || player.isDashing > 0) return;
    player.hp -= amt;
    player.invul = 60;
    screenShake = 15;
    document.getElementById('player-hp').style.width = Math.max(0, player.hp) + '%';
    if (player.hp <= 0) gameOver(false);
}

function damageBoss(amt) {
    boss.hp -= amt;
    document.getElementById('boss-hp').style.width = (boss.hp/1000*100) + '%';
    if (boss.hp <= 0) gameOver(true);
}

function gameOver(win) {
    gameActive = false;
    const overlay = document.getElementById('msg-overlay');
    overlay.style.visibility = 'visible';
    document.getElementById('msg-title').innerText = win ? "擊敗了守望者！" : "你消失在虛空中";
    document.getElementById('msg-title').style.color = win ? "#00ff88" : "#ff0055";
}

function resetGame() {
    player.hp = 100; player.x = 100; player.y = 0; player.vx = 0; player.vy = 0;
    player.projectiles = [];
    boss.hp = 1000; boss.bullets = []; boss.state = 'IDLE';
    document.getElementById('player-hp').style.width = '100%';
    document.getElementById('boss-hp').style.width = '100%';
    document.getElementById('msg-overlay').style.visibility = 'hidden';
    initMap();
    gameActive = true;
    update();
}

function draw() {
    ctx.save();
    if (screenShake > 0) {
        ctx.translate((Math.random()-0.5)*screenShake, (Math.random()-0.5)*screenShake);
        screenShake *= 0.9;
    }
    ctx.clearRect(0, 0, width, height);

    // 繪製裝飾背景 (粒子)
    ctx.fillStyle = "rgba(255, 255, 255, 0.2)";
    bgParticles.forEach(p => ctx.fillRect(p.x, p.y, p.size, p.size));

    // 繪製地圖元素
    platforms.forEach(p => {
        if (p.type === 'trap') {
            ctx.fillStyle = "#ff4444";
            for(let i=0; i<p.w; i+=10) {
                ctx.beginPath();
                ctx.moveTo(p.x + i, p.y + p.h);
                ctx.lineTo(p.x + i + 5, p.y);
                ctx.lineTo(p.x + i + 10, p.y + p.h);
                ctx.fill();
            }
        } else if (p.type === 'jumpPad') {
            ctx.fillStyle = "#00ffff";
            ctx.fillRect(p.x, p.y, p.w, p.h);
            ctx.fillStyle = "white";
            ctx.fillRect(p.x + p.w/4, p.y + p.h/4, p.w/2, p.h/2);
        } else {
            ctx.fillStyle = currentMap ? currentMap.color : "#111";
            ctx.fillRect(p.x, p.y, p.w, p.h);
            ctx.fillStyle = "rgba(255,255,255,0.1)";
            ctx.fillRect(p.x, p.y, p.w, 5); // 平台頂部光澤
        }
    });

    // 玩家殘影
    player.ghosts.forEach(g => {
        ctx.globalAlpha = g.life; ctx.fillStyle = "#00ff88";
        ctx.fillRect(g.x, g.y, player.w, player.h);
    });
    ctx.globalAlpha = 1;

    // 繪製玩家子彈 (發光)
    ctx.shadowBlur = 10; ctx.shadowColor = "#00ff88"; ctx.fillStyle = "#00ff88";
    player.projectiles.forEach(p => ctx.beginPath() || ctx.arc(p.x, p.y, 6, 0, Math.PI*2) || ctx.fill());
    ctx.shadowBlur = 0;

    // 繪製 Boss 子彈
    ctx.fillStyle = "#ff0055";
    boss.bullets.forEach(b => ctx.beginPath() || ctx.arc(b.x, b.y, 8, 0, Math.PI*2) || ctx.fill());

    // 繪製玩家
    if (player.invul % 6 < 3) {
        ctx.fillStyle = "#00ff88";
        ctx.fillRect(player.x, player.y, player.w, player.h);
        ctx.fillStyle = "white"; // 眼睛
        ctx.fillRect(player.facing === 1 ? player.x+20 : player.x+2, player.y+10, 8, 8);
    }

    // 繪製 Boss
    ctx.fillStyle = boss.state === 'PREP' ? "#ffaa00" : "#ff0055";
    ctx.fillRect(boss.x, boss.y, boss.w, boss.h);
    ctx.fillStyle = "black"; // Boss 眼睛
    ctx.fillRect(boss.facing === 1 ? boss.x+boss.w-25 : boss.x+5, boss.y+20, 20, 20);
    
    ctx.restore();
}

/**
 * 輸入監聽
 */
window.addEventListener('keydown', e => {
    const k = e.key.toLowerCase();
    keys[k] = true;
    if ((k === 'w' || k === ' ') && player.jumpCount < 2) {
        player.vy = -14; player.jumpCount++;
    }
    if (k === 'k' || e.shiftKey) {
        if (player.dashCooldown === 0) {
            player.isDashing = 12; player.dashCooldown = 50;
            player.vx = player.facing * 18; player.vy = 0;
        }
    }
    if (k === 'j') playerMelee();
    if (k === 'i') playerShoot();
});
window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);
window.addEventListener('mousedown', e => {
    if (e.button === 0) playerMelee();
    if (e.button === 2) playerShoot();
});
function playerMelee() {
    if (player.attackCooldown === 0) {
        player.isAttacking = 10; player.attackCooldown = 20;
        // 近戰範圍檢測
        const dx = (boss.x + boss.w/2) - (player.x + player.w/2);
        const dy = (boss.y + boss.h/2) - (player.y + player.h/2);
        if (Math.abs(dx) < 100 && Math.abs(dy) < 80) damageBoss(40);
    }
}
function playerShoot() {
    if (player.shootCooldown === 0) {
        player.projectiles.push({ x: player.x+player.w/2, y: player.y+player.h/2, vx: player.facing*13 });
        player.shootCooldown = 30;
    }
}
window.addEventListener('contextmenu', e => e.preventDefault());
window.addEventListener('resize', resize);

resize();
initMap();
update();
</script>
</body>
</html>
