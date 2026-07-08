<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>별사탕 수집가</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #2c3e50;
            font-family: 'Malgun Gothic', sans-serif;
            user-select: none;
        }
        canvas {
            display: block;
        }
        #ui {
            position: absolute;
            top: 20px;
            left: 20px;
            color: white;
            font-size: 24px;
            font-weight: bold;
            pointer-events: none;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
            line-height: 1.5;
        }
        #gameOver {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: white;
            font-size: 48px;
            font-weight: bold;
            text-align: center;
            display: none;
            text-shadow: 2px 2px 8px rgba(0,0,0,0.8);
        }
        .restart {
            font-size: 24px;
            margin-top: 20px;
            cursor: pointer;
            padding: 10px 20px;
            background-color: #e74c3c;
            border-radius: 10px;
            display: inline-block;
            transition: background-color 0.2s;
        }
        .restart:hover {
            background-color: #c0392b;
        }
    </style>
</head>
<body>

    <div id="ui">
        시간: <span id="time">60</span>초<br>
        점수: <span id="score">0</span><br>
        생명: <span id="lives">❤️❤️❤️</span><br>
        <span id="speedStatus" style="font-size: 18px; color: #f1c40f;"></span>
    </div>
    
    <div id="gameOver">
        <span id="endMessage">게임 오버</span><br>
        <span style="font-size: 32px;">최종 점수: <span id="finalScore">0</span></span><br>
        <div class="restart" onclick="location.reload()">다시 시작</div>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreElement = document.getElementById('score');
        const livesElement = document.getElementById('lives');
        const timeElement = document.getElementById('time');
        const speedStatusElement = document.getElementById('speedStatus');
        const gameOverScreen = document.getElementById('gameOver');
        const endMessageElement = document.getElementById('endMessage');
        const finalScoreElement = document.getElementById('finalScore');

        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        const colors = [
            { name: 'Red', code: '#FF6B6B', comp: 'Cyan' },
            { name: 'Cyan', code: '#4ECDC4', comp: 'Red' },
            { name: 'Yellow', code: '#FFE66D', comp: 'Blue' },
            { name: 'Blue', code: '#5A189A', comp: 'Yellow' },
            { name: 'Green', code: '#95D5B2', comp: 'Magenta' },
            { name: 'Magenta', code: '#F72585', comp: 'Green' }
        ];

        let score = 0;
        let lives = 3;
        let timeLeft = 60;
        let isGameOver = false;
        let lastEatenColor = null;
        let colorHistory = []; 

        const keys = {
            ArrowUp: false,
            ArrowDown: false,
            ArrowLeft: false,
            ArrowRight: false
        };

        window.addEventListener('keydown', (e) => {
            if (keys.hasOwnProperty(e.key)) keys[e.key] = true;
        });

        window.addEventListener('keyup', (e) => {
            if (keys.hasOwnProperty(e.key)) keys[e.key] = false;
        });

        const baseSpeed = 8;
        const player = {
            x: 150,
            y: canvas.height / 2,
            radius: 25,
            speed: baseSpeed
        };

        const candies = [];
        let frameCount = 0;

        const timerInterval = setInterval(() => {
            if (!isGameOver) {
                timeLeft--;
                timeElement.innerText = timeLeft;
                
                if (timeLeft <= 0) {
                    endGame("시간 초과!");
                }
            }
        }, 1000);

        function drawStar(cx, cy, spikes, outerRadius, innerRadius, color) {
            let rot = Math.PI / 2 * 3;
            let x = cx;
            let y = cy;
            let step = Math.PI / spikes;

            ctx.beginPath();
            ctx.moveTo(cx, cy - outerRadius);
            for (let i = 0; i < spikes; i++) {
                x = cx + Math.cos(rot) * outerRadius;
                y = cy + Math.sin(rot) * outerRadius;
                ctx.lineTo(x, y);
                rot += step;

                x = cx + Math.cos(rot) * innerRadius;
                y = cy + Math.sin(rot) * innerRadius;
                ctx.lineTo(x, y);
                rot += step;
            }
            ctx.lineTo(cx, cy - outerRadius);
            ctx.closePath();
            ctx.fillStyle = color;
            ctx.fill();
        }

        function drawPlayer() {
            ctx.beginPath();
            ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
            ctx.fillStyle = '#f1c40f';
            ctx.fill();
            ctx.closePath();

            ctx.beginPath();
            ctx.arc(player.x + 8, player.y - 5, 3, 0, Math.PI * 2);
            ctx.arc(player.x - 2, player.y - 5, 3, 0, Math.PI * 2);
            ctx.fillStyle = '#000';
            ctx.fill();
            ctx.closePath();

            ctx.beginPath();
            ctx.arc(player.x + 12, player.y + 2, 4, 0, Math.PI * 2);
            ctx.arc(player.x - 6, player.y + 2, 4, 0, Math.PI * 2);
            ctx.fillStyle = 'rgba(255, 105, 180, 0.6)';
            ctx.fill();
            ctx.closePath();

            ctx.beginPath();
            ctx.arc(player.x + 3, player.y + 2, 3, 0, Math.PI);
            ctx.strokeStyle = '#000';
            ctx.lineWidth = 1.5;
            ctx.stroke();
            ctx.closePath();
        }

        function updatePlayer() {
            if (keys.ArrowUp && player.y - player.radius > 0) player.y -= player.speed;
            if (keys.ArrowDown && player.y + player.radius < canvas.height) player.y += player.speed;
            if (keys.ArrowLeft && player.x - player.radius > 0) player.x -= player.speed;
            if (keys.ArrowRight && player.x + player.radius < canvas.width) player.x += player.speed;
        }

        function handleCandies() {
            if (frameCount % 20 === 0) {
                const isWhite = Math.random() < 0.08; 
                
                if (isWhite) {
                    candies.push({
                        x: canvas.width + 30,
                        y: Math.random() * (canvas.height - 60) + 30,
                        radius: 15,
                        speed: Math.random() * 5 + 7,
                        type: 'white',
                        isBlinking: false
                    });
                } else {
                    const randomColor = colors[Math.floor(Math.random() * colors.length)];
                    const isBlinking = Math.random() < 0.3; 
                    
                    candies.push({
                        x: canvas.width + 30,
                        y: Math.random() * (canvas.height - 60) + 30,
                        radius: 15,
                        speed: Math.random() * 5 + 7,
                        type: 'color',
                        colorInfo: randomColor,
                        isBlinking: isBlinking
                    });
                }
            }

            for (let i = candies.length - 1; i >= 0; i--) {
                const candy = candies[i];
                candy.x -= candy.speed;

                let drawColor;
                if (candy.type === 'white') {
                    drawColor = 'rgba(255, 255, 255, 0.8)';
                } else {
                    drawColor = candy.colorInfo.code;
                    if (candy.isBlinking && frameCount % 60 < 30) {
                        drawColor = 'rgba(150, 150, 150, 0.3)';
                    }
                }

                drawStar(candy.x, candy.y, 5, candy.radius, candy.radius / 2, drawColor);

                const dx = player.x - candy.x;
                const dy = player.y - candy.y;
                const distance = Math.sqrt(dx * dx + dy * dy);

                if (distance < player.radius + candy.radius - 5) {
                    if (candy.type === 'white') {
                        lastEatenColor = null;
                        timeLeft += 3;
                        timeElement.innerText = timeLeft;
                        player.speed = baseSpeed;
                        colorHistory = [];
                        speedStatusElement.innerText = "기억의 소거";
                        setTimeout(() => { if(speedStatusElement.innerText === "기억의 소거") speedStatusElement.innerText = ""; }, 1500);
                        score += 300;
                    } else {
                        if (lastEatenColor && lastEatenColor.comp === candy.colorInfo.name) {
                            lives--;
                            updateLivesDisplay();
                            player.speed = baseSpeed; 
                            colorHistory = [];
                            speedStatusElement.innerText = "";
                            if (lives <= 0) endGame("생명 소진");
                        } else {
                            score += 100;
                            colorHistory.push(candy.colorInfo.name);
                            
                            if (colorHistory.length === 3) {
                                const [c1, c2, c3] = colorHistory;
                                if (c1 === c2 && c2 === c3) {
                                    player.speed = 4;
                                    speedStatusElement.innerText = "과식: 속도 감소";
                                    colorHistory = [];
                                } else if (c1 !== c2 && c2 !== c3 && c1 !== c3) {
                                    player.speed = 13;
                                    speedStatusElement.innerText = "다채로움: 속도 증가";
                                    colorHistory = [];
                                } else {
                                    colorHistory.shift();
                                }
                            }
                        }
                        lastEatenColor = candy.colorInfo;
                    }
                    
                    scoreElement.innerText = score;
                    candies.splice(i, 1);
                } else if (candy.x < -30) {
                    candies.splice(i, 1);
                }
            }
        }

        function updateLivesDisplay() {
            let hearts = '';
            for (let i = 0; i < lives; i++) hearts += '❤️';
            livesElement.innerText = hearts;
        }

        function endGame(message) {
            isGameOver = true;
            clearInterval(timerInterval);
            endMessageElement.innerText = message;
            finalScoreElement.innerText = score;
            gameOverScreen.style.display = 'block';
        }

        function gameLoop() {
            if (isGameOver) return;

            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            updatePlayer();
            drawPlayer();
            handleCandies();

            frameCount++;
            requestAnimationFrame(gameLoop);
        }

        window.addEventListener('resize', () => {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        });

        gameLoop();
    </script>
</body>
</html>
