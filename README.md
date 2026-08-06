# first
第一个仓库
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <title>Canvas 烟花效果</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <style>
    /* 全屏画布、无滚动条、背景为夜空色 */
    html, body {
      height: 100%;
      margin: 0;
      background: radial-gradient(ellipse at center, #0b1630 0%, #030414 70%);
      overflow: hidden;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial;
    }
    canvas { display: block; }
    /* 右上角控制面板（可选）*/
    .ui {
      position: absolute;
      right: 12px;
      top: 12px;
      color: #fff;
      background: rgba(0,0,0,0.25);
      padding: 8px 12px;
      border-radius: 8px;
      font-size: 13px;
      backdrop-filter: blur(4px);
    }
    .ui label { display:inline-block; margin-right:8px; }
  </style>
</head>
<body>
  <canvas id="c"></canvas>
  <div class="ui">
    <label>自动发射<input id="auto" type="checkbox" checked></label>
    <label>强度 <input id="intensity" type="range" min="1" max="6" value="3"></label>
  </div>

  <script>
  // Canvas 烟花效果
  (() => {
    const canvas = document.getElementById('c');
    const ctx = canvas.getContext('2d', { alpha: true });
    let W = 0, H = 0, DPR = Math.max(1, window.devicePixelRatio || 1);

    function resize(){
      W = window.innerWidth;
      H = window.innerHeight;
      canvas.width = Math.floor(W * DPR);
      canvas.height = Math.floor(H * DPR);
      canvas.style.width = W + 'px';
      canvas.style.height = H + 'px';
      ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
    }
    window.addEventListener('resize', resize);
    resize();

    // 工具函数
    function rand(min, max){ return Math.random() * (max - min) + min; }
    function randInt(min, max){ return Math.floor(rand(min, max+1)); }
    function hsla(h, s, l, a){ return `hsla(${h}, ${s}%, ${l}%, ${a})`; }

    // 粒子类（烟花爆炸后的彩色粒子）
    class Particle {
      constructor(x, y, colorHue, speed, angle, life, size){
        this.x = x; this.y = y;
        this.vx = Math.cos(angle) * speed;
        this.vy = Math.sin(angle) * speed;
        this.alpha = 1;
        this.decay = rand(0.008, 0.018);
        this.size = size || rand(1.2, 3.2);
        this.h = colorHue + rand(-16, 16);
        this.s = rand(60, 90);
        this.l = rand(45, 60);
        this.gravity = 0.03;
        this.airFriction = 0.995;
      }
      update(){
        this.vx *= this.airFriction;
        this.vy *= this.airFriction;
        this.vy += this.gravity;
        this.x += this.vx;
        this.y += this.vy;
        this.alpha -= this.decay;
      }
      draw(ctx){
        if (this.alpha <= 0) return;
        ctx.save();
        ctx.globalCompositeOperation = 'lighter';
        ctx.globalAlpha = Math.max(0, this.alpha);
        const g = ctx.createRadialGradient(this.x, this.y, 0, this.x, this.y, this.size*5);
        g.addColorStop(0, hsla(this.h, this.s, this.l, this.alpha));
        g.addColorStop(0.35, hsla(this.h, this.s, this.l, this.alpha*0.6));
        g.addColorStop(1, hsla(this.h, this.s, this.l, 0));
        ctx.fillStyle = g;
        ctx.beginPath();
        ctx.arc(this.x, this.y, this.size*2, 0, Math.PI*2);
        ctx.fill();
        ctx.restore();
      }
    }

    // 火箭类（从底部上升的主体，达到顶端后爆炸成粒子）
    class Rocket {
      constructor(x, y, tx, ty, hue){
        this.x = x; this.y = y;
        this.tx = tx; this.ty = ty;
        this.h = hue;
        this.vx = (tx - x) * 0.02 + rand(-0.5, 0.5);
        this.vy = (ty - y) * 0.02 + rand(-7, -5);
        this.trail = [];
        this.dead = false;
      }
      update(){
        // 记录轨迹用于尾迹绘制
        this.trail.push({ x: this.x, y: this.y });
        if (this.trail.length > 8) this.trail.shift();

        // 简单物理：上升并受重力与空气阻力影响
        this.vy += 0.12; // gravity pulling downwards (positive vy is down)
        this.vx *= 0.999;
        this.vy *= 0.999;
        this.x += this.vx;
        this.y += this.vy;

        // 到达目标或速度方向变向时爆炸
        if (this.y >= this.ty || this.vy >= 0) {
          this.dead = true;
          this.explode();
        }
      }
      explode(){
        // 爆炸产生粒子——由外部管理器接收
        spawnExplosion(this.x, this.y, this.h);
      }
      draw(ctx){
        ctx.save();
        // 火箭尾迹
        for (let i = 0; i < this.trail.length; i++){
          const p = this.trail[i];
          const t = i / this.trail.length;
          ctx.fillStyle = hsla(this.h, 80, 55, 0.6 * t);
          ctx.beginPath();
          ctx.arc(p.x, p.y, (1 + (1 - t) * 2), 0, Math.PI*2);
          ctx.fill();
        }
        // 火箭头（亮点）
        ctx.fillStyle = hsla(this.h, 90, 60, 1);
        ctx.beginPath();
        ctx.arc(this.x, this.y, 3.5, 0, Math.PI*2);
        ctx.fill();
        ctx.restore();
      }
    }

    // 管理所有火箭和粒子
    const rockets = [];
    const particles = [];

    function spawnRocket(x, tx, ty){
      const startY = H + 10;
      const hue = randInt(0, 360);
      rockets.push(new Rocket(x, startY, tx, ty, hue));
    }

    // 当火箭爆炸时调用，产生大量粒子
    function spawnExplosion(x, y, hue){
      const count = Math.floor(rand(20, 120) * intensity);
      for (let i = 0; i < count; i++){
        const angle = rand(0, Math.PI*2);
        const speed = rand(1.6, 6.8) * (0.6 + Math.random()*0.8);
        const p = new Particle(x, y, hue + rand(-8,8), speed, angle);
        particles.push(p);
      }
      // 可选：同时产生少量“闪光”大粒子
      for (let i=0; i<Math.min(6, Math.floor(count/20)); i++){
        const a = rand(0, Math.PI*2), s = rand(2.4, 5.6);
        const big = new Particle(x, y, hue, s*1.2, a);
        big.size = rand(3.5, 6);
        big.decay = rand(0.01, 0.03);
        particles.push(big);
      }
    }

    // 背景淡化，用以形成拖影效果
    function fadeBackground(){
      ctx.fillStyle = 'rgba(1, 6, 20, 0.18)';
      ctx.fillRect(0, 0, W, H);
      // 添加一些微弱星点
      // (每帧不必都画，保持性能)
    }

    // 主循环
    let last = performance.now();
    let autoLaunchTimer = 0;
    let autoLaunchInterval = 600; // ms
    let intensity = 1.0;

    function frame(now){
      const dt = now - last;
      last = now;

      // 画半透明背景制造拖影
      fadeBackground();

      // 更新并绘制火箭
      for (let i = rockets.length - 1; i >= 0; i--){
        const r = rockets[i];
        r.update();
        r.draw(ctx);
        if (r.dead) rockets.splice(i, 1);
      }

      // 更新并绘制粒子
      for (let i = particles.length - 1; i >= 0; i--){
        const p = particles[i];
        p.update();
        p.draw(ctx);
        if (p.alpha <= 0.02 || p.y > H + 50) {
          particles.splice(i, 1);
        }
      }

      // 自动发射逻辑
      if (autoCheckbox.checked) {
        autoLaunchTimer += dt;
        if (autoLaunchTimer >= autoLaunchInterval) {
          autoLaunchTimer = 0;
          // 在随机位置发射若干火箭，数量和强度有关
          const n = Math.max(1, Math.round(rand(1, 3) * intensity));
          for (let i=0; i<n; i++){
            const tx = rand(W*0.15, W*0.85);
            const ty = rand(H*0.12, H*0.45);
            spawnRocket(rand(W*0.1, W*0.9), tx, ty);
          }
        }
      }

      requestAnimationFrame(frame);
    }
    requestAnimationFrame(frame);

    // 鼠标 / 触控交互：点击产生火箭飞向点击位置
    function onPointer(e){
      const rect = canvas.getBoundingClientRect();
      const x = (e.clientX || (e.touches && e.touches[0].clientX)) - rect.left;
      const y = (e.clientY || (e.touches && e.touches[0].clientY)) - rect.top;
      // 发射若干火箭从下方向点击点飞去
      const num = Math.max(1, Math.round(rand(1, 2) * intensity));
      for (let i = 0; i < num; i++){
        const sx = x + rand(-40, 40);
        spawnRocket(sx, x + rand(-20, 20), Math.max(30, y + rand(-20, 20)));
      }
    }
    canvas.addEventListener('pointerdown', onPointer, { passive: true });
    canvas.addEventListener('touchstart', onPointer, { passive: true });

    // UI 控件
    const autoCheckbox = document.getElementById('auto');
    const intensityRange = document.getElementById('intensity');
    intensityRange.addEventListener('input', () => {
      intensity = intensityRange.value / 3; // 将 1..6 映射为 0.33..2
      autoLaunchInterval = Math.max(220, 1200 - intensityRange.value * 160);
    });

    // 初始自动发射参数
    intensity = intensityRange.value / 3;
    autoLaunchInterval = Math.max(220, 1200 - intensityRange.value * 160);

    // 在页面加载时，绘制一些星点作为背景装饰（一次性）
    function drawStars(){
      const starCount = Math.floor((W * H) / 6000);
      for (let i=0; i<starCount; i++){
        const x = Math.random() * W;
        const y = Math.random() * H * 0.6;
        const r = Math.random() * 1.4;
        ctx.beginPath();
        ctx.fillStyle = `rgba(255,255,255,${Math.random()*0.6})`;
        ctx.arc(x, y, r, 0, Math.PI*2);
        ctx.fill();
      }
    }
    // 调整大小后重绘星星
    window.addEventListener('load', () => {
      drawStars();
    });
    window.addEventListener('resize', () => {
      // 画布重置会清空星星，这里简单重新绘制（不影响动画）
      drawStars();
    });

    // 方便外部调试/调用：导出少量方法到 window
    window.__fireworks = { spawnRocket, spawnExplosion, rockets, particles };

    // 小说明（在控制台显示）
    console.log('烟花已启动：点击屏幕发送烟花，或打开/关闭右上角的自动发射。');
  })();
  </script>
</body>
</html>
