# Fluffy-plane
<!doctype html>
<html lang="cs">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Fluffy Plane — Flappy-style letadlo</title>
  <meta name="description" content="Fluffy Plane: Flappy Bird styl, místo ptáka letadlo. HTML/CSS/JS single-file pro GitHub Pages." />
  <style>
    :root{
      --bg1:#87ceeb; /* sky blue */
      --bg2:#b3e5fc; /* lighter */
      --panel: rgba(255,255,255,0.06);
      --accent:#ffcc00;
      --muted:#0b1020;
      font-family: Inter, system-ui, -apple-system, 'Segoe UI', Roboto, Arial;
    }
    html,body{height:100%;margin:0;background:linear-gradient(180deg,var(--bg1),var(--bg2));display:flex;align-items:center;justify-content:center;padding:24px}
    .wrap{width:980px;max-width:96vw;display:grid;grid-template-columns:1fr 300px;gap:18px}
    .gameCard{background:linear-gradient(180deg, rgba(255,255,255,0.03), rgba(255,255,255,0.01));border-radius:12px;padding:12px;box-shadow:0 10px 30px rgba(2,6,23,0.25)}
    canvas{display:block;width:100%;height:640px;border-radius:8px;background:linear-gradient(180deg,#87ceeb,#cdeffd)}
    h1{margin:6px 0;font-size:20px}
    p.lead{margin:0;color:#04314a}

    .sidebar{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));border-radius:12px;padding:12px}
    .stat{display:flex;justify-content:space-between;margin-bottom:8px;color:#04314a}
    .btn{background:var(--accent);border:none;padding:8px 12px;border-radius:8px;font-weight:700;cursor:pointer}
    .btn.ghost{background:transparent;border:1px solid rgba(0,0,0,0.06)}
    .skin{display:flex;gap:8px;align-items:center;padding:8px;background:rgba(255,255,255,0.02);border-radius:8px;margin-bottom:8px}
    .skin button{margin-left:auto}
    footer{margin-top:10px;color:#04314a;font-size:13px}

    @media (max-width:940px){ .wrap{grid-template-columns:1fr} canvas{height:480px} }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="gameCard">
      <h1>Fluffy Plane ✈️</h1>
      <p class="lead">Prolétni mezi sloupci — ovládání: <strong>mezerník</strong>. Získej mince za průlety a kup nové skiny.</p>

      <canvas id="game" width="700" height="640" role="application" aria-label="Fluffy Plane hra"></canvas>

      <div style="display:flex;gap:8px;margin-top:10px;align-items:center">
        <button id="startBtn" class="btn">START</button>
        <button id="restartBtn" class="btn ghost">RESTART</button>
        <div style="margin-left:auto;color:#04314a">Space = tah</div>
      </div>

      <footer>Single-file hra připravená pro GitHub Pages — ulož jako <code>index.html</code>.</footer>
    </div>

    <aside class="sidebar">
      <div style="font-weight:700;margin-bottom:6px">Stav</div>
      <div class="stat"><span>Skóre</span><span id="score">0</span></div>
      <div class="stat"><span>Highscore</span><span id="best">0</span></div>
      <div class="stat"><span>Mince</span><span id="coins">0</span></div>
      <div style="height:1px;background:rgba(0,0,0,0.04);margin:8px 0"></div>

      <div style="font-weight:700;margin-bottom:6px">Obchůdek skinů</div>
      <!-- Skin entries populated by JS -->
      <div id="skins"></div>

      <div style="height:1px;background:rgba(0,0,0,0.04);margin:8px 0"></div>
      <button id="shareBtn" class="btn ghost">Uložit skóre (localStorage)</button>
    </aside>
  </div>

  <script>
    // ---------- Game config ----------
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    let W = canvas.width, H = canvas.height;

    const startBtn = document.getElementById('startBtn');
    const restartBtn = document.getElementById('restartBtn');
    const scoreEl = document.getElementById('score');
    const bestEl = document.getElementById('best');
    const coinsEl = document.getElementById('coins');
    const skinsEl = document.getElementById('skins');

    // Player physics
    const GRAVITY = 1400; // px/s^2
    const FLAP_V = -420; // px/s instant upward velocity

    // Obstacles
    const GAP = 180; // gap height
    const PIPE_W = 86; // obstacle width
    const PIPE_SPACING = 280; // horizontal distance between pipes
    const PIPE_SPEED = 200; // px/s

    // Game state
    let lastTime = 0;
    let player = null;
    let pipes = [];
    let running = false;
    let score = 0;
    let coins = Number(localStorage.getItem('fluffy_coins') || 50);
    let best = Number(localStorage.getItem('fluffy_best') || 0);
    let ownedSkins = JSON.parse(localStorage.getItem('fluffy_owned') || '[]');
    let selectedSkin = localStorage.getItem('fluffy_skin') || 'classic';

    // Skins (two extra: classic and jet)
    const SKINS = {
      classic: {name:'Klasické letadlo', price:0, svg:`<svg width="64" height="40" viewBox="0 0 64 40" xmlns="http://www.w3.org/2000/svg"><g fill="none" fill-rule="evenodd"><rect width="64" height="40" rx="6" fill="#8ecae6"/><path d="M12 20c8-6 28-8 36-2c2 1 2 4 0 5c-8 6-28 8-36 2c-2-1-2-4 0-5z" fill="#023047" opacity="0.95"/></g></svg>`},
      jet: {name:'Jet skin', price:40, svg:`<svg width="64" height="40" viewBox="0 0 64 40" xmlns="http://www.w3.org/2000/svg"><g fill="none" fill-rule="evenodd"><rect width="64" height="40" rx="6" fill="#ffb703"/><path d="M8 22c10-8 36-8 44-2v4c-8 6-34 8-44 2v-4z" fill="#fb8500"/><circle cx="50" cy="18" r="3" fill="#023047"/></g></svg>`}
    };

    // ensure classic owned
    if(!ownedSkins.includes('classic')) ownedSkins.push('classic');
    localStorage.setItem('fluffy_owned', JSON.stringify(ownedSkins));

    // UI init
    coinsEl.textContent = coins;
    bestEl.textContent = best;

    // Player object
    function createPlayer(){
      return {
        x: 150,
        y: H/2,
        vy: 0,
        w: 64,
        h: 40,
        rotation: 0,
        alive: true
      };
    }

    function spawnPipe(x){
      const centerY = 120 + Math.random() * (H - 360);
      pipes.push({x, centerY, passed:false});
    }

    function reset(){
      player = createPlayer();
      pipes = [];
      score = 0;
      running = false;
      lastTime = performance.now();
      // add initial pipes
      for(let i=0;i<3;i++) spawnPipe(W + i*PIPE_SPACING);
      updateUI();
      draw();
    }

    function updateUI(){
      scoreEl.textContent = score;
      coinsEl.textContent = coins;
      bestEl.textContent = best;
    }

    function start(){ if(running) return; running = true; lastTime = performance.now(); loop(lastTime); }
    function restart(){ reset(); start(); }

    // input
    window.addEventListener('keydown', (e)=>{ if(e.code==='Space'){ e.preventDefault(); flap(); } });
    startBtn.addEventListener('click', ()=>{ start(); });
    restartBtn.addEventListener('click', ()=>{ restart(); });

    function flap(){ if(!player || !player.alive) return; player.vy = FLAP_V; }

    // collision
    function rectsIntersect(a,b){ return !(a.x + a.w < b.x || a.x > b.x + b.w || a.y + a.h < b.y || a.y > b.y + b.h); }

    function loop(now){
      const dt = Math.min(0.03, (now - lastTime)/1000); lastTime = now;
      // update player
      player.vy += GRAVITY * dt;
      player.y += player.vy * dt;
      player.rotation = Math.max(-0.6, Math.min(0.6, player.vy / 600));

      // update pipes
      for(const p of pipes){ p.x -= PIPE_SPEED * dt; }
      // spawn
      if(pipes.length && pipes[pipes.length-1].x < W - PIPE_SPACING) spawnPipe(W + 40);

      // remove offscreen
      if(pipes.length && pipes[0].x < -PIPE_W - 40) pipes.shift();

      // scoring & collisions
      for(const p of pipes){
        if(!p.passed && p.x + PIPE_W < player.x){ p.passed = true; score++; coins += 1; updateUI(); localStorage.setItem('fluffy_coins', coins); }

        // build rects for top and bottom
        const topRect = {x: p.x, y:0, w:PIPE_W, h: p.centerY - GAP/2};
        const bottomRect = {x: p.x, y: p.centerY + GAP/2, w: PIPE_W, h: H - (p.centerY + GAP/2)};
        const playerRect = {x: player.x - player.w/2, y: player.y - player.h/2, w: player.w, h: player.h};
        if(rectsIntersect(playerRect, topRect) || rectsIntersect(playerRect, bottomRect)){
          player.alive = false; running = false; onDeath();
        }
      }

      // ground and ceiling
      if(player.y - player.h/2 < 0){ player.y = player.h/2; player.vy = 0; }
      if(player.y + player.h/2 > H - 80){ player.y = H - 80 - player.h/2; player.alive = false; running = false; onDeath(); }

      // draw
      draw();

      if(running) requestAnimationFrame(loop);
    }

    function onDeath(){
      // update best
      if(score > best){ best = score; localStorage.setItem('fluffy_best', best); }
      updateUI();
      // show simple overlay
      setTimeout(()=>{ drawGameOver(); }, 80);
    }

    // Drawing helpers
    function drawBackground(){
      // sky gradient already via canvas background; draw clouds
      ctx.fillStyle = 'rgba(255,255,255,0.6)';
      for(let i=0;i<6;i++){
        const cx = (i*200 + (Date.now()/20 % 200));
        const cy = 40 + (i%3)*40;
        drawCloud(cx % W, cy, 34);
      }
    }
    function drawCloud(x,y,s){ ctx.beginPath(); ctx.fillStyle='rgba(255,255,255,0.85)'; ctx.arc(x,y,s*0.6,0,Math.PI*2); ctx.arc(x+30,y+6,s*0.5,0,Math.PI*2); ctx.arc(x-25,y+8,s*0.45,0,Math.PI*2); ctx.fill(); }

    function drawPipes(){
      for(const p of pipes){
        const x = p.x; const cy = p.centerY;
        // top
        ctx.fillStyle='#2b9348'; ctx.fillRect(x, 0, PIPE_W, cy - GAP/2);
        // bottom
        ctx.fillStyle='#2b9348'; ctx.fillRect(x, cy + GAP/2, PIPE_W, H - (cy + GAP/2));
        // caps
        ctx.fillStyle='#1b5e20'; ctx.fillRect(x, cy - GAP/2 - 8, PIPE_W, 8);
        ctx.fillRect(x, cy + GAP/2, PIPE_W, 8);
      }
    }

    // draw plane skin as SVG path rasterized to canvas
    function drawPlane(x,y,rotation,skinKey){
      const skin = SKINS[skinKey] || SKINS['classic'];
      // render SVG string to image
      const svg = `<?xml version="1.0" encoding="UTF-8"?>` + skin.svg;
      const img = new Image();
      const svg64 = btoa(unescape(encodeURIComponent(svg)));
      img.src = 'data:image/svg+xml;base64,' + svg64;
      const w=64, h=40;
      // draw when ready
      img.onload = ()=>{
        ctx.save(); ctx.translate(x,y); ctx.rotate(rotation); ctx.drawImage(img, -w/2, -h/2, w, h); ctx.restore();
      };
      // if cached, onload may not fire before next frame; to be safe draw immediately if complete
      if(img.complete){ ctx.save(); ctx.translate(x,y); ctx.rotate(rotation); ctx.drawImage(img, -32, -20, 64, 40); ctx.restore(); }
    }

    function drawHUD(){
      ctx.fillStyle='rgba(255,255,255,0.95)'; ctx.font='20px Inter, Arial'; ctx.textAlign='left';
      ctx.fillText('Score: ' + score, 14, 28);
      ctx.fillText('Coins: ' + coins, 14, 54);
    }

    function draw(){
      // clear
      ctx.clearRect(0,0,W,H);
      drawBackground();
      drawPipes();
      // draw ground
      ctx.fillStyle='#7c4dff'; ctx.fillRect(0,H-80,W,80);
      // player
      if(player) drawPlane(player.x, player.y, player.rotation, selectedSkin);
      drawHUD();
    }

    function drawGameOver(){
      ctx.save(); ctx.fillStyle='rgba(0,0,0,0.5)'; ctx.fillRect(0,0,W,H);
      ctx.fillStyle='#fff'; ctx.font='36px Inter, Arial'; ctx.textAlign='center'; ctx.fillText('Game Over', W/2, H/2 - 20);
      ctx.font='20px Inter, Arial'; ctx.fillText('Score: ' + score + '   Best: ' + best, W/2, H/2 + 16);
      ctx.restore();
    }

    // Skins UI
    function renderSkins(){
      skinsEl.innerHTML = '';
      for(const key of Object.keys(SKINS)){
        const s = SKINS[key];
        const div = document.createElement('div'); div.className='skin';
        const thumb = document.createElement('div'); thumb.innerHTML = s.svg; thumb.style.width='64px'; thumb.style.height='40px';
        div.appendChild(thumb);
        const meta = document.createElement('div'); meta.style.flex='1'; meta.innerHTML = `<div style=\"font-weight:700\">${s.name}</div><div style=\"color:#045a7b\">Cena: ${s.price} mincí</div>`;
        div.appendChild(meta);
        const btn = document.createElement('button');
        const owned = ownedSkins.includes(key);
        btn.textContent = (owned ? (selectedSkin===key ? 'Selected' : 'Select') : 'Buy');
        btn.disabled = (selectedSkin===key);
        btn.addEventListener('click', ()=>{
          if(owned){ selectedSkin = key; localStorage.setItem('fluffy_skin', selectedSkin); updateSkinButtons(); }
          else{ // buy
            if(coins >= s.price){ coins -= s.price; ownedSkins.push(key); localStorage.setItem('fluffy_owned', JSON.stringify(ownedSkins)); localStorage.setItem('fluffy_coins', coins); selectedSkin = key; localStorage.setItem('fluffy_skin', selectedSkin); updateSkinButtons(); updateUI(); }
            else{ alert('Nedostatek mincí'); }
          }
        });
        div.appendChild(btn);
        skinsEl.appendChild(div);
      }
      updateSkinButtons();
    }

    function updateSkinButtons(){
      const buttons = skinsEl.querySelectorAll('button');
      Object.keys(SKINS).forEach((key,i)=>{
        const btn = buttons[i]; const owned = ownedSkins.includes(key);
        btn.textContent = owned ? (selectedSkin===key ? 'Selected' : 'Select') : 'Buy';
        btn.disabled = (selectedSkin===key);
      });
    }

    // save best / coins
    document.getElementById('shareBtn').addEventListener('click', ()=>{ localStorage.setItem('fluffy_best', best); localStorage.setItem('fluffy_coins', coins); alert('Skóre a mince uloženy v localStorage.'); });

    // init
    function init(){
      W = canvas.width; H = canvas.height;
      reset();
      renderSkins();
      draw();
    }

    init();

    // responsive: scale canvas to container width keeping logical resolution
    function fitCanvas(){
      const rect = canvas.getBoundingClientRect();
      const ratio = Math.min(1.0, rect.width / W);
      canvas.style.width = (W * ratio) + 'px'; canvas.style.height = (H * ratio) + 'px';
    }
    window.addEventListener('resize', fitCanvas); fitCanvas();

  </script>
</body>
</html>
