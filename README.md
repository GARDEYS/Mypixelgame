<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="utf-8">
<title>Хроники Забытого Света</title>
<style>
  html,body{margin:0;height:100%;background:#03040a;overflow:hidden;
            font-family:"Courier New",monospace;color:#c9d4e8}
  #wrap{position:fixed;inset:0;display:flex;align-items:center;justify-content:center}
  canvas{image-rendering:pixelated;image-rendering:crisp-edges;display:block}
</style>
</head>
<body>
<div id="wrap"><canvas id="c"></canvas></div>
<script>
(() => {
"use strict";

/* =========================================================
   ХРОНИКИ ЗАБЫТОГО СВЕТА — прототип
   ========================================================= */

const VW = 480, VH = 270;
const cvs = document.getElementById('c');
const ctx = cvs.getContext('2d');

const darkCv = document.createElement('canvas');
darkCv.width = VW; darkCv.height = VH;
const dctx = darkCv.getContext('2d');

function resize(){
  const s = Math.max(1, Math.floor(Math.min(innerWidth / VW, innerHeight / VH)));
  cvs.width = VW; cvs.height = VH;
  cvs.style.width  = (VW * s) + 'px';
  cvs.style.height = (VH * s) + 'px';
  ctx.imageSmoothingEnabled = false;
}
addEventListener('resize', resize); resize();

/* ---------------- ВВОД ---------------- */
const keys = {};
let jumpBuffer = 0, restartBuffer = 0;

addEventListener('keydown', e => {
  if(['ArrowLeft','ArrowRight','ArrowUp','ArrowDown','Space'].includes(e.code)) e.preventDefault();
  if(keys[e.code]) return;
  keys[e.code] = true;
  if(e.code === 'KeyW' || e.code === 'ArrowUp' || e.code === 'Space') jumpBuffer = 0.14;
  if(e.code === 'KeyR' || e.code === 'Enter') restartBuffer = 0.14;
});
addEventListener('keyup', e => { keys[e.code] = false; });

const left  = () => keys['KeyA'] || keys['ArrowLeft'];
const right = () => keys['KeyD'] || keys['ArrowRight'];
const focus = () => keys['ShiftLeft'] || keys['ShiftRight'];

/* ---------------- УРОВЕНЬ ---------------- */
const LEVEL_W  = 2600;
const GROUND_Y = 230;

const platforms = [
  {x:0,    y:GROUND_Y, w:400, h:40},
  {x:455,  y:GROUND_Y, w:250, h:40},
  {x:760,  y:GROUND_Y, w:170, h:40},
  {x:985,  y:GROUND_Y, w:400, h:40},
  {x:1440, y:GROUND_Y, w:300, h:40},
  {x:1795, y:GROUND_Y, w:805, h:40},
  // верхние платформы
  {x:250,  y:180, w:70,  h:10},
  {x:390,  y:148, w:60,  h:10},
  {x:545,  y:175, w:80,  h:10},
  {x:695,  y:140, w:70,  h:10},
  {x:845,  y:178, w:90,  h:10},
  {x:990,  y:140, w:80,  h:10},
  {x:1140, y:168, w:90,  h:10},
  {x:1300, y:132, w:80,  h:10},
  {x:1440, y:162, w:90,  h:10},
  {x:1600, y:178, w:90,  h:10},
  {x:1760, y:138, w:100, h:10},
  {x:1950, y:178, w:90,  h:10},
  {x:2100, y:135, w:110, h:10},
  {x:2280, y:172, w:100, h:10},
];

const shroomSpawns = [
  {x:150,  y:222}, {x:600,  y:222}, {x:1050, y:222},
  {x:1520, y:222}, {x:1900, y:222}, {x:2400, y:222},
  {x:415,  y:140}, {x:1015, y:132}, {x:1795, y:130},
];

const shardSpawns = [
  {x:300,  y:150}, {x:760,  y:150}, {x:1220, y:200},
  {x:1660, y:150}, {x:2020, y:190}, {x:2250, y:110},
  {x:1340, y:100},
];

const enemySpawns = [
  {x:330,  y:180}, {x:560,  y:195}, {x:730,  y:185},
  {x:900,  y:160}, {x:1120, y:180}, {x:1320, y:200},
  {x:1530, y:170}, {x:1720, y:190}, {x:1950, y:160},
  {x:2120, y:185}, {x:2300, y:170},
];

const ALTAR = {x:2500, y:186, w:34, h:44};

/* ---------------- СОСТОЯНИЕ ---------------- */
let state = 'title';   // title | play | dead | win
let time = 0;
let cam = {x:0};
let shake = 0;

const player = {
  x:40, y:180, w:10, h:14, vx:0, vy:0,
  onGround:false, jumps:0, face:1,
  light:100, inv:0, dead:false
};

let enemies = [], shrooms = [], shards = [], particles = [], motes = [];

/* ---------------- УТИЛИТЫ ---------------- */
const rand = (a,b) => a + Math.random()*(b-a);
const clamp = (v,a,b) => v < a ? a : v > b ? b : v;

function overlap(a,b){
  return a.x < b.x+b.w && a.x+a.w > b.x && a.y < b.y+b.h && a.y+a.h > b.y;
}

function burst(x,y,color,n,spd=2){
  for(let i=0;i<n;i++){
    const a = Math.random()*Math.PI*2, s = rand(0.4,1)*spd;
    particles.push({x,y,vx:Math.cos(a)*s,vy:Math.sin(a)*s-0.4,
                    life:rand(0.3,0.8),max:0.8,color});
  }
}

/* ---------------- ИНИЦИАЛИЗАЦИЯ ---------------- */
function reset(){
  player.x = 40; player.y = 180;
  player.vx = 0; player.vy = 0;
  player.onGround = false; player.jumps = 0;
  player.light = 100; player.inv = 0; player.face = 1;
  player.dead = false;

  enemies = enemySpawns.map(s => ({
    x:s.x, y:s.y, w:14, h:14, hp:2, t:Math.random()*6.28,
    dead:false, flash:0
  }));

  shrooms = shroomSpawns.map(s => ({x:s.x, y:s.y, r:9, used:false, timer:0}));
  shards  = shardSpawns.map(s => ({x:s.x, y:s.y, taken:false, t:Math.random()*6.28}));

  particles = [];
  motes = [];
  for(let i=0;i<70;i++){
    motes.push({x:rand(0,LEVEL_W), y:rand(0,VH),
                vx:rand(-0.12,0.12), vy:rand(-0.06,0.06),
                s:Math.random()<0.3?2:1, a:rand(0.15,0.5)});
  }
  cam.x = 0; shake = 0;
}

/* ---------------- ФИЗИКА ---------------- */
function moveAndCollide(e){
  e.x += e.vx;
  for(const p of platforms){
    if(overlap(e,p)){
      if(e.vx > 0) e.x = p.x - e.w;
      else if(e.vx < 0) e.x = p.x + p.w;
      e.vx = 0;
    }
  }
  e.y += e.vy;
  e.onGround = false;
  for(const p of platforms){
    if(overlap(e,p)){
      if(e.vy > 0){ e.y = p.y - e.h; e.onGround = true; }
      else if(e.vy < 0){ e.y = p.y + p.h; }
      e.vy = 0;
    }
  }
}

/* ---------------- РАДИУС СВЕТА ---------------- */
function lightRadius(){
  const base = 26 + player.light * 0.62;
  const flick = 1 + Math.sin(time*9) * 0.02 + Math.sin(time*23) * 0.012;
  return base * (focus() && player.light > 0 ? 1.75 : 1) * flick;
}

/* ---------------- ОБНОВЛЕНИЕ ---------------- */
function update(dt){
  time += dt;

  // фоновая пыль
  for(const m of motes){
    m.x += m.vx; m.y += m.vy;
    if(m.y < -5) m.y = VH+5;
    if(m.y > VH+5) m.y = -5;
  }

  // частицы
  for(let i=particles.length-1;i>=0;i--){
    const p = particles[i];
    p.x += p.vx; p.y += p.vy; p.vy += 0.06;
    p.life -= dt;
    if(p.life <= 0) particles.splice(i,1);
  }

  if(state !== 'play') return;

  /* --- управление --- */
  const speed = 2.0;
  const accel = 0.35;

  if(left()){  player.vx -= accel; player.face = -1; }
  if(right()){ player.vx += accel; player.face =  1; }
  if(!left() && !right()) player.vx *= player.onGround ? 0.72 : 0.90;
  player.vx = clamp(player.vx, -speed, speed);

  // прыжок / двойной прыжок
  jumpBuffer -= dt;
  if(jumpBuffer > 0){
    if(player.onGround){
      player.vy = -7.8; player.jumps = 1; jumpBuffer = 0;
      burst(player.x+5, player.y+14, '#6f7fd6', 5, 1.2);
    } else if(player.jumps < 2){
      player.vy = -7.0; player.jumps = 2; jumpBuffer = 0;
      burst(player.x+5, player.y+14, '#9aa8ff', 7, 1.6);
    }
  }

  // гравитация
  player.vy += 0.5;
  if(player.vy > 11) player.vy = 11;

  moveAndCollide(player);
  if(player.onGround) player.jumps = 0;

  // невидимость после удара
  if(player.inv > 0) player.inv -= dt;

  /* --- свет --- */
  const drain = (focus() && player.light > 0 ? 5.2 : 1.3) * dt;
  player.light = Math.max(0, player.light - drain);

  if(player.light <= 0){
    die('Свет угас...');
    return;
  }

  /* --- падение в пропасть --- */
  if(player.y > VH + 30){
    die('Ты растворился во тьме...');
    return;
  }

  /* --- грибы --- */
  for(const s of shrooms){
    if(s.used){
      s.timer -= dt;
      if(s.timer <= 0){ s.used = false; }
      continue;
    }
    if(overlap(player, {x:s.x-s.r, y:s.y-s.r, w:s.r*2, h:s.r*2})){
      s.used = true; s.timer = 14;
      player.light = Math.min(100, player.light + 42);
      burst(s.x, s.y, '#7ff0d8', 14, 2.2);
      shake = Math.max(shake, 2);
    }
  }

  /* --- осколки зари --- */
  for(const sh of shards){
    if(sh.taken) continue;
    if(overlap(player, {x:sh.x-5, y:sh.y-5, w:10, h:10})){
      sh.taken = true;
      player.light = Math.min(100, player.light + 12);
      burst(sh.x, sh.y, '#ffd76a', 9, 1.8);
    }
  }

  /* --- враги --- */
  const R = lightRadius();
  const px = player.x + player.w/2, py = player.y + player.h/2;

  for(const e of enemies){
    if(e.dead) continue;
    e.t += dt * 2.4;

    const ex = e.x + e.w/2, ey = e.y + e.h/2;
    const dx = px - ex, dy = py - ey;
    const dist = Math.hypot(dx, dy) || 1;
    const lit = dist < R;

    if(lit){
      // отступает и получает урон
      e.hp -= 1.4 * dt;
      e.flash = 1;
      const sp = 1.5;
      e.x -= (dx/dist) * sp;
      e.y -= (dy/dist) * sp;
      if(e.hp <= 0){
        e.dead = true;
        burst(ex, ey, '#8ea2ff', 14, 2.4);
        player.light = Math.min(100, player.light + 4);
      }
    } else {
      e.flash = Math.max(0, e.flash - dt*2);
      const sp = 0.62;
      e.x += (dx/dist) * sp;
      e.y += (dy/dist) * sp;
    }

    // столкновение с игроком
    if(player.inv <= 0 && overlap(player, e)){
      player.light = Math.max(0, player.light - 11);
      player.inv = 1.1;
      const k = px < ex ? -1 : 1;
      player.vx = k * 4.2;
      player.vy = -3.6;
      shake = 5;
      burst(px, py, '#ff6b8a', 10, 2);
    }
  }

  /* --- алтарь (победа) --- */
  if(overlap(player, ALTAR)){
    state = 'win';
    burst(ALTAR.x+17, ALTAR.y+10, '#ffe9a3', 40, 3.5);
    shake = 8;
  }

  /* --- камера --- */
  const targetX = clamp(player.x + player.w/2 - VW/2, 0, LEVEL_W - VW);
  cam.x += (targetX - cam.x) * 0.12;
  if(shake > 0) shake = Math.max(0, shake - dt*18);

  restartBuffer -= dt;
  if(restartBuffer > 0){ reset(); state = 'play'; restartBuffer = 0; }
}

function die(msg){
  state = 'dead';
  player.dead = true;
  shake = 9;
  burst(player.x+5, player.y+7, '#ffd76a', 30, 3);
  document.title = msg;
}

/* ---------------- ОТРИСОВКА ---------------- */
function drawBackground(){
  const g = ctx.createLinearGradient(0,0,0,VH);
  g.addColorStop(0, '#0a0d1e');
  g.addColorStop(1, '#04050c');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,VW,VH);

  // дальний слой колонн
  ctx.fillStyle = '#0d1122';
  for(let i=0;i<22;i++){
    let bx = ((i*180 - cam.x*0.25) % (VW+400) + VW+400) % (VW+400) - 200;
    const bw = 32 + (i%3)*20;
    const bh = 95 + (i%4)*38;
    ctx.fillRect(Math.round(bx), VH-bh-45, bw, bh);
  }
  // ближний слой
  ctx.fillStyle = '#131a2e';
  for(let i=0;i<26;i++){
    let bx = ((i*140 - cam.x*0.5) % (VW+320) + VW+320) % (VW+320) - 160;
    const bw = 22 + (i%4)*15;
    const bh = 55 + (i%5)*32;
    ctx.fillRect(Math.round(bx), VH-bh-22, bw, bh);
  }
  // туман у земли
  const f = ctx.createLinearGradient(0, VH-70, 0, VH);
  f.addColorStop(0, 'rgba(30,40,80,0)');
  f.addColorStop(1, 'rgba(40,55,110,0.16)');
  ctx.fillStyle = f;
  ctx.fillRect(0, VH-70, VW, 70);
}

function drawPlatforms(){
  for(const p of platforms){
    const x = Math.round(p.x - cam.x), y = Math.round(p.y);
    if(x + p.w < -10 || x > VW + 10) continue;

    ctx.fillStyle = '#221d33';
    ctx.fillRect(x, y, p.w, p.h);

    // верхняя кромка
    ctx.fillStyle = '#3d3560';
    ctx.fillRect(x, y, p.w, 2);
    ctx.fillStyle = '#514679';
    ctx.fillRect(x, y, p.w, 1);

    // пиксельные крапинки (детерминированные)
    ctx.fillStyle = '#2c2542';
    for(let i = 0; i < p.w; i += 6){
      const h = ((p.x + i) * 7919) % 11;
      if(h < 4){
        ctx.fillRect(x + i, y + 4 + (h*2), 2, 2);
      }
    }
    // нижняя тень
    ctx.fillStyle = '#171327';
    ctx.fillRect(x, y + p.h - 3, p.w, 3);
  }
}

function drawShrooms(){
  for(const s of shrooms){
    const x = Math.round(s.x - cam.x), y = Math.round(s.y);
    if(x < -30 || x > VW+30) continue;

    if(s.used){
      ctx.fillStyle = '#2a3d3d';
      ctx.fillRect(x-2, y-2, 5, 4);
      continue;
    }
    // ножка
    ctx.fillStyle = '#4d7f74';
    ctx.fillRect(x-1, y-1, 3, 5);
    // шляпка
    ctx.fillStyle = '#5fe0c4';
    ctx.fillRect(x-4, y-5, 9, 4);
    ctx.fillStyle = '#a6fff0';
    ctx.fillRect(x-4, y-5, 9, 1);
    // пульс
    const pulse = 0.6 + Math.sin(time*4 + s.x)*0.4;
    ctx.fillStyle = `rgba(120,255,225,${0.10*pulse})`;
    ctx.beginPath();
    ctx.arc(x, y-3, 14 + pulse*3, 0, 6.283);
    ctx.fill();
  }
}

function drawShards(){
  for(const s of shards){
    if(s.taken) continue;
    const x = Math.round(s.x - cam.x);
    const y = Math.round(s.y + Math.sin(time*2.5 + s.t)*3);
    if(x < -20 || x > VW+20) continue;

    ctx.fillStyle = '#ffd76a';
    ctx.fillRect(x-1, y-3, 2, 6);
    ctx.fillRect(x-3, y-1, 6, 2);
    ctx.fillStyle = '#fff3c4';
    ctx.fillRect(x-1, y-1, 2, 2);
  }
}

function drawAltar(){
  const x = Math.round(ALTAR.x - cam.x), y = ALTAR.y;
  if(x < -60 || x > VW+60) return;

  // ступени
  ctx.fillStyle = '#2a2440';
  ctx.fillRect(x-8, y+34, 50, 8);
  ctx.fillRect(x-4, y+28, 42, 6);
  // колонны
  ctx.fillStyle = '#37304f';
  ctx.fillRect(x, y, 6, 34);
  ctx.fillRect(x+28, y, 6, 34);
  ctx.fillStyle = '#4b4270';
  ctx.fillRect(x, y, 2, 34);
  ctx.fillRect(x+28, y, 2, 34);
  // арка
  ctx.fillStyle = '#37304f';
  ctx.fillRect(x, y-4, 34, 5);
  // кристалл
  const pulse = 0.7 + Math.sin(time*3)*0.3;
  ctx.fillStyle = '#ffe9a3';
  ctx.fillRect(x+14, y+10, 6, 10);
  ctx.fillStyle = `rgba(255,220,140,${0.5*pulse})`;
  ctx.fillRect(x+11, y+7, 12, 16);
  ctx.fillStyle = '#fffbe0';
  ctx.fillRect(x+15, y+12, 4, 5);
}

function drawEnemies(){
  for(const e of enemies){
    if(e.dead) continue;
    const bob = Math.sin(e.t)*2;
    const x = Math.round(e.x - cam.x);
    const y = Math.round(e.y + bob);
    if(x < -40 || x > VW+40) continue;

    // тело
    const c = e.flash > 0 ? '#3a3f78' : '#0e0e1a';
    ctx.fillStyle = c;
    ctx.beginPath();
    ctx.arc(x+7, y+7, 7, 0, 6.283);
    ctx.fill();
    // рваные края
    ctx.fillStyle = c;
    ctx.fillRect(x+1, y+9, 12, 5);
    ctx.fillRect(x-1, y+7, 3, 4);
    ctx.fillRect(x+12, y+8, 3, 4);

    // глаза
    ctx.fillStyle = e.flash > 0 ? '#ffffff' : '#ff4d6d';
    ctx.fillRect(x+3, y+5, 2, 2);
    ctx.fillRect(x+9, y+5, 2, 2);

    // вспышка урона
    if(e.flash > 0){
      ctx.fillStyle = `rgba(160,180,255,${e.flash*0.5})`;
      ctx.beginPath();
      ctx.arc(x+7, y+7, 11, 0, 6.283);
      ctx.fill();
    }
  }
}

function drawPlayer(){
  if(state === 'dead') return;
  const x = Math.round(player.x - cam.x);
  const y = Math.round(player.y);

  // мерцание при неуязвимости
  if(player.inv > 0 && Math.floor(time*20) % 2 === 0) return;

  const f = player.face;

  // плащ
  ctx.fillStyle = '#2e2748';
  ctx.fillRect(x, y+4, 10, 10);
  ctx.fillStyle = '#3b3160';
  ctx.fillRect(x+1, y+2, 8, 8);
  // капюшон
  ctx.fillStyle = '#4a3d78';
  ctx.fillRect(x+1, y, 8, 5);
  ctx.fillStyle = '#5a4a8e';
  ctx.fillRect(x+1, y, 8, 1);
  // тень внутри капюшона
  ctx.fillStyle = '#0a0a14';
  ctx.fillRect(x+2, y+2, 6, 3);
  // глаза
  ctx.fillStyle = '#ffd76a';
  ctx.fillRect(x+3, y+2, 2, 2);
  ctx.fillRect(x+6, y+2, 2, 2);
  // ноги
  ctx.fillStyle = '#211c36';
  ctx.fillRect(x+1, y+13, 3, 2);
  ctx.fillRect(x+6, y+13, 3, 2);

  // посох
  const sx = f > 0 ? x + 11 : x - 3;
  ctx.fillStyle = '#7a5c34';
  ctx.fillRect(sx, y-6, 2, 16);
  ctx.fillStyle = '#9a7648';
  ctx.fillRect(sx, y-6, 1, 16);
  // кристалл на посохе
  const p2 = 0.7 + Math.sin(time*7)*0.3;
  ctx.fillStyle = '#ffe9a3';
  ctx.fillRect(sx-1, y-9, 4, 4);
  ctx.fillStyle = `rgba(255,235,170,${0.55*p2})`;
  ctx.fillRect(sx-3, y-11, 8, 8);
}

function drawParticles(){
  for(const p of particles){
    const a = clamp(p.life / p.max, 0, 1);
    ctx.globalAlpha = a;
    ctx.fillStyle = p.color;
    ctx.fillRect(Math.round(p.x - cam.x), Math.round(p.y), 2, 2);
  }
  ctx.globalAlpha = 1;
}

function drawMotes(){
  for(const m of motes){
    const x = Math.round(m.x - cam.x*0.8);
    const y = Math.round(m.y + Math.sin(time + m.x*0.01)*4);
    if(x < -5 || x > VW+5) continue;
    ctx.globalAlpha = m.a * (0.5 + 0.5*Math.sin(time*1.5 + m.x));
    ctx.fillStyle = '#8fa4ff';
    ctx.fillRect(x, y, m.s, m.s);
  }
  ctx.globalAlpha = 1;
}

/* ---- слой тьмы ---- */
function drawDarkness(){
  if(state === 'win') return;

  dctx.globalCompositeOperation = 'source-over';
  dctx.clearRect(0,0,VW,VH);
  dctx.fillStyle = 'rgba(2,3,9,0.965)';
  dctx.fillRect(0,0,VW,VH);

  dctx.globalCompositeOperation = 'destination-out';

  // свет игрока
  if(state !== 'dead'){
    const px = player.x + player.w/2 - cam.x;
    const py = player.y + player.h/2;
    const R = lightRadius();
    const g = dctx.createRadialGradient(px, py, 0, px, py, R);
    g.addColorStop(0,    'rgba(0,0,0,1)');
    g.addColorStop(0.45, 'rgba(0,0,0,0.92)');
    g.addColorStop(0.75, 'rgba(0,0,0,0.45)');
    g.addColorStop(1,    'rgba(0,0,0,0)');
    dctx.fillStyle = g;
    dctx.beginPath(); dctx.arc(px, py, R, 0, 6.283); dctx.fill();
  }

  // грибы
  for(const s of shrooms){
    if(s.used) continue;
    const x = s.x - cam.x, y = s.y;
    if(x < -60 || x > VW+60) continue;
    const g = dctx.createRadialGradient(x, y-3, 0, x, y-3, 42);
    g.addColorStop(0,   'rgba(0,0,0,0.85)');
    g.addColorStop(0.6, 'rgba(0,0,0,0.30)');
    g.addColorStop(1,   'rgba(0,0,0,0)');
    dctx.fillStyle = g;
    dctx.beginPath(); dctx.arc(x, y-3, 42, 0, 6.283); dctx.fill();
  }

  // алтарь
  {
    const x = ALTAR.x + 17 - cam.x, y = ALTAR.y + 12;
    const g = dctx.createRadialGradient(x, y, 0, x, y, 70);
    g.addColorStop(0,   'rgba(0,0,0,0.9)');
    g.addColorStop(0.5, 'rgba(0,0,0,0.35)');
    g.addColorStop(1,   'rgba(0,0,0,0)');
    dctx.fillStyle = g;
    dctx.beginPath(); dctx.arc(x, y, 70, 0, 6.283); dctx.fill();
  }

  dctx.globalCompositeOperation = 'source-over';
  ctx.drawImage(darkCv, 0, 0);

  // тёплое свечение поверх
  if(state !== 'dead'){
    const px = player.x + player.w/2 - cam.x;
    const py = player.y + player.h/2;
    const R = lightRadius();
    ctx.globalCompositeOperation = 'lighter';
    const g = ctx.createRadialGradient(px, py, 0, px, py, R*0.95);
    g.addColorStop(0,   'rgba(255,205,110,0.16)');
    g.addColorStop(0.5, 'rgba(255,175,70,0.06)');
    g.addColorStop(1,   'rgba(255,150,50,0)');
    ctx.fillStyle = g;
    ctx.fillRect(px-R, py-R, R*2, R*2);
    ctx.globalCompositeOperation = 'source-over';
  }
}

/* ---- HUD ---- */
function drawHUD(){
  if(state === 'title') return;

  const w = 92, h = 7, x = 10, y = 10;

  ctx.fillStyle = 'rgba(6,8,18,0.75)';
  ctx.fillRect(x-2, y-2, w+4, h+4);
  ctx.fillStyle = '#1b2138';
  ctx.fillRect(x, y, w, h);

  const pct = player.light / 100;
  const col = pct > 0.5 ? '#ffd76a' : pct > 0.22 ? '#ff9a3c' : '#ff4d5e';
  ctx.fillStyle = col;
  ctx.fillRect(x, y, Math.round(w*pct), h);
  ctx.fillStyle = 'rgba(255,255,255,0.25)';
  ctx.fillRect(x, y, Math.round(w*pct), 2);

  ctx.fillStyle = '#7f8bb0';
  ctx.font = '8px "Courier New", monospace';
  ctx.fillText('СВЕТ', x, y + h + 11);

  // подсказка
  if(player.light < 30 && Math.floor(time*3) % 2 === 0){
    ctx.fillStyle = '#ff8b9c';
    ctx.fillText('НАЙДИ СВЕТЯЩИЙСЯ ГРИБ', VW/2 - 62, 24);
  }

  // индикатор фокуса
  if(focus() && player.light > 0){
    ctx.fillStyle = '#ffe9a3';
    ctx.fillText('ВСПЫШКА', VW - 62, 24);
  }
}

/* ---- ЭКРАНЫ ---- */
function drawTitle(){
  ctx.fillStyle = 'rgba(3,4,12,0.86)';
  ctx.fillRect(0,0,VW,VH);

  ctx.textAlign = 'center';
  ctx.fillStyle = '#ffd76a';
  ctx.font = 'bold 20px "Courier New", monospace';
  ctx.fillText('ХРОНИКИ', VW/2, 70);
  ctx.fillText('ЗАБЫТОГО СВЕТА', VW/2, 96);

  ctx.fillStyle = '#5a6a94';
  ctx.font = '9px "Courier New", monospace';
  ctx.fillText('Солнце погасло. Ты — последняя искра.', VW/2, 124);

  ctx.fillStyle = '#8fa0c8';
  ctx.font = '9px "Courier New", monospace';
  ctx.fillText('A / D  или  ← →   — движение', VW/2, 158);
  ctx.fillText('W / ↑ / ПРОБЕЛ    — прыжок (двойной)', VW/2, 174);
  ctx.fillText('SHIFT             — вспышка (жжёт свет)', VW/2, 190);
  ctx.fillText('R                 — начать заново', VW/2, 206);

  const a = 0.5 + 0.5*Math.sin(time*3);
  ctx.globalAlpha = a;
  ctx.fillStyle = '#ffd76a';
  ctx.font = 'bold 11px "Courier New", monospace';
  ctx.fillText('НАЖМИ ПРОБЕЛ', VW/2, 242);
  ctx.globalAlpha = 1;
  ctx.textAlign = 'left';
}

function drawDead(){
  ctx.fillStyle = 'rgba(20,0,10,0.72)';
  ctx.fillRect(0,0,VW,VH);

  ctx.textAlign = 'center';
  ctx.fillStyle = '#ff6b8a';
  ctx.font = 'bold 18px "Courier New", monospace';
  ctx.fillText('СВЕТ УГАС', VW/2, 118);

  ctx.fillStyle = '#8fa0c8';
  ctx.font = '9px "Courier New", monospace';
  ctx.fillText('Тьма поглотила странника...', VW/2, 142);

  const a = 0.5 + 0.5*Math.sin(time*4);
  ctx.globalAlpha = a;
  ctx.fillStyle = '#ffd76a';
  ctx.font = 'bold 10px "Courier New", monospace';
  ctx.fillText('R — ПОПРОБОВАТЬ СНОВА', VW/2, 178);
  ctx.globalAlpha = 1;
  ctx.textAlign = 'left';
}

function drawWin(){
  ctx.fillStyle = 'rgba(30,20,0,0.68)';
  ctx.fillRect(0,0,VW,VH);

  ctx.textAlign = 'center';
  ctx.fillStyle = '#ffe9a3';
  ctx.font = 'bold 18px "Courier New", monospace';
  ctx.fillText('ИСКРА ВОЗВРАЩЕНА', VW/2, 110);

  ctx.fillStyle = '#c9d4e8';
  ctx.font = '9px "Courier New", monospace';
  ctx.fillText('Алтарь принял твой свет.', VW/2, 136);
  ctx.fillText('Но это лишь начало пути...', VW/2, 152);

  const a = 0.5 + 0.5*Math.sin(time*4);
  ctx.globalAlpha = a;
  ctx.fillStyle = '#ffd76a';
  ctx.font = 'bold 10px "Courier New", monospace';
  ctx.fillText('R — ИГРАТЬ СНОВА', VW/2, 196);
  ctx.globalAlpha = 1;
  ctx.textAlign = 'left';
}

/* ---------------- РЕНДЕР ---------------- */
function render(){
  ctx.save();

  // тряска экрана
  if(shake > 0.2){
    ctx.translate(
      Math.round(rand(-shake, shake)),
      Math.round(rand(-shake, shake))
    );
  }

  drawBackground();
  drawPlatforms();
  drawShrooms();
  drawShards();
  drawAltar();
  drawEnemies();
  drawMotes();
  drawPlayer();
  drawParticles();
  drawDarkness();
  drawHUD();

  ctx.restore();

  if(state === 'title') drawTitle();
  if(state === 'dead')  drawDead();
  if(state === 'win')   drawWin();
}

/* ---------------- ЦИКЛ ---------------- */
let last = performance.now();
let acc = 0;
const STEP = 1/60;

function loop(now){
  requestAnimationFrame(loop);
  let dt = (now - last) / 1000;
  last = now;
  if(dt > 0.1) dt = 0.1;

  acc += dt;
  let guard = 0;
  while(acc >= STEP && guard++ < 5){
    if(state === 'title' && jumpBuffer > 0){
      jumpBuffer = 0;
      reset();
      state = 'play';
    }
    update(STEP);
    acc -= STEP;
  }
  render();
}

/* ---------------- СТАРТ ---------------- */
reset();
requestAnimationFrame(loop);

})();
</script>
</body>
</html>
