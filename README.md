<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mini Physics Soccer</title>
    <style>
        body {
            margin: 0;
            background: #222;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: sans-serif;
            color: white;
            overflow: hidden;
        }
        #gameContainer {
            text-align: center;
        }
        canvas {
            background: #27ae60;
            border: 4px solid #fff;
            box-shadow: 0 10px 20px rgba(0,0,0,0.3);
        }
        .scoreboard {
            font-size: 24px;
            margin-bottom: 10px;
            font-weight: bold;
        }
    </style>
</head>
<body>

<div id="gameContainer">
    <div class="scoreboard">🔴 Player: <span id="pScore">0</span> | 🔵 AI: <span id="aScore">0</span></div>
    <canvas id="gameCanvas" width="800" height="400"></canvas>
    <p>Use <b>Arrow Keys</b> or <b>WASD</b> to move and slide into the ball!</p>
</div>

<script>
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");

// Score tracking
let playerScore = 0;
let aiScore = 0;

// Physics configuration
const FRICTION = 0.98;
const BALL_FRICTION = 0.99;

// Game Entities
const ball = {
    x: 400, y: 200, vx: 0, vy: 0, radius: 12, color: '#fff'
};

const player = {
    x: 200, y: 200, vx: 0, vy: 0, radius: 20, color: '#e74c3c', speed: 0.6
};

const ai = {
    x: 600, y: 200, vx: 0, vy: 0, radius: 20, color: '#2980b9', speed: 0.45
};

// Goals settings
const goalWidth = 10;
const goalHeight = 120;
const leftGoal = { x: 0, y: 140, w: goalWidth, h: goalHeight };
const rightGoal = { x: 790, y: 140, w: goalWidth, h: goalHeight };

// Keyboard Input State
const keys = {};
window.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

function resetLayout() {
    ball.x = canvas.width / 2;
    ball.y = canvas.height / 2;
    ball.vx = 0;
    ball.vy = 0;

    player.x = 200;
    player.y = 200;
    player.vx = 0;
    player.vy = 0;

    ai.x = 600;
    ai.y = 200;
    ai.vx = 0;
    ai.vy = 0;
}

function checkCollision(obj1, obj2, isBall = false) {
    let dx = obj2.x - obj1.x;
    let dy = obj2.y - obj1.y;
    let distance = Math.sqrt(dx * dx + dy * dy);
    let minDistance = obj1.radius + obj2.radius;

    if (distance < minDistance) {
        // Resolve overlap
        let overlap = minDistance - distance;
        let nx = dx / distance;
        let ny = dy / distance;

        if (!isBall) {
            // Push apart static entities roughly
            obj1.x -= nx * overlap / 2;
            obj1.y -= ny * overlap / 2;
            obj2.x += nx * overlap / 2;
            obj2.y += ny * overlap / 2;
        } else {
            // Push ball out of player
            obj2.x += nx * overlap;
            obj2.y += ny * overlap;
            
            // Elastic impulse kick factor
            let kx = obj1.vx - obj2.vx;
            let ky = obj1.vy - obj2.vy;
            let p = 2 * (kx * nx + ky * ny) / 2;
            
            obj2.vx += nx * p * 1.5 + obj1.vx * 0.5;
            obj2.vy += ny * p * 1.5 + obj1.vy * 0.5;
        }
    }
}

function update() {
    // Player Movement (WASD or Arrows)
    if (keys['arrowup'] || keys['w']) player.vy -= player.speed;
    if (keys['arrowdown'] || keys['s']) player.vy += player.speed;
    if (keys['arrowleft'] || keys['a']) player.vx -= player.speed;
    if (keys['arrowright'] || keys['d']) player.vx += player.speed;

    // Basic AI Tracking
    let aiDx = ball.x - ai.x;
    let aiDy = ball.y - ai.y;
    let aiDist = Math.sqrt(aiDx * aiDx + aiDy * aiDy);
    if (aiDist > 10) {
        ai.vx += (aiDx / aiDist) * ai.speed;
        ai.vy += (aiDy / aiDist) * ai.speed;
    }

    // Apply Friction (Creates the slippery/liquid gliding effect)
    player.vx *= FRICTION;
    player.vy *= FRICTION;
    ai.vx *= FRICTION;
    ai.vy *= FRICTION;
    ball.vx *= BALL_FRICTION;
    ball.vy *= BALL_FRICTION;

    // Update positions
    player.x += player.vx;
    player.y += player.vy;
    ai.x += ai.vx;
    ai.y += ai.vy;
    ball.x += ball.vx;
    ball.y += ball.vy;

    // Entity vs Entity Collisions
    checkCollision(player, ball, true);
    checkCollision(ai, ball, true);
    checkCollision(player, ai, false);

    // Boundary Collisions (Walls)
    [player, ai, ball].forEach(obj => {
        if (obj.x - obj.radius < 0) { obj.x = obj.radius; obj.vx *= -0.8; }
        if (obj.x + obj.radius > canvas.width) { obj.x = canvas.width - obj.radius; obj.vx *= -0.8; }
        if (obj.y - obj.radius < 0) { obj.y = obj.radius; obj.vy *= -0.8; }
        if (obj.y + obj.radius > canvas.height) { obj.y = canvas.height - obj.radius; obj.vy *= -0.8; }
    });

    // Check Goals
    if (ball.x - ball.radius <= leftGoal.w && ball.y >= leftGoal.y && ball.y <= leftGoal.y + leftGoal.h) {
        aiScore++;
        document.getElementById('aScore').innerText = aiScore;
        resetLayout();
    }
    if (ball.x + ball.radius >= rightGoal.x && ball.y >= rightGoal.y && ball.y <= rightGoal.y + rightGoal.h) {
        playerScore++;
        document.getElementById('pScore').innerText = playerScore;
        resetLayout();
    }
}

function draw() {
    // Clear pitch
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Draw field lines
    ctx.strokeStyle = "rgba(255,255,255,0.3)";
    ctx.lineWidth = 4;
    ctx.beginPath();
    ctx.moveTo(canvas.width / 2, 0);
    ctx.lineTo(canvas.width / 2, canvas.height);
    ctx.stroke();
    
    ctx.beginPath();
    ctx.arc(canvas.width / 2, canvas.height / 2, 60, 0, Math.PI * 2);
    ctx.stroke();

    // Draw goals
    ctx.fillStyle = "#fff";
    ctx.fillRect(leftGoal.x, leftGoal.y, leftGoal.w, leftGoal.h);
    ctx.fillRect(rightGoal.x, rightGoal.y, rightGoal.w, rightGoal.h);

    // Draw Player
    ctx.fillStyle = player.color;
    ctx.beginPath();
    ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
    ctx.fill();

    // Draw AI
    ctx.fillStyle = ai.color;
    ctx.beginPath();
    ctx.arc(ai.x, ai.y, ai.radius, 0, Math.PI * 2);
    ctx.fill();

    // Draw Ball
    ctx.fillStyle = ball.color;
    ctx.beginPath();
    ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.strokeStyle = "#000";
    ctx.lineWidth = 1;
    ctx.stroke();
}

function gameLoop() {
    update();
    draw();
    requestAnimationFrame(gameLoop);
}

gameLoop();
</script>
</body>
</html>
