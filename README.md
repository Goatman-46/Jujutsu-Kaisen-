<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>PIXEL KAIZEN | JJK AI EDITION</title>
    <style>
        :root { --blue: #00d4ff; --red: #ff2e2e; --purple: #bc00ff; }
        body { margin: 0; background: #080808; color: white; font-family: 'Arial Black', sans-serif; overflow: hidden; touch-action: none; }
        
        #game-container { position: relative; width: 800px; height: 400px; margin: 20px auto; outline: 4px solid #222; overflow: hidden; }
        canvas { background: linear-gradient(to bottom, #111, #222); display: block; }

        /* UI Overlays */
        .hud { position: absolute; top: 15px; width: 100%; display: flex; justify-content: space-between; padding: 0 30px; box-sizing: border-box; pointer-events: none; }
        .stat-group { width: 320px; }
        .label { font-size: 12px; letter-spacing: 2px; margin-bottom: 5px; text-shadow: 2px 2px black; }
        .bar-bg { background: #000; height: 22px; border: 2px solid #fff; position: relative; box-shadow: 0 0 10px rgba(0,0,0,0.5); }
        .hp-fill { background: linear-gradient(90deg, var(--red), #ff7b7b); height: 100%; width: 100%; transition: width 0.1s; }
        .ce-bg { background: #000; height: 8px; margin-top: 6px; border: 1px solid var(--blue); }
        .ce-fill { background: var(--blue); height: 100%; width: 0%; box-shadow: 0 0 8px var(--blue); transition: width 0.3s; }

        /* Domain Expansion Visuals */
        #domain-ui { position: absolute; inset: 0; background: black; opacity: 0; display: flex; align-items: center; justify-content: center; z-index: 100; pointer-events: none; transition: opacity 0.3s; }
        #domain-ui h1 { font-style: italic; font-size: 50px; text-transform: uppercase; text-shadow: 0 0 20px var(--purple); }
        #impact-flash { position: absolute; inset: 0; background: white; opacity: 0; z-index: 101; pointer-events: none; }

        /* Controls */
        .mobile-btns { display: flex; justify-content: space-between; width: 800px; margin: 10px auto; padding: 0 10px; }
        .group { display: flex; gap: 10px; }
        .btn { width: 70px; height: 60px; background: #1a1a1a; border: 2px solid #444; color: white; border-radius: 10px; font-weight: bold; cursor: pointer; display: flex; align-items: center; justify-content: center; user-select: none; }
        .btn:active { background: #444; transform: scale(0.95); }
        .btn-special { border-color: var(--blue); color: var(--blue); }
        .btn-domain { border-color: var(--purple); color: var(--purple); }
    </style>
</head>
<body>

<div id="game-container">
    <div id="impact-flash"></div>
    <div id="domain-ui"><h1>DOMAIN EXPANSION</h1></div>
    
    <div class="hud">
        <div class="stat-group">
            <div class="label">P1: SORCERER</div>
            <div class="bar-bg"><div id="p1-hp" class="hp-fill"></div></div>
            <div class="ce-bg"><div id="p1-ce" class="ce-fill"></div></div>
        </div>
        <div class="stat-group" style="text-align: right;">
            <div class="label">P2: CURSE BOT</div>
            <div class="bar-bg"><div id="p2-hp" class="hp-fill" style="float:right"></div></div>
            <div class="ce-bg"><div id="p2-ce" class="ce-fill" style="float:right"></div></div>
        </div>
    </div>
    <canvas id="gameCanvas" width="800" height="400"></canvas>
</div>

<div class="mobile-btns">
    <div class="group">
        <div class="btn" id="btn-left">◀</div>
        <div class="btn" id="btn-right">▶</div>
        <div class="btn" id="btn-up">UP</div>
    </div>
    <div class="group">
        <div class="btn btn-special" id="btn-charge">⚡</div>
        <div class="btn" id="btn-atk">HIT</div>
        <div class="btn btn-domain" id="btn-dom">DE</div>
    </div>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const gravity = 0.8;
let shake = 0;

class Fighter {
    constructor({ x, y, color, isBot = false }) {
        this.pos = { x, y };
        this.vel = { x: 0, y: 0 };
        this.width = 50; this.height = 100;
        this.color = color;
        this.hp = 100; 
        this.maxHp = 100; // <--- CHANGE BOT HEALTH HERE
        this.ce = 0;
        this.isBot = isBot;
        this.isAttacking = false;
        this.isCharging = false;
        this.dir = isBot ? -1 : 1;
        this.onGround = false;
    }

    draw() {
        ctx.save();
        if (this.isCharging) {
            ctx.shadowBlur = 20;
            ctx.shadowColor = '#00d4ff';
            ctx.fillStyle = '#fff';
        } else {
            ctx.fillStyle = this.color;
        }
        
        ctx.fillRect(this.pos.x, this.pos.y, this.width, this.height);
        
        // Attack visual
        if (this.isAttacking) {
            ctx.fillStyle = "white";
            ctx.fillRect(this.pos.x + (this.dir === 1 ? 50 : -40), this.pos.y + 20, 40, 15);
        }
        ctx.restore();
    }

    update() {
        this.pos.x += this.vel.x;
        this.pos.y += this.vel.y;

        if (this.pos.y + this.height + this.vel.y >= canvas.height - 20) {
            this.vel.y = 0;
            this.pos.y = canvas.height - 20 - this.height;
            this.onGround = true;
        } else {
            this.vel.y += gravity;
            this.onGround = false;
        }

        if (this.isCharging) {
            this.ce = Math.min(100, this.ce + 0.5);
            this.vel.x = 0;
        }
        this.draw();
    }

    attack() {
        if (this.isAttacking) return;
        this.isAttacking = true;
        setTimeout(() => this.isAttacking = false, 150);
    }
}

const p1 = new Fighter({ x: 100, y: 0, color: '#006eff' });
const bot = new Fighter({ x: 650, y: 0, color: '#ff2e2e', isBot: true });

// SET CUSTOM BOT HEALTH
bot.hp = 100; 
bot.maxHp = 100;

function applyHit(atk, vic) {
    const hitX = atk.dir === 1 ? atk.pos.x + atk.width : atk.pos.x - 40;
    if (atk.isAttacking && hitX < vic.pos.x + vic.width && hitX + 40 > vic.pos.x &&
        atk.pos.y < vic.pos.y + vic.height && atk.pos.y + 50 > vic.pos.y) {
        
        vic.hp -= 8;
        atk.ce = Math.min(100, atk.ce + 10);
        atk.isAttacking = false;
        
        // Visual Juice
        shake = 10;
        const flash = document.getElementById('impact-flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 50);
    }
}

function useDomain(caster, target) {
    if (caster.ce < 100) return;
    caster.ce = 0;
    const ui = document.getElementById('domain-ui');
    ui.style.opacity = 1;
    shake = 20;
    
    setTimeout(() => {
        target.hp -= 35;
        shake = 30;
        ui.style.opacity = 0;
        const flash = document.getElementById('impact-flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 100);
    }, 1500);
}

// Bot AI Logic
function botAI() {
    const dist = p1.pos.x - bot.pos.x;
    bot.dir = dist > 0 ? 1 : -1;

    // Movement
    if (Math.abs(dist) > 70) {
        bot.vel.x = dist > 0 ? 3 : -3;
    } else {
        bot.vel.x = 0;
        if (Math.random() < 0.05) bot.attack();
    }

    // AI Charge/Domain
    if (bot.ce < 100 && Math.random() < 0.01) bot.ce += 5;
    if (bot.ce >= 100) useDomain(bot, p1);
}

// Input Handling
const keys = { a: false, d: false };
window.addEventListener('keydown', e => {
    if (e.key === 'a') { keys.a = true; p1.dir = -1; }
    if (e.key === 'd') { keys.d = true; p1.dir = 1; }
    if (e.key === 'w' && p1.onGround) p1.vel.y = -18;
    if (e.key === 'j') p1.attack();
    if (e.key === 'c') p1.isCharging = true;
    if (e.key === 'l') useDomain(p1, bot);
});
window.addEventListener('keyup', e => {
    if (e.key === 'a') keys.a = false;
    if (e.key === 'd') keys.d = false;
    if (e.key === 'c') p1.isCharging = false;
});

// Touch controls
const bind = (id, start, end) => {
    const el = document.getElementById(id);
    el.ontouchstart = (e) => { e.preventDefault(); start(); };
    el.ontouchend = (e) => { e.preventDefault(); if(end) end(); };
};
bind('btn-left', () => { keys.a = true; p1.dir = -1; }, () => keys.a = false);
bind('btn-right', () => { keys.d = true; p1.dir = 1; }, () => keys.d = false);
bind('btn-up', () => { if(p1.onGround) p1.vel.y = -18; });
bind('btn-charge', () => p1.isCharging = true, () => p1.isCharging = false);
bind('btn-atk', () => p1.attack());
bind('btn-dom', () => useDomain(p1, bot));

function animate() {
    ctx.setTransform(1, 0, 0, 1, 0, 0);
    ctx.clearRect(0, 0, 800, 400);

    if (shake > 0) {
        ctx.translate(Math.random() * shake - shake/2, Math.random() * shake - shake/2);
        shake *= 0.9;
    }

    p1.vel.x = 0;
    if (keys.a) p1.vel.x = -6;
    if (keys.d) p1.vel.x = 6;

    p1.update();
    bot.update();
    botAI();

    applyHit(p1, bot);
    applyHit(bot, p1);

    // Update UI
    document.getElementById('p1-hp').style.width = (p1.hp/p1.maxHp*100) + "%";
    document.getElementById('p2-hp').style.width = (bot.hp/bot.maxHp*100) + "%";
    document.getElementById('p1-ce').style.width = p1.ce + "%";
    document.getElementById('p2-ce').style.width = bot.ce + "%";

    requestAnimationFrame(animate);
}
animate();
</script>
</body>
</html>
