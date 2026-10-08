<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Ведьмак: Пиксельный скроллер (Mobile & PC)</title>
    <style>
        body { 
            margin: 0; 
            background-color: #1a1a2e; 
            overflow: hidden; 
            font-family: monospace;
            touch-action: none; /* Отключаем стандартные скроллы и зумы браузера при касаниях */
        }
        canvas { 
            display: block;
            image-rendering: pixelated;
            image-rendering: crisp-edges;
            background-color: #000000;
        }
        #ui {
            position: absolute;
            top: 10px;
            left: 50%;
            transform: translateX(-50%);
            color: #c27c0e;
            background: rgba(0,0,0,0.7);
            padding: 5px 15px;
            border: 2px solid #c27c0e;
            border-radius: 10px;
            z-index: 10;
        }

        /* --- Стили для мобильных контролов --- */
        #controls {
            position: absolute;
            bottom: 20px;
            left: 20px;
            display: flex;
            align-items: flex-end;
            gap: 20px;
            z-index: 20;
            pointer-events: none; /* Пропускаем касания сквозь пустые места */
        }
        .joystick-base, .button {
            pointer-events: all; /* Включаем касания только для самих кнопок */
        }
        .joystick-base {
            width: 100px;
            height: 100px;
            background: rgba(50, 50, 50, 0.6);
            border-radius: 50%;
            border: 3px solid #c27c0e;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .joystick-thumb {
            width: 60px;
            height: 60px;
            background: #c27c0e;
            border-radius: 50%;
            border: 2px solid #8B4513;
        }
        .button {
            width: 80px;
            height: 80px;
            background: rgba(50, 50, 50, 0.6);
            border: 3px solid #c27c0e;
            border-radius: 15px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 14px;
            color: #c27c0e;
            text-align: center;
            line-height: 1.2;
        }
        .button:active { background: rgba(194, 124, 14, 0.3); }

        /* Скрываем джойстик на ПК, если есть мышь (базовая проверка) */
        @media (hover: hover) and (pointer: fine) {
            #controls { display: none; }
        }
    </style>
</head>
<body>
    <div id="ui">Здоровье: <span id="hp">100</span> | Броня: <span id="armor-status">Нет</span></div>
    <canvas id="gameCanvas"></canvas>

    <!-- Мобильные контролы -->
    <div id="controls">
        <div class="joystick-base" id="joystick">
            <div class="joystick-thumb" id="joystick-thumb"></div>
        </div>
        <div class="button" id="btn-jump" style="font-size: 12px;">ПРЫЖОК</div>
        <div class="button" id="btn-sign" style="font-size: 12px;">ААРД</div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const hpText = document.getElementById('hp');
        const armorStatus = document.getElementById('armor-status');

        // Настройка canvas на весь экран
        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        // --- Настройки мира ---
        const world = { width: 3000, height: canvas.height, scrollSpeed: 4 };

        // --- Спрайты (цвета для прототипа) ---
        const sprites = {
            geralt: { w: 40, h: 56, color: '#d4af37' },
            enemy: { w: 32, h: 40, color: '#555555' },
            ground: { w: 60, h: 30, color: '#8B4513' },
            trap: { w: 40, h: 40, color: '#800000' },
            armor: { w: 40, h: 40, color: '#a52a2a' },
            sign: { w: 30, h: 15, color: '#4169E1' }
        };

        // --- Игровые объекты ---
        let player = {
            x: 100, y: canvas.height - 100, 
            w: sprites.geralt.w, h: sprites.geralt.h,
            speed: 4, velY: 0, onGround: false,
            facing: 1, canCast: true, hp: 100,
            hasArmor: false
        };

        let keys = {};
        let enemies = [];
        let grounds = [];
        let traps = [];
        let armorPieces = [];

        // --- Мобильные контролы ---
        const joystickBase = document.getElementById('joystick');
        const joystickThumb = document.getElementById('joystick-thumb');
        let joystickActive = false;
        let joystickCenter = { x: 0, y: 0 };
        let moveInput = { x: 0, y: 0 };

        const btnJump = document.getElementById('btn-jump');
        const btnSign = document.getElementById('btn-sign');

        function updateJoystick(e) {
            const rect = joystickBase.getBoundingClientRect();
            const centerX = rect.left + rect.width / 2;
            const centerY = rect.top + rect.height / 2;
            const touchX = e.touches[0].clientX;
            const touchY = e.touches[0].clientY;

            const deltaX = touchX - centerX;
            const deltaY = touchY - centerY;
            const distance = Math.sqrt(deltaX * deltaX + deltaY * deltaY);
            const maxDistance = 40; // Радиус базы джойстика

            if (distance > maxDistance) {
                moveInput.x = (deltaX / distance) * maxDistance;
                moveInput.y = (deltaY / distance) * maxDistance;
            } else {
                moveInput.x = deltaX;
                moveInput.y = deltaY;
            }

            // Нормализуем для движения (важнее X, чем Y)
            const moveSpeed = 1;
            keys['ArrowLeft'] = moveInput.x < -10;
            keys['ArrowRight'] = moveInput.x > 10;
            
            joystickThumb.style.transform = `translate(${moveInput.x}px, ${moveInput.y}px)`;
        }

        function resetJoystick() {
            moveInput = { x: 0, y: 0 };
            keys['ArrowLeft'] = false;
            keys['ArrowRight'] = false;
            joystickThumb.style.transform = `translate(0px, 0px)`;
        }

        // События для джойстика
        joystickBase.addEventListener('touchstart', (e) => {
            e.preventDefault();
            joystickActive = true;
            const rect = joystickBase.getBoundingClientRect();
            joystickCenter = { x: rect.left + rect.width / 2, y: rect.top + rect.height / 2 };
            updateJoystick(e);
        });

        joystickBase.addEventListener('touchmove', (e) => {
            if (joystickActive) updateJoystick(e);
        });

        document.addEventListener('touchend', () => {
            joystickActive = false;
            resetJoystick();
        });

        // События для кнопок (мобильные)
        btnJump.addEventListener('touchstart', (e) => {
            e.preventDefault();
            if (player.onGround) { player.velY = -14; player.onGround = false; }
        });

        btnSign.addEventListener('touchstart', (e) => {
            e.preventDefault();
            if (player.canCast) {
                player.canCast = false;
                const sign = { x: player.x + (player.facing === 1 ? player.w : -30), y: player.y + 20, w: 30, h: 15, speed: 12 * player.facing };
                // Проверка попадания
                enemies.forEach(enemy => {
                    if (sign.x < enemy.x + enemy.w && sign.x + sign.w > enemy.x &&
                        sign.y < enemy.y + enemy.h && sign.y + sign.h > enemy.y) {
                        enemy.hp -= 1;
                        if (enemy.hp <= 0) enemy.dead = true;
                    }
                });
                setTimeout(() => player.canCast = true, 800);
            }
        });

        // --- Генерация уровня ---
        function generateLevel() {
            for(let i = 0; i < world.width; i += 60) {
                grounds.push({x: i, y: canvas.height - 50, w: 60, h: 30});
            }
            // Ямы
            for(let i = 250; i < world.width; i += 500) {
                traps.push({x: i, y: canvas.height - 70, w: 40, h: 40});
                grounds = grounds.filter(g => g.x < i || g.x > i + 40);
            }
            // Броня
            for(let i = 350; i < world.width; i += 600) {
                armorPieces.push({x: i, y: canvas.height - 90, w: 40, h: 40});
            }
            // Враги
            for(let i = 550; i < world.width; i += 350) {
                enemies.push({x: i, y: canvas.height - 92, w: 32, h: 40, speed: 1.2, dir: -1, hp: 2});
            }
        }

        // --- Игровая логика ---
        function update() {
            // Клавиатура (для ПК)
            if (keys['ArrowLeft']) { player.x -= player.speed; player.facing = -1; }
            if (keys['ArrowRight']) { player.x += player.speed; player.facing = 1; }
            if (keys['ArrowUp'] && player.onGround) { player.velY = -14; player.onGround = false; }
            if (keys[' '] && player.canCast) {
                player.canCast = false;
                const sign = { x: player.x + (player.facing === 1 ? player.w : -30), y: player.y + 20, w: 30, h: 15, speed: 12 * player.facing };
                enemies.forEach(enemy => {
                    if (sign.x < enemy.x + enemy.w && sign.x + sign.w > enemy.x &&
                        sign.y < enemy.y + enemy.h && sign.y + sign.h > enemy.y) {
                        enemy.hp -= 1;
                        if (enemy.hp <= 0) enemy.dead = true;
                    }
                });
                setTimeout(() => player.canCast = true, 800);
            }

            // Гравитация
            player.velY += 0.8;
            player.y += player.velY;

            // Коллизия с землей
            player.onGround = false;
            grounds.forEach(g => {
                if (player.x < g.x + g.w && player.x + player.w > g.x &&
                    player.y < g.y + g.h && player.y + player.h > g.y) {
                    player.velY = 0;
                    player.y = g.y - player.h;
                    player.onGround = true;
                }
            });

            // Скроллинг
            let offsetX = -player.x + canvas.width / 3;
            offsetX = Math.min(0, Math.max(-(world.width - canvas.width), offsetX));

            // Подбор брони
            armorPieces.forEach((armor, index) => {
                if (player.x < armor.x + armor.w && player.x + player.w > armor.x &&
                    player.y < armor.y + armor.h && player.y + player.h > armor.y) {
                    player.hasArmor = true;
                    armorPieces.splice(index, 1); // Удаляем броню с карты
                }
            });

            // Враги
            enemies.forEach(enemy => {
                if (enemy.dead) return;
                enemy.x += enemy.speed * enemy.dir;
                
                // ИИ (разворот у краев)
                let willFall = true;
                grounds.forEach(g => {
                    if (enemy.x + enemy.w > g.x && enemy.x < g.x + g.w && enemy.y + enemy.h === g.y) {
                        willFall = false;
                    }
                });
                if (willFall) enemy.dir *= -1;

                // Урон игроку
                if (player.x < enemy.x + enemy.w && player.x + player.w > enemy.x &&
                    player.y < enemy.y + enemy.h && player.y + player.h > enemy.y) {
                    if (!player.hasArmor) {
                        player.hp -= 1;
                    }
                }
            });
            enemies = enemies.filter(e => !e.dead);

            // Смерть
            if (player.y > canvas.height || player.hp <= 0) {
                alert('Геральт погиб. Перезагрузите страницу.');
                document.removeEventListener('keydown', keyDown);
                document.removeEventListener('keyup', keyUp);
                cancelAnimationFrame(gameLoop);
            }

            // UI
            hpText.innerText = player.hp;
            armorStatus.innerText = player.hasArmor ? 'Есть' : 'Нет';
        }

        // --- Отрисовка ---
        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            let offsetX = -player.x + canvas.width / 3;
            offsetX = Math.min(0, Math.max(-(world.width - canvas.width), offsetX));

            armorPieces.forEach(a => {
                ctx.fillStyle = sprites.armor.color;
                ctx.fillRect(a.x + offsetX, a.y, a.w, a.h);
            });
            traps.forEach(t => {
                ctx.fillStyle = sprites.trap.color;
                ctx.fillRect(t.x + offsetX, t.y, t.w, t.h);
            });
            ctx.fillStyle = sprites.ground.color;
            grounds.forEach(g => {
                ctx.fillRect(g.x + offsetX, g.y, g.w, g.h);
            });
            enemies.forEach(e => {
                ctx.fillStyle = sprites.enemy.color;
                ctx.fillRect(e.x + offsetX, e.y, e.w, e.h);
            });
            ctx.fillStyle = sprites.geralt.color;
            ctx.fillRect(player.x + offsetX, player.y, player.w, player.h);
        }

        // --- Игровой цикл ---
        function gameLoop() {
            update();
            draw();
            requestAnimationFrame(gameLoop);
        }

        // --- Управление с клавиатуры ---
        function keyDown(e) { keys[e.key] = true; }
        function keyUp(e) { keys[e.key] = false; }
        document.addEventListener('keydown', keyDown);
        document.addEventListener('keyup', keyUp);

        generateLevel();
        gameLoop();
    </script>
</body>
</html>
