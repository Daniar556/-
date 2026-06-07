<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Змейка | Классическая игра</title>
    <style>
        * {
            user-select: none;
            -webkit-tap-highlight-color: transparent;
        }

        body {
            background: linear-gradient(145deg, #1a472a 0%, #0e2a1a 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Segoe UI', 'Courier New', monospace;
            margin: 0;
            padding: 20px;
        }

        .game-container {
            background: #0a1f12;
            border-radius: 48px;
            padding: 20px;
            box-shadow: 0 25px 35px rgba(0,0,0,0.5), inset 0 1px 0 rgba(255,255,255,0.1);
        }

        canvas {
            display: block;
            margin: 0 auto;
            border-radius: 24px;
            box-shadow: 0 0 0 4px #2b5e3b, 0 10px 25px rgba(0,0,0,0.3);
            background-color: #1e3a2f;
            cursor: pointer;
        }

        .info-panel {
            display: flex;
            justify-content: space-between;
            align-items: baseline;
            margin-top: 18px;
            margin-bottom: 8px;
            padding: 0 12px;
        }

        .score-box {
            background: #000000aa;
            backdrop-filter: blur(4px);
            padding: 8px 22px;
            border-radius: 60px;
            color: #c0ffb0;
            font-weight: bold;
            font-size: 1.9rem;
            font-family: monospace;
            letter-spacing: 2px;
            text-shadow: 0 0 5px #6fbf4c;
            box-shadow: inset 0 1px 3px #00000055, 0 4px 8px rgba(0,0,0,0.2);
        }

        .restart-btn {
            background: #fdbe5c;
            border: none;
            font-size: 1.2rem;
            font-weight: bold;
            font-family: inherit;
            padding: 8px 24px;
            border-radius: 60px;
            cursor: pointer;
            color: #2d2b1f;
            transition: 0.1s linear;
            box-shadow: 0 5px 0 #8b5a2b;
            transform: translateY(-2px);
        }

        .restart-btn:active {
            transform: translateY(3px);
            box-shadow: 0 2px 0 #8b5a2b;
        }

        .status {
            background: #0b1e13e6;
            padding: 6px 18px;
            border-radius: 40px;
            font-weight: bold;
            font-size: 1rem;
            color: #ffefb9;
            backdrop-filter: blur(2px);
        }

        .controls-tip {
            text-align: center;
            margin-top: 20px;
            font-size: 0.8rem;
            color: #b9e6b3;
            background: #00000066;
            width: fit-content;
            margin-left: auto;
            margin-right: auto;
            padding: 5px 15px;
            border-radius: 60px;
            font-weight: 500;
        }

        @media (max-width: 550px) {
            .score-box { font-size: 1.5rem; padding: 4px 18px; }
            .restart-btn { padding: 6px 18px; font-size: 1rem; }
            .game-container { padding: 12px; }
        }
    </style>
</head>
<body>
<div>
    <div class="game-container">
        <canvas id="snakeCanvas" width="500" height="500"></canvas>
        <div class="info-panel">
            <div class="status">🍎 ЗМЕЙКА</div>
            <div class="score-box">Очки: <span id="scoreValue">0</span></div>
            <button class="restart-btn" id="restartButton">⟳ Новая</button>
        </div>
        <div class="controls-tip">
            ◀ ▲ ▼ ▶ &nbsp;&nbsp;|&nbsp;&nbsp; клавиши WASD &nbsp;&nbsp;|&nbsp;&nbsp; на телефоне: свайп
        </div>
    </div>
</div>

<script>
    (function(){
        // ---------- НАСТРОЙКИ ----------
        const canvas = document.getElementById('snakeCanvas');
        const ctx = canvas.getContext('2d');
        const scoreSpan = document.getElementById('scoreValue');

        // Размеры поля (20x20 клеток, размер клетки 25px)
        const GRID_SIZE = 20;      // 20x20
        const CELL_SIZE = canvas.width / GRID_SIZE; // 25px
        
        // Переменные игры
        let snake = [];     // массив координат {x, y}
        let food = { x: 12, y: 12 };
        let direction = 'RIGHT';   // текущее направление
        let nextDirection = 'RIGHT';
        let score = 0;
        let gameLoop = null;
        let gameRunning = true;
        let gameSpeed = 120;        // миллисекунд на ход (120 = быстро, 150 = норма)
        
        // ---------- ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ ----------
        function randomInt(min, max) {
            return Math.floor(Math.random() * (max - min + 1)) + min;
        }
        
        // Генерация еды в свободной клетке (не на теле змеи)
        function generateRandomFood() {
            // максимум попыток (если змея почти заполнила всё поле)
            const maxAttempts = 5000;
            for (let attempt = 0; attempt < maxAttempts; attempt++) {
                const randX = randomInt(0, GRID_SIZE - 1);
                const randY = randomInt(0, GRID_SIZE - 1);
                if (!snake.some(segment => segment.x === randX && segment.y === randY)) {
                    food = { x: randX, y: randY };
                    return true;
                }
            }
            // если почти нет свободных клеток — ищем вручную
            for (let i = 0; i < GRID_SIZE; i++) {
                for (let j = 0; j < GRID_SIZE; j++) {
                    if (!snake.some(seg => seg.x === i && seg.y === j)) {
                        food = { x: i, y: j };
                        return true;
                    }
                }
            }
            // поле полностью заполнено -> победа!
            return false; // нет свободного места
        }
        
        // Инициализация / сброс игры
        function initGame() {
            // Начальная змейка: 3 клетки в центре
            const center = Math.floor(GRID_SIZE / 2);
            snake = [
                { x: center, y: center },        // голова
                { x: center-1, y: center },
                { x: center-2, y: center }
            ];
            direction = 'RIGHT';
            nextDirection = 'RIGHT';
            score = 0;
            gameRunning = true;
            updateScoreDisplay();
            
            // Генерируем еду, не попадающую на змейку
            if (!generateRandomFood()) {
                // на случай заполнения с начала (невозможно) просто ставим в угол
                food = { x: 5, y: 5 };
                if (snake.some(s => s.x === food.x && s.y === food.y)) food = { x: 15, y: 15 };
            }
            
            // Очищаем старый интервал, если был
            if (gameLoop) clearInterval(gameLoop);
            // Запускаем новый цикл
            gameLoop = setInterval(gameTick, gameSpeed);
            
            drawGame(); // отрисовка сразу
        }
        
        // Обновление счёта на UI
        function updateScoreDisplay() {
            scoreSpan.textContent = score;
        }
        
        // Проверка столкновения головы с телом или стенами
        function checkCollision(headX, headY, snakeArr) {
            // Стены
            if (headX < 0 || headX >= GRID_SIZE || headY < 0 || headY >= GRID_SIZE) {
                return true;
            }
            // Столкновение с телом (пропускаем сравнение с будущей головой? стандартный подход)
            for (let i = 0; i < snakeArr.length; i++) {
                if (snakeArr[i].x === headX && snakeArr[i].y === headY) {
                    return true;
                }
            }
            return false;
        }
        
        // Основной игровой тик (шаг змейки)
        function gameTick() {
            if (!gameRunning) return;
            
            // Применяем накопленное направление (нельзя разворачиваться на 180)
            const opposite = {
                'UP': 'DOWN', 'DOWN': 'UP', 'LEFT': 'RIGHT', 'RIGHT': 'LEFT'
            };
            if (nextDirection && opposite[nextDirection] !== direction) {
                direction = nextDirection;
            }
            
            // Вычисляем новую голову
            let newHead = { ...snake[0] };
            switch(direction) {
                case 'RIGHT': newHead.x += 1; break;
                case 'LEFT':  newHead.x -= 1; break;
                case 'UP':    newHead.y -= 1; break;
                case 'DOWN':  newHead.y += 1; break;
                default: break;
            }
            
            // Проверяем, съест ли еду
            const willEat = (newHead.x === food.x && newHead.y === food.y);
            
            // Движение: создаём новую змейку
            let newSnake = [newHead, ...snake];
            if (!willEat) {
                newSnake.pop();   // убираем хвост, если не съели
            }
            
            // Проверяем коллизию (стены или тело)
            if (checkCollision(newHead.x, newHead.y, willEat ? newSnake.slice(1) : newSnake.slice(1))) {
                gameOver();
                return;
            }
            
            // Применяем новую змейку
            snake = newSnake;
            
            // Если съели еду
            if (willEat) {
                score++;
                updateScoreDisplay();
                const success = generateRandomFood();  // создать новую еду
                if (!success) {
                    // Если нет свободного места - ПОБЕДА!
                    gameWin();
                    return;
                }
            }
            
            // Перерисовываем игровое поле
            drawGame();
        }
        
        // Окончание игры (проигрыш)
        function gameOver() {
            if (!gameRunning) return;
            gameRunning = false;
            if (gameLoop) clearInterval(gameLoop);
            gameLoop = null;
            drawGame(true);  // true = показываем сообщение о проигрыше
        }
        
        // Победа (заполнили поле)
        function gameWin() {
            if (!gameRunning) return;
            gameRunning = false;
            if (gameLoop) clearInterval(gameLoop);
            gameLoop = null;
            drawVictory();
        }
        
        // Отрисовка с сообщением о поражении
        function drawGame(isGameOverFlag = false) {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Рисуем сетку
            ctx.strokeStyle = '#2f6b47';
            ctx.lineWidth = 0.6;
            for (let i = 0; i <= GRID_SIZE; i++) {
                ctx.beginPath();
                ctx.moveTo(i * CELL_SIZE, 0);
                ctx.lineTo(i * CELL_SIZE, canvas.height);
                ctx.stroke();
                ctx.moveTo(0, i * CELL_SIZE);
                ctx.lineTo(canvas.width, i * CELL_SIZE);
                ctx.stroke();
            }
            
            // Рисуем еду (стильное яблоко)
            ctx.shadowBlur = 0;
            ctx.fillStyle = '#ff5252';
            ctx.beginPath();
            ctx.arc(food.x * CELL_SIZE + CELL_SIZE/2, food.y * CELL_SIZE + CELL_SIZE/2, CELL_SIZE * 0.38, 0, 2*Math.PI);
            ctx.fill();
            ctx.fillStyle = '#ff8a8a';
            ctx.beginPath();
            ctx.arc(food.x * CELL_SIZE + CELL_SIZE*0.65, food.y * CELL_SIZE + CELL_SIZE*0.35, CELL_SIZE*0.1, 0, 2*Math.PI);
            ctx.fill();
            ctx.fillStyle = '#6b3e1a';
            ctx.fillRect(food.x * CELL_SIZE + CELL_SIZE*0.45, food.y * CELL_SIZE + CELL_SIZE*0.2, CELL_SIZE*0.12, CELL_SIZE*0.18);
            
            // Рисуем змейку
            for (let i = 0; i < snake.length; i++) {
                const seg = snake[i];
                const grad = ctx.createLinearGradient(seg.x * CELL_SIZE, seg.y * CELL_SIZE, 
                    (seg.x+1) * CELL_SIZE, (seg.y+1) * CELL_SIZE);
                if (i === 0) {
                    grad.addColorStop(0, '#78c46e');
                    grad.addColorStop(1, '#4f9e3c');
                } else {
                    grad.addColorStop(0, '#57b847');
                    grad.addColorStop(1, '#36872a');
                }
                ctx.fillStyle = grad;
                ctx.fillRect(seg.x * CELL_SIZE + 1, seg.y * CELL_SIZE + 1, CELL_SIZE - 2, CELL_SIZE - 2);
                // Глазки головы
                if (i === 0) {
                    ctx.fillStyle = 'white';
                    const eyeOff = CELL_SIZE * 0.2;
                    const eyeSize = CELL_SIZE * 0.2;
                    if (direction === 'RIGHT') {
                        ctx.fillRect(seg.x * CELL_SIZE + CELL_SIZE - eyeOff*2, seg.y * CELL_SIZE + eyeOff, eyeSize, eyeSize);
                        ctx.fillRect(seg.x * CELL_SIZE + CELL_SIZE - eyeOff*2, seg.y * CELL_SIZE + CELL_SIZE - eyeOff - eyeSize, eyeSize, eyeSize);
                    } else if (direction === 'LEFT') {
                        ctx.fillRect(seg.x * CELL_SIZE + eyeOff, seg.y * CELL_SIZE + eyeOff, eyeSize, eyeSize);
                        ctx.fillRect(seg.x * CELL_SIZE + eyeOff, seg.y * CELL_SIZE + CELL_SIZE - eyeOff - eyeSize, eyeSize, eyeSize);
                    } else if (direction === 'UP') {
                        ctx.fillRect(seg.x * CELL_SIZE + eyeOff, seg.y * CELL_SIZE + eyeOff, eyeSize, eyeSize);
                        ctx.fillRect(seg.x * CELL_SIZE + CELL_SIZE - eyeOff - eyeSize, seg.y * CELL_SIZE + eyeOff, eyeSize, eyeSize);
                    } else if (direction === 'DOWN') {
                        ctx.fillRect(seg.x * CELL_SIZE + eyeOff, seg.y * CELL_SIZE + CELL_SIZE - eyeOff*1.8, eyeSize, eyeSize);
                        ctx.fillRect(seg.x * CELL_SIZE + CELL_SIZE - eyeOff - eyeSize, seg.y * CELL_SIZE + CELL_SIZE - eyeOff*1.8, eyeSize, eyeSize);
                    } else {
                        ctx.fillRect(seg.x * CELL_SIZE + CELL_SIZE*0.7, seg.y * CELL_SIZE + CELL_SIZE*0.2, 4, 4);
                        ctx.fillRect(seg.x * CELL_SIZE + CELL_SIZE*0.7, seg.y * CELL_SIZE + CELL_SIZE*0.7, 4, 4);
                    }
                    ctx.fillStyle = '#1f2a0e';
                    // зрачки
                }
            }
            
            // Если игра окончена (проигрыш)
            if (isGameOverFlag && !gameRunning) {
                ctx.font = `bold ${Math.floor(CELL_SIZE * 1.6)}px "Courier New", monospace`;
                ctx.shadowBlur = 0;
                ctx.fillStyle = '#ffd966cc';
                ctx.shadowColor = 'black';
                ctx.fillText("💀 GAME OVER 💀", canvas.width/5.5, canvas.height/2);
                ctx.font = `${CELL_SIZE}px monospace`;
                ctx.fillStyle = '#ffbb88';
                ctx.fillText("нажми НОВАЯ", canvas.width/3.2, canvas.height/1.6);
            }
        }
        
        function drawVictory() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            drawGame(false); // рисуем фон и финальную позицию
            ctx.font = `bold ${Math.floor(CELL_SIZE * 1.3)}px monospace`;
            ctx.fillStyle = '#FDE16D';
            ctx.shadowBlur = 0;
            ctx.fillText("✨ ПОБЕДА! ✨", canvas.width/3.6, canvas.height/2 - 20);
            ctx.font = `${CELL_SIZE-4}px monospace`;
            ctx.fillStyle = '#bfe6a3';
            ctx.fillText("Идеально! 🐍", canvas.width/2.7, canvas.height/1.5);
        }
        
        // ---------- УПРАВЛЕНИЕ ----------
        function changeDirection(newDir) {
            if (!gameRunning) return;
            const opposite = {
                'UP': 'DOWN', 'DOWN': 'UP', 'LEFT': 'RIGHT', 'RIGHT': 'LEFT'
            };
            // Запрещаем разворот на 180 градусов
            if (opposite[newDir] !== direction) {
                nextDirection = newDir;
            }
        }
        
        // Обработка клавиатуры
        function handleKey(e) {
            const key = e.key;
            e.preventDefault();
            if (key === 'ArrowUp' || key === 'w' || key === 'W') changeDirection('UP');
            else if (key === 'ArrowDown' || key === 's' || key === 'S') changeDirection('DOWN');
            else if (key === 'ArrowLeft' || key === 'a' || key === 'A') changeDirection('LEFT');
            else if (key === 'ArrowRight' || key === 'd' || key === 'D') changeDirection('RIGHT');
            else if (key === ' ' && !gameRunning) {
                // Пробел рестарт (удобно)
                document.getElementById('restartButton').click();
            }
        }
        
        // Свайпы для мобильных устройств
        let touchStart = null;
        function handleTouchStart(e) {
            e.preventDefault();
            const rect = canvas.getBoundingClientRect();
            const touch = e.touches[0];
            touchStart = { x: touch.clientX - rect.left, y: touch.clientY - rect.top };
        }
        
        function handleTouchEnd(e) {
            if (!touchStart) return;
            e.preventDefault();
            const rect = canvas.getBoundingClientRect();
            const endX = e.changedTouches[0].clientX - rect.left;
            const endY = e.changedTouches[0].clientY - rect.top;
            const dx = endX - touchStart.x;
            const dy = endY - touchStart.y;
            if (Math.abs(dx) < 15 && Math.abs(dy) < 15) return;
            
            if (Math.abs(dx) > Math.abs(dy)) {
                if (dx > 0) changeDirection('RIGHT');
                else changeDirection('LEFT');
            } else {
                if (dy > 0) changeDirection('DOWN');
                else changeDirection('UP');
            }
            touchStart = null;
        }
        
        // Перезапуск игры
        function restartGame() {
            if (gameLoop) clearInterval(gameLoop);
            initGame();
            drawGame();
        }
        
        // ---------- ПОДКЛЮЧЕНИЕ СОБЫТИЙ И СТАРТ ----------
        window.addEventListener('load', () => {
            initGame();
            // Управление
            window.addEventListener('keydown', handleKey);
            canvas.addEventListener('touchstart', handleTouchStart, {passive: false});
            canvas.addEventListener('touchend', handleTouchEnd);
            canvas.addEventListener('touchcancel', () => { touchStart = null; });
            document.getElementById('restartButton').addEventListener('click', () => restartGame());
            // Защита от прокрутки стрелками страницы
            window.addEventListener('keydown', function(e) {
                if (e.key === 'ArrowUp' || e.key === 'ArrowDown' || e.key === 'ArrowLeft' || e.key === 'ArrowRight' ||
                    e.key === ' ' || e.key === 'w' || e.key === 'W' || e.key === 's' || e.key === 'S' || e.key === 'a' || e.key === 'A' || e.key === 'd' || e.key === 'D') {
                    if (e.target === document.body || e.target === document.documentElement) {
                        e.preventDefault();
                    }
                }
            });
        });
    })();
</script>
</body>
</html>
