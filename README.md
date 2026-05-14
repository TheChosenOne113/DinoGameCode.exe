<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Chrome T-Rex Dino Game</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            background-color: #f7f7f7;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            font-family: 'Courier New', Courier, monospace;
            user-select: none;
            overflow: hidden;
        }
        #game-container {
            position: relative;
            width: 100%;
            max-width: 800px;
        }
        canvas {
            background-color: #ffffff;
            border-bottom: 2px solid #535353;
            display: block;
            width: 100%;
            height: auto;
        }
        #instructions {
            text-align: center;
            margin-top: 15px;
            color: #535353;
            font-size: 14px;
            font-weight: bold;
        }
    </style>
</head>
<body>

    <div id="game-container">
        <canvas id="gameCanvas" width="800" height="250"></canvas>
        <div id="instructions">Press SPACE / UP ARROW or TAP to Jump</div>
    </div>

    <script>
        const canvas = document.getElementById("gameCanvas");
        const ctx = canvas.getContext("2d");

        // Game State Variables
        let isGameStarted = false;
        let isGameOver = false;
        let score = 0;
        let highScore = 0;
        let gameSpeed = 6;
        const baseSpeed = 6;
        const maxSpeed = 15;
        let gameFrame = 0;

        // Physics Configurations
        const gravity = 0.6;
        const groundY = canvas.height - 30;

        // Dinosaur Character Layout & Physics
        const dino = {
            x: 50,
            y: groundY - 40,
            width: 40,
            height: 40,
            velocityY: 0,
            jumpForce: 12,
            isJumping: false,
            // Simple procedural draw loop for the T-Rex sprite silhouette
            draw() {
                ctx.fillStyle = "#535353";
                // Body & Head
                ctx.fillRect(this.x, this.y, this.width, this.height - 10); 
                ctx.fillRect(this.x + 20, this.y - 10, 25, 20); 
                // Eye
                ctx.fillStyle = "#ffffff";
                ctx.fillRect(this.x + 25, this.y - 5, 4, 4);
                // Snout & Jaw
                ctx.fillStyle = "#535353";
                ctx.fillRect(this.x + 35, this.y + 5, 10, 5);
                // Legs alternate positions dynamically over frames if running
                ctx.fillStyle = "#333333";
                if (!this.isJumping && Math.floor(gameFrame / 8) % 2 === 0) {
                    ctx.fillRect(this.x + 5, this.y + this.height - 10, 6, 10); // Leg 1
                    ctx.fillRect(this.x + 25, this.y + this.height - 10, 6, 5); // Leg 2 (bent)
                } else {
                    ctx.fillRect(this.x + 5, this.y + this.height - 10, 6, 5);
                    ctx.fillRect(this.x + 25, this.y + this.height - 10, 6, 10);
                }
            },
            jump() {
                if (!this.isJumping) {
                    this.velocityY = -this.jumpForce;
                    this.isJumping = true;
                }
            },
            update() {
                // Apply Gravity Physics
                this.velocityY += gravity;
                this.y += this.velocityY;

                // Stop falling when touching the structural ground point
                if (this.y >= groundY - this.height) {
                    this.y = groundY - this.height;
                    this.velocityY = 0;
                    this.isJumping = false;
                }
            }
        };

        // Obstacles Array management
        let obstacles = [];

        class Cactus {
            constructor() {
                this.x = canvas.width;
                // Randomly generate sizing varieties
                this.width = 15 + Math.random() * 20; 
                this.height = 30 + Math.random() * 30;
                this.y = groundY - this.height;
            }
            draw() {
                ctx.fillStyle = "#535353";
                // Main stem
                ctx.fillRect(this.x, this.y, this.width, this.height);
                // Left branch arm
                ctx.fillRect(this.x - 6, this.y + this.height * 0.3, 6, 6);
                ctx.fillRect(this.x - 6, this.y + this.height * 0.3, 4, this.height * 0.4);
                // Right branch arm
                ctx.fillRect(this.x + this.width, this.y + this.height * 0.4, 6, 6);
                ctx.fillRect(this.x + this.width + 2, this.y + this.height * 0.4, 4, this.height * 0.3);
            }
            update() {
                this.x -= gameSpeed;
            }
        }

        // Spawn logic manager
        function handleObstacles() {
            // Generate obstacle roughly every 90-150 game loops randomly
            if (gameFrame % Math.floor(90 + Math.random() * 60) === 0) {
                obstacles.push(new Cactus());
            }

            for (let i = obstacles.length - 1; i >= 0; i--) {
                obstacles[i].update();
                obstacles[i].draw();

                // Axis-Aligned Bounding Box (AABB) Collision Detection check
                if (
                    dino.x < obstacles[i].x + obstacles[i].width &&
                    dino.x + dino.width > obstacles[i].x &&
                    dino.y < obstacles[i].y + obstacles[i].height &&
                    dino.y + dino.height > obstacles[i].y
                ) {
                    isGameOver = true;
                }

                // Cleanup out-of-screen objects to preserve computing memory
                if (obstacles[i].x + obstacles[i].width < 0) {
                    obstacles.splice(i, 1);
                }
            }
        }

        // Draw structural scrolling terrain ground line
        function drawGround() {
            ctx.strokeStyle = "#535353";
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.moveTo(0, groundY);
            ctx.lineTo(canvas.width, groundY);
            ctx.stroke();

            // Decorative sand details passing through dynamically
            ctx.fillStyle = "#535353";
            for (let i = 0; i < canvas.width; i += 100) {
                let offset = (gameFrame * gameSpeed) % 100;
                ctx.fillRect(i - offset + 20, groundY + 8, 5, 2);
                ctx.fillRect(i - offset + 60, groundY + 14, 12, 2);
            }
        }

        // Draw text data layers on screen layout
        function drawScore() {
            ctx.fillStyle = "#535353";
            ctx.font = "16px 'Courier New'";
            // Standard dynamic formatting paddings matching original game
            let scoreStr = String(Math.floor(score)).padStart(5, '0');
            let highScoreStr = String(Math.floor(highScore)).padStart(5, '0');
            ctx.fillText(`HI ${highScoreStr}  ${scoreStr}`, canvas.width - 160, 30);
        }

        // Core central render engine sequence loop
        function animate() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            drawGround();
            dino.update();
            dino.draw();

            if (isGameStarted && !isGameOver) {
                gameFrame++;
                score += 0.1; // Accumulate structural score metric points
                
                // Accelerate overall velocity scaling difficulty incrementally over time
                if (gameSpeed < maxSpeed) {
                    gameSpeed = baseSpeed + (score / 200);
                }

                handleObstacles();
            } else if (!isGameStarted) {
                // Splash welcome system
                ctx.fillStyle = "#535353";
                ctx.font = "20px 'Courier New'";
                ctx.fillText("CHROME T-REX RUNNER", canvas.width / 2 - 110, canvas.height / 2 - 20);
                ctx.font = "14px 'Courier New'";
                ctx.fillText("Press Space / Tap Screen to Start", canvas.width / 2 - 130, canvas.height / 2 + 10);
            } else if (isGameOver) {
                // Draw obstacles static position upon loss status trigger
                obstacles.forEach(obs => obs.draw());
                
                if (score > highScore) {
                    highScore = score;
                }

                // Render game over interface messaging blocks
                ctx.fillStyle = "#535353";
                ctx.font = "24px 'Courier New'";
                ctx.fillText("G A M E  O V E R", canvas.width / 2 - 110, canvas.height / 2 - 20);
                ctx.font = "14px 'Courier New'";
                ctx.fillText("Press Space / Tap to Restart", canvas.width / 2 - 110, canvas.height / 2 + 10);
            }

            drawScore();
            requestAnimationFrame(animate);
        }

        // Global Reset Logic Function Instantiation
        function resetGame() {
            isGameOver = false;
            isGameStarted = true;
            score = 0;
            gameSpeed = baseSpeed;
            gameFrame = 0;
            obstacles = [];
            dino.y = groundY - dino.height;
            dino.velocityY = 0;
            dino.isJumping = false;
        }

        // Input Event Listeners
        function triggerAction() {
            if (!isGameStarted) {
                resetGame();
            } else if (isGameOver) {
                resetGame();
            } else {
                dino.jump();
            }
        }

        // Keydown processing
        window.addEventListener("keydown", (e) => {
            if (e.code === "Space" || e.code === "ArrowUp") {
                e.preventDefault(); // Blocks native page scrolling actions
                triggerAction();
            }
        });

        // Touch device mobile canvas tap gesture configuration
        canvas.addEventListener("touchstart", (e) => {
            e.preventDefault(); 
            triggerAction();
        });

        // Initialize execution setup loop sequence
        animate();
    </script>
</body>
</html>

