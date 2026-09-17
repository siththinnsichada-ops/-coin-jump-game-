<!doctype html>
<html lang="th">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover">
<title>กระโดดยิงเหรียญ - Pro Controls</title>
<style>
html,body{margin:0;width:100%;height:100%;overflow:hidden;background:#111;font-family:-apple-system,BlinkMacSystemFont,"Noto Sans Thai",sans-serif;touch-action:none;user-select:none;-webkit-user-select:none}
canvas{display:block;width:100%;height:100%;background:#79d7ff}
#ui{position:fixed;inset:0;pointer-events:none;z-index:10}
#score{position:absolute;top:max(12px,env(safe-area-inset-top));left:16px;background:#ffffffee;padding:8px 16px;border-radius:18px;font-weight:800;font-size:16px;box-shadow:0 3px 10px #0003;backdrop-filter:blur(4px)}
#msg{position:absolute;top:20%;left:50%;transform:translateX(-50%);text-align:center;background:#ffffffee;padding:16px 26px;border-radius:20px;font-weight:800;font-size:20px;display:none;box-shadow:0 8px 20px #0002;backdrop-filter:blur(4px);z-index:20;width:80%;max-width:300px}

/* Joypad Layout */
.controls-left{position:absolute;bottom:max(12px,env(safe-area-inset-bottom));left:max(12px,env(safe-area-inset-left));display:grid;grid-template-columns:repeat(3, 56px);grid-template-rows:repeat(2, 56px);gap:6px;pointer-events:none}
.controls-right{position:absolute;bottom:max(12px,env(safe-area-inset-bottom));right:max(12px,env(safe-area-inset-right));display:flex;flex-direction:column;gap:8px;align-items:flex-end;pointer-events:none}
.action-row{display:flex;gap:8px}

button.btn-pad{pointer-events:auto;border:0;border-radius:16px;width:56px;height:56px;font-size:18px;font-weight:900;color:#fff;background:#2478ffdd;box-shadow:0 4px 0 #0004;outline:none;-webkit-tap-highlight-color:transparent;backdrop-filter:blur(2px)}
button.btn-pad:active{transform:translateY(2px);box-shadow:0 1px 0 #0004;background:#105bd4}

#btn-left{grid-column:1;grid-row:2}
#btn-duck{grid-column:2;grid-row:2;background:#e67e22dd}
#btn-right{grid-column:3;grid-row:2}

.btn-action{pointer-events:auto;border:0;border-radius:50%;width:62px;height:62px;font-size:15px;font-weight:900;color:#fff;box-shadow:0 4px 0 #0004;outline:none;-webkit-tap-highlight-color:transparent}
.btn-action:active{transform:translateY(2px);box-shadow:0 1px 0 #0004}
#btn-jump{background:#2478ff;width:70px;height:70px;font-size:18px}
#btn-shoot-fwd{background:#ef3d42}
#btn-shoot-up{background:#9b59b6}

#restart{position:absolute;left:50%;bottom:28%;transform:translateX(-50%);display:none;border-radius:18px;width:auto;height:auto;padding:14px 28px;background:#222;color:white;font-size:18px;pointer-events:auto;z-index:30;border:none;box-shadow:0 4px 12px #0004}
#start-overlay{position:absolute;inset:0;background:#0007;display:flex;align-items:center;justify-content:center;pointer-events:auto;z-index:40}
.start-card{background:white;padding:28px;border-radius:22px;text-align:center;box-shadow:0 8px 25px #0005;max-width:300px}
small{font-size:12px;color:#666}
</style>
</head>
<body>
<canvas id="game"></canvas>
<div id="ui">
  <div id="start-overlay">
    <div class="start-card">
      <div style="font-size:40px;margin-bottom:8px">🎮🔫</div>
      <b style="font-size:22px">กระโดดยิงเหรียญ</b><br><br>
      <small>
        <b>ฝั่งซ้าย:</b> เดินซ้าย/ขวา, หมอบหลบ<br>
        <b>ฝั่งขวา:</b> กระโดด, ยิงตรง, ยิงขึ้นฟ้า<br>
        เก็บเหรียญครบ 12 เหรียญเพื่อชนะ!
      </small>
    </div>
  </div>
  <div id="score">🪙 0 &nbsp; ❤️ 3 &nbsp; ด่าน 1</div>
  <div id="msg"></div>
  <button id="restart">เล่นอีกครั้ง 🔄</button>

  <!-- D-Pad Left Side -->
  <div class="controls-left">
    <button class="btn-pad" id="btn-left">◀</button>
    <button class="btn-pad" id="btn-duck">▼</button>
    <button class="btn-pad" id="btn-right">▶</button>
  </div>

  <!-- Action Buttons Right Side -->
  <div class="controls-right">
    <button class="btn-action" id="btn-shoot-up">👆🔫</button>
    <div class="action-row">
      <button class="btn-action" id="btn-shoot-fwd">🔫</button>
      <button class="btn-action" id="btn-jump">🦘</button>
    </div>
  </div>
</div>

<script>
const c=document.getElementById('game'),ctx=c.getContext('2d');
let W,H,dpr,scaleFactor=1,coins=0,lives=3,over=false,won=false,started=false;

// Player Properties
const NORMAL_H = 58, DUCK_H = 32;
let player={x:80,y:0,w:42,h:NORMAL_H,vy:0,onGround:false,isDucking:false,speed:5};
let inputState = { left: false, right: false, duck: false };

let gameSpeed=4.2,ground=0,scroll=0,spawn=0,shots=[],coinList=[],blocks=[],birds=[],particles=[];

// Audio System
let audioCtx = null;
function initAudio(){
  try {
    if(!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if(audioCtx.state === 'suspended') audioCtx.resume();
  } catch(e){}
}

function playSfx(type){
  if(!audioCtx) return;
  try {
    const osc = audioCtx.createOscillator(), gain = audioCtx.createGain();
    osc.connect(gain); gain.connect(audioCtx.destination);
    const now = audioCtx.currentTime;
    if(type === 'jump'){
      osc.type = 'sine'; osc.frequency.setValueAtTime(150, now);
      osc.frequency.exponentialRampToValueAtTime(420, now + 0.12);
      gain.gain.setValueAtTime(0.2, now); gain.gain.linearRampToValueAtTime(0.01, now + 0.12);
      osc.start(now); osc.stop(now + 0.12);
    } else if(type === 'coin'){
      osc.type = 'triangle'; osc.frequency.setValueAtTime(523, now);
      osc.frequency.setValueAtTime(659, now + 0.08);
      gain.gain.setValueAtTime(0.2, now); gain.gain.linearRampToValueAtTime(0.01, now + 0.18);
      osc.start(now); osc.stop(now + 0.18);
    } else if(type === 'shoot'){
      osc.type = 'square'; osc.frequency.setValueAtTime(600, now);
      osc.frequency.exponentialRampToValueAtTime(120, now + 0.08);
      gain.gain.setValueAtTime(0.1, now); gain.gain.linearRampToValueAtTime(0.01, now + 0.08);
      osc.start(now); osc.stop(now + 0.08);
    } else if(type === 'hit'||type === 'break'){
      osc.type = 'sawtooth'; osc.frequency.setValueAtTime(220, now);
      osc.frequency.linearRampToValueAtTime(50, now + 0.18);
      gain.gain.setValueAtTime(0.25, now); gain.gain.linearRampToValueAtTime(0.01, now + 0.18);
      osc.start(now); osc.stop(now + 0.18);
    }
  } catch(e){}
}

// Responsive Viewport Scaling
function resize(){
  dpr = Math.min(devicePixelRatio || 1, 2);
  W = innerWidth; H = innerHeight;
  c.width = W * dpr; c.height = H * dpr;
  c.style.width = W + 'px'; c.style.height = H + 'px';
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

  // Dynamic ground & scale based on device screen height
  ground = H - Math.max(85, Math.min(120, H * 0.18));
  scaleFactor = H / 450; 
  if(player.y > ground - player.h) player.y = ground - player.h;
}
addEventListener('resize', resize); resize();

function startGame(){
  initAudio();
  if(!started){
    started = true;
    document.getElementById('start-overlay').style.display = 'none';
  }
}

function reset(){
  coins=0; lives=3; over=false; won=false; started=true; scroll=0; spawn=0;
  shots=[]; coinList=[]; blocks=[]; birds=[]; particles=[];
  player.x=80; player.h=NORMAL_H; player.y=ground-player.h; player.vy=0; player.onGround=true; player.isDucking=false;
  document.getElementById('restart').style.display='none';
  document.getElementById('msg').style.display='none';
  document.getElementById('start-overlay').style.display='none';
}

function jump(){
  startGame();
  if(!over && !won && player.onGround && !player.isDucking){
    player.vy = -13.5;
    player.onGround = false;
    playSfx('jump');
  }
}

function shoot(direction = 'fwd'){
  startGame();
  if(over || won) return;
  if(direction === 'up'){
    shots.push({x: player.x + player.w / 2, y: player.y, vx: 0, vy: -12});
  } else {
    let gunY = player.isDucking ? player.y + 12 : player.y + 24;
    shots.push({x: player.x + player.w, y: gunY, vx: 11, vy: 0});
  }
  playSfx('shoot');
}

function addBurst(x,y,color,count=6){
  for(let i=0;i<count;i++){
    particles.push({x,y,vx:(Math.random()-0.5)*6,vy:(Math.random()-0.5)*6,life:1,color});
  }
}

// Touch Button Event Handlers (Multi-Touch Safe)
function bindTouchBtn(id, onPress, onRelease){
  const btn = document.getElementById(id);
  const start = (e)=>{ e.preventDefault(); e.stopPropagation(); onPress(); };
  const end = (e)=>{ e.preventDefault(); e.stopPropagation(); if(onRelease) onRelease(); };
  btn.addEventListener('pointerdown', start);
  btn.addEventListener('pointerup', end);
  btn.addEventListener('pointercancel', end);
  btn.addEventListener('pointerleave', end);
}

bindTouchBtn('btn-left', ()=>{ inputState.left = true; }, ()=>{ inputState.left = false; });
bindTouchBtn('btn-right', ()=>{ inputState.right = true; }, ()=>{ inputState.right = false; });
bindTouchBtn('btn-duck', ()=>{ inputState.duck = true; }, ()=>{ inputState.duck = false; });

bindTouchBtn('btn-jump', jump);
bindTouchBtn('btn-shoot-fwd', ()=>shoot('fwd'));
bindTouchBtn('btn-shoot-up', ()=>shoot('up'));

document.getElementById('start-overlay').addEventListener('pointerdown', (e)=>{ e.preventDefault(); startGame(); });
document.getElementById('restart').addEventListener('click', reset);

// Keyboard Listeners
addEventListener('keydown', e=>{
  if(e.key==='ArrowLeft'||e.key==='a'||e.key==='A') inputState.left = true;
  if(e.key==='ArrowRight'||e.key==='d'||e.key==='D') inputState.right = true;
  if(e.key==='ArrowDown'||e.key==='s'||e.key==='S') inputState.duck = true;
  if(e.code==='Space'||e.key==='ArrowUp'||e.key==='w'||e.key==='W') jump();
  if(e.key==='x'||e.key==='X'||e.key==='Enter') shoot('fwd');
  if(e.key==='z'||e.key==='Z') shoot('up');
  if(e.key==='r'||e.key==='R') reset();
});

addEventListener('keyup', e=>{
  if(e.key==='ArrowLeft'||e.key==='a'||e.key==='A') inputState.left = false;
  if(e.key==='ArrowRight'||e.key==='d'||e.key==='D') inputState.right = false;
  if(e.key==='ArrowDown'||e.key==='s'||e.key==='S') inputState.duck = false;
});

function rectHit(a,b){
  return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
}

function spawnThings(){
  spawn++;
  if(spawn % 70 === 0){ coinList.push({x:W+40, y:ground-80-Math.random()*110, r:12}); }
  if(spawn % 130 === 0){ blocks.push({x:W+40, y:ground-38, w:32+Math.random()*24, h:38}); }
  // Crows flying low (Need ducking or shooting)
  if(spawn % 160 === 0){ 
    birds.push({x:W+40, y:ground-62, w:32, h:24, wing:0}); 
  }
}

function update(){
  if(!started || over || won) return;
  scroll += gameSpeed; 
  spawnThings();

  // Horizontal Movement Logic
  if(inputState.left) player.x -= player.speed;
  if(inputState.right) player.x += player.speed;
  player.x = Math.max(10, Math.min(W - player.w - 10, player.x));

  // Ducking State Adjustment
  if(inputState.duck && player.onGround){
    if(!player.isDucking){
      player.isDucking = true;
      player.h = DUCK_H;
      player.y = ground - DUCK_H;
    }
  } else {
    if(player.isDucking){
      player.isDucking = false;
      player.h = NORMAL_H;
      player.y = ground - NORMAL_H;
    }
  }

  // Gravity & Physics
  player.vy += 0.65;
  player.y += player.vy;
  if(player.y >= ground - player.h){
    player.y = ground - player.h;
    player.vy = 0;
    player.onGround = true;
  }

  // Bullets Position Update
  shots.forEach(s=>{ s.x += s.vx; s.y += s.vy; }); 
  shots = shots.filter(s=>s.x < W+30 && s.y > -30);

  coinList.forEach(o=>o.x-=gameSpeed);
  blocks.forEach(o=>o.x-=gameSpeed);
  birds.forEach(b=>{ b.x -= (gameSpeed + 1.2); b.wing += 0.25; });

  // Particles
  particles.forEach(p=>{ p.x+=p.vx; p.y+=p.vy; p.life-=0.05; });
  particles=particles.filter(p=>p.life>0);

  let pBox = {x: player.x + 6, y: player.y + 4, w: player.w - 12, h: player.h - 4};

  // 1. Coins Collection
  for(let i=coinList.length-1; i>=0; i--){
    let o = coinList[i], hit = false;
    for(let j=shots.length-1; j>=0; j--){
      if(Math.hypot(shots[j].x-o.x, shots[j].y-o.y) < 18){
        hit = true; shots.splice(j,1); break;
      }
    }
    let cBox = {x: o.x - o.r, y: o.y - o.r, w: o.r * 2, h: o.r * 2};
    if(hit || rectHit(pBox, cBox)){
      coins++;
      playSfx('coin');
      addBurst(o.x, o.y, '#ffd42a');
      coinList.splice(i,1);
    }
  }

  // 2. Blocks Collision / Destroy
  for(let i=blocks.length-1; i>=0; i--){
    let b = blocks[i], shotHit = false;
    for(let j=shots.length-1; j>=0; j--){
      let s = shots[j];
      if(rectHit({x: s.x-5, y: s.y-5, w: 10, h: 10}, b)){
        shotHit = true; shots.splice(j,1); break;
      }
    }
    if(shotHit){
      addBurst(b.x+b.w/2, b.y+b.h/2, '#8d4b2e', 8);
      playSfx('break');
      blocks.splice(i,1);
    } else if(rectHit(pBox, b)){
      addBurst(b.x+b.w/2, b.y+b.h/2, '#8d4b2e');
      blocks.splice(i,1);
      lives--;
      playSfx('hit');
      if(lives <= 0) gameOver();
    }
  }

  // 3. Birds Collision / Destroy
  for(let i=birds.length-1; i>=0; i--){
    let bird = birds[i], shotHit = false;
    for(let j=shots.length-1; j>=0; j--){
      let s = shots[j];
      if(rectHit({x: s.x-5, y: s.y-5, w: 10, h: 10}, bird)){
        shotHit = true; shots.splice(j,1); break;
      }
    }
    if(shotHit){
      addBurst(bird.x+bird.w/2, bird.y+bird.h/2, '#333333', 8);
      playSfx('break');
      birds.splice(i,1);
    } else if(rectHit(pBox, bird)){
      addBurst(bird.x+bird.w/2, bird.y+bird.h/2, '#ef3d42');
      birds.splice(i,1);
      lives--;
      playSfx('hit');
      if(lives <= 0) gameOver();
    }
  }

  coinList = coinList.filter(o=>o.x>-40);
  blocks = blocks.filter(o=>o.x>-60);
  birds = birds.filter(b=>b.x>-50);

  if(coins >= 12){
    won = true;
    showMsg('🎉 ผ่านด่าน 1!<br><small>เก็บเหรียญครบ 12 เหรียญแล้ว</small>');
    document.getElementById('restart').style.display = 'block';
  }

  document.getElementById('score').textContent = `🪙 ${coins}   ❤️ ${lives}   ด่าน 1`;
}

function gameOver(){
  over = true;
  showMsg('เกมจบ 😵<br><small>โดนสิ่งกีดขวางหรืออีกาชน!</small>');
  document.getElementById('restart').style.display = 'block';
}

function showMsg(t){
  let m = document.getElementById('msg');
  m.innerHTML = t;
  m.style.display = 'block';
}

function draw(){
  ctx.clearRect(0,0,W,H);
  
  // Sky Gradient
  let g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0,'#50c4ff'); g.addColorStop(1,'#e4f7ff');
  ctx.fillStyle = g; ctx.fillRect(0,0,W,H);

  // Parallax Clouds
  ctx.fillStyle = '#ffffffbb';
  for(let x = -(scroll * 0.25 % 220) - 40; x < W; x += 220){
    ctx.beginPath();
    ctx.arc(x, 75, 22, 0, 7); ctx.arc(x + 26, 68, 28, 0, 7); ctx.arc(x + 58, 78, 20, 0, 7);
    ctx.fill();
  }

  // Ground
  ctx.fillStyle = '#4CAF50'; ctx.fillRect(0, ground, W, H - ground);
  ctx.fillStyle = '#795548'; ctx.fillRect(0, ground + 10, W, H - ground - 10);

  // Blocks
  blocks.forEach(b=>{
    ctx.fillStyle = '#8d4b2e'; ctx.fillRect(b.x, b.y, b.w, b.h);
    ctx.fillStyle = '#b76b3e'; ctx.fillRect(b.x + 4, b.y + 4, b.w - 8, 6);
  });

  // Birds
  birds.forEach(b=>{
    let wingY = Math.sin(b.wing) * 7;
    ctx.fillStyle = '#222';
    ctx.beginPath(); ctx.ellipse(b.x + 16, b.y + 12, 12, 7, 0, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.arc(b.x + 6, b.y + 9, 6, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#ff9900';
    ctx.beginPath(); ctx.moveTo(b.x + 2, b.y + 9); ctx.lineTo(b.x - 5, b.y + 11); ctx.lineTo(b.x + 2, b.y + 13); ctx.fill();
    ctx.fillStyle = '#111';
    ctx.beginPath(); ctx.moveTo(b.x + 14, b.y + 10); ctx.lineTo(b.x + 22, b.y + 2 - wingY); ctx.lineTo(b.x + 24, b.y + 10); ctx.fill();
  });

  // Coins
  coinList.forEach(o=>{
    ctx.fillStyle = '#ffd42a'; ctx.beginPath(); ctx.arc(o.x, o.y, o.r, 0, 7); ctx.fill();
    ctx.strokeStyle = '#d79600'; ctx.lineWidth = 2.5; ctx.stroke();
    ctx.fillStyle = '#fff2a0'; ctx.font = 'bold 12px sans-serif'; ctx.fillText('$', o.x - 3.5, o.y + 4);
  });

  // Bullets
  shots.forEach(s=>{
    ctx.fillStyle = '#ffec40'; ctx.beginPath(); ctx.arc(s.x, s.y, 5, 0, 7); ctx.fill();
  });

  // Particles
  particles.forEach(p=>{
    ctx.fillStyle = p.color; ctx.globalAlpha = p.life;
    ctx.beginPath(); ctx.arc(p.x, p.y, 3, 0, 7); ctx.fill();
    ctx.globalAlpha = 1;
  });

  // Player Drawing (Normal vs Ducking)
  if(player.isDucking){
    // Ducking Pose
    ctx.fillStyle = '#1d3b72'; ctx.fillRect(player.x + 4, player.y + 10, 32, 22); // Body
    ctx.fillStyle = '#f2b28b'; ctx.beginPath(); ctx.arc(player.x + 28, player.y + 10, 11, 0, 7); ctx.fill(); // Head
    ctx.fillStyle = '#111'; ctx.fillRect(player.x + 16, player.y, 24, 6); // Cap
    ctx.fillStyle = '#333'; ctx.fillRect(player.x + 28, player.y + 12, 22, 6); // Low Gun
  } else {
    // Standing Pose
    ctx.fillStyle = '#1d3b72'; ctx.fillRect(player.x + 8, player.y + 20, 26, 38); // Body
    ctx.fillStyle = '#f2b28b'; ctx.beginPath(); ctx.arc(player.x + 21, player.y + 12, 14, 0, 7); ctx.fill(); // Head
    ctx.fillStyle = '#111'; ctx.fillRect(player.x + 6, player.y - 2, 31, 7); // Cap
    ctx.fillStyle = '#333'; ctx.fillRect(player.x + 29, player.y + 24, 24, 7); ctx.fillRect(player.x + 38, player.y + 30, 6, 10); // Gun
  }
}

function loop(){
  update();
  draw();
  requestAnimationFrame(loop);
}

loop();
</script>
</body>
</html>
