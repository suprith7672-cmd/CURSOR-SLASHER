<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Cursor Slayer: Gritty Battlefield</title>
    <style>
        body { margin: 0; overflow: hidden; background: #0c0f12; font-family: 'Segoe UI', sans-serif; color: white; }
        canvas { display: block; }
        #overlay { 
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            display: flex; flex-direction: column; justify-content: center; align-items: center;
            background: rgba(12, 15, 18, 0.96); z-index: 10;
        }
        .menu-box { text-align: center; background: #161a22; padding: 30px; border: 2px solid #2d3748; border-radius: 15px; box-shadow: 0 0 30px rgba(0,0,0,0.8); }
        .char-grid { display: grid; grid-template-columns: repeat(3, 160px); gap: 12px; margin: 20px 0; }
        .char-card { padding: 15px; border: 1px solid #2d3748; cursor: pointer; border-radius: 8px; background: #1a202c; transition: 0.2s; font-size: 13px; }
        .char-card:hover { border-color: #e53e3e; transform: translateY(-2px); }
        .char-card.selected { border-color: #e53e3e; box-shadow: 0 0 15px rgba(229,62,62,0.3); background: #2d3748; }
        .btn { padding: 12px 35px; font-size: 18px; cursor: pointer; background: transparent; border: 2px solid #e53e3e; color: #e53e3e; text-transform: uppercase; letter-spacing: 2px; border-radius: 5px; font-weight: bold; }
        .btn:hover { background: #e53e3e; color: #0c0f12; box-shadow: 0 0 15px #e53e3e; }
        #ui { position: absolute; top: 20px; left: 20px; z-index: 5; display: none; pointer-events: none; }
        #cooldown-bar { width: 200px; height: 6px; background: #1a202c; border-radius: 3px; overflow: hidden; margin-top: 5px; border: 1px solid #2d3748; }
        #cooldown-fill { width: 100%; height: 100%; background: #e53e3e; }
        #health-container { display: flex; gap: 5px; margin-top: 8px; }
        .health-pip { width: 40px; height: 8px; background: #38a169; box-shadow: 0 0 8px #38a169; border-radius: 2px; transition: 0.3s; }
        .health-pip.lost { background: #4a1525; box-shadow: none; border: 1px solid #742a3a; }
    </style>
</head>
<body>

    <div id="overlay">
        <div class="menu-box">
            <h1 style="color: #e53e3e; font-size: 36px; margin-top: 0; letter-spacing: 3px; text-shadow: 0 0 15px rgba(229,62,62,0.4);">HERO SLAYER</h1>
            <div class="char-grid" id="charGrid">
                <div class="char-card selected" onclick="selectChar('casual', this)"><b style="color:cyan">CASUAL</b><br><small>Sleek Jet Dart</small></div>
                <div class="char-card" onclick="selectChar('titan', this)"><b style="color:#a855f7">TITAN</b><br><small>Iron Bulwark</small></div>
                <div class="char-card" onclick="selectChar('soldier', this)"><b style="color:#fbbf24">SOLDIER</b><br><small>Heavy Chevron</small></div>
                <div class="char-card" onclick="selectChar('warlock', this)"><b style="color:#22c55e">WARLOCK</b><br><small>Astral Star</small></div>
                <div class="char-card" onclick="selectChar('ghost', this)"><b style="color:#e2e8f0">GHOST</b><br><small>Phantom Needle</small></div>
                <div class="char-card" onclick="selectChar('reaper', this)"><b style="color:#ef4444">REAPER</b><br><small>Flying Scythe Waves</small></div>
            </div>
            <button class="btn" onclick="startGame()">START SLAYING</button>
        </div>
    </div>

    <div id="ui">
        <div id="char-name" style="font-weight:bold; font-size: 24px; letter-spacing: 1px;">CLASS</div>
        <div id="kill-count" style="font-size: 18px; color: #a0aec0; margin-top: 2px;">Kills: 0</div>
        
        <div id="health-container">
            <div class="health-pip" id="hp1"></div>
            <div class="health-pip" id="hp2"></div>
            <div class="health-pip" id="hp3"></div>
        </div>

        <div id="cooldown-bar" style="margin-top: 10px;"><div id="cooldown-fill"></div></div>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const cooldownFill = document.getElementById('cooldown-fill');

        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        let score = 0, gameActive = false, selectedClass = 'casual';
        let mousePos = { x: canvas.width / 2, y: canvas.height / 2 }, isMouseDown = false, shakeTime = 0;
        let snapLookTimer = 0, reaperKillTracker = 0; 
        const keys = {};

        const classes = {
            casual: { color: 'cyan', atkSize: 5, atkSpeed: 16, atkLife: 60, cd: 3000, fireRate: 100, duration: 250 },
            titan: { color: '#a855f7', atkSize: 35, atkSpeed: 7, atkLife: 10, cd: 10000, fireRate: 250, duration: 10000 },
            soldier: { color: '#fbbf24', atkSize: 4, atkSpeed: 22, atkLife: 50, cd: 7000, fireRate: 40, duration: 300 },
            warlock: { color: '#22c55e', atkSize: 14, atkSpeed: 6, atkLife: 100, cd: 8000, fireRate: 300, duration: 400 },
            ghost: { color: '#e2e8f0', atkSize: 3, atkSpeed: 24, atkLife: 40, cd: 12000, fireRate: 130, duration: 5000 },
            reaper: { color: '#ef4444', atkSize: 40, atkSpeed: 12, atkLife: 45, cd: 4000, fireRate: 160, duration: 250 }
        };

        const player = { 
            x: 1500, y: 1500, size: 24, angle: 0, state: 'normal', canAbility: true, lastFire: 0,
            health: 3, invincibleFrames: 0
        };
        
        let bullets = [], zombies = [], trail = [], visualEffects = [], particles = [];
        let bloodDecals = []; // Permanent floor stains
        let backgroundCracks = []; // Map noise

        // Generate static battlefield environmental details once
        for(let i=0; i<60; i++) {
            backgroundCracks.push({
                x: Math.random() * 3000,
                y: Math.random() * 3000,
                size: 20 + Math.random() * 40,
                segments: 3 + Math.floor(Math.random() * 3)
            });
        }

        window.addEventListener('mousemove', e => { mousePos = { x: e.clientX, y: e.clientY }; });
        window.addEventListener('mousedown', () => isMouseDown = true);
        window.addEventListener('mouseup', () => isMouseDown = false);
        window.addEventListener('keydown', e => { keys[e.key.toLowerCase()] = true; if(e.code === 'Space' && player.canAbility && gameActive) useAbility(); });
        window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

        function selectChar(c, el) {
            selectedClass = c;
            Array.from(document.getElementById('charGrid').children).forEach(child => child.classList.remove('selected'));
            el.classList.add('selected');
        }

        function startGame() {
            gameActive = true;
            document.getElementById('overlay').style.display = 'none';
            document.getElementById('ui').style.display = 'block';
            document.getElementById('char-name').innerText = selectedClass.toUpperCase();
            document.getElementById('char-name').style.color = classes[selectedClass].color;
            cooldownFill.style.backgroundColor = classes[selectedClass].color;
            updateHealthUI();
        }

        function spawnBullet(isMissile = false) {
            const c = classes[selectedClass];
            const aimX = mousePos.x - canvas.width/2 + player.x;
            const aimY = mousePos.y - canvas.height/2 + player.y;
            const angle = Math.atan2(aimY - player.y, aimX - player.x);
            
            if (selectedClass === 'reaper') {
                bullets.push({ x: player.x, y: player.y, vx: Math.cos(angle) * c.atkSpeed, vy: Math.sin(angle) * c.atkSpeed, life: c.atkLife, size: c.atkSize, color: '#ef4444', isReaperWave: true, angle: angle, pierceCount: 4 });
            } else {
                bullets.push({ x: player.x, y: player.y, vx: Math.cos(angle)* (isMissile ? 12:c.atkSpeed), vy: Math.sin(angle)*(isMissile ? 12:c.atkSpeed), life: 200, size: isMissile ? 18:c.atkSize, color: isMissile ? '#ff4500':c.color, isMissile });
            }
            player.angle = angle; snapLookTimer = 10; 
        }

        function useAbility() {
            const c = classes[selectedClass];
            player.canAbility = false; player.state = 'active';
            cooldownFill.style.width = '0%';
            
            if (selectedClass === 'soldier') spawnBullet(true);
            if (selectedClass === 'warlock') {
                shakeTime = 15;
                visualEffects.push({x: player.x, y: player.y, r: 500, life: 25, color: 'rgba(34, 197, 94, 0.15)'});
                zombies = zombies.filter(z => {
                    if(Math.hypot(z.x-player.x, z.y-player.y) < 500) { handleKill(z.x, z.y); return false; }
                    return true;
                });
            }
            setTimeout(() => { 
                player.state = 'normal'; 
                if(selectedClass === 'reaper') {
                    shakeTime = 15;
                    visualEffects.push({x: player.x, y: player.y, r: 250, life: 20, color: 'rgba(239, 64, 64, 0.25)'});
                    zombies = zombies.filter(z => {
                        if(Math.hypot(z.x-player.x, z.y-player.y) < 250) { handleKill(z.x, z.y); return false; }
                        return true;
                    });
                }
            }, c.duration);
            setTimeout(() => { player.canAbility = true; cooldownFill.style.width = '100%'; }, c.cd);
        }

        function updateHealthUI() {
            for(let i=1; i<=3; i++) {
                const pip = document.getElementById(`hp${i}`);
                if (i <= player.health) { pip.classList.remove('lost'); } else { pip.classList.add('lost'); }
            }
        }

        function createBloodSpray(x, y) {
            // Add static blood splash decal onto map floor history
            bloodDecals.push({ x, y, r: 15 + Math.random()*20, opacity: 0.4 + Math.random()*0.3 });
            if(bloodDecals.length > 300) bloodDecals.shift(); // Prevent memory overflow

            for(let i=0; i<8; i++) {
                const ang = Math.random() * Math.PI * 2;
                const s = 2 + Math.random() * 6;
                particles.push({ x, y, vx: Math.cos(ang)*s, vy: Math.sin(ang)*s, life: 1.0, color: '#9b2c2c', size: Math.random()*4 + 1.5 });
            }
        }

        function handleKill(zx, zy) {
            score++;
            createBloodSpray(zx, zy);
            if (selectedClass === 'reaper') {
                reaperKillTracker++;
                if (reaperKillTracker >= 50) {
                    reaperKillTracker = 0;
                    if (player.health < 3) { player.health++; updateHealthUI(); visualEffects.push({x: player.x, y: player.y, r: 100, life: 15, color: 'rgba(56, 161, 105, 0.25)'}); }
                }
            }
        }

        function update() {
            if (!gameActive) return;
            if (shakeTime > 0) shakeTime--;
            if (snapLookTimer > 0) snapLookTimer--;
            if (player.invincibleFrames > 0) player.invincibleFrames--;

            let mx = 0, my = 0;
            if (keys['w']) my -= 1; if (keys['s']) my += 1;
            if (keys['a']) mx -= 1; if (keys['d']) mx += 1;

            if (player.state === 'active' && (selectedClass === 'casual' || selectedClass === 'reaper')) {
                const screenCenterX = canvas.width / 2; const screenCenterY = canvas.height / 2;
                player.angle = Math.atan2(mousePos.y - screenCenterY, mousePos.x - screenCenterX);
                player.x += Math.cos(player.angle) * 22; player.y += Math.sin(player.angle) * 22;
                trail.push({x: player.x, y: player.y, angle: player.angle, life: 8});
            } else {
                player.x += mx * 6; player.y += my * 6;
                if ((mx !== 0 || my !== 0) && snapLookTimer <= 0) { player.angle = Math.atan2(my, mx); }
            }

            // Lock boundaries within map dimensions
            player.x = Math.max(30, Math.min(2970, player.x));
            player.y = Math.max(30, Math.min(2970, player.y));

            let now = Date.now();
            if (isMouseDown && now - player.lastFire > classes[selectedClass].fireRate) { 
                if (!(selectedClass === 'soldier' && player.state === 'active')) { spawnBullet(); player.lastFire = now; }
            }

            bullets = bullets.filter(b => {
                b.x += b.vx; b.y += b.vy; b.life--;
                let bDestroyed = false;
                for(let i = zombies.length - 1; i >= 0; i--) {
                    let z = zombies[i];
                    if(Math.hypot(z.x-b.x, z.y-b.y) < z.size + b.size) {
                        if (b.isReaperWave) {
                            handleKill(z.x, z.y); zombies.splice(i, 1); b.pierceCount--;
                            if(b.pierceCount <= 0) { bDestroyed = true; break; }
                        } else if(b.isMissile) {
                            bDestroyed = true; shakeTime = 30;
                            visualEffects.push({x: b.x, y: b.y, r: 500, life: 30, color: 'rgba(229, 62, 62, 0.25)'});
                            zombies = zombies.filter(z2 => {
                                if(Math.hypot(z2.x-b.x, z2.y-b.y) < 500) { handleKill(z2.x, z2.y); return false; }
                                return true;
                            });
                            break;
                        } else { bDestroyed = true; handleKill(z.x, z.y); zombies.splice(i, 1); break; }
                    }
                }
                return !bDestroyed && b.life > 0;
            });

            zombies.forEach((z, zi) => {
                if (!(selectedClass === 'ghost' && player.state === 'active')) {
                    z.angle = Math.atan2(player.y-z.y, player.x-z.x);
                    z.x += Math.cos(z.angle) * 2.8; z.y += Math.sin(z.angle) * 2.8;
                }
                if(Math.hypot(z.x-player.x, z.y-player.y) < z.size + 15) {
                    if (player.state === 'active' && (selectedClass === 'casual' || selectedClass === 'reaper')) {
                        zombies.splice(zi, 1); handleKill(z.x, z.y);
                    } else if (!(selectedClass === 'titan' && player.state === 'active')) {
                        if (player.invincibleFrames <= 0) {
                            player.health--; player.invincibleFrames = 60; shakeTime = 15; updateHealthUI();
                            if (player.health <= 0) { location.reload(); }
                        }
                    }
                }
            });

            particles.forEach((p, i) => { p.x += p.vx; p.y += p.vy; p.life -= 0.04; if(p.life <= 0) particles.splice(i,1); });
            visualEffects.forEach((e, i) => { e.life--; if(e.life <= 0) visualEffects.splice(i, 1); });
        }

        function drawCustomCharacter(type, col, sz, isGhostMode) {
            ctx.fillStyle = col; ctx.strokeStyle = '#ffffff'; ctx.lineWidth = 1.5;
            if (isGhostMode) ctx.globalAlpha = 0.25;
            ctx.beginPath();
            if (type === 'casual') {
                ctx.moveTo(sz, 0); ctx.lineTo(-sz, -sz * 0.7); ctx.lineTo(-sz * 0.4, 0); ctx.lineTo(-sz, sz * 0.7);
            } else if (type === 'titan') {
                ctx.moveTo(sz * 1.2, 0); ctx.lineTo(0, -sz * 0.9); ctx.lineTo(-sz * 0.8, -sz * 0.4); ctx.lineTo(-sz * 0.8, sz * 0.4); ctx.lineTo(0, sz * 0.9);
            } else if (type === 'soldier') {
                ctx.moveTo(sz, 0); ctx.lineTo(-sz * 0.3, -sz); ctx.lineTo(-sz, -sz * 0.6); ctx.lineTo(-sz * 0.5, 0); ctx.lineTo(-sz, sz * 0.6); ctx.lineTo(-sz * 0.3, sz);
            } else if (type === 'warlock') {
                ctx.moveTo(sz * 1.3, 0); ctx.lineTo(-sz * 0.2, -sz * 0.4); ctx.lineTo(-sz * 0.8, -sz * 1.1); ctx.lineTo(-sz * 0.5, 0); ctx.lineTo(-sz * 0.8, sz * 1.1); ctx.lineTo(-sz * 0.2, sz * 0.4);
            } else if (type === 'ghost') {
                ctx.moveTo(sz * 1.4, 0); ctx.lineTo(-sz, -sz * 0.3); ctx.lineTo(-sz * 0.6, 0); ctx.lineTo(-sz, sz * 0.3);
            } else if (type === 'reaper') {
                ctx.moveTo(sz, 0); ctx.lineTo(-sz * 0.5, -sz * 1.2); ctx.lineTo(-sz * 0.3, -sz * 0.4); ctx.lineTo(-sz, 0); ctx.lineTo(-sz * 0.3, sz * 0.4); ctx.lineTo(-sz * 0.5, sz * 1.2);
            }
            ctx.closePath(); ctx.fill(); ctx.stroke(); ctx.globalAlpha = 1.0;
        }

        function draw() {
            ctx.setTransform(1,0,0,1,0,0);
            ctx.fillStyle = "#12161a"; ctx.fillRect(0,0,canvas.width,canvas.height); // Smooth asphalt hue
            
            let sx = (Math.random()-0.5) * shakeTime; let sy = (Math.random()-0.5) * shakeTime;
            const camX = -player.x + canvas.width / 2;
            const camY = -player.y + canvas.height / 2;
            ctx.translate(camX + sx, camY + sy);

            // DRAW BATTLEFIELD FLOOR DETAILS (No neon grid grids)
            ctx.fillStyle = "#171d24";
            ctx.fillRect(0, 0, 3000, 3000);

            // Distant Concrete Wall Border Lines
            ctx.strokeStyle = "#4a1515"; ctx.lineWidth = 8;
            ctx.strokeRect(0, 0, 3000, 3000);

            // Ground Cracks
            ctx.strokeStyle = "#1b222a"; ctx.lineWidth = 2;
            backgroundCracks.forEach(c => {
                ctx.beginPath(); ctx.moveTo(c.x, c.y);
                for(let j=0; j<c.segments; j++) {
                    ctx.lineTo(c.x + (Math.random()-0.5)*c.size, c.y + (Math.random()-0.5)*c.size);
                }
                ctx.stroke();
            });

            // Permanent Blood Splatters
            bloodDecals.forEach(d => {
                ctx.fillStyle = `rgba(106, 25, 25, ${d.opacity})`;
                ctx.beginPath(); ctx.arc(d.x, d.y, d.r, 0, Math.PI*2); ctx.fill();
            });

            visualEffects.forEach(e => { ctx.fillStyle = e.color; ctx.beginPath(); ctx.arc(e.x, e.y, e.r, 0, Math.PI*2); ctx.fill(); });
            particles.forEach(p => { ctx.fillStyle = `rgba(155, 44, 44, ${p.life})`; ctx.fillRect(p.x, p.y, p.size, p.size); });
            
            bullets.forEach(b => {
                if (b.isReaperWave) {
                    ctx.save(); ctx.strokeStyle = '#ef4444'; ctx.lineWidth = 5; ctx.shadowBlur = 15; ctx.shadowColor = '#ef4444';
                    ctx.beginPath(); ctx.arc(b.x, b.y, b.size, b.angle - Math.PI / 4, b.angle + Math.PI / 4); ctx.stroke(); ctx.restore();
                } else {
                    ctx.fillStyle = b.color; ctx.save(); ctx.shadowBlur = b.isMissile ? 25 : 8; ctx.shadowColor = b.color;
                    ctx.beginPath(); ctx.arc(b.x, b.y, b.size, 0, Math.PI*2); ctx.fill(); ctx.restore();
                }
            });
            
            // Fleshy-textured Zombie designs
            zombies.forEach(z => {
                ctx.save(); ctx.translate(z.x, z.y); ctx.rotate(z.angle);
                ctx.fillStyle = "#2d3748"; ctx.strokeStyle = "#9b2c2c"; ctx.lineWidth = 2;
                ctx.beginPath(); ctx.moveTo(22,0); ctx.lineTo(-12,-16); ctx.lineTo(-12,16); ctx.closePath(); ctx.fill(); ctx.stroke(); 
                ctx.fillStyle = "#e53e3e"; ctx.fillRect(6, -6, 3, 2); ctx.fillRect(6, 4, 3, 2); 
                ctx.restore();
            });

            if(selectedClass === 'titan' && player.state === 'active') {
                ctx.beginPath(); ctx.arc(player.x, player.y, 75, 0, Math.PI*2);
                ctx.strokeStyle = "rgba(168, 85, 247, 0.4)"; ctx.lineWidth = 6; ctx.stroke();
            }

            trail.forEach((t, i) => {
                ctx.save(); ctx.translate(t.x, t.y); ctx.rotate(t.angle);
                drawCustomCharacter(selectedClass, selectedClass === 'reaper' ? `rgba(239, 68, 68, ${t.life/10})` : `rgba(0, 255, 204, ${t.life/10})`, player.size, false);
                ctx.restore();
                t.life--; if(t.life <= 0) trail.splice(i, 1);
            });

            if (player.invincibleFrames > 0 && Math.floor(player.invincibleFrames / 4) % 2 === 0) {} 
            else {
                ctx.save(); ctx.translate(player.x, player.y); ctx.rotate(player.angle);
                const isGhostInvisible = (selectedClass === 'ghost' && player.state === 'active');
                drawCustomCharacter(selectedClass, classes[selectedClass].color, player.size, isGhostInvisible);
                ctx.restore();
            }

            document.getElementById('kill-count').innerText = "Kills: " + score;
            update(); requestAnimationFrame(draw);
        }

        window.addEventListener('resize', () => { canvas.width = window.innerWidth; canvas.height = window.innerHeight; });

        setInterval(() => { 
            if (gameActive) {
                const spawnAngle = Math.random() * Math.PI * 2;
                const safeDistance = 700 + Math.random() * 300; 
                zombies.push({ x: player.x + Math.cos(spawnAngle) * safeDistance, y: player.y + Math.sin(spawnAngle) * safeDistance, size: 20 });
            } 
        }, 650);

        draw();
    </script>
</body>
</html>
