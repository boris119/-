# -
Это симулятор
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, touch-action: manipulation">
  <title>🌌 Космический симулятор на максималках</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
      -webkit-tap-highlight-color: transparent;
      user-select: none;
      font-family: 'Courier New', 'Courier', monospace;
    }
    body {
      background: #0a0a1a;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      width: 100vw;
      overflow: hidden;
      touch-action: pan-x pan-y; 
    }
    #game-container {
      position: relative;
      width: 100vw;
      height: 100vh;
      max-width: 500px;
      max-height: 900px;
      background: radial-gradient(circle at 20% 20%, #1a1a3a, #050510);
      box-shadow: 0 0 40px rgba(80, 120, 255, 0.3);
      overflow: hidden;
      display: flex;
      flex-direction: column;
    }
    canvas {
      display: block;
      width: 100%;
      height: auto;
      flex: 2;
      touch-action: none;
    }
    .ui-panel {
      background: rgba(10, 10, 30, 0.85);
      backdrop-filter: blur(20px);
      border-top: 2px solid #4a6fff;
      padding: 10px 12px;
      display: flex;
      flex-direction: column;
      gap: 6px;
      color: #ccddff;
      font-size: 14px;
      flex-shrink: 0;
      z-index: 10;
    }
    .stats {
      display: flex;
      justify-content: space-between;
      align-items: center;
      flex-wrap: wrap;
      background: rgba(0, 10, 40, 0.7);
      padding: 8px 12px;
      border-radius: 20px;
    }
    .stat-item {
      display: flex;
      align-items: center;
      gap: 4px;
      font-weight: bold;
    }
    .currency {
      color: #ffd966;
      text-shadow: 0 0 10px #ffaa00;
      font-size: 16px;
    }
    .dark-matter {
      color: #cc88ff;
      text-shadow: 0 0 10px #aa44ff;
    }
    .upgrades {
      display: flex;
      gap: 8px;
      overflow-x: auto;
      padding: 4px 0;
      scrollbar-width: thin;
      scrollbar-color: #4a6fff #0a0a2a;
    }
    .upgrade-btn {
      background: linear-gradient(180deg, #1e2a5a, #0b1030);
      border: 1px solid #4a6fff;
      color: #ccddff;
      padding: 8px 12px;
      border-radius: 16px;
      font-size: 12px;
      font-weight: bold;
      white-space: nowrap;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 4px;
      min-width: 80px;
      transition: all 0.2s;
      box-shadow: 0 4px 8px rgba(0,0,0,0.5);
      cursor: pointer;
    }
    .upgrade-btn:active {
      background: #2a3a7a;
      transform: scale(0.94);
      border-color: #8ab0ff;
    }
    .upgrade-btn.locked {
      opacity: 0.5;
      filter: grayscale(0.5);
      border-color: #4a4a6a;
    }
    .prestige-row {
      display: flex;
      justify-content: space-between;
      align-items: center;
      gap: 10px;
    }
    .prestige-btn {
      background: linear-gradient(180deg, #4a2060, #1e0a30);
      border: 2px solid #cc88ff;
      color: #eaccff;
      padding: 10px 18px;
      border-radius: 24px;
      font-weight: bold;
      font-size: 14px;
      text-shadow: 0 0 8px #b366ff;
      box-shadow: 0 0 15px #5500aa;
      cursor: pointer;
    }
    .prestige-btn:active {
      background: #6a30a0;
    }
    button {
      touch-action: manipulation;
    }
  </style>
</head>
<body>
<div id="game-container">
  <canvas id="gameCanvas"></canvas>
  <div class="ui-panel">
    <div class="stats">
      <span class="stat-item">✨ <span id="stardustDisplay" class="currency">0</span></span>
      <span class="stat-item">🌀/сек <span id="perSecDisplay">0</span></span>
      <span class="stat-item">🌑 <span id="darkMatterDisplay" class="dark-matter">0</span></span>
    </div>
    <div class="upgrades" id="upgradeContainer"></div>
    <div class="prestige-row">
      <span style="font-size:12px;">🌀Престиж: +<span id="dmGain">0</span>🌑</span>
      <button class="prestige-btn" id="prestigeButton">✨ СЖАТЬ ВСЕЛЕННУЮ ✨</button>
    </div>
  </div>
</div>
<script>
  (function() {
    // ---------- ЗВУКОВОЙ ДВИЖОК (WEB AUDIO API) ----------
    const AudioCtx = window.AudioContext || window.webkitAudioContext;
    let audioCtx;
    function initAudio() {
      if (!audioCtx) {
        audioCtx = new AudioCtx();
      }
      if (audioCtx.state === 'suspended') audioCtx.resume();
    }
    function playTone(freq, type = 'sine', duration = 0.09, vol = 0.08) {
      if (!audioCtx) return;
      const t = audioCtx.currentTime;
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      osc.type = type;
      osc.frequency.setValueAtTime(freq, t);
      gain.gain.setValueAtTime(vol, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + duration);
      osc.connect(gain);
      gain.connect(audioCtx.destination);
      osc.start(t);
      osc.stop(t + duration);
    }
    function sfxClick() { playTone(780, 'sine', 0.07, 0.09); playTone(1200, 'triangle', 0.06, 0.05); }
    function sfxBuy() { playTone(500, 'square', 0.1, 0.07); playTone(900, 'sine', 0.08, 0.06); }
    function sfxPrestige() { playTone(200, 'sawtooth', 0.4, 0.12); playTone(600, 'triangle', 0.2, 0.1); }

    // ---------- ОСНОВНЫЕ ПЕРЕМЕННЫЕ ИГРЫ ----------
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    // Данные
    let stardust = 100;          // начальная звёздная пыль
    let totalStardustEarned = 100;
    let darkMatter = 0;         // валюта престижа
    let clickPower = 1;
    let autoCollectors = 0;     // количество авто-сборщиков
    let collectorLevel = 1;     // уровень эффективности сборщиков
    let collectorBaseCost = 25;
    let upgradeClickCost = 50;
    let upgradeCollectorCost = 120;

    // Множители от престижа
    let prestigeMultiplier = 1.0;

    // Обновление UI элементов
    const stardustDisplay = document.getElementById('stardustDisplay');
    const perSecDisplay = document.getElementById('perSecDisplay');
    const darkMatterDisplay = document.getElementById('darkMatterDisplay');
    const dmGainSpan = document.getElementById('dmGain');
    const upgradeContainer = document.getElementById('upgradeContainer');
    const prestigeButton = document.getElementById('prestigeButton');

    // ---------- ВИЗУАЛИЗАЦИЯ ПЛАНЕТЫ И ЗВЁЗД ----------
    let planetRotation = 0;
    let stars = [];
    let particles = [];

    function resizeCanvas() {
      const container = document.getElementById('game-container');
      const rect = container.getBoundingClientRect();
      canvas.width = rect.width * (window.devicePixelRatio || 1);
      canvas.height = (rect.height * 0.55) * (window.devicePixelRatio || 1);
      canvas.style.width = rect.width + 'px';
      canvas.style.height = (rect.height * 0.55) + 'px';
      generateStars();
    }

    function generateStars() {
      stars = [];
      for (let i = 0; i < 180; i++) {
        stars.push({
          x: Math.random() * canvas.width,
          y: Math.random() * canvas.height,
          radius: Math.random() * 2.2 + 0.6,
          brightness: Math.random() * 0.7 + 0.3,
          speed: Math.random() * 0.3 + 0.05
        });
      }
    }

    function createParticles(x, y, count = 14) {
      for (let i = 0; i < count; i++) {
        particles.push({
          x: x, y: y,
          vx: (Math.random() - 0.5) * 5,
          vy: (Math.random() - 0.5) * 5 - 1.5,
          life: 1.0,
          decay: 0.02 + Math.random() * 0.04,
          size: Math.random() * 4 + 2,
          color: `hsl(${50 + Math.random()*30}, 100%, 70%)`
        });
      }
    }

    function drawPlanet(cx, cy, radius) {
      // Псевдо-3D планета с кратерами
      ctx.save();
      ctx.translate(cx, cy);
      ctx.rotate(planetRotation);
      
      // Основной шар
      const gradient = ctx.createRadialGradient(-radius*0.3, -radius*0.3, radius*0.1, 0, 0, radius);
      gradient.addColorStop(0, '#5b9bd5');
      gradient.addColorStop(0.4, '#2e6eb5');
      gradient.addColorStop(0.85, '#0f3b6b');
      gradient.addColorStop(1, '#051a30');
      ctx.fillStyle = gradient;
      ctx.beginPath();
      ctx.arc(0, 0, radius, 0, Math.PI*2);
      ctx.fill();
      
      // Кратеры
      for (let i = 0; i < 16; i++) {
        let angle = (i * 1.8 + 0.7) % (Math.PI*2);
        let dist = radius * (0.5 + (i%3)*0.1);
        let craterX = Math.cos(angle) * dist;
        let craterY = Math.sin(angle) * dist * 0.8;
        ctx.fillStyle = 'rgba(0,20,40,0.5)';
        ctx.beginPath();
        ctx.ellipse(craterX, craterY, radius*0.08, radius*0.06, 0, 0, Math.PI*2);
        ctx.fill();
        ctx.fillStyle = 'rgba(180,210,255,0.25)';
        ctx.beginPath();
        ctx.ellipse(craterX-0.5, craterY-0.8, radius*0.04, radius*0.03, 0, 0, Math.PI*2);
        ctx.fill();
      }
      
      // Атмосфера
      ctx.strokeStyle = 'rgba(140,200,255,0.4)';
      ctx.lineWidth = 2.5;
      ctx.beginPath();
      ctx.arc(0, 0, radius+3, 0, Math.PI*2);
      ctx.stroke();
      ctx.restore();
    }

    function drawStarfield() {
      stars.forEach(s => {
        ctx.fillStyle = `rgba(255,240,200,${s.brightness})`;
        ctx.beginPath();
        ctx.arc(s.x, s.y, s.radius, 0, Math.PI*2);
        ctx.fill();
        // мерцание
        s.y += s.speed * 0.15;
        if (s.y > canvas.height + 5) { s.y = -5; s.x = Math.random() * canvas.width; }
      });
    }

    function drawParticles() {
      for (let i = particles.length-1; i >= 0; i--) {
        let p = particles[i];
        p.x += p.vx;
        p.y += p.vy;
        p.life -= p.decay;
        if (p.life <= 0) {
          particles.splice(i,1);
          continue;
        }
        ctx.fillStyle = p.color.replace('70%)', `${70 + p.life*20}%)`).replace('100%', `${80+p.life*20}%`);
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.size * p.life, 0, 2*Math.PI);
        ctx.fill();
      }
    }

    function updateCanvas() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      drawStarfield();
      
      const planetX = canvas.width * 0.5;
      const planetY = canvas.height * 0.5;
      const planetRadius = Math.min(canvas.width, canvas.height) * 0.28;
      
      drawPlanet(planetX, planetY, planetRadius);
      drawParticles();
      
      // Вращение планеты
      planetRotation += 0.004;
    }

    // ---------- ИГРОВАЯ ЛОГИКА ----------
    function calculatePerSecond() {
      return autoCollectors * collectorLevel * prestigeMultiplier;
    }

    function calculateDarkMatterGain() {
      if (totalStardustEarned < 8000) return 0;
      let base = Math.floor(Math.sqrt(totalStardustEarned / 8000)) ;
      return Math.max(0, base);
    }

    function addStardust(amount) {
      stardust += amount;
      totalStardustEarned += amount;
      updateUI();
    }

    function spendStardust(amount) {
      if (stardust >= amount) {
        stardust -= amount;
        updateUI();
        return true;
      }
      return false;
    }

    function buyAutoCollector() {
      let cost = Math.floor(collectorBaseCost * Math.pow(1.22, autoCollectors));
      if (spendStardust(cost)) {
        autoCollectors++;
        sfxBuy();
        updateUI();
      }
    }

    function upgradeClick() {
      if (spendStardust(upgradeClickCost)) {
        clickPower++;
        upgradeClickCost = Math.floor(upgradeClickCost * 1.45);
        sfxBuy();
        updateUI();
      }
    }

    function upgradeCollector() {
      if (spendStardust(upgradeCollectorCost)) {
        collectorLevel++;
        upgradeCollectorCost = Math.floor(upgradeCollectorCost * 1.6);
        sfxBuy();
        updateUI();
      }
    }

    function performPrestige() {
      let gain = calculateDarkMatterGain();
      if (gain <= 0) return;
      darkMatter += gain;
      prestigeMultiplier = 1 + darkMatter * 0.15;
      
      // Сброс
      stardust = 100;
      totalStardustEarned = 100;
      clickPower = 1;
      autoCollectors = 0;
      collectorLevel = 1;
      collectorBaseCost = 25;
      upgradeClickCost = 50;
      upgradeCollectorCost = 120;
      
      sfxPrestige();
      updateUI();
      saveGame();
    }

    function updateUI() {
      stardustDisplay.textContent = Math.floor(stardust).toLocaleString();
      perSecDisplay.textContent = calculatePerSecond().toFixed(1);
      darkMatterDisplay.textContent = Math.floor(darkMatter);
      let dmGain = calculateDarkMatterGain();
      dmGainSpan.textContent = dmGain;
      
      // Обновление кнопок апгрейдов
      const collectorCost = Math.floor(collectorBaseCost * Math.pow(1.22, autoCollectors));
      document.getElementById('btnCollector').textContent = `🤖Сборщик\n${collectorCost}✨`;
      document.getElementById('btnClick').textContent = `💥Тап\n${upgradeClickCost}✨`;
      document.getElementById('btnCollectorUp').textContent = `⚡Уровень\n${upgradeCollectorCost}✨`;
      
      // Блокировка кнопок по деньгам
      document.getElementById('btnCollector').style.opacity = stardust >= collectorCost ? '1' : '0.5';
      document.getElementById('btnClick').style.opacity = stardust >= upgradeClickCost ? '1' : '0.5';
      document.getElementById('btnCollectorUp').style.opacity = stardust >= upgradeCollectorCost ? '1' : '0.5';
      
      prestigeButton.style.opacity = dmGain > 0 ? '1' : '0.4';
    }

    // ---------- СОХРАНЕНИЕ / ЗАГРУЗКА ----------
    function saveGame() {
      const save = {
        stardust, totalStardustEarned, darkMatter, clickPower,
        autoCollectors, collectorLevel, collectorBaseCost,
        upgradeClickCost, upgradeCollectorCost, prestigeMultiplier
      };
      localStorage.setItem('cosmicSimMax', JSON.stringify(save));
    }

    function loadGame() {
      const raw = localStorage.getItem('cosmicSimMax');
      if (raw) {
        try {
          const s = JSON.parse(raw);
          stardust = s.stardust || 100;
          totalStardustEarned = s.totalStardustEarned || 100;
          darkMatter = s.darkMatter || 0;
          clickPower = s.clickPower || 1;
          autoCollectors = s.autoCollectors || 0;
          collectorLevel = s.collectorLevel || 1;
          collectorBaseCost = s.collectorBaseCost || 25;
          upgradeClickCost = s.upgradeClickCost || 50;
          upgradeCollectorCost = s.upgradeCollectorCost || 120;
          prestigeMultiplier = s.prestigeMultiplier || 1.0;
        } catch(e){}
      }
      updateUI();
    }

    // ---------- ОБРАБОТЧИКИ СОБЫТИЙ ----------
    function handleCanvasTap(e) {
      e.preventDefault();
      initAudio();
      const rect = canvas.getBoundingClientRect();
      const scaleX = canvas.width / rect.width;
      const scaleY = canvas.height / rect.height;
      
      let clientX, clientY;
      if (e.touches) {
        clientX = e.touches[0].clientX;
        clientY = e.touches[0].clientY;
      } else {
        clientX = e.clientX;
        clientY = e.clientY;
      }
      
      const canvasX = (clientX - rect.left) * scaleX;
      const canvasY = (clientY - rect.top) * scaleY;
      
      const planetX = canvas.width * 0.5;
      const planetY = canvas.height * 0.5;
      const planetRadius = Math.min(canvas.width, canvas.height) * 0.28;
      
      const dist = Math.hypot(canvasX - planetX, canvasY - planetY);
      if (dist < planetRadius + 10) {
        // Попадание по планете
        addStardust(clickPower * prestigeMultiplier);
        createParticles(canvasX, canvasY, 12);
        sfxClick();
      }
    }

    // ---------- ИНИЦИАЛИЗАЦИЯ UI КНОПОК ----------
    function buildUpgradeButtons() {
      upgradeContainer.innerHTML = `
        <button class="upgrade-btn" id="btnCollector">🤖Сборщик<br>25✨</button>
        <button class="upgrade-btn" id="btnClick">💥Тап<br>50✨</button>
        <button class="upgrade-btn" id="btnCollectorUp">⚡Уровень<br>120✨</button>
      `;
      document.getElementById('btnCollector').addEventListener('click', buyAutoCollector);
      document.getElementById('btnClick').addEventListener('click', upgradeClick);
      document.getElementById('btnCollectorUp').addEventListener('click', upgradeCollector);
    }

    // ---------- ИГРОВОЙ ЦИКЛ ----------
    let lastTimestamp = 0;
    function gameLoop(ts) {
      if (!lastTimestamp) lastTimestamp = ts;
      const delta = Math.min(0.1, (ts - lastTimestamp) / 1000);
      if (delta > 0) {
        // Авто-сбор
        if (autoCollectors > 0) {
          addStardust(calculatePerSecond() * delta);
        }
        updateCanvas();
      }
      lastTimestamp = ts;
      requestAnimationFrame(gameLoop);
    }

    // Автосохранение раз в 5 секунд
    setInterval(saveGame, 5000);

    // События
    canvas.addEventListener('touchstart', handleCanvasTap, {passive: false});
    canvas.addEventListener('mousedown', handleCanvasTap);
    prestigeButton.addEventListener('click', performPrestige);
    
    window.addEventListener('resize', () => {
      resizeCanvas();
      updateUI();
    });

    // Старт
    resizeCanvas();
    buildUpgradeButtons();
    loadGame();
    updateUI();
    requestAnimationFrame(gameLoop);
  })();
</script>
</body>
</html>
