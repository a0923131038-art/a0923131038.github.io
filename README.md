# a0923131038.github.io[tetris.html](https://github.com/user-attachments/files/32411985/tetris.html)
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>手機俄羅斯方塊</title>
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
            margin: 0;
            padding: 0;
        }
        body {
            background-color: #1a1a1a;
            color: #fff;
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: space-between;
            height: 100vh;
            overflow: hidden;
        }
        /* 上方資訊欄 */
        header {
            width: 100%;
            padding: 10px;
            display: flex;
            justify-content: space-around;
            align-items: center;
            background-color: #222;
        }
        .score-board {
            font-size: 1.2rem;
            font-weight: bold;
        }
        #start-btn {
            padding: 8px 16px;
            font-size: 1rem;
            background-color: #28a745;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        /* 遊戲畫布容器 */
        .game-container {
            flex: 1;
            display: flex;
            justify-content: center;
            align-items: center;
            width: 100%;
            max-height: 55vh;
            padding: 5px;
        }
        canvas {
            background-color: #000;
            border: 4px solid #333;
            max-height: 100%;
            max-width: 100%;
            aspect-ratio: 10 / 20;
        }
        /* 下方觸控按鈕區 */
        .controls {
            width: 100%;
            max-width: 400px;
            padding: 10px;
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            grid-gap: 10px;
            background-color: #222;
        }
        .btn {
            background-color: #444;
            color: white;
            border: none;
            border-radius: 10px;
            font-size: 1.5rem;
            font-weight: bold;
            padding: 15px 0;
            text-align: center;
            box-shadow: 0 4px #222;
            touch-action: manipulation;
        }
        .btn:active {
            background-color: #666;
            transform: translateY(2px);
            box-shadow: 0 2px #222;
        }
        .btn-rotate { background-color: #007bff; grid-column: span 3; padding: 12px 0; }
        .btn-drop { background-color: #dc3545; }
    </style>
</head>
<body>

    <header>
        <div class="score-board">分數: <span id="score">0</span></div>
        <button id="start-btn">開始遊戲</button>
    </header>

    <div class="game-container">
        <canvas id="tetris" width="240" height="480"></canvas>
    </div>

    <!-- 手機虛擬按鈕 -->
    <div class="controls">
        <button class="btn btn-rotate" id="ctrl-rotate">旋轉 (Rotate)</button>
        <button class="btn" id="ctrl-left">◀</button>
        <button class="btn btn-drop" id="ctrl-drop">▼▼</button>
        <button class="btn" id="ctrl-right">▶</button>
    </div>

    <script>
        const canvas = document.getElementById('tetris');
        const context = canvas.getContext('2d');
        const scoreElement = document.getElementById('score');
        const startBtn = document.getElementById('start-btn');

        // 方塊放大倍率 (240x480 畫布 / 10x20 格子 = 每格 24 像素)
        context.scale(24, 24);

        // 7種經典方塊形狀
        const SHAPES = {
            'I': [[0,1,0,0],[0,1,0,0],[0,1,0,0],[0,1,0,0]],
            'L': [[0,2,0],[0,2,0],[0,2,2]],
            'J': [[0,3,0],[0,3,0],[3,3,0]],
            'O': [[4,4],[4,4]],
            'Z': [[5,5,0],[0,5,5],[0,0,0]],
            'S': [[0,6,6],[6,6,0],[0,0,0]],
            'T': [[0,7,0],[7,7,7],[0,0,0]]
        };

        const COLORS = [
            null, '#00f0f0', '#f0a000', '#0000f0', 
            '#f0f000', '#f00000', '#00f000', '#a000f0'
        ];

        // 建立遊戲地圖 (10欄 x 20列)
        function createMatrix(w, h) {
            const matrix = [];
            while (h--) matrix.push(new Array(w).fill(0));
            return matrix;
        }

        let arena = createMatrix(10, 20);
        let player = { pos: {x: 0, y: 0}, matrix: null, score: 0 };
        let gameOver = true;

        // 隨機取得新方塊
        function playerReset() {
            const pieces = 'ILJOZST';
            const randPiece = pieces[pieces.length * Math.random() | 0];
            player.matrix = SHAPES[randPiece];
            player.pos.y = 0;
            player.pos.x = (arena[0].length / 2 | 0) - (player.matrix[0].length / 2 | 0);

            if (collide(arena, player)) {
                alert('遊戲結束！最終得分: ' + player.score);
                arena.forEach(row => row.fill(0));
                player.score = 0;
                gameOver = true;
                startBtn.innerText = '重新開始';
                updateScore();
            }
        }

        // 碰撞偵測
        function collide(arena, player) {
            const [m, o] = [player.matrix, player.pos];
            for (let y = 0; y < m.length; ++y) {
                for (let x = 0; x < m[y].length; ++x) {
                    if (m[y][x] !== 0 && (arena[y + o.y] && arena[y + o.y][x + o.x]) !== 0) {
                        return true;
                    }
                }
            }
            return false;
        }

        // 將落下的方塊固定到地圖上
        function merge(arena, player) {
            player.matrix.forEach((row, y) => {
                row.forEach((value, x) => {
                    if (value !== 0) {
                        arena[y + player.pos.y][x + player.pos.x] = value;
                    }
                });
            });
        }

        // 消除整行並計分
        function arenaSweep() {
            let rowCount = 1;
            outer: for (let y = arena.length - 1; y > 0; --y) {
                for (let x = 0; x < arena[y].length; ++x) {
                    if (arena[y][x] === 0) continue outer;
                }
                const row = arena.splice(y, 1)[0].fill(0);
                arena.unshift(row);
                ++y;
                player.score += rowCount * 10;
                rowCount *= 2;
            }
        }

        // 繪製單一矩陣
        function drawMatrix(matrix, offset) {
            matrix.forEach((row, y) => {
                row.forEach((value, x) => {
                    if (value !== 0) {
                        context.fillStyle = COLORS[value];
                        context.fillRect(x + offset.x, y + offset.y, 1, 1);
                        // 加點邊框質感
                        context.strokeStyle = '#1a1a1a';
                        context.lineWidth = 0.05;
                        context.strokeRect(x + offset.x, y + offset.y, 1, 1);
                    }
                });
            });
        }

        // 畫面渲染
        function draw() {
            context.fillStyle = '#000';
            context.fillRect(0, 0, canvas.width, canvas.height);
            drawMatrix(arena, {x: 0, y: 0});
            if (player.matrix) drawMatrix(player.matrix, player.pos);
        }

        // 方塊下落邏輯
        function playerDrop() {
            if (gameOver) return;
            player.pos.y++;
            if (collide(arena, player)) {
                player.pos.y--;
                merge(arena, player);
                playerReset();
                arenaSweep();
                updateScore();
            }
            dropCounter = 0;
        }

        // 方塊左右移動
        function playerMove(dir) {
            if (gameOver) return;
            player.pos.x += dir;
            if (collide(arena, player)) player.pos.x -= dir;
        }

        // 方塊旋轉
        function playerRotate() {
            if (gameOver) return;
            const matrix = player.matrix;
            for (let y = 0; y < matrix.length; ++y) {
                for (let x = 0; x < y; ++x) {
                    [matrix[x][y], matrix[y][x]] = [matrix[y][x], matrix[x][y]];
                }
            }
            matrix.forEach(row => row.reverse());

            // 旋轉若碰撞，嘗試左右彈開修正
            const pos = player.pos.x;
            let offset = 1;
            while (collide(arena, player)) {
                player.pos.x += offset;
                offset = -(offset + (offset > 0 ? 1 : -1));
                if (offset > matrix[0].length) {
                    // 無法修正則轉回原樣
                    matrix.forEach(row => row.reverse());
                    for (let y = 0; y < matrix.length; ++y) {
                        for (let x = 0; x < y; ++x) {
                            [matrix[x][y], matrix[y][x]] = [matrix[y][x], matrix[x][y]];
                        }
                    }
                    player.pos.x = pos;
                    return;
                }
            }
        }

        // 瞬間下落 (Hard Drop)
        function playerHardDrop() {
            if (gameOver) return;
            while (!collide(arena, player)) {
                player.pos.y++;
            }
            player.pos.y--;
            merge(arena, player);
            playerReset();
            arenaSweep();
            updateScore();
        }

        function updateScore() {
            scoreElement.innerText = player.score;
        }

        // 遊戲主迴圈時間控制
        let dropCounter = 0;
        let dropInterval = 1000; // 預設 1 秒下一格
        let lastTime = 0;

        function update(time = 0) {
            if (gameOver) return;
            const deltaTime = time - lastTime;
            lastTime = time;

            dropCounter += deltaTime;
            if (dropCounter > dropInterval) {
                playerDrop();
            }
            draw();
            requestAnimationFrame(update);
        }

        // 綁定鍵盤事件 (留給電腦網頁版測試用)
        document.addEventListener('keydown', event => {
            if (event.keyCode === 37) playerMove(-1);


