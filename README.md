<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Simple Tank Game — Click to Fire</title>
<style>
  html,body { height:100%; margin:0; background:#0b1220; color:#fff; font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial; }
  #game { display:block; width:100%; height:100vh; cursor: crosshair; background: linear-gradient(#0b1220, #16304a 60%); }
  .hud { position:fixed; left:12px; top:12px; z-index:10; font-weight:700; }
  .controls { position:fixed; right:12px; top:12px; z-index:10; }
  button { padding:8px 12px; border-radius:8px; border:none; cursor:pointer; }
</style>
</head>
<body>
<canvas id="game"></canvas>
<div class="hud">Score: <span id="score">0</span></div>
<div class="controls">
  <button id="restart">Restart</button>
  <button id="soundToggle">Sound: ON</button>
</div>

<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const scoreEl = document.getElementById('score');
  const restartBtn = document.getElementById('restart');
  const soundToggle = document.getElementById('soundToggle');

  let W = canvas.width = innerWidth;
  let H = canvas.height = innerHeight;

  // Game state
  let bullets = [];
  let enemies = [];
  let particles = [];
  let score = 0;
  let lastSpawn = 0;
  let spawnInterval = 1400; // ms
  let lastTime = performance.now();
  let gameOver = false;
  let soundOn = true;

  // Player tank at bottom center
  const tank = {
    x: () => W/2,
    y: () => H - 70,
    baseRadius: 36,
    turretLength: 44
  };

  // Mouse
  const mouse = { x: W/2, y: H/2 };

  // Resize
  function resize(){ W = canvas.width = innerWidth; H = canvas.height = innerHeight; }
  addEventListener('resize', resize);

  // Mouse events - click to fire, move to aim
  canvas.addEventListener('mousemove', e => {
    const r = canvas.getBoundingClientRect();
    mouse.x = e.clientX - r.left;
    mouse.y = e.clientY - r.top;
  });

  canvas.addEventListener('mousedown', e => {
    if (gameOver) return;
    const r = canvas.getBoundingClientRect();
    const mx = e.clientX - r.left, my = e.clientY - r.top;
    mouse.x = mx; mouse.y = my;
    fireShell(mx, my);
  });

  // Touch support
  canvas.addEventListener('touchstart', e => {
    e.preventDefault();
    const t = e.touches[0];
    const r = canvas.getBoundingClientRect();
    const tx = t.clientX - r.left, ty = t.clientY - r.top;
    mouse.x = tx; mouse.y = ty;
    fireShell(tx, ty);
  }, {passive:false});
  canvas.addEventListener('touchmove', e => {
    const t = e.touches[0];
    const r = canvas.getBoundingClientRect();
    mouse.x = t.clientX - r.left; mouse.y = t.clientY - r.top;
  }, {passive:false});

  // Sound (tiny WebAudio)
  const AudioCtx = window.AudioContext || window.webkitAudioContext;
  let audioCtx = null;
  function ensureAudioCtx() { if (!audioCtx) audioCtx = new AudioCtx(); if (audioCtx.state === 'suspended') audioCtx.resume(); }

  function playBeep(freq, dur=0.06, type='sine', vol=0.06){
    if (!soundOn) return;
    ensureAudioCtx();
    const o = audioCtx.createOscillator();
    const g = audioCtx.createGain();
    o.type = type; o.frequency.value = freq;
    g.gain.value = vol;
    o.connect(g); g.connect(audioCtx.destination);
    o.start();
    o.stop(audioCtx.currentTime + dur);
  }

  // Fire a shell
  function fireShell(tx, ty) {
    const sx = tank.x();
    const sy = tank.y();
    const dx = tx - sx, dy = ty - sy;
    const len = Math.hypot(dx, dy) || 1;
    const speed = 720; // px/s
    const vx = (dx/len) * speed;
    const vy = (dy/len) * speed;
    bullets.push({
      x: sx + (dx/len) * tank.turretLength,
      y: sy + (dy/len) * tank.turretLength,
      vx, vy, r: 6,
      life: 2000
    });
    playBeep(700 + Math.random()*200, 0.06, 'square', 0.08);
    // small recoil particle
    for (let i=0;i<8;i++){
      particles.push({ x: sx + (dx/len)*(tank.turretLength-6), y: sy + (dy/len)*(tank.turretLength-6), vx:(Math.random()-0.5)*120 - vx*0.02, vy:(Math.random()-0.5)*120 - vy*0.02, life: 300 + Math.random()*300, r:2 + Math.random()*2 });
    }
  }

  // Spawn enemies from top or sides
  function spawnEnemy(){
    const type = Math.random() < 0.7 ? 'light' : 'heavy';
    const r = type === 'light' ? 18 + Math.random()*12 : 28 + Math.random()*8;
    // spawn at random top x
    const x = Math.random()* (W - 2*r) + r;
    const y = -r - 10;
    const speed = type === 'light' ? 40 + Math.random()*80 : 20 + Math.random()*40;
    enemies.push({ x, y, r, speed, hp: type==='light' ? 1 : 2, type });
  }

  // Update loop
  function update(dt){
    // spawn
    lastSpawn += dt*1000;
    if (lastSpawn >= spawnInterval) { lastSpawn = 0; spawnEnemy(); if (spawnInterval>600) spawnInterval *= 0.98; }

    // bullets
    for (let i=bullets.length-1;i>=0;i--){
      const b = bullets[i];
      b.x += b.vx * dt; b.y += b.vy * dt; b.life -= dt*1000;
      if (b.life <=0 || b.x < -50 || b.x > W+50 || b.y < -50 || b.y > H+50) bullets.splice(i,1);
    }

    // enemies
    for (let i=enemies.length-1;i>=0;i--){
      const e = enemies[i];
      // move downwards slowly and slightly wiggle
      e.y += e.speed * dt;
      e.x += Math.sin((performance.now()+i*100)/700)*10*dt;
      // reach bottom -> game over
      if (e.y - e.r > H - 20) { gameOver = true; }
    }

    // particles
    for (let i=particles.length-1;i>=0;i--){
      const p = particles[i];
      p.x += p.vx * dt; p.y += p.vy * dt; p.life -= dt*1000;
      p.vy += 400 * dt; // gravity
      if (p.life <= 0) particles.splice(i,1);
    }

    // collisions (bullets vs enemies)
    for (let i=bullets.length-1;i>=0;i--){
      const b = bullets[i];
      for (let j=enemies.length-1;j>=0;j--){
        const e = enemies[j];
        const dx = b.x - e.x, dy = b.y - e.y;
        const dist = Math.hypot(dx, dy);
        if (dist < b.r + e.r) {
          // hit
          bullets.splice(i,1);
          e.hp -= 1;
          playBeep(200 + Math.random()*200, 0.08, 'sawtooth', 0.08);
          // spawn impact particles
          for (let k=0;k<18;k++){
            const ang = Math.random()*Math.PI*2;
            const sp = 60 + Math.random()*160;
            particles.push({ x: b.x, y: b.y, vx: Math.cos(ang)*sp, vy: Math.sin(ang)*sp, life: 300 + Math.random()*500, r:2 + Math.random()*2 });
          }
          if (e.hp <= 0) {
            score += Math.round(10 + e.r);
            scoreEl.textContent = score;
            enemies.splice(j,1);
            playBeep(120 + Math.random()*80, 0.12, 'triangle', 0.12);
          }
          break;
        }
      }
    }
  }

  // Draw tank, enemies, bullets
  function draw(){
    ctx.clearRect(0,0,W,H);

    // ground band
    ctx.fillStyle = '#0b1726';
    ctx.fillRect(0, H - 80, W, 80);

    // draw enemies
    for (const e of enemies){
      // body
      ctx.beginPath();
      const grad = ctx.createLinearGradient(e.x - e.r, e.y - e.r, e.x + e.r, e.y + e.r);
      grad.addColorStop(0, e.type==='light' ? '#5eead4' : '#fca5a5');
      grad.addColorStop(1, e.type==='light' ? '#2dd4bf' : '#ef4444');
      ctx.fillStyle = grad;
      ctx.arc(e.x, e.y, e.r, 0, Math.PI*2);
      ctx.fill();
      // turret/eye
      ctx.beginPath();
      ctx.fillStyle = 'rgba(0,0,0,0.15)';
      ctx.fillRect(e.x - e.r*0.25, e.y - e.r*0.2, e.r*0.5, e.r*0.25);
    }

    // draw particles
    for (const p of particles){
      ctx.beginPath();
      const lifePct = Math.max(0, p.life/800);
      ctx.fillStyle = `rgba(255,220,120,${Math.min(1, lifePct)})`;
      ctx.arc(p.x, p.y, p.r, 0, Math.PI*2);
      ctx.fill();
    }

    // draw bullets
    for (const b of bullets){
      ctx.beginPath();
      ctx.fillStyle = '#ffd86b';
      ctx.arc(b.x, b.y, b.r, 0, Math.PI*2);
      ctx.fill();
      // trail
      ctx.beginPath();
      ctx.strokeStyle = 'rgba(255,215,110,0.4)';
      ctx.lineWidth = 2;
      ctx.moveTo(b.x - b.vx*0.012, b.y - b.vy*0.012);
      ctx.lineTo(b.x, b.y);
      ctx.stroke();
    }

    // draw tank base
    const sx = tank.x(), sy = tank.y();
    ctx.save();
    // base plate
    ctx.beginPath();
    ctx.fillStyle = '#2b6cb0';
    roundRect(ctx, sx - tank.baseRadius - 10, sy - 18, tank.baseRadius*2 + 20, 36, 12, true, false);
    // tracks
    ctx.fillStyle = '#12324a';
    ctx.fillRect(sx - tank.baseRadius - 6, sy - 12, (tank.baseRadius*2 + 12), 24);
    ctx.restore();

    // turret aim
    const dx = mouse.x - sx, dy = mouse.y - sy;
    const angle = Math.atan2(dy, dx);

    // turret
    ctx.save();
    ctx.translate(sx, sy);
    ctx.rotate(angle);
    // barrel
    ctx.fillStyle = '#0ea5a4';
    ctx.fillRect(0, -8, tank.turretLength + 8, 16);
    // turret dome
    ctx.beginPath();
    ctx.fillStyle = '#06b6d4';
    ctx.arc(0, 0, 16, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();

    // crosshair at mouse
    ctx.beginPath();
    ctx.strokeStyle = 'rgba(255,255,255,0.12)';
    ctx.arc(mouse.x, mouse.y, 18, 0, Math.PI*2);
    ctx.stroke();

    // if game over overlay
    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.6)';
      ctx.fillRect(0,0,W,H);
      ctx.fillStyle = '#fff';
      ctx.textAlign = 'center';
      ctx.font = 'bold 48px system-ui, Arial';
      ctx.fillText('Game Over', W/2, H/2 - 20);
      ctx.font = '24px system-ui, Arial';
      ctx.fillText('Score: ' + score, W/2, H/2 + 28);
      ctx.font = '16px system-ui, Arial';
      ctx.fillText('Click Restart to play again', W/2, H/2 + 68);
    }
  }

  // helper: rounded rect
  function roundRect(ctx, x, y, w, h, r, fill, stroke) {
    if (typeof r === 'undefined') r = 5;
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.arcTo(x + w, y,     x + w, y + h, r);
    ctx.arcTo(x + w, y + h, x,     y + h, r);
    ctx.arcTo(x,     y + h, x,     y,     r);
    ctx.arcTo(x,     y,     x + w, y,     r);
    ctx.closePath();
    if (fill) ctx.fill();
    if (stroke) ctx.stroke();
  }

  // main loop
  function loop(now){
    const dt = Math.min(0.04, (now - lastTime)/1000);
    lastTime = now;
    if (!gameOver) update(dt);
    draw();
    requestAnimationFrame(loop);
  }

  // restart
  function resetGame(){
    bullets = []; enemies = []; particles = []; score = 0; lastSpawn = 0; spawnInterval = 1400; gameOver = false;
    scoreEl.textContent = score;
    cart = null;
  }
  restartBtn.addEventListener('click', ()=>{ resetGame(); });

  soundToggle.addEventListener('click', ()=>{ soundOn = !soundOn; soundToggle.textContent = 'Sound: ' + (soundOn?'ON':'OFF'); });

  // enable audio on first interaction (some browsers)
  window.addEventListener('click', ()=>{ if (audioCtx && audioCtx.state === 'suspended') audioCtx.resume(); }, { once:true });

  // initial spawns
  for (let i=0;i<2;i++) spawnEnemy();

  lastTime = performance.now();
  requestAnimationFrame(loop);

})();
</script>
</body>
</html>
