<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>🦕 Dino Runner</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');

  * { margin: 0; padding: 0; box-sizing: border-box; }

  body {
    background: #0a0a12;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    font-family: 'Press Start 2P', monospace;
    overflow: hidden;
    touch-action: none;
  }

  #gameCanvas {
    display: block;
    border-radius: 12px;
    border: 2px solid #1a2a1a;
    box-shadow: 0 0 40px #00ff6622;
  }

  #controls {
    display: grid;
    grid-template-columns: 70px 70px 70px;
    grid-template-rows: 70px 70px;
    gap: 8px;
    margin-top: 18px;
  }

  .btn {
    background: #111a11;
    border: 2px solid #2a4a2a;
    border-radius: 12px;
    color: #50e878;
    font-size: 22px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    user-select: none;
    -webkit-user-select: none;
    transition: background 0.1s, transform 0.1s;
    box-shadow: 0 4px 0 #0a120a;
    active: none;
  }

  .btn:active, .btn.pressed {
    background: #1a3a1a;
    transform: translateY(3px);
    box-shadow: 0 1px 0 #0a120a;
  }

  #btnUp    { grid-column: 2; grid-row: 1; }
  #btnLeft  { grid-column: 1; grid-row: 2; }
  #btnDown  { grid-column: 2; grid-row: 2; }
  #btnRight { grid-column: 3; grid-row: 2; }
</style>
</head>
<body>

<canvas id="gameCanvas"></canvas>

<div id="controls">
  <div class="btn" id="btnUp">▲</div>
  <div class="btn" id="btnLeft">◀</div>
  <div class="btn" id="btnDown">▼</div>
  <div class="btn" id="btnRight">▶</div>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Responsive sizing
const isMobile = window.innerWidth < 600;
const CW = Math.min(window.innerWidth - 16, 520);
const CH = isMobile ? 220 : 280;
canvas.width  = CW;
canvas.height = CH;

const GROUND_Y  = CH - 55;
const DINO_X    = 80;
const GRAVITY   = 0.7;
const JUMP_VEL  = -13;

// State
let dino, obstacles, clouds, particles;
let score, hiScore = 0, speed, baseSpeed;
let obsTimer, obsInterval;
let gameOver, started;
let keys = { up: false, down: false, left: false, right: false };

function init() {
  dino = { y: GROUND_Y - 52, vy: 0, ducking: false, onGround: true, w: 38, h: 52 };
  obstacles = [];
  clouds    = Array.from({length:4}, makeCloud);
  particles = [];
  score     = 0;
  speed     = 5;
  baseSpeed = 5;
  obsTimer  = 0;
  obsInterval = 80;
  gameOver  = false;
  started   = false;
}

function makeCloud() {
  return {
    x: CW + Math.random() * 300,
    y: 20 + Math.random() * (GROUND_Y / 2 - 20),
    w: 50 + Math.random() * 80,
    h: 18 + Math.random() * 22,
    sp: 0.3 + Math.random() * 0.5
  };
}

function makeObstacle() {
  const types = [
    { w:18, h:50, yOff:0,   kind:'cactus' },
    { w:32, h:28, yOff:0,   kind:'rock'   },
    { w:46, h:22, yOff:-48, kind:'bird'   },
  ];
  const t = types[Math.floor(Math.random() * types.length)];
  return { x: CW + 10, y: GROUND_Y - t.h + t.yOff, w: t.w, h: t.h, kind: t.kind, sp: speed };
}

function spawnParticles(x, y) {
  for (let i = 0; i < 12; i++) {
    particles.push({
      x, y,
      vx: (Math.random() - 0.5) * 5,
      vy: -Math.random() * 5,
      life: 1,
      r: 2 + Math.random() * 4
    });
  }
}

// Stars
const stars = Array.from({length:60}, () => ({
  x: Math.random() * CW,
  y: Math.random() * (GROUND_Y - 10),
  r: Math.random() * 1.5 + 0.5
}));

// Drawing helpers
function roundRect(x, y, w, h, r) {
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + r);
  ctx.lineTo(x + w, y + h - r);
  ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
  ctx.lineTo(x + r, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - r);
  ctx.lineTo(x, y + r);
  ctx.quadraticCurveTo(x, y, x + r, y);
  ctx.closePath();
}

function drawDino() {
  const dy = Math.round(dino.y);
  const dw = dino.ducking ? 54 : dino.w;
  const dh = dino.ducking ? 28 : dino.h;
  const dx = DINO_X;
  const groundY = GROUND_Y;

  ctx.fillStyle = '#50e878';
  roundRect(dx, dino.ducking ? groundY - dh : dy, dw, dh, 6);
  ctx.fill();

  if (!dino.ducking) {
    // Eye
    ctx.fillStyle = '#0a0a12';
    ctx.beginPath();
    ctx.arc(dx + dw - 8, dy + 10, 5, 0, Math.PI*2);
    ctx.fill();
    ctx.fillStyle = '#50e878';
    ctx.beginPath();
    ctx.arc(dx + dw - 6, dy + 8, 2, 0, Math.PI*2);
    ctx.fill();

    // Legs
    const t = Date.now();
    const l1 = 8 + 6 * Math.abs(Math.sin(t / 100));
    const l2 = 8 + 6 * Math.abs(Math.cos(t / 100));
    ctx.fillStyle = '#50e878';
    roundRect(dx + 6, dy + dh, 9, l1, 3); ctx.fill();
    roundRect(dx + 22, dy + dh, 9, l2, 3); ctx.fill();

    // Arm
    roundRect(dx + dw, dy + 18, 10, 6, 3); ctx.fill();
  } else {
    // Duck eye
    ctx.fillStyle = '#0a0a12';
    ctx.beginPath();
    ctx.arc(dx + dw - 8, groundY - dh + 9, 5, 0, Math.PI*2);
    ctx.fill();
  }
}

function drawObstacle(o) {
  if (o.kind === 'cactus') {
    ctx.fillStyle = '#e84040';
    roundRect(o.x, o.y, o.w, o.h, 4); ctx.fill();
    roundRect(o.x - 9, o.y + 10, 9, 16, 3); ctx.fill();
    roundRect(o.x + o.w, o.y + 16, 9, 12, 3); ctx.fill();
  } else if (o.kind === 'rock') {
    ctx.fillStyle = '#cc3333';
    ctx.beginPath();
    ctx.ellipse(o.x + o.w/2, o.y + o.h/2, o.w/2, o.h/2, 0, 0, Math.PI*2);
    ctx.fill();
  } else {
    // Bird
    const flap = Math.floor(Date.now() / 200) % 2;
    ctx.fillStyle = '#ffb020';
    roundRect(o.x, o.y, o.w, o.h, 5); ctx.fill();
    ctx.fillStyle = '#ffd060';
    const wy = flap ? o.y - 10 : o.y + 5;
    roundRect(o.x + 6, wy, o.w - 12, 12, 4); ctx.fill();
  }
}

function drawGround() {
  ctx.fillStyle = '#28c850';
  ctx.fillRect(0, GROUND_Y, CW, 5);
  ctx.fillStyle = '#0a1a0a';
  ctx.fillRect(0, GROUND_Y + 5, CW, CH - GROUND_Y - 5);
}

function drawCloud(c) {
  ctx.fillStyle = '#1a2a2a';
  ctx.beginPath();
  ctx.ellipse(c.x + c.w/2, c.y + c.h/2, c.w/2, c.h/2, 0, 0, Math.PI*2);
  ctx.fill();
}

function drawStars() {
  const t = Date.now() / 1000;
  stars.forEach((s, i) => {
    const alpha = 0.3 + 0.4 * Math.abs(Math.sin(t + i));
    ctx.fillStyle = `rgba(180,200,255,${alpha})`;
    ctx.beginPath();
    ctx.arc(s.x, s.y, s.r, 0, Math.PI*2);
    ctx.fill();
  });
}

function drawParticles() {
  particles.forEach(p => {
    ctx.fillStyle = `rgba(80,232,120,${p.life})`;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r * p.life, 0, Math.PI*2);
    ctx.fill();
  });
}

function dinoRect() {
  if (dino.ducking) return { x: DINO_X, y: GROUND_Y - 28, w: 54, h: 28 };
  return { x: DINO_X, y: dino.y, w: dino.w, h: dino.h };
}

function collides(a, b) {
  const pad = 6;
  return a.x + pad < b.x + b.w - pad &&
         a.x + a.w - pad > b.x + pad &&
         a.y + pad < b.y + b.h - pad &&
         a.y + a.h - pad > b.y + pad;
}

function update() {
  if (!started || gameOver) return;

  score++;

  // Dino physics
  if (!dino.onGround) {
    dino.vy += GRAVITY;
    dino.y  += dino.vy;
    if (dino.y >= GROUND_Y - dino.h) {
      dino.y = GROUND_Y - dino.h;
      dino.vy = 0;
      dino.onGround = true;
    }
  }

  // Ducking
  if (keys.down && dino.onGround) dino.ducking = true;
  else if (!keys.down) dino.ducking = false;

  // Speed control
  if (keys.right) speed = Math.min(baseSpeed + 5, speed + 0.06);
  else if (keys.left) speed = Math.max(2.5, speed - 0.06);
  else speed = baseSpeed;

  baseSpeed = 5 + score / 700;

  // Spawn
  obsTimer++;
  if (obsTimer >= obsInterval) {
    obstacles.push(makeObstacle());
    obsTimer = 0;
    obsInterval = 50 + Math.random() * 60;
  }

  // Update obstacles
  obstacles.forEach(o => { o.sp = speed; o.x -= o.sp; });
  obstacles = obstacles.filter(o => o.x + o.w > 0);

  // Collision
  const dr = dinoRect();
  for (const o of obstacles) {
    if (collides(dr, o)) {
      gameOver = true;
      hiScore = Math.max(hiScore, score);
      spawnParticles(DINO_X + 20, dino.y + 20);
      return;
    }
  }

  // Clouds & particles
  clouds.forEach(c => {
    c.x -= c.sp;
    if (c.x + c.w < 0) { Object.assign(c, makeCloud()); c.x = CW + 10; }
  });

  particles.forEach(p => {
    p.x += p.vx; p.y += p.vy; p.vy += 0.15; p.life -= 0.04;
  });
  particles = particles.filter(p => p.life > 0);
}

function draw() {
  // Background
  ctx.fillStyle = '#0a0a12';
  ctx.fillRect(0, 0, CW, CH);

  drawStars();
  clouds.forEach(drawCloud);
  drawGround();
  obstacles.forEach(drawObstacle);
  drawDino();
  drawParticles();

  // HUD
  ctx.fillStyle = '#50e878';
  ctx.font = `${isMobile ? 7 : 9}px 'Press Start 2P'`;
  ctx.fillText(`SCORE ${String(score).padStart(6,'0')}`, CW - (isMobile?140:170), 18);
  ctx.fillStyle = '#2a8a40';
  ctx.fillText(`BEST  ${String(hiScore).padStart(6,'0')}`, CW - (isMobile?140:170), 32);

  ctx.fillStyle = '#3060ff';
  ctx.fillText(`SPD ${speed.toFixed(1)}`, 10, 18);

  if (!started) {
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(0, 0, CW, CH);
    ctx.fillStyle = '#50e878';
    ctx.font = `${isMobile ? 13 : 18}px 'Press Start 2P'`;
    ctx.textAlign = 'center';
    ctx.fillText('DINO RUNNER', CW/2, CH/2 - 30);
    ctx.font = `${isMobile ? 7 : 9}px 'Press Start 2P'`;
    ctx.fillStyle = '#aaffaa';
    ctx.fillText('TAP  ▲  TO  START', CW/2, CH/2 + 4);
    ctx.textAlign = 'left';
  }

  if (gameOver) {
    ctx.fillStyle = 'rgba(0,0,0,0.6)';
    ctx.fillRect(0, 0, CW, CH);
    ctx.fillStyle = '#e84040';
    ctx.font = `${isMobile ? 13 : 18}px 'Press Start 2P'`;
    ctx.textAlign = 'center';
    ctx.fillText('GAME OVER', CW/2, CH/2 - 28);
    ctx.fillStyle = '#ffffff';
    ctx.font = `${isMobile ? 7 : 9}px 'Press Start 2P'`;
    ctx.fillText(`SCORE: ${score}`, CW/2, CH/2);
    ctx.fillText(`BEST:  ${hiScore}`, CW/2, CH/2 + 18);
    ctx.fillStyle = '#50e878';
    ctx.fillText('TAP  ▲  TO  RESTART', CW/2, CH/2 + 42);
    ctx.textAlign = 'left';
  }
}

function loop() {
  update();
  draw();
  requestAnimationFrame(loop);
}

// Touch controls
function setupBtn(id, key) {
  const el = document.getElementById(id);
  const press   = (e) => { e.preventDefault(); keys[key] = true;
    if (key === 'up') {
      if (!started) { started = true; return; }
      if (gameOver)  { init(); started = true; return; }
      if (dino.onGround) { dino.vy = JUMP_VEL; dino.onGround = false; dino.ducking = false; }
    }
  };
  const release = (e) => { e.preventDefault(); keys[key] = false; };
  el.addEventListener('touchstart', press,   { passive: false });
  el.addEventListener('touchend',   release, { passive: false });
  el.addEventListener('mousedown',  press);
  el.addEventListener('mouseup',    release);
}

setupBtn('btnUp',    'up');
setupBtn('btnDown',  'down');
setupBtn('btnLeft',  'left');
setupBtn('btnRight', 'right');

// Keyboard fallback
document.addEventListener('keydown', e => {
  if (e.key === 'ArrowUp')    { keys.up    = true;
    if (!started) { started = true; return; }
    if (gameOver) { init(); started = true; return; }
    if (dino.onGround) { dino.vy = JUMP_VEL; dino.onGround = false; dino.ducking = false; }
  }
  if (e.key === 'ArrowDown')  keys.down  = true;
  if (e.key === 'ArrowLeft')  keys.left  = true;
  if (e.key === 'ArrowRight') keys.right = true;
});
document.addEventListener('keyup', e => {
  if (e.key === 'ArrowUp')    keys.up    = false;
  if (e.key === 'ArrowDown')  keys.down  = false;
  if (e.key === 'ArrowLeft')  keys.left  = false;
  if (e.key === 'ArrowRight') keys.right = false;
});

init();
loop();
</script>
</body>
</html>
