<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Super Square Adventure - Sound Effects Edition</title>

  <style>
    body {
      margin: 0;
      background: #111;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      overflow: hidden;
    }

    canvas {
      border: 4px solid #000;
      background: linear-gradient(to bottom, #74b9ff, #a29bfe);
      display: block;
      box-shadow: 0 10px 30px rgba(0,0,0,0.5);
    }
  </style>
</head>
<body>

<canvas id="gameCanvas" width="800" height="500"></canvas>

<script>
  const canvas = document.getElementById("gameCanvas");
  const ctx = canvas.getContext("2d");

  // Screen/Canvas dimensions
  const WIDTH = canvas.width;
  const HEIGHT = canvas.height;

  // Level dimensions (Vast scrolling world)
  const WORLD_WIDTH = 3200;

  // Game States Framework: "START", "PLAYING", "GAME_OVER", "WON"
  let gameState = "START";

  // Game Systems State variables
  let score = 0;
  let lives = 3;

  // Ground settings
  const groundHeight = 40;
  const groundY = HEIGHT - groundHeight;

  // Side-Scrolling Camera Object
  const camera = {
    x: 0
  };

  // Goal Flag Config at x=3100
  const goalFlag = {
    x: 3100,
    y: groundY - 200, 
    poleWidth: 8,
    poleHeight: 200,
    flagWidth: 40,
    flagHeight: 30
  };

  // Platforms Array (World coordinates)
  const platforms = [
    { x: 200, y: 380, width: 120, height: 20 },
    { x: 400, y: 300, width: 120, height: 20 },
    { x: 600, y: 220, width: 120, height: 20 },
    { x: 900, y: 380, width: 120, height: 20 },
    { x: 1100, y: 320, width: 120, height: 20 },
    { x: 1300, y: 260, width: 120, height: 20 },
    { x: 1500, y: 320, width: 200, height: 20 },
    { x: 1800, y: 400, width: 100, height: 20 },
    { x: 2000, y: 320, width: 100, height: 20 },
    { x: 2200, y: 250, width: 100, height: 20 },
    { x: 2500, y: 350, width: 200, height: 20 },
    { x: 2800, y: 280, width: 150, height: 20 }
  ];

  // Static Visual World Assets
  const clouds = [
    { x: 150, y: 80, size: 30 },
    { x: 450, y: 50, size: 40 },
    { x: 800, y: 90, size: 25 },
    { x: 1200, y: 60, size: 35 },
    { x: 1600, y: 100, size: 45 },
    { x: 2000, y: 40, size: 30 },
    { x: 2400, y: 75, size: 35 },
    { x: 2800, y: 60, size: 40 }
  ];

  const backgroundHills = [
    { x: 100, cx: 350, r: 240 },
    { x: 650, cx: 900, r: 280 },
    { x: 1300, cx: 1600, r: 260 },
    { x: 2000, cx: 2350, r: 310 },
    { x: 2650, cx: 3000, r: 250 }
  ];

  // Player Starting Position Constants
  const PLAYER_START_X = 100;
  const PLAYER_START_Y = groundY - 40;

  // Player Object
  const player = {
    x: PLAYER_START_X,
    y: PLAYER_START_Y,
    width: 40,
    height: 40,
    color: "#ff2a2a",

    velocityY: 0,
    speed: 5,            
    jumpStrength: 13.5,  
    gravity: 0.6,

    isJumping: false
  };

  // Enemies Master Blueprint Context Array
  let enemies = [];

  // Coins Master Blueprint Context Array
  let coins = [];

  // --- WEB AUDIO API SOUND SYSTEM ---
  let audioCtx = null;

  function initAudio() {
    if (!audioCtx) {
      audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    }
    if (audioCtx.state === "suspended") {
      audioCtx.resume();
    }
  }

  function playJumpSound() {
    if (!audioCtx) return;
    const now = audioCtx.currentTime;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    
    osc.type = "triangle";
    osc.frequency.setValueAtTime(150, now);
    osc.frequency.exponentialRampToValueAtTime(400, now + 0.1);
    
    gain.gain.setValueAtTime(0.15, now);
    gain.gain.linearRampToValueAtTime(0.01, now + 0.1);
    
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    
    osc.start(now);
    osc.stop(now + 0.1);
  }

  function playCoinSound() {
    if (!audioCtx) return;
    const now = audioCtx.currentTime;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    
    osc.type = "sine";
    // Traditional arcade chime arpeggio: B5 then E6
    osc.frequency.setValueAtTime(987.77, now); 
    osc.frequency.setValueAtTime(1318.51, now + 0.07);
    
    gain.gain.setValueAtTime(0.12, now);
    gain.gain.linearRampToValueAtTime(0.01, now + 0.3);
    
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    
    osc.start(now);
    osc.stop(now + 0.3);
  }

  function playStompSound() {
    if (!audioCtx) return;
    const now = audioCtx.currentTime;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    
    osc.type = "triangle";
    osc.frequency.setValueAtTime(300, now);
    osc.frequency.linearRampToValueAtTime(80, now + 0.15);
    
    gain.gain.setValueAtTime(0.2, now);
    gain.gain.linearRampToValueAtTime(0.01, now + 0.15);
    
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    
    osc.start(now);
    osc.stop(now + 0.15);
  }

  function playDeathSound() {
    if (!audioCtx) return;
    const now = audioCtx.currentTime;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    
    osc.type = "square";
    osc.frequency.setValueAtTime(180, now);
    osc.frequency.linearRampToValueAtTime(60, now + 0.25);
    
    gain.gain.setValueAtTime(0.15, now);
    gain.gain.linearRampToValueAtTime(0.01, now + 0.25);
    
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    
    osc.start(now);
    osc.stop(now + 0.25);
  }

  function playWinSound() {
    if (!audioCtx) return;
    const now = audioCtx.currentTime;
    
    // Notes mapping: C5 (523.25Hz), E5 (659.25Hz), G5 (783.99Hz)
    const notes = [523.25, 659.25, 783.99];
    const noteDuration = 0.12;

    notes.forEach((freq, index) => {
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      const startTime = now + (index * noteDuration);
      
      osc.type = "triangle";
      osc.frequency.setValueAtTime(freq, startTime);
      
      gain.gain.setValueAtTime(0.15, startTime);
      gain.gain.linearRampToValueAtTime(0.01, startTime + noteDuration * 1.5);
      
      osc.connect(gain);
      gain.connect(audioCtx.destination);
      
      osc.start(startTime);
      osc.stop(startTime + noteDuration * 1.5);
    });
  }

  // Dedicated Engine System Reset Functionality
  function resetGame() {
    score = 0;
    lives = 3;
    
    // Reposition Avatar to Initial States
    player.x = PLAYER_START_X;
    player.y = PLAYER_START_Y;
    player.velocityY = 0;
    player.isJumping = false;

    // Reset Camera Position
    camera.x = 0;

    // Completely restore fresh Enemies array states
    enemies = [
      { startX: 300, x: 300, y: groundY - 32, width: 32, height: 32, speed: 1.5, direction: 1, patrolRange: 150, color: "#4a2711" },
      { startX: 550, x: 550, y: groundY - 32, width: 32, height: 32, speed: 1.5, direction: -1, patrolRange: 150, color: "#4a2711" },
      { startX: 1000, x: 1000, y: groundY - 32, width: 32, height: 32, speed: 1.5, direction: 1, patrolRange: 150, color: "#4a2711" },
      { startX: 1600, x: 1600, y: groundY - 32, width: 32, height: 32, speed: 1.5, direction: 1, patrolRange: 150, color: "#4a2711" },
      { startX: 2100, x: 2100, y: groundY - 32, width: 32, height: 32, speed: 1.5, direction: 1, patrolRange: 150, color: "#4a2711" },
      { startX: 2700, x: 2700, y: groundY - 32, width: 32, height: 32, speed: 1.5, direction: 1, patrolRange: 150, color: "#4a2711" }
    ];

    // Completely restore fresh Collectibles map placement configurations
    coins = [
      { x: 260, y: 350, size: 20 }, { x: 310, y: 350, size: 20 },
      { x: 450, y: 270, size: 20 }, { x: 500, y: 270, size: 20 },
      { x: 650, y: 190, size: 20 }, { x: 700, y: 190, size: 20 },
      { x: 150, y: 420, size: 20 }, { x: 730, y: 420, size: 20 },
      { x: 1150, y: 290, size: 20 }, { x: 1350, y: 230, size: 20 },
      { x: 1550, y: 290, size: 20 }, { x: 1850, y: 370, size: 20 },
      { x: 2050, y: 290, size: 20 }, { x: 2250, y: 220, size: 20 },
      { x: 2600, y: 320, size: 20 }, { x: 2900, y: 250, size: 20 }
    ];
  }

  // Keyboard control tracking
  const keys = {
    left: false,
    right: false
  };

  // Input Intercept Listeners
  window.addEventListener("keydown", (e) => {
    // Standard contextual authorization catch for running Web Audio nodes on viewport triggers
    initAudio();

    // Intercept SPACE on Menu Loading Frame
    if (gameState === "START") {
      if (e.code === "Space") {
        e.preventDefault();
        gameState = "PLAYING";
      }
      return; 
    }

    // Intercept SPACE on Terminal Overlays to trigger hard structural resets
    if (gameState === "GAME_OVER" || gameState === "WON") {
      if (e.code === "Space") {
        e.preventDefault();
        resetGame();
        gameState = "PLAYING";
      }
      return; 
    }

    // Standard Game Loop Controls Mapping
    if (e.code === "ArrowLeft") keys.left = true;
    if (e.code === "ArrowRight") keys.right = true;

    if (e.code === "Space") {
      e.preventDefault();
      if (!player.isJumping) {
        player.velocityY = -player.jumpStrength;
        player.isJumping = true;
        playJumpSound();
      }
    }
  });

  window.addEventListener("keyup", (e) => {
    if (e.code === "ArrowLeft") keys.left = false;
    if (e.code === "ArrowRight") keys.right = false;

    // Variable bounce vector modifiers
    if (e.code === "Space") {
      if (player.velocityY < -3) {
        player.velocityY = -3; 
      }
    }
  });

  function handlePlayerDeath() {
    playDeathSound();
    lives--;
    if (lives <= 0) {
      lives = 0;
      gameState = "GAME_OVER";
    } else {
      player.x = PLAYER_START_X;
      player.y = PLAYER_START_Y;
      player.velocityY = 0;
      player.isJumping = false;
    }
  }

  // Engine Physics and Logic Computations
  function update() {
    if (gameState !== "PLAYING") return;

    // Horizontal velocity applications
    if (keys.left) player.x -= player.speed;
    if (keys.right) player.x += player.speed;

    // Dynamic map boundaries constraint limits
    if (player.x < 0) {
      player.x = 0;
    }
    if (player.x + player.width > WORLD_WIDTH) {
      player.x = WORLD_WIDTH - player.width;
    }

    // Capture vertical positioning baseline context frame step
    const previousY = player.y;

    // Standard gravity acceleration tracking equations
    player.velocityY += player.gravity;
    player.y += player.velocityY;

    // Base horizon grounding
    if (player.y + player.height >= groundY) {
      player.y = groundY - player.height;
      player.velocityY = 0;
      player.isJumping = false;
    }

    // Horizontal structural platform evaluation loops
    for (const platform of platforms) {
      const horizontalOverlap =
        player.x + player.width > platform.x &&
        player.x < platform.x + platform.width;

      const wasAbovePlatform =
        previousY + player.height <= platform.y;

      const isFallingOntoPlatform =
        player.y + player.height >= platform.y;

      if (
        horizontalOverlap &&
        wasAbovePlatform &&
        isFallingOntoPlatform &&
        player.velocityY >= 0
      ) {
        player.y = platform.y - player.height;
        player.velocityY = 0;
        player.isJumping = false;
      }
    }

    // Goal Flag pole overlap box intersections
    const isOverlappingGoalPole = 
      player.x + player.width > goalFlag.x &&
      player.x < goalFlag.x + goalFlag.poleWidth &&
      player.y + player.height > goalFlag.y &&
      player.y < goalFlag.y + goalFlag.poleHeight;

    if (isOverlappingGoalPole) {
      gameState = "WON";
      playWinSound();
      return; 
    }

    // Enemy state updates and collision checking loops
    for (let i = enemies.length - 1; i >= 0; i--) {
      const enemy = enemies[i];

      enemy.x += enemy.speed * enemy.direction;

      const distanceFromStart = enemy.x - enemy.startX;
      if (enemy.direction === 1 && distanceFromStart >= enemy.patrolRange) {
        enemy.direction = -1; 
      } else if (enemy.direction === -1 && distanceFromStart <= -enemy.patrolRange) {
        enemy.direction = 1;  
      }

      const isOverlappingEnemy = 
        player.x + player.width > enemy.x &&
        player.x < enemy.x + enemy.width &&
        player.y + player.height > enemy.y &&
        player.y < enemy.y + enemy.height;

      if (isOverlappingEnemy) {
        const wasAboveEnemy = previousY + player.height <= enemy.y;

        if (wasAboveEnemy && player.velocityY >= 0) {
          enemies.splice(i, 1);
          score += 200;                       
          player.y = enemy.y - player.height; 
          player.velocityY = -8;              
          player.isJumping = true;
          playStompSound();
        } else {
          handlePlayerDeath();
        }
      }
    }

    // Item intersection evaluations
    for (let i = coins.length - 1; i >= 0; i--) {
      const coin = coins[i];

      const isOverlappingCoin =
        player.x + player.width > coin.x &&
        player.x < coin.x + coin.size &&
        player.y + player.height > coin.y &&
        player.y < coin.y + coin.size;

      if (isOverlappingCoin) {
        coins.splice(i, 1); 
        score += 100;       
        playCoinSound();
      }
    }

    // Align camera position to follow the center axis of player body
    camera.x = player.x - WIDTH / 2 + player.width / 2;

    // Bound the camera scope so it stays restricted within level frame
    if (camera.x < 0) {
      camera.x = 0;
    }
    if (camera.x > WORLD_WIDTH - WIDTH) {
      camera.x = WORLD_WIDTH - WIDTH;
    }
  }

  // Screen Graphics Construction Loop
  function draw() {
    ctx.clearRect(0, 0, WIDTH, HEIGHT);

    // 1. BACKGROUND LAYERS: Skies, Clouds & Parallax Hills
    
    // Clouds Layer (Scrolls with camera foreground)
    ctx.fillStyle = "rgba(255, 255, 255, 0.75)";
    for (const cloud of clouds) {
      const cx = cloud.x - camera.x;
      if (cx + cloud.size * 2 < 0 || cx - cloud.size > WIDTH) continue;
      
      ctx.beginPath();
      ctx.arc(cx, cloud.y, cloud.size, 0, Math.PI * 2);
      ctx.arc(cx + cloud.size * 0.6, cloud.y - cloud.size * 0.4, cloud.size * 0.8, 0, Math.PI * 2);
      ctx.arc(cx + cloud.size * 1.2, cloud.y, cloud.size * 0.7, 0, Math.PI * 2);
      ctx.fill();
    }

    // Parallax Dark Green Hills Layer (Scrolls at HALF speed)
    ctx.fillStyle = "#27ae60";
    const parallaxOffsetX = camera.x * 0.5;
    for (const hill of backgroundHills) {
      const hillX = hill.cx - parallaxOffsetX;
      ctx.beginPath();
      ctx.arc(hillX, groundY + 80, hill.r, 0, Math.PI * 2);
      ctx.fill();
    }

    // 2. MIDDLEGROUND LAYERS: Platforms, Hazards, Flag & Items
    
    // Draw platforms with 3D shadow baselines
    for (const platform of platforms) {
      const px = platform.x - camera.x;
      
      // Main Platform Body Face
      ctx.fillStyle = "#8a5429";
      ctx.fillRect(px, platform.y, platform.width, platform.height);
      
      // Platform 3D Dark Under-Shading Strip
      ctx.fillStyle = "#5c3414";
      ctx.fillRect(px, platform.y + platform.height - 6, platform.width, 6);
    }

    // Draw long landscape base flooring with integrated 3D grass top edge
    ctx.fillStyle = "#6d3e1a";
    ctx.fillRect(0 - camera.x, groundY, WORLD_WIDTH, groundHeight);
    ctx.fillStyle = "#2ecc71";
    ctx.fillRect(0 - camera.x, groundY, WORLD_WIDTH, 8);

    // Draw Goal Flag components
    ctx.fillStyle = "#4a2f13"; 
    ctx.fillRect(
      goalFlag.x - camera.x,
      goalFlag.y,
      goalFlag.poleWidth,
      goalFlag.poleHeight
    );
    ctx.fillStyle = "#e74c3c";
    ctx.beginPath();
    ctx.moveTo(goalFlag.x - camera.x + goalFlag.poleWidth, goalFlag.y); 
    ctx.lineTo(goalFlag.x - camera.x + goalFlag.poleWidth + goalFlag.flagWidth, goalFlag.y + goalFlag.flagHeight / 2); 
    ctx.lineTo(goalFlag.x - camera.x + goalFlag.poleWidth, goalFlag.y + goalFlag.flagHeight); 
    ctx.fill();
    ctx.closePath();

    // Draw coins
    ctx.fillStyle = "gold";
    ctx.strokeStyle = "#d4af37";
    ctx.lineWidth = 2;
    for (const coin of coins) {
      const radius = coin.size / 2;
      const centerX = (coin.x + radius) - camera.x;
      const centerY = coin.y + radius;

      ctx.beginPath();
      ctx.arc(centerX, centerY, radius, 0, Math.PI * 2);
      ctx.fill();
      ctx.stroke();
      ctx.closePath();
    }

    // Draw roaming enemies + ANGRY FACIAL GRAPHICS
    for (const enemy of enemies) {
      const ex = enemy.x - camera.x;
      
      // Base Enemy Square Block
      ctx.fillStyle = enemy.color;
      ctx.fillRect(ex, enemy.y, enemy.width, enemy.height);

      // Angry Eyes Drawing Routine
      ctx.fillStyle = "white";
      // Left White Eye Shape
      ctx.beginPath();
      ctx.moveTo(ex + 4, enemy.y + 6);
      ctx.lineTo(ex + 13, enemy.y + 11);
      ctx.lineTo(ex + 11, enemy.y + 16);
      ctx.lineTo(ex + 3, enemy.y + 12);
      ctx.fill();

      // Right White Eye Shape
      ctx.beginPath();
      ctx.moveTo(ex + enemy.width - 4, enemy.y + 6);
      ctx.lineTo(ex + enemy.width - 13, enemy.y + 11);
      ctx.lineTo(ex + enemy.width - 11, enemy.y + 16);
      ctx.lineTo(ex + enemy.width - 3, enemy.y + 12);
      ctx.fill();

      // Tilted Inner Angry Black Pupils
      ctx.fillStyle = "black";
      ctx.fillRect(ex + 7, enemy.y + 11, 3, 3);
      ctx.fillRect(ex + enemy.width - 10, enemy.y + 11, 3, 3);
    }

    // 3. FOREGROUND LAYER: Player Avatar Block + FACE GRAPHICS
    const rx = player.x - camera.x;
    ctx.fillStyle = player.color;
    ctx.fillRect(rx, player.y, player.width, player.height);

    // Player Face Overlay Detail Matrix
    ctx.fillStyle = "white";
    ctx.fillRect(rx + 8, player.y + 10, 8, 8);  
    ctx.fillRect(rx + 24, player.y + 10, 8, 8); 

    ctx.fillStyle = "black";
    ctx.fillRect(rx + 11, player.y + 13, 4, 4); 
    ctx.fillRect(rx + 25, player.y + 13, 4, 4); 

    // Open Mouth Accent Path
    ctx.fillStyle = "#4a0000";
    ctx.beginPath();
    ctx.arc(rx + 20, player.y + 26, 5, 0, Math.PI, false);
    ctx.fill();

    // 4. STATIC HEADS-UP DISPLAY OVERLAYS (HUD / Menus)
    ctx.lineWidth = 5;
    ctx.strokeStyle = "black";
    ctx.fillStyle = "white";

    if (gameState !== "START") {
      ctx.font = "bold 28px sans-serif";
      
      // Score UI (Top-Left)
      ctx.textBaseline = "top";
      ctx.textAlign = "left";
      ctx.strokeText("Score: " + score, 20, 20);
      ctx.fillText("Score: " + score, 20, 20);

      // Lives UI (Top-Right)
      ctx.textAlign = "right";
      ctx.strokeText("Lives: " + lives, WIDTH - 20, 20);
      ctx.fillText("Lives: " + lives, WIDTH - 20, 20);
    }

    // Start Screen UI layer properties
    if (gameState === "START") {
      ctx.fillStyle = "rgba(0, 0, 0, 0.45)";
      ctx.fillRect(0, 0, WIDTH, HEIGHT);

      ctx.textAlign = "center";
      ctx.textBaseline = "alphabetic"; 
      
      ctx.font = "bold 56px sans-serif";
      ctx.lineWidth = 8;
      ctx.strokeStyle = "black";
      ctx.fillStyle = "white";
      ctx.strokeText("Super Square Adventure", WIDTH / 2, HEIGHT / 2 - 80);
      ctx.fillText("Super Square Adventure", WIDTH / 2, HEIGHT / 2 - 80);

      ctx.font = "bold 20px sans-serif";
      ctx.lineWidth = 5;
      ctx.strokeText("Arrow keys to move. Space to jump. Stomp enemies, grab coins, reach the flag!", WIDTH / 2, HEIGHT / 2 + 10);
      ctx.fillStyle = "#ffeb3b"; 
      ctx.fillText("Arrow keys to move. Space to jump. Stomp enemies, grab coins, reach the flag!", WIDTH / 2, HEIGHT / 2 + 10);

      ctx.font = "bold 28px sans-serif";
      ctx.strokeStyle = "black";
      ctx.strokeText("Press SPACE to start", WIDTH / 2, HEIGHT / 2 + 90);
      ctx.fillStyle = "white";
      ctx.fillText("Press SPACE to start", WIDTH / 2, HEIGHT / 2 + 90);
    }

    // Terminal Game Over State
    if (gameState === "GAME_OVER") {
      ctx.fillStyle = "rgba(0, 0, 0, 0.55)"; 
      ctx.fillRect(0, 0, WIDTH, HEIGHT);

      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      
      ctx.font = "bold 64px sans-serif";
      ctx.lineWidth = 8;
      ctx.strokeText("GAME OVER", WIDTH / 2, HEIGHT / 2 - 30);
      ctx.fillStyle = "#ff3838"; 
      ctx.fillText("GAME OVER", WIDTH / 2, HEIGHT / 2 - 30);

      ctx.font = "bold 26px sans-serif";
      ctx.lineWidth = 5;
      ctx.strokeText("Press SPACE to play again", WIDTH / 2, HEIGHT / 2 + 40);
      ctx.fillStyle = "white";
      ctx.fillText("Press SPACE to play again", WIDTH / 2, HEIGHT / 2 + 40);
    }

    // Terminal Winning Celebration State
    if (gameState === "WON") {
      ctx.fillStyle = "rgba(0, 0, 0, 0.35)"; 
      ctx.fillRect(0, 0, WIDTH, HEIGHT);

      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      
      ctx.font = "bold 64px sans-serif";
      ctx.lineWidth = 8;
      ctx.strokeText("YOU WIN!", WIDTH / 2, HEIGHT / 2 - 60);
      ctx.fillStyle = "#f1c40f"; 
      ctx.fillText("YOU WIN!", WIDTH / 2, HEIGHT / 2 - 60);

      ctx.font = "bold 32px sans-serif";
      ctx.lineWidth = 6;
      ctx.strokeText("Final Score: " + score, WIDTH / 2, HEIGHT / 2 + 10);
      ctx.fillStyle = "white";
      ctx.fillText("Final Score: " + score, WIDTH / 2, HEIGHT / 2 + 10);

      ctx.font = "bold 26px sans-serif";
      ctx.lineWidth = 5;
      ctx.strokeText("Press SPACE to play again", WIDTH / 2, HEIGHT / 2 + 75);
      ctx.fillStyle = "#ffeb3b"; 
      ctx.fillText("Press SPACE to play again", WIDTH / 2, HEIGHT / 2 + 75);
    }
  }

  // Animation configuration framework loop execution
  function gameLoop() {
    update();
    draw();
    requestAnimationFrame(gameLoop);
  }

  // Pre-seed baseline arrays with objects before firing execution loops
  resetGame();

  // Fire execution
  gameLoop();
</script>

</body>
</html>
