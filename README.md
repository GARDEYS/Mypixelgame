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
   ХРОНИКИ ЗАБЫТОГО СВЕТА
   Платформер-головоломка со светом, 4 типами врагов,
   процедурным звуком, чиптюн-музыкой и эхом.
   ========================================================= */

/* ====================== АУДИО-ДВИЖОК ====================== */
const SFX = (() => {
  let ac = null, master = null, input = null;
  let reverb = null, drySend = null, wetSend = null;
  let ambientGain = null, musicGain = null;
  let layerBase = null, layerPad = null, layerArp = null, layerPerc = null;
  let enabled = true, muted = false;
  let noiseBuf = null, lastHitT = 0;

  let musicIntensity = 0;
  let musicIntensitySm = 0;

  let musicTimer = null;
  let nextNoteTime = 0;
  let step = 0;
  const BPM = 72;
  const STEP_DUR = 60 / BPM / 2;
  const LOOKAHEAD = 0.12;
  const INTERVAL = 25;
  const TOTAL_STEPS = 32;

  const N = {
    'A2':110.00,'F2':87.31,'C3':130.81,'G2':98.00,
    'F3':174.61,'A3':220.00,'B3':246.94,'C4':261.63,'D4':293.66,'E4':329.63,
    'F4':349.23,'G4':392.00,'A4':440.00,'B4':493.88,
    'C5':523.25,'D5':587.33,'E5':659.25,'G5':783.99,'A5':880.00
  };

  const MELODY = [
    'A4',null,'E5',null, 'C5',null,'A4',null,
    'F4',null,'C5',null, 'A4',null,'F4',null,
    'C4',null,'G4',null, 'E5',null,'C5',null,
    'G3',null,'D4',null, 'B4',null,'G4',null
  ];
  const BASS = [
    'A2',null,null,null,'A2',null,null,null,
    'F2',null,null,null,'F2',null,null,null,
    'C3',null,null,null,'C3',null,null,null,
    'G2',null,null,null,'G2',null,null,null
  ];
  const PADS = [
    ['A3','C4','E4'],
    ['F3','A3','C4'],
    ['C4','E4','G4'],
    ['G3','B3','D4']
  ];
  const ARP = ['A5','C5','E5','G5'];

  function ensure(){
    if(!enabled) return null;
    if(ac){ if(ac.state === 'suspended') ac.resume(); return ac; }
    try{
      ac = new (window.AudioContext || window.webkitAudioContext)();

      master = ac.createGain();
      master.gain.value = 0.34;
      master.connect(ac.destination);

      reverb = ac.createConvolver();
      reverb.buffer = makeImpulse(ac, 2.6, 3.2);

      input = ac.createGain();
      drySend = ac.createGain(); drySend.gain.value = 0.78;
      wetSend = ac.createGain(); wetSend.gain.value = 0.62;
      input.connect(drySend); drySend.connect(master);
      input.connect(wetSend); wetSend.connect(reverb);
      reverb.connect(master);

      musicGain = ac.createGain();
      musicGain.gain.value = 0.0001;
      musicGain.connect(input);

      layerBase = ac.createGain(); layerBase.gain.value = 1.0;
      layerPad  = ac.createGain(); layerPad.gain.value  = 0.0;
      layerArp  = ac.createGain(); layerArp.gain.value  = 0.0;
      layerPerc = ac.createGain(); layerPerc.gain.value = 0.0;

      layerBase.connect(musicGain);
      layerPad.connect(musicGain);
      layerArp.connect(musicGain);
      layerPerc.connect(musicGain);
    }catch(e){ enabled = false; }
    return ac;
  }

  function makeImpulse(c, dur, decay){
    const rate = c.sampleRate;
    const len = Math.floor(rate * dur);
    const buf = c.createBuffer(2, len, rate);
    for(let ch = 0; ch < 2; ch++){
      const d = buf.getChannelData(ch);
      for(let i = 0; i < len; i++){
        const t = i / len;
        let v = (Math.random()*2 - 1) * Math.pow(1 - t, decay);
        if(i < rate * 0.04) v *= 1.7;
        d[i] = v;
      }
    }
    return buf;
  }

  function getNoise(c){
    if(noiseBuf) return noiseBuf;
    const len = Math.floor(c.sampleRate * 0.6);
    noiseBuf = c.createBuffer(1, len, c.sampleRate);
    const d = noiseBuf.getChannelData(0);
    for(let i = 0; i < len; i++) d[i] = Math.random()*2 - 1;
    return noiseBuf;
  }

  function env(g, t0, attack, peak, decay){
    g.gain.cancelScheduledValues(t0);
    g.gain.setValueAtTime(0.0001, t0);
    g.gain.exponentialRampToValueAtTime(Math.max(0.0002, peak), t0 + attack);
    g.gain.exponentialRampToValueAtTime(0.0001, t0 + attack + decay);
  }

  function tone(freq, dur, type='square', vol=0.25, slideTo=null, delay=0){
    const c = ensure(); if(!c || muted) return;
    const t = c.currentTime + delay;
    const o = c.createOscillator(), g = c.createGain();
    o.type = type;
    o.frequency.setValueAtTime(freq, t);
    if(slideTo) o.frequency.exponentialRampToValueAtTime(Math.max(20, slideTo), t + dur);
    env(g, t, 0.005, vol, dur);
    o.connect(g); g.connect(input);
    o.start(t); o.stop(t + dur + 0.06);
  }

  function noise(dur, vol=0.18, fFrom=3000, fTo=180, q=1.2, delay=0){
    const c = ensure(); if(!c || muted) return;
    const t = c.currentTime + delay;
    const src = c.createBufferSource();
    src.buffer = getNoise(c);
    const flt = c.createBiquadFilter();
    flt.type = 'lowpass'; flt.Q.value = q;
    flt.frequency.setValueAtTime(fFrom, t);
    flt.frequency.exponentialRampToValueAtTime(Math.max(60, fTo), t + dur);
    const g = c.createGain();
    env(g, t, 0.004, vol, dur);
    src.connect(flt); flt.connect(g); g.connect(input);
    src.start(t); src.stop(t + dur + 0.05);
  }

  function startAmbient(){
    const c = ensure(); if(!c || ambientGain) return;
    const t = c.currentTime;
    ambientGain = c.createGain();
    ambientGain.gain.setValueAtTime(0.0001, t);
    ambientGain.gain.exponentialRampToValueAtTime(0.055, t + 4);
    ambientGain.connect(input);

    const lp = c.createBiquadFilter();
    lp.type = 'lowpass'; lp.frequency.value = 320;
    lp.connect(ambientGain);

    [110, 110.7, 164.8].forEach((f, i) => {
      const o = c.createOscillator();
      o.type = i === 2 ? 'triangle' : 'sine';
      o.frequency.value = f;
      const g = c.createGain();
      g.gain.value = i === 2 ? 0.35 : 0.55;
      o.connect(g); g.connect(lp);
      o.start(t);
    });

    const lfo = c.createOscillator();
    lfo.frequency.value = 0.06;
    const lfoG = c.createGain();
    lfoG.gain.value = 0.018;
    lfo.connect(lfoG); lfoG.connect(ambientGain.gain);
    lfo.start(t);
  }

  function playNote(freq, t, dur, type, vol, dest){
    const o = ac.createOscillator();
    o.type = type;
    o.frequency.value = freq;
    const g = ac.createGain();
    g.gain.setValueAtTime(0.0001, t);
    g.gain.exponentialRampToValueAtTime(vol, t + 0.015);
    g.gain.setValueAtTime(vol, t + dur * 0.55);
    g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
    o.connect(g); g.connect(dest);
    o.start(t); o.stop(t + dur + 0.05);
  }

  function kickPerc(t){
    const o = ac.createOscillator();
    const g = ac.createGain();
    o.frequency.setValueAtTime(120, t);
    o.frequency.exponentialRampToValueAtTime(40, t + 0.12);
    g.gain.setValueAtTime(0.0001, t);
    g.gain.exponentialRampToValueAtTime(0.20, t + 0.005);
    g.gain.exponentialRampToValueAtTime(0.0001, t + 0.18);
    o.connect(g); g.connect(layerPerc);
    o.start(t); o.stop(t + 0.22);
  }

  function hatPerc(t){
    const src = ac.createBufferSource();
    src.buffer = getNoise(ac);
    const flt = ac.createBiquadFilter();
    flt.type = 'highpass'; flt.frequency.value = 6500;
    const g = ac.createGain();
    g.gain.setValueAtTime(0.0001, t);
    g.gain.exponentialRampToValueAtTime(0.055, t + 0.003);
    g.gain.exponentialRampToValueAtTime(0.0001, t + 0.05);
    src.connect(flt); flt.connect(g); g.connect(layerPerc);
    src.start(t); src.stop(t + 0.1);
  }

  function scheduleStep(i, t){
    const mel = MELODY[i];
    if(mel){
      playNote(N[mel], t, STEP_DUR * 1.7, 'triangle', 0.16, layerBase);
      playNote(N[mel] * 2, t, STEP_DUR * 0.9, 'sine', 0.045, layerBase);
    }
    const bass = BASS[i];
    if(bass){
      playNote(N[bass], t, STEP_DUR * 3.2, 'sawtooth', 0.10, layerBase);
      playNote(N[bass] / 2, t, STEP_DUR * 3.2, 'sine', 0.08, layerBase);
    }
    if(i % 8 === 0){
      const chord = PADS[Math.floor(i / 8)];
      if(chord) chord.forEach(f =>
        playNote(N[f], t, STEP_DUR * 7.6, 'sine', 0.06, layerPad));
    }
    if(i % 2 === 1){
      const n = ARP[Math.floor(i / 2) % ARP.length];
      playNote(N[n], t, STEP_DUR * 0.75, 'square', 0.038, layerArp);
    }
    if(i % 4 === 0) kickPerc(t);
    if(i % 4 === 2) hatPerc(t);
  }

  function updateLayers(){
    musicIntensitySm += (musicIntensity - musicIntensitySm) * 0.06;
    const I = musicIntensitySm;
    layerPad.gain.value  = Math.min(1, I * 1.1);
    layerArp.gain.value  = Math.max(0, (I - 0.35) / 0.65);
    layerPerc.gain.value = Math.max(0, (I - 0.55) / 0.45);
  }

  function scheduler(){
    if(!ac || muted) return;
    updateLayers();
    if(nextNoteTime < ac.currentTime) nextNoteTime = ac.currentTime + 0.05;
    let guard = 0;
    while(nextNoteTime < ac.currentTime + LOOKAHEAD && guard++ < 32){
      scheduleStep(step, nextNoteTime);
      nextNoteTime += STEP_DUR;
      step = (step + 1) % TOTAL_STEPS;
    }
  }

  function startMusic(){
    const c = ensure(); if(!c || musicTimer) return;
    step = 0;
    nextNoteTime = c.currentTime + 0.15;
    musicTimer = setInterval(scheduler, INTERVAL);
    const t = c.currentTime;
    musicGain.gain.cancelScheduledValues(t);
    musicGain.gain.setValueAtTime(Math.max(0.0001, musicGain.gain.value), t);
    musicGain.gain.exponentialRampToValueAtTime(0.85, t + 3.5);
  }

  return {
    init(){ ensure(); startAmbient(); startMusic(); },

    setIntensity(v){
      musicIntensity = v < 0 ? 0 : v > 1 ? 1 : v;
    },
    getIntensity(){ return musicIntensitySm; },

    toggleMute(){
      muted = !muted;
      if(master) master.gain.value = muted ? 0 : 0.34;
      return muted;
    },
    isMuted(){ return muted; },

    jump(){     tone(320, 0.10, 'square',   0.16, 520); },
    djump(){    tone(480, 0.12, 'square',   0.15, 760);
                tone(720, 0.09, 'triangle', 0.09, 900, 0.03); },
    land(){     noise(0.10, 0.16, 900, 90, 1.0); },

    footstep(side){
      const c = ensure(); if(!c || muted) return;
      const t = c.currentTime;
      const src = c.createBufferSource();
      src.buffer = getNoise(c);
      const flt = c.createBiquadFilter();
      flt.type = 'lowpass';
      const f0 = side ? 780 : 1000;
      flt.frequency.setValueAtTime(f0, t);
      flt.frequency.exponentialRampToValueAtTime(180, t + 0.06);
      const g = c.createGain();
      g.gain.setValueAtTime(0.0001, t);
      g.gain.exponentialRampToValueAtTime(0.055, t + 0.004);
      g.gain.exponentialRampToValueAtTime(0.0001, t + 0.07);
      src.connect(flt); flt.connect(g); g.connect(input);
      src.start(t); src.stop(t + 0.1);
    },

    flash(){    noise(0.28, 0.20, 5000, 400, 1.5);
                tone(880, 0.22, 'triangle', 0.10, 1600, 0.01);
                tone(1320, 0.18, 'sine',    0.06, 2200, 0.03); },
    hitEnemy(){ const now = performance.now();
                if(now - lastHitT < 55) return; lastHitT = now;
                noise(0.06, 0.13, 2400, 500);
                tone(420, 0.05, 'square', 0.08, 260); },
    killEnemy(){ tone(260, 0.18, 'sawtooth', 0.18, 90);
                 noise(0.16, 0.14, 1800, 200, 1.1); },
    hurt(){     tone(180, 0.30, 'sawtooth', 0.22, 70);
                noise(0.22, 0.18, 1400, 120, 1.0); },
    shroom(){   [523.25, 659.25, 783.99, 1046.5].forEach((f, i) =>
                  tone(f, 0.22, 'triangle', 0.14, null, i*0.05)); },
    shard(){    tone(880, 0.07, 'triangle', 0.14, 1320);
                tone(1320, 0.12, 'sine', 0.10, null, 0.05); },
    death(){    tone(220, 0.9, 'sawtooth', 0.22, 40);
                noise(0.8, 0.16, 1200, 60, 0.9);
                tone(110, 1.2, 'sine', 0.14, 30, 0.1); },
    win(){      [523.25, 659.25, 783.99, 1046.5, 1318.5].forEach((f, i) => {
                  tone(f, 0.4, 'triangle', 0.16, null, i*0.11);
                  tone(f*2, 0.3, 'sine', 0.05, null, i*0.11 + 0.02);
                });
                noise(0.6, 0.10, 6000, 800, 1.4, 0.05); },
  };
})();

/* ====================== КАНВАС ====================== */
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

/* ====================== ВВОД ====================== */
const keys = {};
let jumpBuffer = 0, restartBuffer = 0;
let prevFocus = false;
let mutedBannerT = 0;

addEventListener('keydown', e => {
  if(['ArrowLeft','ArrowRight','ArrowUp','ArrowDown','Space'].includes(e.code)) e.preventDefault();
  SFX.init();
  if(keys[e.code]) return;
  keys[e.code] = true;
  if(e.code === 'KeyW' || e.code === 'ArrowUp' || e.code === 'Space') jumpBuffer = 0.14;
  if(e.code === 'KeyR' || e.code === 'Enter') restartBuffer = 0.14;
  if(e.code === 'KeyM'){ SFX.toggleMute(); mutedBannerT = 1.6; }
});
addEventListener('keyup', e => { keys[e.code] = false; });

const left  = () => keys['KeyA'] || keys['ArrowLeft'];
const right = () => keys['KeyD'] || keys['ArrowRight'];
const focus = () => keys['ShiftLeft'] || keys['ShiftRight'];

/* ====================== УРОВЕНЬ ====================== */
const LEVEL_W  = 2600;
const GROUND_Y = 230;

const platforms = [
  {x:0,    y:GROUND_Y, w:400, h:40},{x:455,  y:GROUND_Y, w:250, h:40},
  {x:760,  y:GROUND_Y, w:170, h:40},{x:985,  y:GROUND_Y, w:400, h:40},
  {x:1440, y:GROUND_Y, w:300, h:40},{x:1795, y:GROUND_Y, w:805, h:40},
  {x:250,  y:180, w:70,  h:10},{x:390,  y:148, w:60,  h:10},
  {x:545,  y:175, w:80,  h:10},{x:695,  y:140, w:70,  h:10},
  {x:845,  y:178, w:90,  h:10},{x:990,  y:140, w:80,  h:10},
  {x:1140, y:168, w:90,  h:10},{x:1300, y:132, w:80,  h:10},
  {x:1440, y:162, w:90,  h:10},{x:1600, y:178, w:90,  h:10},
  {x:1760, y:138, w:100, h:10},{x:1950, y:178, w:90,  h:10},
  {x:2100, y:135, w:110, h:10},{x:2280, y:172, w:100, h:10},
];

const shroomSpawns = [
  {x:150,y:222},{x:600,y:222},{x:1050,y:222},{x:1520,y:222},
  {x:1900,y:222},{x:2400,y:222},{x:415,y:140},{x:1015,y:132},{x:1795,y:130},
];
const shardSpawns = [
  {x:300,y:150},{x:760,y:150},{x:1220,y:200},{x:1660,y:150},
  {x:2020,y:190},{x:2250,y:110},{x:1340,y:100},
];

const enemySpawns = [
  {x:330,  y:180, type:'crawler'},
  {x:560,  y:195, type:'crawler'},
  {x:730,  y:185, type:'hopper'},
  {x:900,  y:160, type:'crawler'},
  {x:1120, y:180, type:'sentinel'},
  {x:1320, y:200, type:'hopper'},
  {x:1530, y:170, type:'devourer'},
  {x:1720, y:190, type:'crawler'},
  {x:1950, y:160, type:'hopper'},
  {x:2120, y:185, type:'sentinel'},
  {x:2300, y:170, type:'devourer'},
];
const ALTAR = {x:2500, y:186, w:34, h:44};

/* ====================== СОСТОЯНИЕ ====================== */
let state = 'title';
let time = 0;
let cam = {x:0};
let shake = 0;

const player = {
  x:40, y:180, w:10, h:14, vx:0, vy:0,
  onGround:false, jumps:0, face:1,
  light:70, inv:0, dead:false
};

let enemies=[], shrooms=[], shards=[], particles=[], motes=[];
let projectiles = [];
let stepTimer = 0, stepSide = 0;
let flashWave = null;
let flashCooldown = 0;

/* ====================== УТИЛИТЫ ====================== */
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

/* ====================== ФАБРИКА ВРАГОВ ====================== */
function makeEnemy(spawn){
  const base = {
    x: spawn.x, y: spawn.y, type: spawn.type,
    t: Math.random()*6.28, dead:false, flash:0
  };
  switch(spawn.type){
    case 'crawler':
      return { ...base, w:14, h:14, hp:2.0, maxHp:2.0,
               dmgMult:1.0, hitDmg:2.2, speed:0.62, retreat:1.5 };
    case 'hopper':
      return { ...base, w:12, h:12, hp:1.6, maxHp:1.6,
               dmgMult:1.0, hitDmg:2.6, speed:0.35, retreat:1.8,
               jumpTimer: rand(0.4, 1.0), jumpVx:0, jumpVy:0 };
    case 'devourer':
      return { ...base, w:22, h:22, hp:5.5, maxHp:5.5,
               dmgMult:2.0, hitDmg:1.5, speed:0.32, retreat:0.9 };
    case 'sentinel':
      return { ...base, w:14, h:20, hp:3.5, maxHp:3.5,
               dmgMult:1.0, hitDmg:2.0, speed:0, retreat:0,
               shootTimer: rand(0.8, 1.8) };
    default:
      return { ...base, w:14, h:14, hp:2.0, maxHp:2.0,
               dmgMult:1.0, hitDmg:2.2, speed:0.62, retreat:1.5 };
  }
}

/* ====================== ИНИЦИАЛИЗАЦИЯ ====================== */
function reset(){
  player.x = 40; player.y = 180;
  player.vx = 0; player.vy = 0;
  player.onGround = false; player.jumps = 0;
  player.light = 70; player.inv = 0; player.face = 1;
  player.dead = false;

  enemies = enemySpawns.map(s => makeEnemy(s));
  projectiles = [];
  shrooms = shroomSpawns.map(s => ({x:s.x, y:s.y, r:9, used:false, timer:0}));
  shards  = shardSpawns.map(s => ({x:s.x, y:s.y, taken:false, t:Math.random()*6.28}));

  particles = []; motes = [];
  for(let i=0;i<70;i++){
    motes.push({x:rand(0,LEVEL_W), y:rand(0,VH),
                vx:rand(-0.12,0.12), vy:rand(-0.06,0.06),
                s:Math.random()<0.3?2:1, a:rand(0.15,0.5)});
  }
  cam.x = 0; shake = 0;
  stepTimer = 0; stepSide = 0;
  flashWave = null; flashCooldown = 0;
  SFX.setIntensity(0);
}

/* ====================== ФИЗИКА ====================== */
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

function lightRadius(){
  const base = 14 + player.light * 0.34;
  const flick = 1 + Math.sin(time*9)*0.02 + Math.sin(time*23)*0.012;
  let mult = 1;
  if(focus() && player.light > 0) mult = 1.5;
  if(flashWave){
    const t = flashWave.life / flashWave.maxLife;
    mult += t * 1.8;
  }
  return base * mult * flick;
}

/* ====================== ОБНОВЛЕНИЕ ====================== */
function update(dt){
  time += dt;
  if(mutedBannerT > 0) mutedBannerT -= dt;

  if(flashCooldown > 0) flashCooldown -= dt;
  if(flashWave){
    flashWave.life -= dt;
    flashWave.r = flashWave.maxR * (1 - flashWave.life / flashWave.maxLife);
    if(flashWave.life <= 0) flashWave = null;
  }

  for(const m of motes){
    m.x += m.vx; m.y += m.vy;
    if(m.y < -5) m.y = VH+5;
    if(m.y > VH+5) m.y = -5;
  }
  for(let i=particles.length-1;i>=0;i--){
    const p = particles[i];
    p.x += p.vx; p.y += p.vy; p.vy += 0.06;
    p.life -= dt;
    if(p.life <= 0) particles.splice(i,1);
  }

  /* === R — рестарт из любого состояния === */
  restartBuffer -= dt;
  if(restartBuffer > 0){
    restartBuffer = 0;
    reset();
    state = 'play';
    return;
  }

  if(state !== 'play') return;

  /* --- управление --- */
  const speed = 2.0, accel = 0.35;
  if(left()){  player.vx -= accel; player.face = -1; }
  if(right()){ player.vx += accel; player.face =  1; }
  if(!left() && !right()) player.vx *= player.onGround ? 0.72 : 0.90;
  player.vx = clamp(player.vx, -speed, speed);

  const focusing = focus() && player.light > 0;

  /* === ВЗРЫВ СВЕТА === */
  if(focusing && !prevFocus && flashCooldown <= 0 && player.light > 15){
    SFX.flash();
    shake = Math.max(shake, 5);
    flashCooldown = 0.55;

    const cx = player.x + player.w/2;
    const cy = player.y + player.h/2;
    flashWave = { x:cx, y:cy, r:20, maxR:180, life:0.55, maxLife:0.55 };

    player.light = Math.max(0, player.light - 18);

    for(const e of enemies){
      if(e.dead) continue;
      const dx = (e.x + e.w/2) - cx;
      const dy = (e.y + e.h/2) - cy;
      const d = Math.hypot(dx, dy) || 1;
      if(d < 180){
        const power = 1 - d/180;
        e.hp -= 1.5 + 3.5 * power;
        e.flash = 1;
        e.x += (dx/d) * (3 + 5*power);
        e.y += (dy/d) * (3 + 5*power);
        if(e.hp <= 0){
          e.dead = true;
          burst(e.x+e.w/2, e.y+e.h/2, '#8ea2ff', 14, 2.4);
          player.light = Math.min(100, player.light + 4);
        }
      }
    }
  }
  prevFocus = focusing;

  jumpBuffer -= dt;
  if(jumpBuffer > 0){
    if(player.onGround){
      player.vy = -7.8; player.jumps = 1; jumpBuffer = 0;
      burst(player.x+5, player.y+14, '#6f7fd6', 5, 1.2);
      SFX.jump();
    } else if(player.jumps < 2){
      player.vy = -7.0; player.jumps = 2; jumpBuffer = 0;
      burst(player.x+5, player.y+14, '#9aa8ff', 7, 1.6);
      SFX.djump();
    }
  }

  player.vy += 0.5;
  if(player.vy > 11) player.vy = 11;

  const wasGround = player.onGround;
  const fallSpeed = player.vy;
  moveAndCollide(player);
  if(player.onGround && !wasGround && fallSpeed > 4){
    SFX.land();
    burst(player.x+5, player.y+14, '#4a5070', 4, 1.0);
    stepTimer = 0;
  }
  if(player.onGround) player.jumps = 0;
  if(player.inv > 0) player.inv -= dt;

  /* --- шаги --- */
  const walking = player.onGround && Math.abs(player.vx) > 0.5;
  if(walking){
    stepTimer -= dt;
    if(stepTimer <= 0){
      SFX.footstep(stepSide);
      stepSide ^= 1;
      stepTimer = 0.34 - Math.abs(player.vx) * 0.04;
    }
  } else {
    stepTimer = 0;
  }

  /* --- свет --- */
  const drain = (focusing ? 8.0 : 2.6) * dt;
  player.light = Math.max(0, player.light - drain);

  if(player.light <= 0){ die('Свет угас...'); return; }
  if(player.y > VH + 30){ die('Ты растворился во тьме...'); return; }

  /* --- грибы --- */
  for(const s of shrooms){
    if(s.used){
      s.timer -= dt;
      if(s.timer <= 0) s.used = false;
      continue;
    }
    if(overlap(player, {x:s.x-s.r, y:s.y-s.r, w:s.r*2, h:s.r*2})){
      s.used = true; s.timer = 14;
      player.light = Math.min(100, player.light + 55);
      burst(s.x, s.y, '#7ff0d8', 14, 2.2);
      shake = Math.max(shake, 2);
      SFX.shroom();
    }
  }

  /* --- осколки --- */
  for(const sh of shards){
    if(sh.taken) continue;
    if(overlap(player, {x:sh.x-5, y:sh.y-5, w:10, h:10})){
      sh.taken = true;
      player.light = Math.min(100, player.light + 12);
      burst(sh.x, sh.y, '#ffd76a', 9, 1.8);
      SFX.shard();
    }
  }

  /* --- враги --- */
  const R = lightRadius();
  const px = player.x + player.w/2, py = player.y + player.h/2;
  let nearestDist = 9999;

  for(const e of enemies){
    if(e.dead) continue;
    e.t += dt * 2.4;
    const ex = e.x + e.w/2, ey = e.y + e.h/2;
    const dx = px - ex, dy = py - ey;
    const dist = Math.hypot(dx, dy) || 1;
    if(dist < nearestDist) nearestDist = dist;

    const lit = dist < R;

    if(e.type === 'hopper'){
      if(lit){
        e.hp -= e.hitDmg * dt;
        e.flash = 1;
        SFX.hitEnemy();
        e.x -= (dx/dist) * e.retreat;
        e.y -= (dy/dist) * e.retreat;
        e.jumpVx *= 0.8; e.jumpVy *= 0.8;
      } else {
        e.flash = Math.max(0, e.flash - dt*2);
        e.jumpTimer -= dt;
        if(e.jumpTimer <= 0){
          e.jumpTimer = rand(0.9, 1.4);
          e.jumpVx = (dx/dist) * 3.2;
          e.jumpVy = (dy/dist) * 3.2 - 2.2;
          burst(ex, ey, '#7c5cff', 5, 1.2);
        }
        e.x += e.jumpVx * dt * 30;
        e.y += e.jumpVy * dt * 30;
        e.jumpVx *= Math.pow(0.92, dt*60);
        e.jumpVy += 6 * dt;
      }
      if(e.hp <= 0){
        e.dead = true;
        burst(ex, ey, '#c08cff', 12, 2.2);
        player.light = Math.min(100, player.light + 4);
        SFX.killEnemy();
      }
    }
    else if(e.type === 'sentinel'){
      if(lit){
        e.hp -= e.hitDmg * dt;
        e.flash = 1;
        SFX.hitEnemy();
      } else {
        e.flash = Math.max(0, e.flash - dt*2);
        e.shootTimer -= dt;
        if(e.shootTimer <= 0 && dist < 220){
          e.shootTimer = 1.8;
          projectiles.push({
            x: ex, y: ey,
            vx: (dx/dist) * 1.6,
            vy: (dy/dist) * 1.6,
            life: 3.0
          });
          burst(ex, ey, '#ff8c5a', 4, 1.2);
        }
      }
      if(e.hp <= 0){
        e.dead = true;
        burst(ex, ey, '#ff9a5c', 16, 2.6);
        player.light = Math.min(100, player.light + 6);
        SFX.killEnemy();
      }
    }
    else {
      // crawler + devourer
      if(lit){
        e.hp -= e.hitDmg * dt;
        e.flash = 1;
        SFX.hitEnemy();
        e.x -= (dx/dist) * e.retreat;
        e.y -= (dy/dist) * e.retreat;
        if(e.hp <= 0){
          e.dead = true;
          burst(ex, ey, e.type === 'devourer' ? '#7c5cff' : '#8ea2ff',
                e.type === 'devourer' ? 20 : 14, 2.4);
          player.light = Math.min(100, player.light + (e.type === 'devourer' ? 8 : 4));
          SFX.killEnemy();
        }
      } else {
        e.flash = Math.max(0, e.flash - dt*2);
        e.x += (dx/dist) * e.speed;
        e.y += (dy/dist) * e.speed;
      }
    }

    if(player.inv <= 0 && overlap(player, e)){
      const lightLoss = 11 * e.dmgMult;
      player.light = Math.max(0, player.light - lightLoss);
      player.inv = 1.1;
      const k = px < ex ? -1 : 1;
      player.vx = k * 4.2; player.vy = -3.6;
      shake = Math.max(shake, 5);
      burst(px, py, e.dmgMult > 1 ? '#ff4d7a' : '#ff6b8a', 10, 2);
      SFX.hurt();
    }
  }

  /* --- снаряды стражей --- */
  for(let i = projectiles.length - 1; i >= 0; i--){
    const p = projectiles[i];
    p.x += p.vx;
    p.y += p.vy;
    p.life -= dt;

    let hitWall = false;
    for(const pl of platforms){
      if(p.x > pl.x && p.x < pl.x+pl.w && p.y > pl.y && p.y < pl.y+pl.h){
        hitWall = true; break;
      }
    }

    if(!hitWall && player.inv <= 0 &&
       p.x > player.x && p.x < player.x+player.w &&
       p.y > player.y && p.y < player.y+player.h){
      player.light = Math.max(0, player.light - 8);
      player.inv = 1.1;
      burst(p.x, p.y, '#ff5c8a', 8, 1.8);
      SFX.hurt();
      shake = Math.max(shake, 4);
      projectiles.splice(i, 1);
      continue;
    }

    if(p.life <= 0 || hitWall){
      burst(p.x, p.y, '#8f6aff', 4, 1.0);
      projectiles.splice(i, 1);
    }
  }

  /* --- реактивная музыка --- */
  const progress = clamp(player.x / LEVEL_W, 0, 1);
  const danger   = clamp(1 - nearestDist / 150, 0, 1);
  SFX.setIntensity(clamp(progress * 0.5 + danger * 0.7, 0, 1));

  /* --- победа --- */
  if(overlap(player, ALTAR)){
    state = 'win';
    burst(ALTAR.x+17, ALTAR.y+10, '#ffe9a3', 40, 3.5);
    shake = 8;
    SFX.win();
  }

  /* --- камера --- */
  const targetX = clamp(player.x + player.w/2 - VW/2, 0, LEVEL_W - VW);
  cam.x += (targetX - cam.x) * 0.12;
  if(shake > 0) shake = Math.max(0, shake - dt*18);
}

function die(msg){
  state = 'dead';
  player.dead = true;
  shake = 9;
  burst(player.x+5, player.y+7, '#ffd76a', 30, 3);
  SFX.death();
  SFX.setIntensity(1);
  document.title = msg;
}

/* ====================== ОТРИСОВКА ====================== */
function drawBackground(){
  const g = ctx.createLinearGradient(0,0,0,VH);
  g.addColorStop(0, '#0a0d1e'); g.addColorStop(1, '#04050c');
  ctx.fillStyle = g; ctx.fillRect(0,0,VW,VH);

  ctx.fillStyle = '#0d1122';
  for(let i=0;i<22;i++){
    let bx = ((i*180 - cam.x*0.25) % (VW+400) + VW+400) % (VW+400) - 200;
    ctx.fillRect(Math.round(bx), VH-(95+(i%4)*38)-45, 32+(i%3)*20, 95+(i%4)*38);
  }
  ctx.fillStyle = '#131a2e';
  for(let i=0;i<26;i++){
    let bx = ((i*140 - cam.x*0.5) % (VW+320) + VW+320) % (VW+320) - 160;
    ctx.fillRect(Math.round(bx), VH-(55+(i%5)*32)-22, 22+(i%4)*15, 55+(i%5)*32);
  }
  const f = ctx.createLinearGradient(0, VH-70, 0, VH);
  f.addColorStop(0, 'rgba(30,40,80,0)');
  f.addColorStop(1, 'rgba(40,55,110,0.16)');
  ctx.fillStyle = f; ctx.fillRect(0, VH-70, VW, 70);
}

function drawPlatforms(){
  for(const p of platforms){
    const x = Math.round(p.x - cam.x), y = Math.round(p.y);
    if(x + p.w < -10 || x > VW + 10) continue;
    ctx.fillStyle = '#221d33'; ctx.fillRect(x, y, p.w, p.h);
    ctx.fillStyle = '#3d3560'; ctx.fillRect(x, y, p.w, 2);
    ctx.fillStyle = '#514679'; ctx.fillRect(x, y, p.w, 1);
    ctx.fillStyle = '#2c2542';
    for(let i = 0; i < p.w; i += 6){
      const h = ((p.x + i) * 7919) % 11;
      if(h < 4) ctx.fillRect(x + i, y + 4 + (h*2), 2, 2);
    }
    ctx.fillStyle = '#171327'; ctx.fillRect(x, y + p.h - 3, p.w, 3);
  }
}

function drawShrooms(){
  for(const s of shrooms){
    const x = Math.round(s.x - cam.x), y = Math.round(s.y);
    if(x < -30 || x > VW+30) continue;
    if(s.used){ ctx.fillStyle = '#2a3d3d'; ctx.fillRect(x-2, y-2, 5, 4); continue; }
    ctx.fillStyle = '#4d7f74'; ctx.fillRect(x-1, y-1, 3, 5);
    ctx.fillStyle = '#5fe0c4'; ctx.fillRect(x-4, y-5, 9, 4);
    ctx.fillStyle = '#a6fff0'; ctx.fillRect(x-4, y-5, 9, 1);
    const pulse = 0.6 + Math.sin(time*4 + s.x)*0.4;
    ctx.fillStyle = `rgba(120,255,225,${0.10*pulse})`;
    ctx.beginPath(); ctx.arc(x, y-3, 14 + pulse*3, 0, 6.283); ctx.fill();
  }
}

function drawShards(){
  for(const s of shards){
    if(s.taken) continue;
    const x = Math.round(s.x - cam.x);
    const y = Math.round(s.y + Math.sin(time*2.5 + s.t)*3);
    if(x < -20 || x > VW+20) continue;
    ctx.fillStyle = '#ffd76a'; ctx.fillRect(x-1, y-3, 2, 6); ctx.fillRect(x-3, y-1, 6, 2);
    ctx.fillStyle = '#fff3c4'; ctx.fillRect(x-1, y-1, 2, 2);
  }
}

function drawAltar(){
  const x = Math.round(ALTAR.x - cam.x), y = ALTAR.y;
  if(x < -60 || x > VW+60) return;
  ctx.fillStyle = '#2a2440'; ctx.fillRect(x-8, y+34, 50, 8); ctx.fillRect(x-4, y+28, 42, 6);
  ctx.fillStyle = '#37304f'; ctx.fillRect(x, y, 6, 34); ctx.fillRect(x+28, y, 6, 34);
  ctx.fillStyle = '#4b4270'; ctx.fillRect(x, y, 2, 34); ctx.fillRect(x+28, y, 2, 34);
  ctx.fillStyle = '#37304f'; ctx.fillRect(x, y-4, 34, 5);
  const pulse = 0.7 + Math.sin(time*3)*0.3;
  ctx.fillStyle = '#ffe9a3'; ctx.fillRect(x+14, y+10, 6, 10);
  ctx.fillStyle = `rgba(255,220,140,${0.5*pulse})`; ctx.fillRect(x+11, y+7, 12, 16);
  ctx.fillStyle = '#fffbe0'; ctx.fillRect(x+15, y+12, 4, 5);
}

function drawEnemies(){
  for(const e of enemies){
    if(e.dead) continue;
    const bob = e.type === 'hopper' ? 0 : Math.sin(e.t)*2;
    const x = Math.round(e.x - cam.x), y = Math.round(e.y + bob);
    if(x < -50 || x > VW+50) continue;

    if(e.type === 'crawler'){
      const c = e.flash > 0 ? '#3a3f78' : '#0e0e1a';
      ctx.fillStyle = c;
      ctx.beginPath(); ctx.arc(x+7, y+7, 7, 0, 6.283); ctx.fill();
      ctx.fillRect(x+1, y+9, 12, 5);
      ctx.fillRect(x-1, y+7, 3, 4);
      ctx.fillRect(x+12, y+8, 3, 4);
      ctx.fillStyle = e.flash > 0 ? '#ffffff' : '#ff4d6d';
      ctx.fillRect(x+3, y+5, 2, 2);
      ctx.fillRect(x+9, y+5, 2, 2);
    }
    else if(e.type === 'hopper'){
      const c = e.flash > 0 ? '#7c6aff' : '#1a0f2e';
      ctx.fillStyle = c;
      ctx.beginPath(); ctx.arc(x+6, y+6, 6, 0, 6.283); ctx.fill();
      ctx.fillRect(x+1, y-2, 3, 4);
      ctx.fillRect(x+8, y-2, 3, 4);
      ctx.fillStyle = e.flash > 0 ? '#ffffff' : '#c08cff';
      ctx.fillRect(x+2, y+4, 2, 2);
      ctx.fillRect(x+8, y+4, 2, 2);
    }
    else if(e.type === 'devourer'){
      const c = e.flash > 0 ? '#5c4aaa' : '#0a0a18';
      ctx.fillStyle = c;
      ctx.beginPath(); ctx.arc(x+11, y+11, 11, 0, 6.283); ctx.fill();
      ctx.fillRect(x-2, y+14, 26, 8);
      ctx.fillRect(x+1, y+20, 4, 5);
      ctx.fillRect(x+17, y+20, 4, 5);
      ctx.fillRect(x+7, y+22, 4, 4);
      ctx.fillStyle = e.flash > 0 ? '#ffffff' : '#b060ff';
      ctx.fillRect(x+4, y+8, 3, 3);
      ctx.fillRect(x+15, y+8, 3, 3);
      ctx.fillRect(x+9, y+13, 3, 3);
      ctx.fillStyle = `rgba(120,80,200,${0.10 + 0.05*Math.sin(e.t*3)})`;
      ctx.beginPath(); ctx.arc(x+11, y+11, 18, 0, 6.283); ctx.fill();
    }
    else if(e.type === 'sentinel'){
      const c = e.flash > 0 ? '#ff8060' : '#1a0f0a';
      ctx.fillStyle = c;
      ctx.fillRect(x+3, y, 8, 20);
      ctx.fillRect(x+2, y+2, 10, 4);
      ctx.fillRect(x+2, y+16, 10, 4);
      const glow = e.shootTimer < 0.4 ? '#ffdd66' : '#ff5c2a';
      ctx.fillStyle = e.flash > 0 ? '#ffffff' : glow;
      ctx.fillRect(x+5, y+8, 4, 4);
      ctx.fillStyle = 'rgba(255,100,60,0.25)';
      ctx.fillRect(x+3, y+6, 8, 8);
    }

    if(e.flash > 0){
      ctx.fillStyle = `rgba(160,180,255,${e.flash*0.35})`;
      ctx.beginPath();
      ctx.arc(x + e.w/2, y + e.h/2, e.w, 0, 6.283);
      ctx.fill();
    }
  }
}

function drawPlayer(){
  if(state === 'dead') return;
  const x = Math.round(player.x - cam.x), y = Math.round(player.y);
  if(player.inv > 0 && Math.floor(time*20) % 2 === 0) return;
  const f = player.face;

  ctx.fillStyle = '#2e2748'; ctx.fillRect(x, y+4, 10, 10);
  ctx.fillStyle = '#3b3160'; ctx.fillRect(x+1, y+2, 8, 8);
  ctx.fillStyle = '#4a3d78'; ctx.fillRect(x+1, y, 8, 5);
  ctx.fillStyle = '#5a4a8e'; ctx.fillRect(x+1, y, 8, 1);
  ctx.fillStyle = '#0a0a14'; ctx.fillRect(x+2, y+2, 6, 3);
  ctx.fillStyle = '#ffd76a';
  ctx.fillRect(x+3, y+2, 2, 2); ctx.fillRect(x+6, y+2, 2, 2);
  ctx.fillStyle = '#211c36';
  ctx.fillRect(x+1, y+13, 3, 2); ctx.fillRect(x+6, y+13, 3, 2);

  const sx = f > 0 ? x + 11 : x - 3;
  ctx.fillStyle = '#7a5c34'; ctx.fillRect(sx, y-6, 2, 16);
  ctx.fillStyle = '#9a7648'; ctx.fillRect(sx, y-6, 1, 16);
  const p2 = 0.7 + Math.sin(time*7)*0.3;
  ctx.fillStyle = '#ffe9a3'; ctx.fillRect(sx-1, y-9, 4, 4);
  ctx.fillStyle = `rgba(255,235,170,${0.55*p2})`;
  ctx.fillRect(sx-3, y-11, 8, 8);
}

function drawProjectiles(){
  for(const p of projectiles){
    const x = Math.round(p.x - cam.x), y = Math.round(p.y);
    if(x < -20 || x > VW+20) continue;
    ctx.fillStyle = 'rgba(150,110,255,0.35)';
    ctx.fillRect(x-4, y-4, 8, 8);
    ctx.fillStyle = '#c8a4ff';
    ctx.fillRect(x-2, y-2, 4, 4);
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(x-1, y-1, 2, 2);
  }
}

function drawFlashWave(){
  if(!flashWave) return;
  const x = flashWave.x - cam.x;
  const y = flashWave.y;
  const a = flashWave.life / flashWave.maxLife;

  ctx.globalCompositeOperation = 'lighter';

  ctx.strokeStyle = `rgba(255,230,160,${a*0.9})`;
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.arc(x, y, flashWave.r, 0, 6.283);
  ctx.stroke();

  ctx.strokeStyle = `rgba(170,200,255,${a*0.55})`;
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.arc(x, y, flashWave.r * 0.72, 0, 6.283);
  ctx.stroke();

  const g = ctx.createRadialGradient(x, y, 0, x, y, flashWave.r);
  g.addColorStop(0,   `rgba(255,240,190,${a*0.35})`);
  g.addColorStop(0.6, `rgba(255,200,120,${a*0.12})`);
  g.addColorStop(1,   'rgba(255,180,80,0)');
  ctx.fillStyle = g;
  ctx.beginPath();
  ctx.arc(x, y, flashWave.r, 0, 6.283);
  ctx.fill();

  ctx.globalCompositeOperation = 'source-over';
}

function drawParticles(){
  for(const p of particles){
    ctx.globalAlpha = clamp(p.life / p.max, 0, 1);
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
    ctx.fillStyle = '#8fa4ff'; ctx.fillRect(x, y, m.s, m.s);
  }
  ctx.globalAlpha = 1;
}

function drawDarkness(){
  if(state === 'win') return;
  dctx.globalCompositeOperation = 'source-over';
  dctx.clearRect(0,0,VW,VH);
  dctx.fillStyle = 'rgba(2,3,9,0.965)'; dctx.fillRect(0,0,VW,VH);
  dctx.globalCompositeOperation = 'destination-out';

  if(state !== 'dead'){
    const px = player.x + player.w/2 - cam.x, py = player.y + player.h/2;
    const R = lightRadius();
    const g = dctx.createRadialGradient(px, py, 0, px, py, R);
    g.addColorStop(0,'rgba(0,0,0,1)'); g.addColorStop(0.45,'rgba(0,0,0,0.92)');
    g.addColorStop(0.75,'rgba(0,0,0,0.45)'); g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle = g;
    dctx.beginPath(); dctx.arc(px, py, R, 0, 6.283); dctx.fill();
  }
  for(const s of shrooms){
    if(s.used) continue;
    const x = s.x - cam.x, y = s.y;
    if(x < -60 || x > VW+60) continue;
    const g = dctx.createRadialGradient(x, y-3, 0, x, y-3, 28);
    g.addColorStop(0,'rgba(0,0,0,0.85)'); g.addColorStop(0.6,'rgba(0,0,0,0.30)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle = g;
    dctx.beginPath(); dctx.arc(x, y-3, 28, 0, 6.283); dctx.fill();
  }
  {
    const x = ALTAR.x + 17 - cam.x, y = ALTAR.y + 12;
    const g = dctx.createRadialGradient(x, y, 0, x, y, 45);
    g.addColorStop(0,'rgba(0,0,0,0.9)'); g.addColorStop(0.5,'rgba(0,0,0,0.35)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle = g;
    dctx.beginPath(); dctx.arc(x, y, 45, 0, 6.283); dctx.fill();
  }
  dctx.globalCompositeOperation = 'source-over';
  ctx.drawImage(darkCv, 0, 0);

  if(state !== 'dead'){
    const px = player.x + player.w/2 - cam.x, py = player.y + player.h/2;
    const R = lightRadius();
    ctx.globalCompositeOperation = 'lighter';
    const g = ctx.createRadialGradient(px, py, 0, px, py, R*0.95);
    g.addColorStop(0,'rgba(255,205,110,0.16)'); g.addColorStop(0.5,'rgba(255,175,70,0.06)');
    g.addColorStop(1,'rgba(255,150,50,0)');
    ctx.fillStyle = g; ctx.fillRect(px-R, py-R, R*2, R*2);
    ctx.globalCompositeOperation = 'source-over';
  }
}

function drawHUD(){
  if(state === 'title') return;
  const w = 92, h = 7, x = 10, y = 10;
  ctx.fillStyle = 'rgba(6,8,18,0.75)'; ctx.fillRect(x-2, y-2, w+4, h+4);
  ctx.fillStyle = '#1b2138'; ctx.fillRect(x, y, w, h);
  const pct = player.light / 100;
  const col = pct > 0.5 ? '#ffd76a' : pct > 0.22 ? '#ff9a3c' : '#ff4d5e';
  ctx.fillStyle = col; ctx.fillRect(x, y, Math.round(w*pct), h);
  ctx.fillStyle = 'rgba(255,255,255,0.25)';
  ctx.fillRect(x, y, Math.round(w*pct), 2);
  ctx.fillStyle = '#7f8bb0'; ctx.font = '8px "Courier New", monospace';
  ctx.fillText('СВЕТ', x, y + h + 11);

  if(player.light < 25 && Math.floor(time*3) % 2 === 0){
    ctx.fillStyle = '#ff8b9c';
    ctx.fillText('НАЙДИ СВЕТЯЩИЙСЯ ГРИБ', VW/2 - 62, 24);
  }
  if(focus() && player.light > 15 && flashCooldown <= 0){
    ctx.fillStyle = '#ffe9a3';
    ctx.fillText('ВСПЫШКА ГОТОВА', VW - 96, 24);
  } else if(flashCooldown > 0){
    ctx.fillStyle = '#5a6a94';
    ctx.fillText('ПЕРЕЗАРЯДКА', VW - 82, 24);
  }

  const ix = 10, iy = 32, iw = 60, ih = 4;
  ctx.fillStyle = 'rgba(6,8,18,0.65)'; ctx.fillRect(ix-1, iy-1, iw+2, ih+2);
  ctx.fillStyle = '#1b2138'; ctx.fillRect(ix, iy, iw, ih);
  ctx.fillStyle = '#8f7bff';
  ctx.fillRect(ix, iy, Math.round(iw * SFX.getIntensity()), ih);

  const sx = VW - 12, sy = 12;
  ctx.fillStyle = SFX.isMuted() ? '#5a3040' : '#4a6a4a';
  ctx.fillRect(sx - 5, sy - 4, 6, 6);
  if(!SFX.isMuted()){
    ctx.fillStyle = '#8fe09a'; ctx.fillRect(sx - 4, sy - 3, 4, 4);
  }

  if(mutedBannerT > 0){
    ctx.globalAlpha = Math.min(1, mutedBannerT * 2);
    ctx.fillStyle = 'rgba(6,8,18,0.85)';
    ctx.fillRect(VW/2 - 42, VH - 30, 84, 16);
    ctx.fillStyle = '#ffd76a'; ctx.textAlign = 'center';
    ctx.font = '9px "Courier New", monospace';
    ctx.fillText(SFX.isMuted() ? 'ЗВУК ВЫКЛ' : 'ЗВУК ВКЛ', VW/2, VH - 19);
    ctx.textAlign = 'left'; ctx.globalAlpha = 1;
  }
}

function drawTitle(){
  ctx.fillStyle = 'rgba(3,4,12,0.86)'; ctx.fillRect(0,0,VW,VH);
  ctx.textAlign = 'center';
  ctx.fillStyle = '#ffd76a'; ctx.font = 'bold 20px "Courier New", monospace';
  ctx.fillText('ХРОНИКИ', VW/2, 70);
  ctx.fillText('ЗАБЫТОГО СВЕТА', VW/2, 96);
  ctx.fillStyle = '#5a6a94'; ctx.font = '9px "Courier New", monospace';
  ctx.fillText('Солнце погасло. Ты — последняя искра.', VW/2, 124);
  ctx.fillStyle = '#8fa0c8';
  ctx.fillText('A / D  или  ← →   — движение', VW/2, 158);
  ctx.fillText('W / ↑ / ПРОБЕЛ    — прыжок (двойной)', VW/2, 174);
  ctx.fillText('SHIFT             — ВЗРЫВ СВЕТА', VW/2, 190);
  ctx.fillText('R — заново        M — звук', VW/2, 206);
  const a = 0.5 + 0.5*Math.sin(time*3);
  ctx.globalAlpha = a;
  ctx.fillStyle = '#ffd76a'; ctx.font = 'bold 11px "Courier New", monospace';
  ctx.fillText('НАЖМИ ПРОБЕЛ', VW/2, 242);
  ctx.globalAlpha = 1; ctx.textAlign = 'left';
}

function drawDead(){
  ctx.fillStyle = 'rgba(20,0,10,0.72)'; ctx.fillRect(0,0,VW,VH);
  ctx.textAlign = 'center';
  ctx.fillStyle = '#ff6b8a'; ctx.font = 'bold 18px "Courier New", monospace';
  ctx.fillText('СВЕТ УГАС', VW/2, 118);
  ctx.fillStyle = '#8fa0c8'; ctx.font = '9px "Courier New", monospace';
  ctx.fillText('Тьма поглотила странника...', VW/2, 142);
  const a = 0.5 + 0.5*Math.sin(time*4);
  ctx.globalAlpha = a;
  ctx.fillStyle = '#ffd76a'; ctx.font = 'bold 10px "Courier New", monospace';
  ctx.fillText('R — ПОПРОБОВАТЬ СНОВА', VW/2, 178);
  ctx.globalAlpha = 1; ctx.textAlign = 'left';
}

function drawWin(){
  ctx.fillStyle = 'rgba(30,20,0,0.68)'; ctx.fillRect(0,0,VW,VH);
  ctx.textAlign = 'center';
  ctx.fillStyle = '#ffe9a3'; ctx.font = 'bold 18px "Courier New", monospace';
  ctx.fillText('ИСКРА ВОЗВРАЩЕНА', VW/2, 110);
  ctx.fillStyle = '#c9d4e8'; ctx.font = '9px "Courier New", monospace';
  ctx.fillText('Алтарь принял твой свет.', VW/2, 136);
  ctx.fillText('Но это лишь начало пути...', VW/2, 152);
  const a = 0.5 + 0.5*Math.sin(time*4);
  ctx.globalAlpha = a;
  ctx.fillStyle = '#ffd76a'; ctx.font = 'bold 10px "Courier New", monospace';
  ctx.fillText('R — ИГРАТЬ СНОВА', VW/2, 196);
  ctx.globalAlpha = 1; ctx.textAlign = 'left';
}

function render(){
  ctx.save();
  if(shake > 0.2){
    ctx.translate(Math.round(rand(-shake, shake)), Math.round(rand(-shake, shake)));
  }
  drawBackground(); drawPlatforms(); drawShrooms(); drawShards(); drawAltar();
  drawEnemies(); drawMotes(); drawPlayer(); drawProjectiles();
  drawParticles();
  drawFlashWave();
  drawDarkness(); drawHUD();
  ctx.restore();

  if(state === 'title') drawTitle();
  if(state === 'dead')  drawDead();
  if(state === 'win')   drawWin();
}

/* ====================== ЦИКЛ ====================== */
let last = performance.now(), acc = 0;
const STEP = 1/60;

function loop(now){
  requestAnimationFrame(loop);
  let dt = (now - last) / 1000; last = now;
  if(dt > 0.1) dt = 0.1;

  acc += dt;
  let guard = 0;
  while(acc >= STEP && guard++ < 5){
    if(state === 'title' && jumpBuffer > 0){
      jumpBuffer = 0;
      reset(); state = 'play'; SFX.init();
    }
    update(STEP);
    acc -= STEP;
  }
  render();
}

reset();
requestAnimationFrame(loop);

})();
</script>
</body>
</html>
