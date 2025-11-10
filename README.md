# Shooter
Game
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Asteroid Shooter</title>
    <!-- Use Inter font and dark mode theme -->
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap" rel="stylesheet">
    <style>
        :root {
            --ship-color: #38bdf8; /* Sky 400 */
            --bullet-color: #fcd34d; /* Amber 300 */
            --asteroid-color: #9ca3af; /* Gray 400 */
            --background-color: #030712; /* Gray 900 */
            --text-color: #f9fafb; /* Gray 50 */
        }

        body {
            font-family: 'Inter', sans-serif;
            background-color: var(--background-color);
            color: var(--text-color);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            padding: 10px;
            overflow: hidden; /* Prevent body scroll on mobile */
        }

        #game-container {
            width: 100%;
            max-width: 600px;
            display: flex;
            flex-direction: column;
            align-items: center;
            border: 2px solid var(--text-color);
            border-radius: 12px;
            box-shadow: 0 0 20px rgba(56, 189, 248, 0.5);
            background: linear-gradient(180deg, #1f2937 0%, #030712 100%);
            margin-bottom: 16px;
        }

        #gameCanvas {
            width: 100%;
            display: block;
            background-color: #030712;
            border-bottom-left-radius: 10px;
            border-bottom-right-radius: 10px;
        }

        .hud {
            width: 100%;
            display: flex;
            justify-content: space-between;
            padding: 8px 16px;
            background-color: #1f2937;
            border-top-left-radius: 10px;
            border-top-right-radius: 10px;
        }

        .controls {
            display: flex;
            justify-content: center;
            width: 100%;
            padding: 10px 0;
        }

        .fire-button {
            padding: 15px 30px;
            font-size: 1.25rem;
            font-weight: bold;
            color: var(--text-color);
            background: #ef4444; /* Red 500 */
            border: none;
            border-radius: 9999px; /* Fully rounded */
            box-shadow: 0 4px 15px rgba(239, 68, 68, 0.4);
            cursor: pointer;
            transition: all 0.15s ease-in-out;
            touch-action: manipulation; /* Improves touch responsiveness */
        }

        .fire-button:active {
            background: #dc2626; /* Red 600 */
            box-shadow: 0 2px 5px rgba(239, 68, 68, 0.6);
            transform: scale(0.98);
        }

        /* Game Over Modal Styling */
        #messageBox {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background-color: rgba(0, 0, 0, 0.9);
            padding: 30px;
            border: 3px solid #fcd34d;
            border-radius: 12px;
            text-align: center;
            z-index: 100;
            box-shadow: 0 0 40px rgba(252, 211, 77, 0.7);
            width: 80%;
            max-width: 400px;
            display: none;
        }

        #messageBox h2 {
            font-size: 2rem;
            color: #fcd34d;
            margin-bottom: 10px;
        }

        #messageBox p {
            font-size: 1.2rem;
            margin-bottom: 20px;
        }

        #messageBox button {
            background-color: #10b981; /* Emerald 500 */
            color: white;
            padding: 10px 20px;
            border: none;
            border-radius: 6px;
            font-weight: bold;
            cursor: pointer;
            transition: background-color 0.2s;
        }

        #messageBox button:hover {
            background-color: #059669; /* Emerald 600 */
        }

    </style>
</head>
<body>

    <h1 class="text-3xl font-bold mb-4 text-sky-400">ASTEROID FRONTIER</h1>

    <div id="game-container">
        <div class="hud">
            <span id="scoreDisplay" class="font-bold text-lg">SCORE: 0</span>
            <span id="livesDisplay" class="font-bold text-lg text-red-400">LIVES: 3</span>
        </div>
        <canvas id="gameCanvas"></canvas>
    </div>

    <div class="controls">
        <button id="fireButton" class="fire-button">FIRE</button>
    </div>

    <!-- Game Over Message Box -->
    <div id="messageBox">
        <h2 id="messageTitle">Game Over</h2>
        <p id="messageText">Your final score is: 0</p>
        <button id="restartButton">Restart Game</button>
    </div>

    <script>
        // Global variables provided by the environment are not needed for this client-side game.

        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreDisplay = document.getElementById('scoreDisplay');
        const livesDisplay = document.getElementById('livesDisplay');
        const fireButton = document.getElementById('fireButton');
        const messageBox = document.getElementById('messageBox');
        const messageTitle = document.getElementById('messageTitle');
        const messageText = document.getElementById('messageText');
        const restartButton = document.getElementById('restartButton');

        // Game Constants
        const SHIP_SIZE = 20;
        const BULLET_SPEED = 7;
        const BULLET_LIFESPAN = 120; // frames
        const ASTEROID_SPEED = 1.5;
        const ASTEROID_SPAWN_RATE = 150; // frames per spawn attempt
        const ASTEROID_MIN_SIZE = 15;
        const ASTEROID_MAX_SIZE = 40;
        const ASTEROID_SPAWN_MARGIN = 50; // distance from center to spawn

        // Game State
        let player = {};
        let bullets = [];
        let asteroids = [];
        let score = 0;
        let lives = 3;
        let gameRunning = false;
        let spawnCounter = 0;
        let mousePos = { x: 0, y: 0 };
        let isMouseDown = false;
        let lastFireTime = 0;
        const fireCooldown = 200; // milliseconds

        // Utility: Distance calculation
        function dist(x1, y1, x2, y2) {
            return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
        }

        // --- Game Objects ---

        class Ship {
            constructor() {
                this.x = canvas.width / 2;
                this.y = canvas.height / 2;
                this.size = SHIP_SIZE;
                this.angle = 0;
                this.speed = 3;
                this.thrustX = 0;
                this.thrustY = 0;
                this.isAlive = true;
            }

            draw() {
                if (!this.isAlive) return;

                ctx.save();
                ctx.translate(this.x, this.y);
                ctx.rotate(this.angle);

                // Draw Triangle Ship (inline SVG-like)
                ctx.beginPath();
                ctx.strokeStyle = 'var(--ship-color)';
                ctx.lineWidth = 2;
                ctx.moveTo(0, -this.size * 1.5); // Tip
                ctx.lineTo(-this.size, this.size); // Bottom Left
                ctx.lineTo(this.size, this.size); // Bottom Right
                ctx.closePath();
                ctx.stroke();

                ctx.restore();
            }

            update() {
                if (!this.isAlive) return;

                // Calculate angle towards the target (mouse/touch)
                this.angle = Math.atan2(mousePos.y - this.y, mousePos.x - this.x) + Math.PI / 2;

                if (isMouseDown) {
                    // Move towards the target position
                    const dx = mousePos.x - this.x;
                    const dy = mousePos.y - this.y;
                    const distance = Math.sqrt(dx * dx + dy * dy);

                    if (distance > 5) { // Only move if not too close
                        this.thrustX = (dx / distance) * this.speed;
                        this.thrustY = (dy / distance) * this.speed;

                        this.x += this.thrustX;
                        this.y += this.thrustY;
                    } else {
                        this.thrustX = 0;
                        this.thrustY = 0;
                    }
                } else {
                    this.thrustX = 0;
                    this.thrustY = 0;
                }

                // Keep player within bounds
                this.x = Math.max(0, Math.min(canvas.width, this.x));
                this.y = Math.max(0, Math.min(canvas.height, this.y));
            }
        }

        class Bullet {
            constructor(x, y, angle) {
                this.x = x + Math.cos(angle - Math.PI / 2) * SHIP_SIZE * 1.5;
                this.y = y + Math.sin(angle - Math.PI / 2) * SHIP_SIZE * 1.5;
                this.vx = Math.cos(angle - Math.PI / 2) * BULLET_SPEED;
                this.vy = Math.sin(angle - Math.PI / 2) * BULLET_SPEED;
                this.size = 3;
                this.lifespan = BULLET_LIFESPAN;
            }

            draw() {
                ctx.fillStyle = 'var(--bullet-color)';
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
                ctx.fill();
            }

            update() {
                this.x += this.vx;
                this.y += this.vy;
                this.lifespan--;
            }

            isExpired() {
                return this.lifespan <= 0 ||
                       this.x < 0 || this.x > canvas.width ||
                       this.y < 0 || this.y > canvas.height;
            }
        }

        class Asteroid {
            constructor(x, y, size, vx, vy) {
                this.x = x;
                this.y = y;
                this.size = size;
                this.vx = vx;
                this.vy = vy;
                this.rotation = Math.random() * Math.PI * 2;
                this.rotationSpeed = (Math.random() - 0.5) * 0.02;
                this.segments = Math.floor(Math.random() * 4) + 6; // 6 to 9 sides
                this.offset = [];
                for (let i = 0; i < this.segments; i++) {
                    this.offset.push((Math.random() * 0.4 + 0.8) * this.size); // Randomize shape
                }
            }

            draw() {
                ctx.save();
                ctx.translate(this.x, this.y);
                ctx.rotate(this.rotation);

                ctx.beginPath();
                ctx.strokeStyle = 'var(--asteroid-color)';
                ctx.lineWidth = 2;

                for (let i = 0; i < this.segments; i++) {
                    const angle = (i / this.segments) * Math.PI * 2;
                    const radius = this.offset[i];
                    const x = Math.cos(angle) * radius;
                    const y = Math.sin(angle) * radius;
                    if (i === 0) {
                        ctx.moveTo(x, y);
                    } else {
                        ctx.lineTo(x, y);
                    }
                }
                ctx.closePath();
                ctx.stroke();
                ctx.restore();
            }

            update() {
                this.x += this.vx;
                this.y += this.vy;
                this.rotation += this.rotationSpeed;
            }

            isOffScreen() {
                const margin = this.size + 10;
                return this.x < -margin || this.x > canvas.width + margin ||
                       this.y < -margin || this.y > canvas.height + margin;
            }
        }

        // --- Core Game Logic ---

        function resizeCanvas() {
            const container = document.getElementById('game-container');
            const aspectRatio = 16 / 9; // Target aspect ratio
            const containerWidth = container.clientWidth;
            
            // Set canvas size based on container width and aspect ratio
            canvas.width = containerWidth;
            canvas.height = containerWidth / aspectRatio;
            
            // Re-center player on resize
            if (player.x) {
                player.x = canvas.width / 2;
                player.y = canvas.height / 2;
            }
        }

        function initGame() {
            gameRunning = true;
            lives = 3;
            score = 0;
            bullets = [];
            asteroids = [];
            spawnCounter = 0;
            player = new Ship();
            updateHUD();
            messageBox.style.display = 'none';

            // Initial resize and start mouse position
            resizeCanvas();
            mousePos.x = canvas.width / 2;
            mousePos.y = canvas.height / 2;
        }

        function spawnAsteroid() {
            const size = Math.random() * (ASTEROID_MAX_SIZE - ASTEROID_MIN_SIZE) + ASTEROID_MIN_SIZE;

            // Randomly pick a side (0: top, 1: right, 2: bottom, 3: left)
            const side = Math.floor(Math.random() * 4);
            let x, y, angle;

            if (side === 0) { // Top
                x = Math.random() * canvas.width;
                y = -ASTEROID_SPAWN_MARGIN;
            } else if (side === 1) { // Right
                x = canvas.width + ASTEROID_SPAWN_MARGIN;
                y = Math.random() * canvas.height;
            } else if (side === 2) { // Bottom
                x = Math.random() * canvas.width;
                y = canvas.height + ASTEROID_SPAWN_MARGIN;
            } else { // Left
                x = -ASTEROID_SPAWN_MARGIN;
                y = Math.random() * canvas.height;
            }

            // Aim for a random point near the center, weighted towards the ship's center
            const targetX = canvas.width / 2 + (Math.random() - 0.5) * canvas.width * 0.3;
            const targetY = canvas.height / 2 + (Math.random() - 0.5) * canvas.height * 0.3;

            angle = Math.atan2(targetY - y, targetX - x);

            const speedMultiplier = 0.5 + Math.random(); // 0.5x to 1.5x base speed
            const vx = Math.cos(angle) * ASTEROID_SPEED * speedMultiplier;
            const vy = Math.sin(angle) * ASTEROID_SPEED * speedMultiplier;

            asteroids.push(new Asteroid(x, y, size, vx, vy));
        }

        function checkCollisions() {
            // 1. Bullet vs Asteroid
            for (let i = bullets.length - 1; i >= 0; i--) {
                const bullet = bullets[i];
                for (let j = asteroids.length - 1; j >= 0; j--) {
                    const asteroid = asteroids[j];
                    // Simple circle collision using max offset as radius approximation
                    if (dist(bullet.x, bullet.y, asteroid.x, asteroid.y) < asteroid.size) {
                        // Hit!
                        score += 10;
                        updateHUD();
                        bullets.splice(i, 1);
                        asteroids.splice(j, 1);
                        break; // Go to next bullet
                    }
                }
            }

            // 2. Player vs Asteroid
            if (player.isAlive) {
                for (let i = asteroids.length - 1; i >= 0; i--) {
                    const asteroid = asteroids[i];
                    // Collision check (distance between ship center and asteroid center)
                    if (dist(player.x, player.y, asteroid.x, asteroid.y) < player.size + asteroid.size * 0.8) {
                        // Player hit!
                        lives--;
                        updateHUD();
                        asteroids.splice(i, 1); // Destroy asteroid

                        if (lives <= 0) {
                            gameOver();
                        } else {
                            // Briefly make the ship invincible and reset position
                            player.isAlive = false;
                            setTimeout(() => {
                                player.x = canvas.width / 2;
                                player.y = canvas.height / 2;
                                player.isAlive = true;
                            }, 1500); // 1.5 seconds invincibility
                        }
                        break;
                    }
                }
            }
        }

        function update() {
            if (!gameRunning) return;

            // Update player
            player.update();

            // Update bullets
            bullets.forEach(bullet => bullet.update());
            bullets = bullets.filter(bullet => !bullet.isExpired());

            // Update asteroids
            asteroids.forEach(asteroid => asteroid.update());
            asteroids = asteroids.filter(asteroid => !asteroid.isOffScreen());

            // Spawn logic
            spawnCounter++;
            if (spawnCounter > ASTEROID_SPAWN_RATE) {
                spawnAsteroid();
                spawnCounter = 0;
            }

            checkCollisions();
        }

        function draw() {
            // Clear canvas
            ctx.fillStyle = 'var(--background-color)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Draw game objects
            asteroids.forEach(asteroid => asteroid.draw());
            bullets.forEach(bullet => bullet.draw());

            // Only draw player if alive or flashing
            if (player.isAlive || (lives > 0 && Math.floor(Date.now() / 200) % 2 === 0)) {
                 player.draw();
            }
        }

        function gameLoop() {
            if (gameRunning) {
                update();
                draw();
                requestAnimationFrame(gameLoop);
            }
        }

        function updateHUD() {
            scoreDisplay.textContent = `SCORE: ${score}`;
            livesDisplay.textContent = `LIVES: ${lives}`;
        }

        function gameOver() {
            gameRunning = false;
            messageTitle.textContent = "GAME OVER!";
            messageText.textContent = `You protected the frontier for a little while. Your final score is: ${score}.`;
            messageBox.style.display = 'block';
        }

        function fireBullet() {
            if (!gameRunning || !player.isAlive) return;

            const now = Date.now();
            if (now - lastFireTime > fireCooldown) {
                bullets.push(new Bullet(player.x, player.y, player.angle));
                lastFireTime = now;
            }
        }

        // --- Event Listeners (Input) ---

        function getMousePos(e) {
            const rect = canvas.getBoundingClientRect();
            let clientX, clientY;

            if (e.touches && e.touches.length > 0) {
                clientX = e.touches[0].clientX;
                clientY = e.touches[0].clientY;
            } else {
                clientX = e.clientX;
                clientY = e.clientY;
            }

            mousePos.x = clientX - rect.left;
            mousePos.y = clientY - rect.top;
        }

        function handleStart(e) {
            if (e.target === canvas) {
                e.preventDefault(); // Prevent scrolling on touch
                isMouseDown = true;
                getMousePos(e);
            }
        }

        function handleEnd(e) {
            if (e.target === canvas) {
                isMouseDown = false;
            }
        }

        function handleMove(e) {
            if (e.target === canvas) {
                e.preventDefault();
                getMousePos(e);
            }
        }

        // Mouse/Touch events for Movement
        canvas.addEventListener('mousedown', handleStart);
        canvas.addEventListener('touchstart', handleStart);
        canvas.addEventListener('mouseup', handleEnd);
        canvas.addEventListener('touchend', handleEnd);
        canvas.addEventListener('mousemove', handleMove);
        canvas.addEventListener('touchmove', handleMove);

        // Fire butt
