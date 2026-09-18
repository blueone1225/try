<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>惡魔城：五大關卡完整冒險版</title>
    <style>
        * { box-sizing: border-box; }
        body {
            background-color: #050308;
            color: #fff;
            font-family: monospace;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            margin: 0;
            padding: 10px;
        }
        h1 { color: #d4af37; font-size: 16px; margin: 0 0 4px 0; text-shadow: 1px 1px #000; }
        .instructions { color: #aaa; font-size: 11px; margin-bottom: 8px; }
        canvas {
            border: 2px solid #554422;
            box-shadow: 0 0 20px rgba(180, 50, 50, 0.4);
            background: #000;
            width: 100%;
            max-width: 800px;
            height: auto;
            aspect-ratio: 16 / 9;
            image-rendering: pixelated;
            touch-action: none;
        }
        .controls {
            margin-top: 10px;
            display: flex;
            gap: 8px;
            flex-wrap: wrap;
            justify-content: center;
        }
        button {
            padding: 10px 14px;
            background: #22152e;
            color: #d4af37;
            border: 1px solid #775599;
            border-radius: 4px;
            font-weight: bold;
            font-size: 13px;
            user-select: none;
        }
        button:active { background: #d4af37; color: #000; }
    </style>
</head>
<body>

    <h1>惡魔城：五大關卡完整冒險</h1>
    <div class="instructions">[A/D] 移動 | [W] 跳躍 | [J] 揮劍 | [K] 投擲戰斧 | 擊敗守門怪取得鑰匙開門過關</div>

    <canvas id="gameCanvas" width="800" height="450"></canvas>

    <div class="controls">
        <button id="btnLeft">← 左移</button>
        <button id="btnRight">右移 →</button>
        <button id="btnJump">跳躍 (W)</button>
        <button id="btnAttack">揮劍 (J)</button>
        <button id="btnSkill">投斧 (K)</button>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        ctx.imageSmoothingEnabled = false;

        // 離線點陣圖 Atlas
        function buildHighResSpriteAtlas() {
            const atlas = document.createElement('canvas');
            atlas.width = 512;
            atlas.height = 512;
            const aCtx = atlas.getContext('2d');

            const drawPixelArt = (offsetX, offsetY, scale, matrix, colorMap) => {
                for (let r = 0; r < matrix.length; r++) {
                    for (let c = 0; c < matrix[r].length; c++) {
                        const code = matrix[r][c];
                        if (code !== '.') {
                            aCtx.fillStyle = colorMap[code];
                            aCtx.fillRect(offsetX + c * scale, offsetY + r * scale, scale, scale);
                        }
                    }
                }
            };

            // 1. 瑪利亞 (HERO)
            const mariaMatrix = [
                "..YYYYY..",
                ".YYYYYYY.",
                ".YOSSSOY.",
                "..SSSS...",
                ".GGGGGGG.",
                "GGGGGGGGG",
                "GGGGGGGGG",
                ".WWWWWW..",
                ".WW..WW..",
                ".SS..SS.."
            ];
            drawPixelArt(0, 0, 4, mariaMatrix, { 'Y': '#ffd700', 'O': '#ffb080', 'S': '#ffe0bd', 'G': '#1b9aaa', 'W': '#ffffff' });

            // 2. 狼人 (ENEMY/AI)
            const wolfMatrix = [
                "...P...P...",
                "..PPPPPPP..",
                ".PPRPPPPPR.",
                ".PPPPPPPPP.",
                "PPPPPPPPPPP",
                "PPPPPPPPPPP",
                ".PPPPPPPPP.",
                "..PPP.PPP..",
                "..PP...PP.."
            ];
            drawPixelArt(100, 0, 5, wolfMatrix, { 'P': '#6a0da5', 'R': '#ff2222' });

            // 3. 骷髏兵 (BOSS/STAGE)
            const skelMatrix = [
                "..WWWW..",
                ".WWRRW..",
                "..WWWW..",
                "...WW...",
                ".WWWWWW.",
                "..WWWW..",
                "..W..W..",
                ".WW..WW."
            ];
            drawPixelArt(200, 0, 5, skelMatrix, { 'W': '#e0e0e0', 'R': '#ff0000' });

            // 4. 戰斧 (ATTACK)
            const axeMatrix = [
                "...CC...",
                "..CCCC..",
                ".CCCCCC.",
                "...HH...",
                "...HH...",
                "...HH..."
            ];
            drawPixelArt(300, 0, 5, axeMatrix, { 'C': '#d4af37', 'H': '#8b4513' });

            // 5. 愛心與鑰匙 (ITEMS)
            drawPixelArt(0, 100, 4, [".RR.RR.", "RRRRRRR", "RRRRRRR", ".RRRRR.", "..RRR..", "...R..."], { 'R': '#ff0055' });
            drawPixelArt(50, 100, 4, [".YYY.", "Y...Y", ".YYY.", "..Y..", "..YY.", "..Y..", "..YY."], { 'Y': '#ffd700' });

            // 6. 惡魔城大門 (DOOR)
            drawPixelArt(100, 100, 6, [
                "DDDDDDDD", "D......D", "D.D..D.D", "D.DDDD.D",
                "D..K...D", "D.DDDD.D", "D......D", "DDDDDDDD"
            ], { 'D': '#4a2c11', 'K': '#ffd700', '.': '#2b180a' });

            return atlas;
        }

        const atlas = buildHighResSpriteAtlas();

        // 5 個地圖關卡資料庫
        const stageConfigs = [
            {
                name: "STAGE 1: 惡魔城大門前庭",
                bg: "#0f0817",
                platformColor: "#4a4459",
                platforms: [
                    { x: 0, y: 340, w: 800, h: 110 },
                    { x: 200, y: 240, w: 180, h: 20 },
                    { x: 450, y: 180, w: 140, h: 20 }
                ],
                door: { x: 720, y: 244, w: 48, h: 96 },
                heartPos: { x: 270, y: 200 },
                werewolves: [{ x: 300, y: 185, minX: 200, maxX: 380, hp: 40 }],
                boss: { x: 630, y: 270, hp: 60, name: "門衛骷髏兵", type: "skeleton" }
            },
            {
                name: "STAGE 2: 大理石廊柱通道",
                bg: "#150d24",
                platformColor: "#3a506b",
                platforms: [
                    { x: 0, y: 340, w: 800, h: 110 },
                    { x: 120, y: 260, w: 120, h: 20 },
                    { x: 320, y: 190, w: 160, h: 20 },
                    { x: 550, y: 250, w: 120, h: 20 }
                ],
                door: { x: 730, y: 244, w: 48, h: 96 },
                heartPos: { x: 380, y: 150 },
                werewolves: [
                    { x: 140, y: 205, minX: 120, maxX: 240, hp: 50 },
                    { x: 350, y: 135, minX: 320, maxX: 480, hp: 50 }
                ],
                boss: { x: 650, y: 270, hp: 90, name: "巡廊長槍骷髏", type: "skeleton" }
            },
            {
                name: "STAGE 3: 廢棄地牢層",
                bg: "#0b131f",
                platformColor: "#2b3a41",
                platforms: [
                    { x: 0, y: 340, w: 800, h: 110 },
                    { x: 150, y: 270, w: 100, h: 20 },
                    { x: 300, y: 210, w: 100, h: 20 },
                    { x: 450, y: 150, w: 100, h: 20 },
                    { x: 600, y: 220, w: 140, h: 20 }
                ],
                door: { x: 720, y: 124, w: 48, h: 96 },
                heartPos: { x: 490, y: 110 },
                werewolves: [
                    { x: 160, y: 215, minX: 150, maxX: 250, hp: 60 },
                    { x: 460, y: 95, minX: 450, maxX: 550, hp: 60 }
                ],
                boss: { x: 640, y: 150, hp: 120, name: "地牢狂暴狼王", type: "werewolf" }
            },
            {
                name: "STAGE 4: 鐘樓高塔迴廊",
                bg: "#1f100b",
                platformColor: "#5c3d2e",
                platforms: [
                    { x: 0, y: 340, w: 800, h: 110 },
                    { x: 100, y: 250, w: 140, h: 20 },
                    { x: 280, y: 180, w: 140, h: 20 },
                    { x: 480, y: 240, w: 120, h: 20 },
                    { x: 650, y: 170, w: 120, h: 20 }
                ],
                door: { x: 710, y: 74, w: 48, h: 96 },
                heartPos: { x: 330, y: 140 },
                werewolves: [
                    { x: 120, y: 195, minX: 100, maxX: 240, hp: 70 },
                    { x: 500, y: 185, minX: 480, maxX: 600, hp: 70 }
                ],
                boss: { x: 680, y: 100, hp: 150, name: "鐘樓守護骷髏將", type: "skeleton" }
            },
            {
                name: "STAGE 5: 德古拉王座大廳",
                bg: "#240008",
                platformColor: "#6b0f1a",
                platforms: [
                    { x: 0, y: 340, w: 800, h: 110 },
                    { x: 180, y: 230, w: 440, h: 20 }
                ],
                door: { x: 720, y: 244, w: 48, h: 96 },
                heartPos: { x: 400, y: 190 },
                werewolves: [
                    { x: 200, y: 175, minX: 180, maxX: 350, hp: 90 },
                    { x: 450, y: 175, minX: 380, maxX: 600, hp: 90 }
                ],
                boss: { x: 620, y: 270, hp: 250, name: "惡魔城大領主", type: "skeleton" }
            }
        ];

        let currentStageIdx = 0;

        // 控制與按鍵
        const keys = {};
        window.addEventListener('keydown', e => keys[e.code] = true);
        window.addEventListener('keyup', e => keys[e.code] = false);

        const bindBtn = (id, code) => {
            const btn = document.getElementById(id);
            btn.addEventListener('touchstart', (e) => { e.preventDefault(); keys[code] = true; });
            btn.addEventListener('touchend', (e) => { e.preventDefault(); keys[code] = false; });
            btn.addEventListener('mousedown', () => keys[code] = true);
            btn.addEventListener('mouseup', () => keys[code] = false);
        };
        bindBtn('btnLeft', 'KeyA');
        bindBtn('btnRight', 'KeyD');
        bindBtn('btnJump', 'KeyW');
        bindBtn('btnAttack', 'KeyJ');
        bindBtn('btnSkill', 'KeyK');

        // 遊戲角色與物件實體
        const player = {
            x: 80, y: 290, w: 36, h: 48,
            vx: 0, vy: 0, speed: 3.5, gravity: 0.55,
            isGrounded: true, isAttacking: false, facingRight: true,
            hp: 10, maxHp: 10, mp: 8, lives: 3,
            hasKey: false, hurtTimer: 0
        };

        let currentStage, doorObj, items, werewolves, bossObj;
        let axes = [];
        let enemyProjectiles = [];
        let stageClear = false;
        let gameFinished = false;
        let gameOver = false;

        // 初始化/載入關卡
        function loadStage(idx) {
            currentStageIdx = idx;
            currentStage = stageConfigs[idx];
            player.x = 50;
            player.y = 280;
            player.vx = 0;
            player.vy = 0;
            player.hasKey = false;
            axes = [];
            enemyProjectiles = [];
            stageClear = false;

            doorObj = { ...currentStage.door, locked: true };
            items = [
                { x: currentStage.heartPos.x, y: currentStage.heartPos.y, w: 24, h: 24, type: 'heart', active: true }
            ];

            werewolves = currentStage.werewolves.map(w => ({
                x: w.x, y: w.y, w: 55, h: 55,
                vx: 1.8, speed: 2.0,
                hp: w.hp, maxHp: w.hp, active: true,
                minX: w.minX, maxX: w.maxX, detectRange: 220
            }));

            const b = currentStage.boss;
            bossObj = {
                x: b.x, y: b.y, w: 45, h: 65,
                vx: 0, speed: 1.6,
                hp: b.hp, maxHp: b.hp, active: true,
                name: b.name, type: b.type,
                attackCooldown: 0, facingLeft: true,
                hasDroppedKey: false
            };
        }

        loadStage(0);

        function update() {
            if (stageClear || gameOver || gameFinished) return;

            if (player.hurtTimer > 0) player.hurtTimer--;

            // 玩家移動操作
            if (keys['KeyA'] || keys['ArrowLeft']) { player.vx = -player.speed; player.facingRight = false; }
            else if (keys['KeyD'] || keys['ArrowRight']) { player.vx = player.speed; player.facingRight = true; }
            else player.vx = 0;

            if ((keys['KeyW'] || keys['Space']) && player.isGrounded) {
                player.vy = -10.5;
                player.isGrounded = false;
            }

            // 揮劍
            if (keys['KeyJ'] && !player.isAttacking) {
                player.isAttacking = true;
                const hitRange = player.facingRight ? 
                    { x: player.x + player.w, y: player.y, w: 42, h: player.h } :
                    { x: player.x - 42, y: player.y, w: 42, h: player.h };

                werewolves.forEach(w => {
                    if (w.active && checkHit(hitRange, w)) w.hp -= 20;
                });
                if (bossObj.active && checkHit(hitRange, bossObj)) bossObj.hp -= 25;

                setTimeout(() => player.isAttacking = false, 150);
            }

            // 投斧技能
            if (keys['KeyK'] && player.mp > 0) {
                keys['KeyK'] = false;
                player.mp--;
                axes.push({
                    x: player.x + (player.facingRight ? 20 : -10),
                    y: player.y,
                    vx: player.facingRight ? 5.5 : -5.5,
                    vy: -7,
                    rot: 0
                });
            }

            // 玩家斧頭碰撞
            for (let i = axes.length - 1; i >= 0; i--) {
                const axe = axes[i];
                axe.x += axe.vx;
                axe.y += axe.vy;
                axe.vy += 0.4;
                axe.rot += 0.25;

                let hit = false;
                werewolves.forEach(w => {
                    if (w.active && checkHit({x: axe.x, y: axe.y, w: 24, h: 24}, w)) {
                        w.hp -= 25;
                        hit = true;
                    }
                });
                if (!hit && bossObj.active && checkHit({x: axe.x, y: axe.y, w: 24, h: 24}, bossObj)) {
                    bossObj.hp -= 30;
                    hit = true;
                }
                if (hit || axe.y > 420) axes.splice(i, 1);
            }

            // 重力物理與地形碰撞
            player.vy += player.gravity;
            player.x += player.vx;
            player.y += player.vy;

            player.isGrounded = false;
            currentStage.platforms.forEach(p => {
                if (player.vy >= 0 && 
                    player.x + player.w > p.x && 
                    player.x < p.x + p.w &&
                    player.y + player.h >= p.y && 
                    player.y + player.h <= p.y + 12) {
                    player.y = p.y - player.h;
                    player.vy = 0;
                    player.isGrounded = true;
                }
            });

            // 門區域判定
            if (doorObj.locked && checkHit(player, doorObj)) {
                if (player.hasKey) {
                    doorObj.locked = false;
                } else {
                    player.x = doorObj.x - player.w;
                }
            } else if (!doorObj.locked && checkHit(player, doorObj)) {
                if (currentStageIdx < stageConfigs.length - 1) {
                    stageClear = true;
                    setTimeout(() => loadStage(currentStageIdx + 1), 1200);
                } else {
                    gameFinished = true;
                }
            }

            if (player.x < 0) player.x = 0;
            if (player.x + player.w > canvas.width) player.x = canvas.width - player.w;

            // 狼人 AI
            werewolves.forEach(w => {
                if (w.hp <= 0) w.active = false;
                if (!w.active) return;

                const distToPlayerX = player.x - w.x;
                const distToPlayerY = Math.abs(player.y - w.y);

                if (Math.abs(distToPlayerX) < w.detectRange && distToPlayerY < 80) {
                    w.vx = distToPlayerX > 0 ? w.speed : -w.speed;
                } else {
                    if (w.x < w.minX) w.vx = w.speed;
                    if (w.x > w.maxX) w.vx = -w.speed;
                }
                w.x += w.vx;

                if (checkHit(player, w)) takeDamage(2);
            });

            // 守門 BOSS AI
            if (bossObj.hp <= 0 && bossObj.active) {
                bossObj.active = false;
                // 擊敗 BOSS 掉落過關鑰匙！
                if (!bossObj.hasDroppedKey) {
                    items.push({ x: bossObj.x, y: bossObj.y + 20, w: 20, h: 28, type: 'key', active: true });
                    bossObj.hasDroppedKey = true;
                }
            }

            if (bossObj.active) {
                const distToPlayer = player.x - bossObj.x;
                bossObj.facingLeft = distToPlayer < 0;

                if (Math.abs(distToPlayer) < 320) {
                    if (Math.abs(distToPlayer) > 70) {
                        bossObj.x += distToPlayer > 0 ? bossObj.speed : -bossObj.speed;
                    }
                    bossObj.attackCooldown++;
                    if (bossObj.attackCooldown > 80) {
                        enemyProjectiles.push({
                            x: bossObj.x + (bossObj.facingLeft ? -10 : 40),
                            y: bossObj.y + 20,
                            vx: bossObj.facingLeft ? -4.5 : 4.5,
                            vy: -0.5
                        });
                        bossObj.attackCooldown = 0;
                    }
                }
                if (checkHit(player, bossObj)) takeDamage(2);
            }

            // 敵方子彈
            for (let i = enemyProjectiles.length - 1; i >= 0; i--) {
                const proj = enemyProjectiles[i];
                proj.x += proj.vx;
                proj.y += proj.vy;

                if (checkHit(player, { x: proj.x, y: proj.y, w: 12, h: 12 })) {
                    takeDamage(1);
                    enemyProjectiles.splice(i, 1);
                    continue;
                }
                if (proj.x < 0 || proj.x > canvas.width) enemyProjectiles.splice(i, 1);
            }

            // 道具拾取
            items.forEach(item => {
                if (item.active && checkHit(player, item)) {
                    item.active = false;
                    if (item.type === 'heart') {
                        player.hp = Math.min(player.hp + 4, player.maxHp);
                        player.mp += 3;
                    } else if (item.type === 'key') {
                        player.hasKey = true;
                    }
                }
            });
        }

        function takeDamage(dmg) {
            if (player.hurtTimer > 0) return;
            player.hp -= dmg;
            player.hurtTimer = 40;
            player.vy = -4;
            player.vx = player.facingRight ? -3 : 3;
            if (player.hp <= 0) {
                player.hp = 0;
                gameOver = true;
            }
        }

        function checkHit(r1, r2) {
            return !(r1.x > r2.x + r2.w || r1.x + r1.w < r2.x ||
                     r1.y > r2.y + r2.h || r1.y + r1.h < r2.y);
        }

        function render() {
            // 背景
            ctx.fillStyle = currentStage.bg;
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // 背景法陣
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.08)';
            ctx.lineWidth = 4;
            ctx.beginPath();
            ctx.arc(400, 200, 140, 0, Math.PI * 2);
            ctx.stroke();

            // 1. 地形
            currentStage.platforms.forEach((p, idx) => {
                ctx.fillStyle = currentStage.platformColor;
                ctx.fillRect(p.x, p.y, p.w, p.h);
                ctx.strokeStyle = 'rgba(0,0,0,0.4)';
                ctx.lineWidth = 2;
                for(let bx = p.x; bx < p.x + p.w; bx += 20) {
                    ctx.strokeRect(bx, p.y, 20, p.h);
                }
                if (idx === 0) {
                    ctx.fillStyle = '#0a0510';
                    ctx.fillRect(p.x, p.y + p.h, p.w, canvas.height - p.y);
                }
            });

            // 2. 門
            ctx.drawImage(atlas, 100, 100, 48, 48, doorObj.x, doorObj.y, doorObj.w, doorObj.h);
            if (doorObj.locked) {
                ctx.strokeStyle = '#ff2222';
                ctx.lineWidth = 2;
                ctx.strokeRect(doorObj.x, doorObj.y, doorObj.w, doorObj.h);
                ctx.fillStyle = '#ff2222';
                ctx.font = '10px monospace';
                ctx.fillText("LOCKED", doorObj.x + 4, doorObj.y - 6);
            } else {
                ctx.strokeStyle = '#00ffcc';
                ctx.lineWidth = 2;
                ctx.strokeRect(doorObj.x, doorObj.y, doorObj.w, doorObj.h);
            }

            // 3. 道具
            items.forEach(item => {
                if (!item.active) return;
                if (item.type === 'heart') {
                    ctx.drawImage(atlas, 0, 100, 28, 24, item.x, item.y, item.w, item.h);
                } else if (item.type === 'key') {
                    ctx.drawImage(atlas, 50, 100, 20, 28, item.x, item.y, item.w, item.h);
                }
            });

            // 4. 狼人敵人
            werewolves.forEach(w => {
                if (!w.active) return;
                ctx.save();
                if (w.vx < 0) {
                    ctx.translate(w.x + w.w, w.y);
                    ctx.scale(-1, 1);
                    ctx.drawImage(atlas, 100, 0, 55, 45, 0, 0, w.w, w.h);
                } else {
                    ctx.drawImage(atlas, 100, 0, 55, 45, w.x, w.y, w.w, w.h);
                }
                ctx.restore();
                ctx.fillStyle = 'red';
                ctx.fillRect(w.x, w.y - 8, w.w * (w.hp / w.maxHp), 4);
            });

            // 5. BOSS 守門怪
            if (bossObj.active) {
                ctx.save();
                if (!bossObj.facingLeft) {
                    ctx.translate(bossObj.x + bossObj.w, bossObj.y);
                    ctx.scale(-1, 1);
                    ctx.drawImage(atlas, bossObj.type === 'werewolf' ? 100 : 200, 0, 45, 45, 0, 0, bossObj.w, bossObj.h);
                } else {
                    ctx.drawImage(atlas, bossObj.type === 'werewolf' ? 100 : 200, 0, 45, 45, bossObj.x, bossObj.y, bossObj.w, bossObj.h);
                }
                ctx.restore();

                // 長槍
                if (bossObj.type === 'skeleton') {
                    ctx.fillStyle = '#d4af37';
                    ctx.fillRect(bossObj.facingLeft ? bossObj.x - 15 : bossObj.x + 20, bossObj.y + 30, 35, 4);
                }

                ctx.fillStyle = 'red';
                ctx.fillRect(bossObj.x - 10, bossObj.y - 12, (bossObj.w + 20) * (bossObj.hp / bossObj.maxHp), 5);
                ctx.fillStyle = '#ffd700';
                ctx.font = '10px monospace';
                ctx.fillText(bossObj.name, bossObj.x - 10, bossObj.y - 18);
            }

            // 敵子彈
            enemyProjectiles.forEach(p => {
                ctx.fillStyle = '#ff3300';
                ctx.fillRect(p.x, p.y, 10, 6);
            });

            // 6. 主角 (瑪利亞)
            if (player.hurtTimer % 4 < 2) {
                ctx.save();
                if (!player.facingRight) {
                    ctx.translate(player.x + player.w, player.y);
                    ctx.scale(-1, 1);
                    ctx.drawImage(atlas, 0, 0, 36, 40, 0, 0, player.w, player.h);
                } else {
                    ctx.drawImage(atlas, 0, 0, 36, 40, player.x, player.y, player.w, player.h);
                }
                ctx.restore();
            }

            // 揮劍
            if (player.isAttacking) {
                ctx.strokeStyle = '#00ffff';
                ctx.lineWidth = 4;
                ctx.beginPath();
                const slashX = player.facingRight ? player.x + player.w + 10 : player.x - 10;
                ctx.arc(slashX, player.y + 20, 20, player.facingRight ? -Math.PI/3 : Math.PI*2/3, player.facingRight ? Math.PI/3 : Math.PI*4/3);
                ctx.stroke();
            }

            // 玩家戰斧
            axes.forEach(axe => {
                ctx.save();
                ctx.translate(axe.x, axe.y);
                ctx.rotate(axe.rot);
                ctx.drawImage(atlas, 300, 0, 30, 30, -12, -12, 24, 24);
                ctx.restore();
            });

            // 7. HUD
            ctx.fillStyle = 'rgba(0, 0, 0, 0.75)';
            ctx.fillRect(10, 10, 320, 55);
            ctx.strokeStyle = '#d4af37';
            ctx.strokeRect(10, 10, 320, 55);

            ctx.fillStyle = '#ffd700';
            ctx.font = '11px monospace';
            ctx.fillText(currentStage.name, 20, 24);

            ctx.fillStyle = '#fff';
            ctx.fillText(`MP: ${player.mp}`, 180, 24);
            ctx.fillText(player.hasKey ? "KEY: YES" : "KEY: NO", 240, 24);

            for (let i = 0; i < player.maxHp; i++) {
                ctx.fillStyle = i < player.hp ? '#e63946' : '#333';
                ctx.fillRect(20 + (i * 9), 34, 7, 14);
            }

            // 關卡切換 / 遊戲通關 / 失敗畫面
            if (stageClear) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.8)';
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                ctx.fillStyle = '#ffd700';
                ctx.font = '24px monospace';
                ctx.textAlign = 'center';
                ctx.fillText("STAGE CLEAR !", canvas.width / 2, 210);
                ctx.fillStyle = '#fff';
                ctx.font = '14px monospace';
                ctx.fillText("進入下一個關卡...", canvas.width / 2, 240);
            } else if (gameFinished) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.9)';
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                ctx.fillStyle = '#ffd700';
                ctx.font = '28px monospace';
                ctx.textAlign = 'center';
                ctx.fillText("VICTORY ! 惡魔城完全征服 !", canvas.width / 2, 200);
                ctx.fillStyle = '#fff';
                ctx.font = '15px monospace';
                ctx.fillText("你已成功突破五大關卡並擊敗惡魔城領主！", canvas.width / 2, 240);
            } else if (gameOver) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.85)';
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                ctx.fillStyle = '#ff2222';
                ctx.font = '28px monospace';
                ctx.textAlign = 'center';
                ctx.fillText("YOU DIED", canvas.width / 2, 200);
                ctx.fillStyle = '#fff';
                ctx.font = '15px monospace';
                ctx.fillText("重新載入頁面可再次挑戰五大關卡", canvas.width / 2, 240);
            }
        }

        function gameLoop() {
            update();
            render();
            requestAnimationFrame(gameLoop);
        }

        gameLoop();
    </script>
</body>
</html>
