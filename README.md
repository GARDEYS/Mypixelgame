<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="theme-color" content="#03040a">
<title>Хроники Забытого Света: Полное Издание</title>
<style>
  html,body{margin:0;height:100vh;height:100dvh;background:#03040a;overflow:hidden;
    font-family:"Courier New",monospace;color:#c9d4e8;touch-action:none;
    overscroll-behavior:none;-webkit-tap-highlight-color:transparent;
    -webkit-user-select:none;user-select:none;position:fixed;width:100%}
  #wrap{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;
    padding:env(safe-area-inset-top,0) env(safe-area-inset-right,0)
           env(safe-area-inset-bottom,0) env(safe-area-inset-left,0);box-sizing:border-box}
  canvas{image-rendering:pixelated;image-rendering:crisp-edges;display:block;
    max-width:100%;max-height:100%}
  #touch{position:fixed;inset:0;pointer-events:none;z-index:10;display:none}
  #touch.on{display:block}
  .btn{position:absolute;pointer-events:auto;width:66px;height:66px;border-radius:50%;
    background:rgba(120,140,220,0.24);border:2px solid rgba(180,200,255,0.45);
    color:#dfe6f7;font:bold 22px "Courier New",monospace;
    display:flex;align-items:center;justify-content:center;
    user-select:none;-webkit-user-select:none;touch-action:none;
    box-shadow:0 0 12px rgba(120,140,220,0.2);transition:background 0.06s,transform 0.06s}
  .btn.press{background:rgba(200,220,255,0.55);transform:scale(0.94)}
  #bL{left:calc(14px + env(safe-area-inset-left,0));bottom:calc(110px + env(safe-area-inset-bottom,0))}
  #bR{left:calc(94px + env(safe-area-inset-left,0));bottom:calc(110px + env(safe-area-inset-bottom,0))}
  #bJ{right:calc(14px + env(safe-area-inset-right,0));bottom:calc(130px + env(safe-area-inset-bottom,0));width:82px;height:82px;font-size:26px}
  #bF{right:calc(110px + env(safe-area-inset-right,0));bottom:calc(60px + env(safe-area-inset-bottom,0));width:62px;height:62px;font-size:16px}
  #bRst{right:calc(14px + env(safe-area-inset-right,0));top:calc(14px + env(safe-area-inset-top,0));width:44px;height:44px;font-size:14px}
  #bMute{right:calc(14px + env(safe-area-inset-right,0));top:calc(66px + env(safe-area-inset-top,0));width:44px;height:44px;font-size:16px}
  #bP{right:calc(14px + env(safe-area-inset-right,0));top:calc(118px + env(safe-area-inset-top,0));width:44px;height:44px;font-size:18px}
  @media (max-height:420px){
    .btn{width:56px;height:56px;font-size:18px}
    #bJ{width:72px;height:72px;font-size:22px}
    #bF{width:54px;height:54px;font-size:14px}
    #bRst,#bMute,#bP{width:38px;height:38px;font-size:12px}
  }
</style>
</head>
<body>
<div id="wrap"><canvas id="c"></canvas></div>
<div id="touch">
  <div class="btn" id="bL">◀</div>
  <div class="btn" id="bR">▶</div>
  <div class="btn" id="bJ">⤒</div>
  <div class="btn" id="bF">☀</div>
  <div class="btn" id="bRst">R</div>
  <div class="btn" id="bMute">♪</div>
  <div class="btn" id="bP">⏸</div>
</div>
<script>
(()=>{
"use strict";

/* ==================== SFX ==================== */
const SFX=(()=>{
  let ac=null,master=null,input=null,reverb=null,dry=null,wet=null;
  let amb=null,mus=null,lB=null,lP=null,lA=null,lR=null;
  let on=true,mt=false,nb=null,lastHit=0,mv=0.85;
  let mi=0,mis=0,mTimer=null,nnt=0,stp=0;
  const BPM=72,SD=60/BPM/2,LA=0.12,IT=25,TS=32;
  const N={'A2':110,'F2':87.31,'C3':130.81,'G2':98,'F3':174.61,'A3':220,'B3':246.94,
    'C4':261.63,'D4':293.66,'E4':329.63,'F4':349.23,'G4':392,'A4':440,'B4':493.88,
    'C5':523.25,'D5':587.33,'E5':659.25,'G5':783.99,'A5':880};
  const M=['A4',null,'E5',null,'C5',null,'A4',null,'F4',null,'C5',null,'A4',null,'F4',null,
    'C4',null,'G4',null,'E5',null,'C5',null,'G3',null,'D4',null,'B4',null,'G4',null];
  const B=['A2',null,null,null,'A2',null,null,null,'F2',null,null,null,'F2',null,null,null,
    'C3',null,null,null,'C3',null,null,null,'G2',null,null,null,'G2',null,null,null];
  const PD=[['A3','C4','E4'],['F3','A3','C4'],['C4','E4','G4'],['G3','B3','D4']];
  const AR=['A5','C5','E5','G5'];
  const MF=['A4','C5','E5','D5','C5'];
  
  function ens(){
    if(!on)return null;
    if(ac){if(ac.state==='suspended')ac.resume();return ac;}
    try{
      ac=new (window.AudioContext||window.webkitAudioContext)();
      master=ac.createGain();master.gain.value=0.34;master.connect(ac.destination);
      reverb=ac.createConvolver();reverb.buffer=imp(ac,2.6,3.2);
      input=ac.createGain();dry=ac.createGain();dry.gain.value=0.78;
      wet=ac.createGain();wet.gain.value=0.62;
      input.connect(dry);dry.connect(master);
      input.connect(wet);wet.connect(reverb);reverb.connect(master);
      mus=ac.createGain();mus.gain.value=0.0001;mus.connect(input);
      lB=ac.createGain();lB.gain.value=1;lP=ac.createGain();lP.gain.value=0;
      lA=ac.createGain();lA.gain.value=0;lR=ac.createGain();lR.gain.value=0;
      lB.connect(mus);lP.connect(mus);lA.connect(mus);lR.connect(mus);
    }catch(e){on=false;}
    return ac;
  }
  function imp(c,dur,dc){
    const r=c.sampleRate,len=Math.floor(r*dur),buf=c.createBuffer(2,len,r);
    for(let ch=0;ch<2;ch++){const d=buf.getChannelData(ch);
      for(let i=0;i<len;i++){const t=i/len;let v=(Math.random()*2-1)*Math.pow(1-t,dc);
        if(i<r*0.04)v*=1.7;d[i]=v;}}
    return buf;
  }
  function gN(c){
    if(nb)return nb;
    const len=Math.floor(c.sampleRate*0.6);
    nb=c.createBuffer(1,len,c.sampleRate);
    const d=nb.getChannelData(0);
    for(let i=0;i<len;i++)d[i]=Math.random()*2-1;
    return nb;
  }
  function en(g,t,a,pk,dc){
    g.gain.cancelScheduledValues(t);g.gain.setValueAtTime(0.0001,t);
    g.gain.exponentialRampToValueAtTime(Math.max(0.0002,pk),t+a);
    g.gain.exponentialRampToValueAtTime(0.0001,t+a+dc);
  }
  function tn(f,dur,ty,vol,sl,dl){
    const c=ens();if(!c||mt)return;
    const t=c.currentTime+(dl||0);
    const o=c.createOscillator(),g=c.createGain();
    o.type=ty||'square';o.frequency.setValueAtTime(f,t);
    if(sl)o.frequency.exponentialRampToValueAtTime(Math.max(20,sl),t+dur);
    en(g,t,0.005,vol||0.25,dur);
    o.connect(g);g.connect(input);o.start(t);o.stop(t+dur+0.06);
  }
  function nz(dur,vol,fr,to,q,dl){
    const c=ens();if(!c||mt)return;
    const t=c.currentTime+(dl||0);
    const src=c.createBufferSource();src.buffer=gN(c);
    const fl=c.createBiquadFilter();fl.type='lowpass';fl.Q.value=q||1.2;
    fl.frequency.setValueAtTime(fr||3000,t);
    fl.frequency.exponentialRampToValueAtTime(Math.max(60,to||180),t+dur);
    const g=c.createGain();en(g,t,0.004,vol||0.18,dur);
    src.connect(fl);fl.connect(g);g.connect(input);
    src.start(t);src.stop(t+dur+0.05);
  }
  function stAmb(){
    const c=ens();if(!c||amb)return;
    const t=c.currentTime;
    amb=c.createGain();amb.gain.setValueAtTime(0.0001,t);
    amb.gain.exponentialRampToValueAtTime(0.055,t+4);amb.connect(input);
    const lp=c.createBiquadFilter();lp.type='lowpass';lp.frequency.value=320;lp.connect(amb);
    [110,110.7,164.8].forEach((f,i)=>{
      const o=c.createOscillator();o.type=i===2?'triangle':'sine';o.frequency.value=f;
      const g=c.createGain();g.gain.value=i===2?0.35:0.55;
      o.connect(g);g.connect(lp);o.start(t);});
    const l=c.createOscillator();l.frequency.value=0.06;
    const lg=c.createGain();lg.gain.value=0.018;
    l.connect(lg);lg.connect(amb.gain);l.start(t);
  }
  function pn(f,t,dur,ty,vol,dst){
    const o=ac.createOscillator();o.type=ty;o.frequency.value=f;
    const g=ac.createGain();g.gain.setValueAtTime(0.0001,t);
    g.gain.exponentialRampToValueAtTime(vol,t+0.015);
    g.gain.setValueAtTime(vol,t+dur*0.55);
    g.gain.exponentialRampToValueAtTime(0.0001,t+dur);
    o.connect(g);g.connect(dst);o.start(t);o.stop(t+dur+0.05);
  }
  function kck(t){
    const o=ac.createOscillator(),g=ac.createGain();
    o.frequency.setValueAtTime(120,t);o.frequency.exponentialRampToValueAtTime(40,t+0.12);
    g.gain.setValueAtTime(0.0001,t);g.gain.exponentialRampToValueAtTime(0.20,t+0.005);
    g.gain.exponentialRampToValueAtTime(0.0001,t+0.18);
    o.connect(g);g.connect(lR);o.start(t);o.stop(t+0.22);
  }
  function ht(t){
    const s=ac.createBufferSource();s.buffer=gN(ac);
    const f=ac.createBiquadFilter();f.type='highpass';f.frequency.value=6500;
    const g=ac.createGain();g.gain.setValueAtTime(0.0001,t);
    g.gain.exponentialRampToValueAtTime(0.055,t+0.003);
    g.gain.exponentialRampToValueAtTime(0.0001,t+0.05);
    s.connect(f);f.connect(g);g.connect(lR);s.start(t);s.stop(t+0.1);
  }
  function sSt(i,t){
    const m=M[i];if(m){pn(N[m],t,SD*1.7,'triangle',0.16,lB);pn(N[m]*2,t,SD*0.9,'sine',0.045,lB);}
    const b=B[i];if(b){pn(N[b],t,SD*3.2,'sawtooth',0.10,lB);pn(N[b]/2,t,SD*3.2,'sine',0.08,lB);}
    if(i%8===0){const ch=PD[Math.floor(i/8)];if(ch)ch.forEach(f=>pn(N[f],t,SD*7.6,'sine',0.06,lP));}
    if(i%2===1){const n=AR[Math.floor(i/2)%AR.length];pn(N[n],t,SD*0.75,'square',0.038,lA);}
    if(i%4===0)kck(t);if(i%4===2)ht(t);
  }
  function uL(){
    mis+=(mi-mis)*0.06;const I=mis;
    lP.gain.value=Math.min(1,I*1.1);
    lA.gain.value=Math.max(0,(I-0.35)/0.65);
    lR.gain.value=Math.max(0,(I-0.55)/0.45);
  }
  function sch(){
    if(!ac||mt)return;uL();
    if(nnt<ac.currentTime)nnt=ac.currentTime+0.05;
    let g=0;
    while(nnt<ac.currentTime+LA&&g++<32){sSt(stp,nnt);nnt+=SD;stp=(stp+1)%TS;}
  }
  function stM(){
    const c=ens();if(!c||mTimer)return;
    stp=0;nnt=c.currentTime+0.15;mTimer=setInterval(sch,IT);
    const t=c.currentTime;mus.gain.cancelScheduledValues(t);
    mus.gain.setValueAtTime(Math.max(0.0001,mus.gain.value),t);
    mus.gain.exponentialRampToValueAtTime(mv,t+3.5);
  }
  return {
    init(){ens();stAmb();stM();},
    setIntensity(v){mi=v<0?0:v>1?1:v;},getIntensity(){return mis;},
    toggleMute(){mt=!mt;if(master)master.gain.value=mt?0:0.34;return mt;},
    isMuted(){return mt;},
    setMusicVolume(v){mv=v;if(mus&&!mt)mus.gain.value=v;},getMusicVolume(){return mv;},
    motif(){MF.forEach((f,i)=>{tn(N[f],0.35,'triangle',0.14,null,i*0.14);tn(N[f]*2,0.25,'sine',0.04,null,i*0.14+0.02);});},
    choir(){[523.25,659.25,783.99].forEach((f,i)=>{tn(f,0.5,'sine',0.09,null,i*0.03);tn(f*1.5,0.4,'triangle',0.03,null,i*0.05);});},
    jump(){tn(320,0.10,'square',0.16,520);},
    djump(){tn(480,0.12,'square',0.15,760);tn(720,0.09,'triangle',0.09,900,0.03);},
    land(){nz(0.10,0.16,900,90,1.0);},
    dash(){nz(0.15,0.18,2000,400,1.2);tn(600,0.12,'sine',0.12,900);},
    chargeUp(){tn(220,0.4,'sine',0.08,440);},
    chargedFlash(){nz(0.4,0.25,4000,300,1.8);tn(440,0.3,'triangle',0.18,880);tn(880,0.25,'sine',0.12,1760,0.05);},
    wallBreak(){nz(0.3,0.22,1200,100,1.5);tn(150,0.2,'sawtooth',0.16,60);},
    heartbeat(rate){tn(60,0.15,'sine',0.12+rate*0.05,30);tn(55,0.12,'sine',0.08+rate*0.03,25,0.15);},
    footstep(s,sf){
      const c=ens();if(!c||mt)return;const t=c.currentTime;
      const src=c.createBufferSource();src.buffer=gN(c);
      const fl=c.createBiquadFilter();fl.type='lowpass';
      let f0=s?780:1000;
      if(sf==='water')f0=1400;else if(sf==='ice')f0=1800;
      else if(sf==='moss')f0=500;else if(sf==='metal')f0=2200;
      fl.frequency.setValueAtTime(f0,t);
      fl.frequency.exponentialRampToValueAtTime(sf==='water'?300:150,t+0.06);
      const g=c.createGain();g.gain.setValueAtTime(0.0001,t);
      g.gain.exponentialRampToValueAtTime(0.055,t+0.004);
      g.gain.exponentialRampToValueAtTime(0.0001,t+0.07);
      src.connect(fl);fl.connect(g);g.connect(input);src.start(t);src.stop(t+0.1);
    },
    flash(){nz(0.28,0.20,5000,400,1.5);
      tn(880,0.22,'triangle',0.10,1600,0.01);
      tn(1320,0.18,'sine',0.06,2200,0.03);
      [523.25,659.25,783.99].forEach((f,i)=>tn(f,0.4,'sine',0.06,null,i*0.02));},
    iceFlash(){nz(0.4,0.18,3000,600,1.5);
      tn(1400,0.35,'sine',0.12,2200,0.01);
      tn(2100,0.3,'triangle',0.08,2800,0.04);},
    bolt(){nz(0.15,0.2,6000,800,1.8);
      tn(1800,0.15,'sawtooth',0.14,600,0.01);
      tn(900,0.25,'square',0.10,200,0.05);},
    freeze(){nz(0.2,0.12,800,400);tn(300,0.3,'sine',0.10,150);},
    drown(){nz(0.5,0.2,400,80,1.2);tn(80,0.6,'sine',0.15,40);},
    burn(){nz(0.35,0.2,1800,200,1.4);tn(200,0.4,'sawtooth',0.15,80);},
    wind(){nz(0.7,0.15,500,200,0.9);},
    hitEnemy(){const n=performance.now();if(n-lastHit<55)return;lastHit=n;
      nz(0.06,0.13,2400,500);tn(420,0.05,'square',0.08,260);},
    killEnemy(){tn(260,0.18,'sawtooth',0.18,90);nz(0.16,0.14,1800,200,1.1);},
    hurt(){tn(180,0.30,'sawtooth',0.22,70);nz(0.22,0.18,1400,120,1.0);},
    shroom(){[523.25,659.25,783.99,1046.5].forEach((f,i)=>tn(f,0.22,'triangle',0.14,null,i*0.05));},
    shard(){tn(880,0.07,'triangle',0.14,1320);tn(1320,0.12,'sine',0.10,null,0.05);},
    death(){tn(220,0.9,'sawtooth',0.22,40);nz(0.8,0.16,1200,60,0.9);tn(110,1.2,'sine',0.14,30,0.1);},
    bossRoar(){tn(80,1.4,'sawtooth',0.28,50);tn(60,1.6,'sine',0.18,40,0.05);nz(0.9,0.20,900,80,0.9);},
    bossHit(){nz(0.12,0.14,3000,400);tn(300,0.10,'square',0.12,180);},
    bossPhase(){tn(120,0.6,'sawtooth',0.24,60);tn(180,0.4,'square',0.14,90,0.1);nz(0.5,0.18,2200,200,1.2);},
    storyBlip(){tn(660,0.05,'triangle',0.10,720);},
    storyNext(){tn(440,0.10,'triangle',0.12,660);},
    achievement(){[880,1175,1568].forEach((f,i)=>tn(f,0.3,'triangle',0.15,null,i*0.08));},
    whisper(){nz(0.6,0.05,600,200,0.8);},
    diary(){tn(440,0.4,'triangle',0.15,550);tn(660,0.5,'sine',0.08,null,0.1);},
    secretReveal(){nz(0.3,0.15,1200,300,1.0);tn(300,0.2,'sawtooth',0.12,150);tn(150,0.35,'sine',0.10,80,0.05);},
    trialStart(){[300,400,500].forEach((f,i)=>tn(f,0.2,'square',0.14,null,i*0.08));tn(700,0.3,'sawtooth',0.10,500,0.2);},
    trialEnd(){[500,650,800,1000].forEach((f,i)=>tn(f,0.25,'triangle',0.15,null,i*0.08));},
    menuSelect(){tn(660,0.08,'triangle',0.14,880);},
    menuMove(){tn(440,0.05,'square',0.10,520);},
    piston(){nz(0.2,0.18,800,200,1.2);tn(150,0.15,'square',0.14,100);},
    upgrade(){[440,554,659,880].forEach((f,i)=>tn(f,0.25,'triangle',0.12,null,i*0.1));},
    win(){[523.25,659.25,783.99,1046.5,1318.5].forEach((f,i)=>{
      tn(f,0.4,'triangle',0.16,null,i*0.11);tn(f*2,0.3,'sine',0.05,null,i*0.11+0.02);});
      nz(0.6,0.10,6000,800,1.4,0.05);}
  };
})();

/* ==================== КАНВАС ==================== */
const BASE_H=270,MIN_VW=320,MAX_VW=720,MAX_SCALE=4.5;
let VW=480,VH=BASE_H;
const cvs=document.getElementById('c'),ctx=cvs.getContext('2d');
const darkCv=document.createElement('canvas'),dctx=darkCv.getContext('2d');
function resize(){
  const cW=Math.max(1,innerWidth),cH=Math.max(1,innerHeight);
  let sc=cH/BASE_H;if(sc>MAX_SCALE)sc=MAX_SCALE;if(sc<1)sc=1;
  let vw=Math.ceil(cW/sc);
  if(vw<MIN_VW)vw=MIN_VW;if(vw>MAX_VW)vw=MAX_VW;
  VW=vw;VH=BASE_H;
  cvs.width=VW;cvs.height=VH;ctx.imageSmoothingEnabled=false;
  const ds=Math.min(cW/VW,cH/VH);
  cvs.style.width=Math.floor(VW*ds)+'px';cvs.style.height=Math.floor(VH*ds)+'px';
  darkCv.width=VW;darkCv.height=VH;dctx.imageSmoothingEnabled=false;
}
addEventListener('resize',resize);
addEventListener('orientationchange',()=>{setTimeout(resize,150);setTimeout(resize,450);});
resize();

/* ==================== ВВОД И НАСТРОЙКИ КЛАВИШ ==================== */
const DEFAULT_KEYS={left:'KeyA',right:'KeyD',jump:'Space',focus:'ShiftLeft',dash:'ShiftLeft'};
let keyMap={...DEFAULT_KEYS};
const keys={};
let jBuf=0,rBuf=0,cBuf=0,pBuf=0,paused=false;
let mUB=0,mDB=0,mLB=0,mRB=0,mBB=0;
let prevF=false,mBannerT=0,staffBuffer=0;
let focusHoldTime=0; // Для заряженной вспышки

// Маппинг действий
const act=k=>{
  if(k===keyMap.left||k==='ArrowLeft')return'left';
  if(k===keyMap.right||k==='ArrowRight')return'right';
  if(k===keyMap.jump||k==='ArrowUp'||k==='Space')return'jump';
  if(k===keyMap.focus||k==='ShiftLeft'||k==='ShiftRight')return'focus';
  return null;
};

addEventListener('keydown',e=>{
  if(['ArrowLeft','ArrowRight','ArrowUp','ArrowDown','Space'].includes(e.code))e.preventDefault();
  SFX.init();
  if(keys[e.code])return;keys[e.code]=true;
  const a=act(e.code);
  if(a==='jump'){jBuf=0.14;cBuf=0.14;}
  if(e.code==='KeyR'||e.code==='Enter')rBuf=0.14;
  if(e.code==='KeyP'||e.code==='Escape'){pBuf=0.14;mBB=0.14;}
  if(e.code==='ArrowUp')mUB=0.14;
  if(e.code==='ArrowDown')mDB=0.14;
  if(act(e.code)==='left')mLB=0.14;
  if(act(e.code)==='right')mRB=0.14;
  if(e.code==='KeyM'){SFX.toggleMute();mBannerT=1.6;}
  if(e.code==='Digit1')staffBuffer=1;
  if(e.code==='Digit2')staffBuffer=2;
  if(e.code==='Digit3')staffBuffer=3;
});
addEventListener('keyup',e=>{keys[e.code]=false;});

const lft=()=>keys[keyMap.left]||keys['ArrowLeft'];
const rgt=()=>keys[keyMap.right]||keys['ArrowRight'];
const foc=()=>keys[keyMap.focus]||keys['ShiftLeft']||keys['ShiftRight'];
const jmp=()=>keys[keyMap.jump]||keys['ArrowUp']||keys['Space'];

(function touch(){
  const t=('ontouchstart' in window)||(navigator.maxTouchPoints>0);
  if(!t)return;
  document.getElementById('touch').classList.add('on');
  function b(id,dn,up){
    const el=document.getElementById(id);
    const pr=e=>{e.preventDefault();SFX.init();el.classList.add('press');dn&&dn();};
    const rl=e=>{e.preventDefault();el.classList.remove('press');up&&up();};
    el.addEventListener('touchstart',pr,{passive:false});
    el.addEventListener('touchend',rl,{passive:false});
    el.addEventListener('touchcancel',rl,{passive:false});
    el.addEventListener('mousedown',pr);el.addEventListener('mouseup',rl);
    el.addEventListener('mouseleave',rl);
  }
  b('bL',()=>{keys[keyMap.left]=true;mLB=0.14;},()=>{keys[keyMap.left]=false;});
  b('bR',()=>{keys[keyMap.right]=true;mRB=0.14;},()=>{keys[keyMap.right]=false;});
  b('bJ',()=>{keys[keyMap.jump]=true;jBuf=0.14;cBuf=0.14;},()=>{keys[keyMap.jump]=false;});
  b('bF',()=>{keys[keyMap.focus]=true;},()=>{keys[keyMap.focus]=false;});
  b('bRst',()=>{rBuf=0.14;cBuf=0.14;},null);
  b('bMute',()=>{SFX.toggleMute();mBannerT=1.6;},null);
  b('bP',()=>{pBuf=0.14;mBB=0.14;},null);
})();

/* ==================== ДНЕВНИКИ ==================== */
const DIARIES=[
  {id:'d1',level:1,x:220,y:170,title:'Запись I — Начало',secret:true,text:'«Солнце угасло не за один день. Оно уходило медленно, как свеча в сырой комнате.»'},
  {id:'d2',level:1,x:1010,y:120,title:'Запись II — Колодец',text:'«Вода в колодцах застыла. Хлеб не поднимался. Я видел, как люди ели землю.»'},
  {id:'d3',level:1,x:1500,y:90,title:'Запись III — Искра',secret:true,text:'«Говорят, Искра передаётся по крови. Но у меня не было детей. Только посох.»'},
  {id:'d4',level:'1.5',x:400,y:150,title:'Запись IV — Лес',text:'«Деревья здесь шепчут. Они помнят свет. И они голодны.»'},
  {id:'d5',level:'1.5',x:1200,y:120,title:'Запись V — Хранитель',text:'«Я видел его. Он был как мы — когда-то. Но свет сжёг его изнутри.»'},
  {id:'d6',level:'1.5',x:1750,y:100,title:'Запись VI — Молитва',secret:true,text:'«Если ты читаешь это — значит дошёл дальше меня. Не отдавай Искру.»'},
  {id:'d7',level:4,x:400,y:170,title:'Запись VII — Потоп',text:'«Море поглотило наш город за одну ночь. Мы думали — тьма спасёт нас. Не спасла.»'},
  {id:'d8',level:4,x:1400,y:110,title:'Запись VIII — Глубина',secret:true,text:'«Здесь, в глубине, свет тонет вместе с тобой. Держись за пузырь. Держись за жизнь.»'},
  {id:'d9',level:5,x:600,y:150,title:'Запись IX — Лёд',text:'«Холод не убивает. Он сохраняет. Пока ты идёшь — ты живой.»'},
  {id:'d10',level:5,x:1800,y:120,title:'Запись X — Ветер',secret:true,text:'«Ветер здесь сдувает даже свет. Прячься в паузах — и дыши.»'},
  {id:'d11',level:6,x:400,y:150,title:'Запись XI — Кузнец',text:'«Кузнец выковал Хранителя. Он сам стал пеплом. Но его молот ещё стучит.»'},
  {id:'d12',level:2,x:820,y:150,title:'Запись XII — Последний',secret:true,text:'«Когда я умру, свет не угаснет. Он будет жить, пока ты идёшь. Спасибо, Люмен.»'}
];

/* ==================== ДОСТИЖЕНИЯ ==================== */
const ACHS=[
  {id:'first_light',name:'Первый свет',desc:'Активируй Алтарь'},
  {id:'all_diaries',name:'Хроностраник',desc:'Собери все 12 дневников'},
  {id:'no_damage',name:'Не тронь меня',desc:'Пройди уровень без урона'},
  {id:'fast',name:'Скороход',desc:'Уровень 1 за 3 минуты'},
  {id:'pyromancer',name:'Пироман',desc:'20 вспышек за уровень'},
  {id:'trial_master',name:'Испытание',desc:'Пройди комнату-испытание'},
  {id:'boss_slayer',name:'Убийца Хранителя',desc:'Убей Хранителя Пепла'},
  {id:'smith_slayer',name:'Кузнец мёртв',desc:'Убей Кузнеца Тьмы'},
  {id:'secret_hunter',name:'Искатель',desc:'Найди 3 секретные комнаты'},
  {id:'ice_master',name:'Ледяной',desc:'Заморозь 10 врагов'},
  {id:'storm',name:'Гроза',desc:'Убей молнией 3 врага одним ударом'},
  {id:'true_sun',name:'Истинный Свет',desc:'Пройди без смертей'},
  {id:'ng_plus',name:'Угасающее Солнце',desc:'Заверши NG+'},
  {id:'wall_breaker',name:'Разрушитель',desc:'Разбей 5 стен'},
  {id:'shadow_dancer',name:'Танцующий с тенью',desc:'Убей 10 сталкеров'}
];

/* ==================== АПГРЕЙДЫ ==================== */
const UPGRADES=[
  {id:'light_max',name:'Яркая искра',desc:'+20 макс. свет',cost:10,max:3,icon:'☀'},
  {id:'drain_res',name:'Стойкость',desc:'-15% утечка света',cost:15,max:3,icon:'🛡'},
  {id:'jump_pow',name:'Легкость',desc:'+10% высота прыжка',cost:12,max:2,icon:'⤒'},
  {id:'dash_cd',name:'Быстрый шаг',desc:'-20% КД рывка',cost:20,max:2,icon:'💨'}
];

/* ==================== УРОВНИ ==================== */
const LEVELS={
  1:{
    width:2600,groundY:230,spawn:{x:40,y:180},surface:'stone',
    staff:'fire',theme:'default',
    platforms:[
      {x:0,y:230,w:400,h:40},{x:455,y:230,w:250,h:40},{x:760,y:230,w:170,h:40},
      {x:985,y:230,w:400,h:40},{x:1440,y:230,w:300,h:40},{x:1795,y:230,w:805,h:40},
      {x:250,y:180,w:70,h:10},{x:390,y:148,w:60,h:10},{x:545,y:175,w:80,h:10},
      {x:695,y:140,w:70,h:10},{x:845,y:178,w:90,h:10},{x:990,y:140,w:80,h:10},
      {x:1140,y:168,w:90,h:10},{x:1300,y:132,w:80,h:10},{x:1440,y:162,w:90,h:10},
      {x:1600,y:178,w:90,h:10},{x:1760,y:138,w:100,h:10},{x:1950,y:178,w:90,h:10},
      {x:2100,y:135,w:110,h:10},{x:2280,y:172,w:100,h:10}
    ],
    breakables:[{x:1350,y:132,w:40,h:40,drop:'shard'},{x:2150,y:135,w:40,h:40,drop:'secret'}],
    secrets:[{x:200,y:200,w:14,h:30},{x:1490,y:120,w:14,h:42}],
    shrooms:[{x:150,y:222},{x:600,y:222},{x:1050,y:222},{x:1520,y:222},
      {x:1900,y:222},{x:2400,y:222},{x:415,y:140},{x:1015,y:132},{x:1795,y:130}],
    shards:[{x:300,y:150},{x:760,y:150},{x:1220,y:200},{x:1660,y:150},
      {x:2020,y:190},{x:2250,y:110},{x:1340,y:100}],
    lanterns:[{x:700,y:180},{x:1600,y:150}],
    enemies:[
      {x:330,y:180,type:'crawler'},{x:560,y:195,type:'crawler'},
      {x:730,y:185,type:'hopper'},{x:900,y:160,type:'crawler'},
      {x:1120,y:180,type:'sentinel'},{x:1320,y:200,type:'mimic'},
      {x:1530,y:170,type:'devourer'},{x:1720,y:190,type:'crawler'},
      {x:1950,y:160,type:'swarm'},{x:2120,y:185,type:'sentinel'},
      {x:2300,y:170,type:'devourer'},{x:1800,y:160,type:'leech'}
    ],
    trialZone:{x:1560,y:180,w:80,h:50,spawns:[
      {x:1600,y:200,type:'crawler'},{x:1640,y:200,type:'crawler'},
      {x:1600,y:150,type:'hopper'},{x:1660,y:150,type:'hopper'},
      {x:1560,y:160,type:'swarm'},{x:1700,y:180,type:'swarm'}
    ]},
    altar:{x:2500,y:186,w:34,h:44},hasBoss:false
  },
  '1.5':{
    width:1900,groundY:230,spawn:{x:40,y:180},surface:'moss',
    staff:'fire',theme:'forest',
    platforms:[
      {x:0,y:230,w:300,h:40},{x:380,y:230,w:220,h:40},
      {x:680,y:230,w:280,h:40},{x:1040,y:230,w:200,h:40},
      {x:1320,y:230,w:580,h:40},
      {x:150,y:170,w:80,h:10},{x:280,y:140,w:70,h:10},
      {x:450,y:180,w:80,h:10},{x:560,y:150,w:80,h:10},
      {x:720,y:180,w:80,h:10},{x:840,y:140,w:80,h:10},
      {x:980,y:170,w:70,h:10},{x:1120,y:150,w:80,h:10},
      {x:1240,y:180,w:80,h:10},{x:1400,y:150,w:90,h:10},
      {x:1540,y:120,w:90,h:10},{x:1700,y:160,w:80,h:10}
    ],
    breakables:[{x:880,y:140,w:40,h:40,drop:'shard'}],
    secrets:[{x:1730,y:130,w:14,h:42}],
    shrooms:[{x:100,y:222},{x:440,y:222},{x:740,y:222},{x:1100,y:222},
      {x:1500,y:222},{x:1800,y:222},{x:580,y:142},{x:860,y:132},{x:1570,y:112}],
    shards:[{x:200,y:160},{x:480,y:170},{x:1000,y:160},{x:1440,y:140},{x:1740,y:150}],
    lanterns:[{x:900,y:180},{x:1400,y:150}],
    enemies:[
      {x:250,y:190,type:'crawler'},{x:420,y:190,type:'hopper'},
      {x:600,y:140,type:'mimic'},{x:760,y:190,type:'crawler'},
      {x:920,y:150,type:'swarm'},{x:1080,y:190,type:'hopper'},
      {x:1260,y:180,type:'sentinel'},{x:1480,y:140,type:'crawler'},
      {x:1650,y:170,type:'swarm'},{x:1780,y:190,type:'devourer'},
      {x:500,y:180,type:'stalker'},{x:1300,y:160,type:'stalker'}
    ],
    altar:null,hasBoss:false,exit:{x:1850,y:186,w:34,h:44}
  },
  4:{
    width:2200,groundY:230,spawn:{x:40,y:180},surface:'water',
    staff:'ice',theme:'water',rain:true,
    platforms:[
      {x:0,y:230,w:280,h:40},{x:360,y:230,w:180,h:40},
      {x:620,y:230,w:220,h:40},{x:920,y:230,w:200,h:40},
      {x:1200,y:230,w:160,h:40},{x:1440,y:230,w:240,h:40},
      {x:1760,y:230,w:440,h:40},
      {x:140,y:170,w:80,h:10},{x:280,y:140,w:70,h:10},
      {x:420,y:180,w:80,h:10},{x:540,y:150,w:80,h:10},
      {x:680,y:180,w:80,h:10},{x:800,y:140,w:80,h:10},
      {x:950,y:170,w:70,h:10},{x:1080,y:140,w:80,h:10},
      {x:1240,y:180,w:80,h:10},{x:1380,y:150,w:90,h:10},
      {x:1560,y:180,w:90,h:10},{x:1720,y:140,w:90,h:10},
      {x:1880,y:170,w:100,h:10},{x:2030,y:150,w:80,h:10}
    ],
    breakables:[{x:1150,y:140,w:40,h:40,drop:'shard'}],
    secrets:[{x:450,y:180,w:14,h:30},{x:1990,y:130,w:14,h:40}],
    shrooms:[{x:110,y:222},{x:400,y:222},{x:660,y:222},{x:960,y:222},
      {x:1260,y:222},{x:1500,y:222},{x:1850,y:222},{x:2100,y:222},
      {x:570,y:142},{x:1110,y:132},{x:1750,y:132}],
    shards:[{x:200,y:170},{x:500,y:160},{x:850,y:170},{x:1200,y:160},{x:1600,y:130},{x:2050,y:140}],
    lanterns:[{x:700,y:180},{x:1300,y:180}],
    currents:[{x:900,y:150,w:100,h:100,vx:1.5,vy:0}],
    enemies:[
      {x:180,y:200,type:'murena'},{x:400,y:190,type:'crawler'},
      {x:600,y:150,type:'jellyfish'},{x:780,y:200,type:'murena'},
      {x:940,y:180,type:'jellyfish'},{x:1100,y:150,type:'crawler'},
      {x:1300,y:200,type:'murena'},{x:1500,y:170,type:'jellyfish'},
      {x:1700,y:200,type:'devourer'},{x:1900,y:170,type:'murena'},
      {x:2100,y:200,type:'crawler'},{x:1000,y:180,type:'leech'}
    ],
    altar:null,hasBoss:false,exit:{x:2150,y:186,w:34,h:44}
  },
  5:{
    width:2400,groundY:230,spawn:{x:40,y:180},surface:'ice',
    staff:'ice',theme:'ice',snow:true,
    platforms:[
      {x:0,y:230,w:240,h:40},{x:320,y:230,w:160,h:40},
      {x:560,y:230,w:140,h:40},{x:780,y:230,w:180,h:40},
      {x:1040,y:230,w:160,h:40},{x:1280,y:230,w:200,h:40},
      {x:1560,y:230,w:160,h:40},{x:1800,y:230,w:600,h:40},
      {x:100,y:170,w:80,h:10},{x:240,y:140,w:70,h:10},
      {x:400,y:170,w:80,h:10},{x:520,y:130,w:80,h:10},
      {x:660,y:160,w:80,h:10},{x:800,y:120,w:80,h:10},
      {x:940,y:150,w:70,h:10},{x:1080,y:120,w:80,h:10},
      {x:1220,y:150,w:80,h:10},{x:1360,y:120,w:90,h:10},
      {x:1520,y:150,w:80,h:10},{x:1660,y:120,w:80,h:10},
      {x:1820,y:150,w:90,h:10},{x:1980,y:120,w:100,h:10},
      {x:2140,y:150,w:80,h:10},{x:2280,y:180,w:80,h:10}
    ],
    breakables:[{x:1400,y:120,w:40,h:40,drop:'shard'}],
    secrets:[{x:2200,y:110,w:14,h:40}],
    shrooms:[{x:120,y:222},{x:360,y:222},{x:600,y:222},{x:840,y:222},
      {x:1100,y:222},{x:1350,y:222},{x:1620,y:222},{x:1900,y:222},{x:2250,y:222},
      {x:540,y:122},{x:1100,y:112},{x:2000,y:112}],
    shards:[{x:250,y:130},{x:700,y:150},{x:950,y:140},{x:1240,y:140},
      {x:1550,y:140},{x:1850,y:140},{x:2160,y:140}],
    lanterns:[{x:800,y:180},{x:1600,y:180}],
    enemies:[
      {x:200,y:200,type:'iceWolf'},{x:400,y:200,type:'crawler'},
      {x:600,y:150,type:'iceWolf'},{x:750,y:200,type:'crawler'},
      {x:950,y:140,type:'iceWolf'},{x:1150,y:200,type:'hopper'},
      {x:1400,y:200,type:'iceWolf'},{x:1600,y:180,type:'sentinel'},
      {x:1800,y:200,type:'iceWolf'},{x:2000,y:150,type:'swarm'},
      {x:2200,y:200,type:'devourer'},{x:1200,y:180,type:'stalker'}
    ],
    icicles:[{x:350,y:180},{x:900,y:180},{x:1450,y:180},{x:2050,y:180}],
    altar:null,hasBoss:false,exit:{x:2340,y:186,w:34,h:44}
  },
  6:{
    width:1800,groundY:230,spawn:{x:40,y:180},surface:'metal',
    staff:'lightning',theme:'forge',
    platforms:[
      {x:0,y:230,w:200,h:40},{x:280,y:230,w:180,h:40},
      {x:540,y:230,w:160,h:40},{x:780,y:230,w:180,h:40},
      {x:1040,y:230,w:160,h:40},{x:1300,y:230,w:500,h:40},
      {x:140,y:170,w:70,h:10},{x:280,y:140,w:80,h:10},
      {x:440,y:180,w:80,h:10},{x:580,y:150,w:80,h:10},
      {x:720,y:120,w:80,h:10},{x:860,y:150,w:80,h:10},
      {x:1000,y:120,w:80,h:10},{x:1160,y:150,w:80,h:10},
      {x:1340,y:120,w:90,h:10},{x:1480,y:170,w:80,h:10}
    ],
    lava:[{x:200,y:260,w:80,h:20},{x:460,y:260,w:80,h:20},
          {x:700,y:260,w:80,h:20},{x:940,y:260,w:100,h:20},
          {x:1200,y:260,w:100,h:20}],
    hot:[{x:1040,y:230,w:160,h:40}],
    pistons:[{x:640,y:130,w:20,h:50,phase:0,speed:1.5,amp:30}],
    breakables:[{x:1200,y:150,w:40,h:40,drop:'shard'}],
    secrets:[{x:1470,y:140,w:14,h:40}],
    shrooms:[{x:130,y:222},{x:350,y:222},{x:600,y:222},{x:850,y:222},
      {x:1100,y:222},{x:1450,y:222},{x:1650,y:222},
      {x:310,y:132},{x:890,y:142},{x:1370,y:112}],
    shards:[{x:170,y:160},{x:480,y:170},{x:740,y:110},{x:1120,y:140},{x:1520,y:160}],
    lanterns:[{x:800,y:180},{x:1400,y:180}],
    enemies:[
      {x:230,y:200,type:'smith'},{x:400,y:200,type:'spark'},
      {x:560,y:200,type:'crawler'},{x:700,y:150,type:'spark'},
      {x:900,y:200,type:'smith'},{x:1100,y:140,type:'spark'},
      {x:1300,y:200,type:'crawler'},{x:1500,y:200,type:'spark'},
      {x:1650,y:200,type:'devourer'},{x:600,y:180,type:'leech'}
    ],
    altar:null,hasBoss:true,
    boss:{x:1650,y:160,type:'smith'}
  },
  2:{
    width:900,groundY:230,spawn:{x:60,y:180},surface:'stone',
    staff:'all',theme:'boss',
    platforms:[
      {x:0,y:230,w:900,h:40},{x:80,y:170,w:90,h:10},
      {x:730,y:170,w:90,h:10},{x:340,y:130,w:70,h:10},{x:490,y:130,w:70,h:10}
    ],
    secrets:[{x:790,y:160,w:14,h:42}],
    shrooms:[{x:130,y:222},{x:770,y:222},{x:280,y:162},{x:620,y:162},{x:450,y:122}],
    shards:[{x:375,y:105},{x:525,y:105}],
    lanterns:[{x:400,y:180}],
    enemies:[],altar:null,hasBoss:true,boss:{x:720,y:166,type:'keeper'}
  }
};

/* ==================== СЮЖЕТ ==================== */
const STORY={
  intro:[
    {art:'stars',lines:['Тысячу лет назад солнце погасло.','Королевство Аврора погрузилось во тьму.']},
    {art:'void',lines:['Люди забыли тепло.','Дети рождались, не увидев света.']},
    {art:'hero',lines:['Ты — Люмен, последний носитель','Искры Первого Пламени.']},
    {art:'staff',lines:['Найди Алтарь Древних.','Зажги солнце заново — или растворись во тьме.']}
  ],
  interlude:[
    {art:'altar',lines:['Первый Алтарь вспыхнул.','Но свет не вернулся — лишь тень отступила на шаг.']},
    {art:'void',lines:['Из глубины поднялся Хранитель Пепла —','тот, кто пожрал само Солнце.']},
    {art:'hero',lines:['Но путь к нему долог.','Затонувший город, льды, кузня...']}
  ],
  interlude2:[
    {art:'void',lines:['Кузнец Тьмы пал.','Молот замолчал навсегда.']},
    {art:'altar',lines:['Но Хранитель стал лишь сильнее —','гнев его отца питает его.']},
    {art:'hero',lines:['Сердце Тьмы ждёт.','Иди. Ты почти дошёл.']}
  ],
  ending:[
    {art:'altar',lines:['Хранитель пал.','Его тело рассыпалось искрами, и тьма отступила.']},
    {art:'staff',lines:['Ты поднял посох. Последний луч','вырвался из него и коснулся неба.']},
    {art:'sun',lines:['Солнце вернулось.','Дети впервые увидели свет.']},
    {art:'hero',lines:['Но ты знал: где-то в глубине, во тьме,','оно снова ждёт своего часа.']},
    {art:'sun',lines:['КОНЕЦ','','Но свет всегда возвращается.']}
  ],
  ng_intro:[
    {art:'void',lines:['Свет вернулся... но тьма стала гуще.','Угасающее Солнце взошло над миром.']},
    {art:'hero',lines:['В этом мире враги жаждут света сильнее.','Каждая искра на вес золота.']},
    {art:'staff',lines:['Докажи, что ты достоин Истинного Света.','Выживи в вечной ночи.']}
  ]
};

/* ==================== СОСТОЯНИЕ ==================== */
let state='title',titleMenu='main',titleCursor=0;
let storyPhase='intro',storySlide=0,storyTimer=0,storyDone=false;
let level=1,time=0,cam={x:0},shake=0;
let deaths=0,runTime=0,deathsThisLevel=0,damagedThisLevel=false,flashesThisLevel=0;
let secretsFound=0,frozenKills=0,stalkerKills=0,wallsBroken=0;
let bestTimes={},diariesFound=[],achievements=[],skins=['default'],
  currentSkin='default',difficulty='normal';
let staffType='fire';
let windTimer=0,windActive=0,windDir=1;
let currency=0; // Осколки как валюта
let upgrades={light_max:0,drain_res:0,jump_pow:0,dash_cd:0};
let ngPlus=false,ngPlusComplete=false;
let idleTimer=0,compassTarget=null;
let heartTimer=0;
let rain=[];
let snowTracks=[];

const player={x:40,y:180,w:10,h:14,vx:0,vy:0,onGround:false,jumps:0,face:1,
  light:70,maxLight:100,inv:0,dead:false,coyote:0,frozen:0,bubble:0,
  dashCd:0,dashing:0,wallSlide:false};

let enemies=[],shrooms=[],shards=[],lanterns=[],particles=[],motes=[],
  projectiles=[],secrets=[],icePlatforms=[],hammers=[],silhouette=null,
  currents=[],lavaZones=[],hotZones=[],pistons=[],icicles=[],breakables=[];
let platforms=[],LEVEL_W=2600,GROUND_Y=230,ALTAR=null,EXIT=null;
let boss=null,bossIntroT=0,stepTimer=0,stepSide=0;
let flashWave=null,flashCooldown=0,chainBolts=[];
let whisperT=0,whisperMsg='';
let trialActive=false,trialTimer=0,trialZone=null;
let achievementPopup=null;
let levelSurface='stone',levelTheme='default';
let hitFlash=0;

/* ==================== УТИЛИТЫ ==================== */
const rnd=(a,b)=>a+Math.random()*(b-a);
const clamp=(v,a,b)=>v<a?a:v>b?b:v;
function overlap(a,b){return a.x<b.x+b.w&&a.x+a.w>b.x&&a.y<b.y+b.h&&a.y+a.h>b.y;}
function burst(x,y,c,n,spd){
  for(let i=0;i<n;i++){const a=Math.random()*Math.PI*2,s=rnd(0.4,1)*(spd||2);
    particles.push({x,y,vx:Math.cos(a)*s,vy:Math.sin(a)*s-0.4,life:rnd(0.3,0.8),max:0.8,color:c});}
}
function unlockAch(id){
  if(achievements.includes(id))return;
  achievements.push(id);
  const a=ACHS.find(x=>x.id===id);
  if(a){achievementPopup={name:a.name,desc:a.desc,t:3};SFX.achievement();}
  saveGame();
}

/* ==================== СОХРАНЕНИЕ ==================== */
function saveGame(){
  try{localStorage.setItem('fl_save2',JSON.stringify({
    level,deaths,bestTimes,diariesFound,achievements,skins,currentSkin,difficulty,secretsFound,
    currency,upgrades,ngPlus,ngPlusComplete,stalkerKills,wallsBroken
  }));}catch(e){}
}
function loadGame(){
  try{const s=JSON.parse(localStorage.getItem('fl_save2')||'{}');
    if(s.level)level=s.level;if(s.deaths)deaths=s.deaths;
    if(s.bestTimes)bestTimes=s.bestTimes;if(s.diariesFound)diariesFound=s.diariesFound;
    if(s.achievements)achievements=s.achievements;if(s.skins)skins=s.skins;
    if(s.currentSkin)currentSkin=s.currentSkin;if(s.difficulty)difficulty=s.difficulty;
    if(s.secretsFound)secretsFound=s.secretsFound;
    if(s.currency!==undefined)currency=s.currency;
    if(s.upgrades)upgrades={...upgrades,...s.upgrades};
    if(s.ngPlus)ngPlus=s.ngPlus;if(s.ngPlusComplete)ngPlusComplete=s.ngPlusComplete;
    if(s.stalkerKills)stalkerKills=s.stalkerKills;
    if(s.wallsBroken)wallsBroken=s.wallsBroken;
  }catch(e){}
}
loadGame();

/* ==================== ФАБРИКИ ==================== */
function makeEnemy(sp){
  const b={x:sp.x,y:sp.y,type:sp.type,t:Math.random()*6.28,dead:false,flash:0,frozen:0};
  switch(sp.type){
    case 'crawler':return{...b,w:14,h:14,hp:2.0,dmgMult:1.0,hitDmg:2.2,speed:0.62,retreat:1.5};
    case 'hopper':return{...b,w:12,h:12,hp:1.6,dmgMult:1.0,hitDmg:2.6,speed:0.35,retreat:1.8,
      jumpTimer:rnd(0.4,1.0),jumpVx:0,jumpVy:0};
    case 'devourer':return{...b,w:22,h:22,hp:5.5,dmgMult:2.0,hitDmg:1.5,speed:0.32,retreat:0.9};
    case 'sentinel':return{...b,w:14,h:20,hp:3.5,dmgMult:1.0,hitDmg:2.0,speed:0,retreat:0,
      shootTimer:rnd(0.8,1.8)};
    case 'mimic':return{...b,w:12,h:12,hp:1.5,dmgMult:1.0,hitDmg:2.4,speed:0.5,retreat:1.6,
      revealed:false,revealT:0,baseX:sp.x,baseY:sp.y};
    case 'swarm':return{...b,w:8,h:8,hp:0.9,dmgMult:0.5,hitDmg:2.5,speed:0.75,retreat:1.2,
      swarmPhase:Math.random()*6.28};
    case 'murena':return{...b,w:18,h:10,hp:2.5,dmgMult:1.0,hitDmg:2.4,speed:1.0,retreat:1.6,
      zig:Math.random()*6.28};
    case 'jellyfish':return{...b,w:14,h:16,hp:2.2,dmgMult:1.0,hitDmg:2.8,speed:0,retreat:0,
      pullRad:70};
    case 'iceWolf':return{...b,w:16,h:12,hp:1.4,dmgMult:1.0,hitDmg:2.5,speed:1.3,retreat:2.0};
    case 'smith':return{...b,w:16,h:18,hp:4.0,dmgMult:1.0,hitDmg:2.2,speed:0,retreat:0,
      throwTimer:rnd(1.5,2.5)};
    case 'spark':return{...b,w:10,h:10,hp:1.2,dmgMult:1.0,hitDmg:2.8,speed:0.9,retreat:1.4,
      orb:Math.random()*6.28,orbR:40,baseX:sp.x,baseY:sp.y};
    case 'stalker':return{...b,w:14,h:16,hp:2.8,dmgMult:2.0,hitDmg:3.0,speed:1.8,retreat:2.5,
      visible:false};
    case 'leech':return{...b,w:16,h:16,hp:3.0,dmgMult:0,hitDmg:0,speed:0.3,retreat:0,
      auraRad:60};
    default:return{...b,w:14,h:14,hp:2.0,dmgMult:1.0,hitDmg:2.2,speed:0.62,retreat:1.5};
  }
}
function makeBoss(sp){
  const base={x:sp.x,y:sp.y,phase:1,state:'idle',stateT:0,idleDur:1.4,
    flash:0,vx:0,vy:0,dead:false,facing:-1,hitFlash:0,flashDone:false,type:sp.type,frozen:0,
    attackCount:0};
  if(sp.type==='smith'){
    return{...base,w:48,h:56,hp:45,maxHp:45,idleDur:1.0};
  }
  return{...base,w:56,h:64,hp:60,maxHp:60,idleDur:1.4};
}

/* ==================== ЗАГРУЗКА УРОВНЯ ==================== */
function loadLevel(n){
  level=n;
  const L=LEVELS[n];
  if(!L)return;
  LEVEL_W=L.width;GROUND_Y=L.groundY;levelSurface=L.surface||'stone';
  levelTheme=L.theme||'default';
  platforms=L.platforms.slice();ALTAR=L.altar;EXIT=L.exit||null;
  
  // Применяем апгрейды
  player.maxLight=100+upgrades.light_max*20;
  player.x=L.spawn.x;player.y=L.spawn.y;
  player.vx=0;player.vy=0;player.onGround=false;player.jumps=0;
  player.light=n===2?player.maxLight:Math.min(player.maxLight,70);
  player.inv=0;player.face=1;player.dead=false;
  player.coyote=0;player.frozen=0;player.bubble=0;
  player.dashCd=0;player.dashing=0;player.wallSlide=false;
  
  staffType=L.staff||'fire';
  enemies=L.enemies.map(s=>makeEnemy(s));
  projectiles=[];hammers=[];chainBolts=[];icePlatforms=[];
  shrooms=L.shrooms.map(s=>({x:s.x,y:s.y,r:9,used:false,timer:0}));
  shards=L.shards.map(s=>({x:s.x,y:s.y,taken:false,t:Math.random()*6.28}));
  lanterns=(L.lanterns||[]).map(l=>({x:l.x,y:l.y,lit:false}));
  secrets=(L.secrets||[]).map(s=>({...s,done:false,flash:0}));
  breakables=(L.breakables||[]).map(b=>({...b,broken:false}));
  currents=(L.currents||[]).slice();
  lavaZones=(L.lava||[]).slice();
  hotZones=(L.hot||[]).slice();
  pistons=(L.pistons||[]).map(p=>({...p,y0:p.y}));
  icicles=(L.icicles||[]).map(i=>({x:i.x,y:i.y,vy:0,falling:false,respawn:0,y0:i.y}));
  boss=L.hasBoss?makeBoss(L.boss):null;bossIntroT=0;
  if(L.trialZone){
    const t=L.trialZone;
    trialZone={x:t.x,y:t.y,w:t.w,h:t.h,triggered:achievements.includes('trial_master'),spawns:t.spawns};
    trialActive=false;trialTimer=0;
  }else{trialZone=null;trialActive=false;}
  particles=[];motes=[];snowTracks=[];
  rain=[];
  if(L.rain){for(let i=0;i<80;i++)rain.push({x:rnd(0,VW),y:rnd(-VH,VH),s:rnd(3,6),sp:rnd(4,7)});}
  
  const mc=Math.min(90,Math.max(40,Math.floor(VW*0.18)));
  for(let i=0;i<mc;i++)motes.push({x:rnd(0,Math.max(LEVEL_W,VW*2)),y:rnd(0,VH),
    vx:rnd(-0.12,0.12),vy:rnd(-0.06,0.06),s:Math.random()<0.3?2:1,a:rnd(0.15,0.5)});
  cam.x=0;shake=0;stepTimer=0;stepSide=0;
  flashWave=null;flashCooldown=0;paused=false;hitFlash=0;
  silhouette=null;whisperT=0;windTimer=6;windActive=0;windDir=1;
  deathsThisLevel=0;damagedThisLevel=false;flashesThisLevel=0;
  runTime=0;idleTimer=0;compassTarget=null;heartTimer=0;
  focusHoldTime=0;
  jBuf=0;rBuf=0;cBuf=0;pBuf=0;prevF=false;staffBuffer=0;
  SFX.setIntensity(0);
}

/* ==================== ФИЗИКА ==================== */
function moveAndCollide(e){
  e.x+=e.vx;
  if(e===player){
    for(const s of secrets){
      if(s.done)continue;
      if(overlap(e,s)){
        const pushR=(s.x+e.w)>s.x&&player.vx>0;
        const pushL=e.x<s.x+s.w&&player.vx<0;
        if(pushR||pushL){s.done=true;s.flash=1;SFX.secretReveal();
          burst(s.x+s.w/2,s.y+s.h/2,'#c9a05a',20,2.5);secretsFound++;
          if(secretsFound>=3)unlockAch('secret_hunter');saveGame();}
      }
    }
  }
  
  for(const p of platforms){
    if(overlap(e,p)){
      if(e.vx > 0) e.x = p.x - e.w;
      else if(e.vx < 0) e.x = p.x + p.w;
      else {
        const ol=(e.x+e.w)-p.x,or2=(p.x+p.w)-e.x;
        if(ol<or2)e.x=p.x-e.w;else e.x=p.x+p.w;
      }
      e.vx = 0;
    }
  }
  for(const ip of icePlatforms){
    if(overlap(e,ip)){
      if(e.vx > 0) e.x = ip.x - e.w;
      else if(e.vx < 0) e.x = ip.x + ip.w;
      else {
        const ol=(e.x+e.w)-ip.x,or2=(ip.x+ip.w)-e.x;
        if(ol<or2)e.x=ip.x-e.w;else e.x=ip.x+ip.w;
      }
      e.vx = 0;
    }
  }

  e.y+=e.vy;e.onGround=false;
  for(const p of platforms){if(overlap(e,p)){
    if(e.vy>0){e.y=p.y-e.h;e.onGround=true;}
    else if(e.vy<0){e.y=p.y+p.h;}e.vy=0;}}
  for(const ip of icePlatforms){if(overlap(e,ip)){
    if(e.vy>0){e.y=ip.y-e.h;e.onGround=true;}
    else if(e.vy<0){e.y=ip.y+ip.h;}e.vy=0;}}
}

function lightRadius(){
  const base=14+player.light*0.34;
  const flick=1+Math.sin(time*9)*0.02+Math.sin(time*23)*0.012;
  let mult=1;
  if(foc()&&player.light>0)mult=1.5;
  if(flashWave){const t=flashWave.life/flashWave.maxLife;mult+=t*1.8;}
  return base*mult*flick;
}

/* ==================== БОССЫ ==================== */
function bossSpread(){
  const cx=boss.x+boss.w/2,cy=boss.y+boss.h/2;
  const px=player.x+player.w/2,py=player.y+player.h/2;
  const ba=Math.atan2(py-cy,px-cx);
  const cnt=boss.phase===1?5:7,sp=0.6;
  for(let i=0;i<cnt;i++){const a=ba+(i/(cnt-1)-0.5)*sp;
    projectiles.push({x:cx,y:cy,vx:Math.cos(a)*1.7,vy:Math.sin(a)*1.7,life:4,from:'boss'});}
  burst(cx,cy,'#ff8c5a',8,1.6);SFX.bossHit();
}
function bossSummon(){
  const cnt=boss.phase===1?2:3;
  for(let i=0;i<cnt;i++){
    const sx=boss.x+rnd(-60,60),sy=boss.y+30;
    const type=boss.type==='smith'?(Math.random()<0.5?'spark':'crawler'):
      (Math.random()<0.6?'crawler':'hopper');
    enemies.push(makeEnemy({x:sx,y:sy,type}));
    burst(sx+7,sy+7,'#c08cff',10,1.8);}
  SFX.bossRoar();
}
function bossSlam(){boss.vy=-6.5;boss.vx=0;burst(boss.x+boss.w/2,boss.y+boss.h,'#ff6a2a',12,2.2);}
function bossSlamImpact(){
  const cx=boss.x+boss.w/2,cy=boss.y+boss.h;
  const cnt=boss.phase===1?8:12;
  for(let i=0;i<cnt;i++){const a=(i/cnt)*Math.PI*2;
    projectiles.push({x:cx,y:cy,vx:Math.cos(a)*1.5,vy:Math.sin(a)*1.5,life:3,from:'boss'});}
  shake=Math.max(shake,8);SFX.bossPhase();
  for(let i=0;i<24;i++)burst(cx,cy,'#ff9a5c',1,3);
}
function bossCharge(){
  const px=player.x+player.w/2,cx=boss.x+boss.w/2;
  boss.vx=(px<cx?-1:1)*3.4;boss.facing=px<cx?-1:1;
  burst(boss.x+boss.w/2,boss.y+boss.h-4,'#ff6a2a',14,2);SFX.bossRoar();
}
function smithThrow(){
  const cx=boss.x+boss.w/2,cy=boss.y+boss.h/2;
  const px=player.x+player.w/2,py=player.y+player.h/2;
  const dx=px-cx,dy=py-cy,d=Math.hypot(dx,dy)||1;
  hammers.push({x:cx,y:cy,vx:(dx/d)*2.2,vy:(dy/d)*2.2-1.5,life:4,rot:0});
  SFX.bossHit();
}
function smithWave(){
  const cx=boss.x+boss.w/2,cy=boss.y+boss.h/2;
  for(let i=0;i<6;i++){
    const a=(i/6)*Math.PI*2;
    projectiles.push({x:cx,y:cy,vx:Math.cos(a)*1.9,vy:Math.sin(a)*1.9,life:3.5,from:'boss'});}
  SFX.bossPhase();
}
function updateBoss(dt){
  if(!boss||boss.dead)return;
  boss.hitFlash=Math.max(0,boss.hitFlash-dt*2);
  boss.flash=Math.max(0,boss.flash-dt*2);
  if(boss.phase===1&&boss.hp<=boss.maxHp*0.5){
    boss.phase=2;boss.idleDur=0.75;shake=Math.max(shake,10);SFX.bossPhase();
    burst(boss.x+boss.w/2,boss.y+boss.h/2,'#ff9a5c',30,3);
    player.vx=(player.x<boss.x?-1:1)*5;player.vy=-5;player.inv=1.2;
  }
  const cx=boss.x+boss.w/2,cy=boss.y+boss.h/2;
  const px=player.x+player.w/2,py=player.y+player.h/2;
  const d=Math.hypot(px-cx,py-cy)||1;
  const R=lightRadius();
  if(d<R*1.15){
    boss.hp-=3.0*dt;boss.flash=1;
    if(Math.random()<0.15)SFX.hitEnemy();
  }
  if(flashWave&&!boss.flashDone&&flashWave.life>flashWave.maxLife-0.1){
    const fd=Math.hypot(cx-flashWave.x,cy-flashWave.y);
    if(fd<180){
      const p=1-fd/180;
      if(staffType==='fire')boss.hp-=4+8*p;
      else if(staffType==='ice'){boss.hp-=2+4*p;boss.frozen=1.5;}
      else if(staffType==='lightning')boss.hp-=3+6*p;
      boss.hitFlash=1;shake=Math.max(shake,6);
    }
    boss.flashDone=true;
  }
  if(boss.frozen>0)boss.frozen-=dt;
  if(boss.hp<=0){
    boss.dead=true;shake=16;SFX.killEnemy();SFX.bossRoar();
    for(let i=0;i<60;i++)burst(cx,cy,i%2?'#ff9a5c':'#ffe9a3',1,4);
    if(boss.type==='smith'){
      unlockAch('smith_slayer');
      setTimeout(()=>{if(state==='play'&&!paused)goToInterlude2();},1500);
    } else {
      unlockAch('boss_slayer');
      setTimeout(()=>{if(state==='play'&&!paused)winLevel2();},1500);
    }
    return;
  }
  if(boss.frozen>0){boss.vx=0;return;}
  boss.stateT-=dt;
  if(boss.state==='idle'){boss.vx*=0.85;
    if(boss.stateT<=0){boss.attackCount++;
      const roll=Math.random();
      if(boss.type==='smith'){
        if(roll<0.5)boss.state='throw';
        else if(roll<0.8)boss.state='wave';
        else boss.state='charge';
      } else {
        if(boss.phase===1){
          if(roll<0.55)boss.state='spread';
          else if(roll<0.85)boss.state='summon';
          else boss.state='charge';
        } else {
          if(roll<0.35)boss.state='spread';
          else if(roll<0.55)boss.state='summon';
          else if(roll<0.75)boss.state='charge';
          else boss.state='slam';
        }
      }
      boss.stateT=boss.state==='idle'?boss.idleDur:0.6;}}
  else if(boss.state==='spread'){
    if(boss.stateT<0.55&&!boss._fired){bossSpread();boss._fired=true;}
    if(boss.stateT<=0){boss.state='idle';boss.stateT=boss.idleDur;boss._fired=false;}}
  else if(boss.state==='summon'){
    if(boss.stateT<0.4&&!boss._fired){bossSummon();boss._fired=true;}
    if(boss.stateT<=0){boss.state='idle';boss.stateT=boss.idleDur;boss._fired=false;}}
  else if(boss.state==='throw'){
    if(boss.stateT<0.4&&!boss._fired){smithThrow();boss._fired=true;}
    if(boss.stateT<=0){boss.state='idle';boss.stateT=boss.idleDur;boss._fired=false;}}
  else if(boss.state==='wave'){
    if(boss.stateT<0.4&&!boss._fired){smithWave();boss._fired=true;}
    if(boss.stateT<=0){boss.state='idle';boss.stateT=boss.idleDur;boss._fired=false;}}
  else if(boss.state==='charge'){
    if(!boss._fired){bossCharge();boss._fired=true;}
    if(boss.x<=4||boss.x+boss.w>=LEVEL_W-4)boss.stateT=0;
    if(boss.stateT<=0){boss.state='idle';boss.stateT=boss.idleDur;boss._fired=false;boss.vx=0;}}
  else if(boss.state==='slam'){
    if(!boss._fired){bossSlam();boss._fired=true;}
    if(boss.onGround&&boss.vy===0&&boss.stateT<0.5){bossSlamImpact();
      boss.state='idle';boss.stateT=boss.idleDur;boss._fired=false;}
    if(boss.stateT<=0){boss.state='idle';boss.stateT=boss.idleDur;boss._fired=false;}}
  boss.vy+=0.55;if(boss.vy>12)boss.vy=12;
  boss.x+=boss.vx;boss.y+=boss.vy;
  boss.onGround=false;
  for(const p of platforms){if(overlap(boss,p)){
    if(boss.vy>0){boss.y=p.y-boss.h;boss.onGround=true;boss.vy=0;}
    else if(boss.vy<0){boss.y=p.y+p.h;boss.vy=0;}}}
  if(boss.x<4){boss.x=4;boss.vx=0;}
  if(boss.x+boss.w>LEVEL_W-4){boss.x=LEVEL_W-4-boss.w;boss.vx=0;}
  if(player.inv<=0&&overlap(player,boss)&&boss.frozen<=0){
    player.light=Math.max(0,player.light-20);player.inv=1.3;
    const k=px<cx?-1:1;player.vx=k*5;player.vy=-4;
    shake=Math.max(shake,7);burst(px,py,'#ff4d7a',14,2.4);SFX.hurt();damagedThisLevel=true;}
}

/* ==================== МЕНЮ ==================== */
function updateTitleMenu(dt){
  let opts=[];
  if(titleMenu==='main'){
    opts=['Продолжить','Новая игра'];
    if(ngPlus||ngPlusComplete)opts.push('Угасающее Солнце');
    opts.push('Костер','Скины','Настройки','Сброс');
  } else if(titleMenu==='skins') opts=skins.map(s=>skinName(s));
  else if(titleMenu==='settings') opts=['Сложность: '+diffName(difficulty),'Громкость музыки','Клавиши','Назад'];
  else if(titleMenu==='bonfire') opts=UPGRADES.map(u=>`${u.icon} ${u.name} (${upgrades[u.id]}/${u.max}) - ${u.cost*(upgrades[u.id]+1)}💎`);
  else if(titleMenu==='keys') opts=['Прыжок: '+keyMap.jump,'Фокус/Рывок: '+keyMap.focus,'Сброс клавиш','Назад'];
  
  if(!opts.length)return;
  mUB-=dt;mDB-=dt;
  if(mUB>0){mUB=0;titleCursor=(titleCursor-1+opts.length)%opts.length;SFX.menuMove();}
  if(mDB>0){mDB=0;titleCursor=(titleCursor+1)%opts.length;SFX.menuMove();}
  
  if(titleMenu==='settings'&&titleCursor===1){
    mLB-=dt;mRB-=dt;
    if(mLB>0){mLB=0;SFX.setMusicVolume(Math.max(0,SFX.getMusicVolume()-0.1));SFX.menuMove();}
    if(mRB>0){mRB=0;SFX.setMusicVolume(Math.min(1,SFX.getMusicVolume()+0.1));SFX.menuMove();}
  }
  
  if(cBuf>0){
    cBuf=0;SFX.menuSelect();
    if(titleMenu==='main'){
      if(titleCursor===0){loadLevel(level||1);state='play';SFX.motif();}
      else if(titleCursor===1){level=1;runTime=0;deaths=0;currency=0;
        upgrades={light_max:0,drain_res:0,jump_pow:0,dash_cd:0};ngPlus=false;
        loadLevel(1);storyPhase='intro';storySlide=0;storyTimer=0;storyDone=false;state='story';}
      else if(titleCursor===2&&opts[titleCursor]==='Угасающее Солнце'){
        ngPlus=true;level=1;loadLevel(1);storyPhase='ng_intro';storySlide=0;storyTimer=0;storyDone=false;state='story';}
      else if(opts[titleCursor]==='Костер'){titleMenu='bonfire';titleCursor=0;}
      else if(opts[titleCursor]==='Скины'){titleMenu='skins';titleCursor=0;}
      else if(opts[titleCursor]==='Настройки'){titleMenu='settings';titleCursor=0;}
      else if(opts[titleCursor]==='Сброс'){
        try{localStorage.removeItem('fl_save2');}catch(e){}
        achievements=[];diariesFound=[];skins=['default'];currentSkin='default';
        deaths=0;level=1;secretsFound=0;currency=0;ngPlus=false;ngPlusComplete=false;
        upgrades={light_max:0,drain_res:0,jump_pow:0,dash_cd:0};saveGame();
      }
    } else if(titleMenu==='skins'){currentSkin=skins[titleCursor];saveGame();}
    else if(titleMenu==='bonfire'){
      const u=UPGRADES[titleCursor];
      const cost=u.cost*(upgrades[u.id]+1);
      if(upgrades[u.id]<u.max&&currency>=cost){
        currency-=cost;upgrades[u.id]++;SFX.upgrade();saveGame();
      }
    }
    else if(titleMenu==='settings'){
      if(titleCursor===0){difficulty=difficulty==='easy'?'normal':difficulty==='normal'?'hard':'easy';saveGame();}
      else if(titleCursor===2){titleMenu='keys';titleCursor=0;}
      else if(titleCursor===3){titleMenu='main';titleCursor=0;}
    }
    else if(titleMenu==='keys'){
      if(titleCursor===2){keyMap={...DEFAULT_KEYS};saveGame();}
      else if(titleCursor===3){titleMenu='settings';titleCursor=2;}
    }
  }
  if(mBB>0){mBB=0;if(titleMenu!=='main'){titleMenu='main';titleCursor=0;SFX.menuMove();}}
}
function skinName(s){
  return s==='default'?'Обычный':s==='shadow'?'Тень':s==='gold'?'Золотой':
    s==='phantom'?'Призрак':s==='eclipse'?'Затмение':s;
}
function diffName(d){return d==='easy'?'Новичок':d==='normal'?'Обычный':d==='hard'?'Хардкор':d;}

/* ==================== ОБНОВЛЕНИЕ ==================== */
function update(dt){
  time+=dt;
  if(mBannerT>0)mBannerT-=dt;
  if(hitFlash>0)hitFlash-=dt;
  if(flashCooldown>0)flashCooldown-=dt;
  if(flashWave){flashWave.life-=dt;
    flashWave.r=flashWave.maxR*(1-flashWave.life/flashWave.maxLife);
    if(flashWave.life<=0)flashWave=null;}
  if(bossIntroT>0)bossIntroT-=dt;
  if(achievementPopup){achievementPopup.t-=dt;if(achievementPopup.t<=0)achievementPopup=null;}
  if(paused&&state!=='play')paused=false;
  pBuf-=dt;
  if(pBuf>0){pBuf=0;if(state==='play'){paused=!paused;SFX.storyNext();}}
  if(state==='play')runTime+=dt;

  for(let i=icePlatforms.length-1;i>=0;i--){
    icePlatforms[i].life-=dt;
    if(icePlatforms[i].life<=0)icePlatforms.splice(i,1);
  }
  for(let i=chainBolts.length-1;i>=0;i--){
    chainBolts[i].life-=dt;
    if(chainBolts[i].life<=0)chainBolts.splice(i,1);
  }
  for(const m of motes){m.x+=m.vx;m.y+=m.vy;
    if(m.y<-5)m.y=VH+5;if(m.y>VH+5)m.y=-5;}
  for(let i=particles.length-1;i>=0;i--){const p=particles[i];
    p.x+=p.vx;p.y+=p.vy;p.vy+=0.06;p.life-=dt;
    if(p.life<=0)particles.splice(i,1);}
  for(const s of secrets){if(s.flash>0)s.flash-=dt*2;}
  for(let i=snowTracks.length-1;i>=0;i--){snowTracks[i].life-=dt;if(snowTracks[i].life<=0)snowTracks.splice(i,1);}

  if(state==='title'){updateTitleMenu(dt);return;}

  if(state==='story'){
    const slides=STORY[storyPhase],cur=slides[storySlide];
    if(!cur)return;
    const tC=cur.lines.join('').length,cT=0.035;
    if(!storyDone){storyTimer+=dt;
      const rev=Math.floor(storyTimer/cT),prv=Math.floor((storyTimer-dt)/cT);
      if(rev>prv&&rev<tC&&rev%3===0)SFX.storyBlip();
      if(rev>=tC)storyDone=true;}
    if(cBuf>0){
      cBuf=0;
      if(!storyDone){storyDone=true;storyTimer=tC*cT+0.1;}
      else{storySlide++;storyTimer=0;storyDone=false;
        if(storySlide>=slides.length){
          if(storyPhase==='intro'){loadLevel(1);state='play';SFX.motif();}
          else if(storyPhase==='interlude'){loadLevel('1.5');state='play';SFX.motif();}
          else if(storyPhase==='interlude2'){loadLevel(2);bossIntroT=2.5;state='play';SFX.bossRoar();}
          else if(storyPhase==='ending'){
            if(!ngPlus){ngPlus=true;saveGame();}
            state='title';storySlide=0;titleMenu='main';}
          else if(storyPhase==='ng_intro'){loadLevel(1);state='play';SFX.motif();}
        }else SFX.storyNext();}
    }
    return;
  }

  rBuf-=dt;
  if(rBuf>0){
    rBuf=0;
    if(state==='dead'){deaths++;deathsThisLevel++;saveGame();loadLevel(level);state='play';}
    else if(state==='win'){state='title';titleMenu='main';}
    return;
  }
  if(paused)return;
  if(state!=='play')return;

  if(staffBuffer>0){
    const unlock=LEVELS[level].staff||'fire';
    let ok=false;
    if(unlock==='all')ok=true;
    else if(unlock==='ice'&&staffBuffer===2)ok=true;
    else if(unlock==='lightning'&&staffBuffer===3)ok=true;
    else if(unlock==='fire'&&staffBuffer===1)ok=true;
    if(ok){staffType=['fire','ice','lightning'][staffBuffer-1];SFX.menuSelect();}
    staffBuffer=0;
  }

  if(player.frozen>0)player.frozen-=dt;
  if(player.dashCd>0)player.dashCd-=dt;
  if(player.dashing>0){player.dashing-=dt;if(player.dashing<=0)player.inv=0;}

  const speed=2.0,accel=0.35;
  const jumpPow=7.8*(1+upgrades.jump_pow*0.1);
  
  if(player.frozen<=0&&player.dashing<=0){
    if(lft()){player.vx-=accel;player.face=-1;}
    if(rgt()){player.vx+=accel;player.face=1;}
  }
  const fr=levelSurface==='ice'?0.98:(player.onGround?0.72:0.90);
  if(!lft()&&!rgt()&&player.dashing<=0)player.vx*=fr;
  player.vx=clamp(player.vx,-speed*1.3,speed*1.3);
  const focusing=foc()&&player.light>0;

  // Заряд вспышки
  if(focusing&&player.dashing<=0){
    focusHoldTime+=dt;
    if(focusHoldTime>0.6&&focusHoldTime-dt<=0.6)SFX.chargeUp();
  } else {
    focusHoldTime=0;
  }

  // Рывок
  if(focusing&&!prevF&&player.dashCd<=0&&player.light>=15&&player.dashing<=0&&focusHoldTime<0.3){
    const dashCdBase=0.8*(1-upgrades.dash_cd*0.2);
    player.dashCd=dashCdBase;player.dashing=0.15;player.inv=0.2;
    player.vx=player.face*6;player.light=Math.max(0,player.light-15);
    SFX.dash();burst(player.x+5,player.y+7,'#ffffff',8,2);
    for(let i=0;i<5;i++)particles.push({x:player.x+5,y:player.y+7,
      vx:-player.face*rnd(1,3),vy:rnd(-0.5,0.5),life:0.3,max:0.3,color:'rgba(255,255,255,0.5)'});
  }

  if(level===5){
    windTimer-=dt;
    if(windTimer<=0){windTimer=8;windActive=2;windDir=Math.random()<0.5?-1:1;SFX.wind();}
    if(windActive>0){windActive-=dt;player.vx+=windDir*0.08;}
  }
  if(currents.length){
    for(const c of currents){
      if(player.x+player.w>c.x&&player.x<c.x+c.w&&
         player.y+player.h>c.y&&player.y<c.y+c.h){
        player.vx+=c.vx*0.05;player.vy+=c.vy*0.05;
      }
    }
  }

  // === ВСПЫШКА (обычная и заряженная) ===
  if(!focusing&&prevF&&flashCooldown<=0&&player.dashing<=0){
    const charged=focusHoldTime>=0.6;
    const rad=charged?260:180;
    const cost=charged?35:18;
    
    if(player.light>=cost){
      flashCooldown=charged?0.8:0.55;flashesThisLevel++;
      const cx=player.x+player.w/2,cy=player.y+player.h/2;
      flashWave={x:cx,y:cy,r:20,maxR:rad,life:charged?0.7:0.55,maxLife:charged?0.7:0.55};
      if(boss)boss.flashDone=false;
      
      if(charged){
        SFX.chargedFlash();shake=Math.max(shake,4);
        player.light=Math.max(0,player.light-cost);
        // Разрушение стен
        for(const b of breakables){
          if(b.broken)continue;
          const bd=Math.hypot((b.x+b.w/2)-cx,(b.y+b.h/2)-cy);
          if(bd<rad){b.broken=true;wallsBroken++;SFX.wallBreak();
            burst(b.x+b.w/2,b.y+b.h/2,'#8a7a60',20,3);
            if(wallsBroken>=5)unlockAch('wall_breaker');
            if(b.drop==='shard')shards.push({x:b.x+b.w/2,y:b.y+b.h/2,taken:false,t:0});
            if(b.drop==='secret'){secrets.push({x:b.x,y:b.y,w:14,h:30,done:false,flash:1});SFX.secretReveal();}
            saveGame();}
        }
      } else {
        if(staffType==='fire')SFX.flash();
        else if(staffType==='ice')SFX.iceFlash();
        else SFX.bolt();
        player.light=Math.max(0,player.light-(staffType==='ice'?22:cost));
      }
      
      // Урон врагам
      for(const e of enemies){if(e.dead)continue;
        const dx=(e.x+e.w/2)-cx,dy=(e.y+e.h/2)-cy,d=Math.hypot(dx,dy)||1;
        if(d<rad){
          const p=1-d/rad;
          const dmgMult=charged?3:1;
          
          if(staffType==='fire'){
            e.hp-=(1.5+3.5*p)*dmgMult;e.flash=1;
            e.x+=(dx/d)*(3+5*p);e.y+=(dy/d)*(3+5*p);
          } else if(staffType==='ice'){
            e.hp-=(0.5+1.5*p)*dmgMult;e.flash=1;e.frozen=charged?3:2;
          } else if(staffType==='lightning'){
            e.hp-=(2.0+3.0*p)*dmgMult;e.flash=1;
          }
          
          if(e.hp<=0&&!e.dead){e.dead=true;
            burst(e.x+e.w/2,e.y+e.h/2,e.type==='stalker'?'#6a4a8a':'#8ea2ff',14,2.4);
            player.light=Math.min(player.maxLight,player.light+4);
            if(e.type==='stalker'){stalkerKills++;if(stalkerKills>=10)unlockAch('shadow_dancer');}
            if(e.frozen>0){frozenKills++;if(frozenKills>=10)unlockAch('ice_master');}
            SFX.killEnemy();saveGame();}
        }
      }
      
      // Цепная молния
      if(staffType==='lightning'){
        const hit=[];
        for(const e of enemies){if(e.dead)continue;
          const dx=(e.x+e.w/2)-cx,dy=(e.y+e.h/2)-cy,d=Math.hypot(dx,dy)||1;
          if(d<rad)hit.push({e,d});}
        hit.sort((a,b)=>a.d-b.d);
        if(hit.length>0){
          const first=hit[0];
          chainBolts.push({x1:cx,y1:cy,x2:first.e.x+first.e.w/2,y2:first.e.y+first.e.h/2,life:0.3});
          let prevX=first.e.x+first.e.w/2,prevY=first.e.y+first.e.h/2;
          for(let i=1;i<Math.min(hit.length,4);i++){
            const cur=hit[i];
            const dd=Math.hypot((cur.e.x+cur.e.w/2)-prevX,(cur.e.y+cur.e.h/2)-prevY);
            if(dd<90){
              chainBolts.push({x1:prevX,y1:prevY,x2:cur.e.x+cur.e.w/2,y2:cur.e.y+cur.e.h/2,life:0.3});
              prevX=cur.e.x+cur.e.w/2;prevY=cur.e.y+cur.e.h/2;
            }
          }
          if(hit.length>=3)unlockAch('storm');
        }
      }
      
      // Ледяные платформы
      if(staffType==='ice'){
        for(let i=0;i<(charged?3:2);i++){
          icePlatforms.push({x:cx+player.face*(30+i*35)-20,y:cy+10,w:34,h:8,life:charged?6:4.5});
        }
      }
      
      if(flashesThisLevel>=20)unlockAch('pyromancer');
    }
  }
  prevF=focusing;

  // Wall Slide
  player.wallSlide=false;
  if(!player.onGround&&player.vy>0&&player.dashing<=0){
    for(const p of platforms){
      if(player.y+player.h>p.y&&player.y<p.y+p.h){
        if(Math.abs(player.x-p.x-p.w)<3&&lft()){player.wallSlide=true;break;}
        if(Math.abs(player.x+player.w-p.x)<3&&rgt()){player.wallSlide=true;break;}
      }
    }
  }
  if(player.wallSlide)player.vy=Math.min(player.vy,1.5);

  if(player.onGround)player.coyote=0.1;
  else player.coyote=Math.max(0,player.coyote-dt);
  jBuf-=dt;
  if(jBuf>0&&player.frozen<=0&&player.dashing<=0){
    if(player.onGround||player.coyote>0){
      player.vy=-jumpPow;player.jumps=1;jBuf=0;player.coyote=0;
      burst(player.x+5,player.y+14,'#6f7fd6',5,1.2);SFX.jump();
    }else if(player.wallSlide){
      player.vy=-jumpPow*0.9;player.vx=-player.face*4;player.jumps=1;jBuf=0;
      burst(player.x+5,player.y+7,'#9aa8ff',5,1.2);SFX.jump();
    }else if(player.jumps<2){
      player.vy=-jumpPow*0.9;player.jumps=2;jBuf=0;
      burst(player.x+5,player.y+14,'#9aa8ff',7,1.6);SFX.djump();}
  }
  player.vy+=0.5;if(player.vy>11)player.vy=11;
  const wasG=player.onGround,fS=player.vy;
  moveAndCollide(player);
  if(player.onGround&&!wasG&&fS>4){SFX.land();stepTimer=0;}
  if(player.onGround)player.jumps=0;
  if(player.inv>0&&player.dashing<=0)player.inv-=dt;

  // Следы на снегу
  if(levelSurface==='ice'&&player.onGround&&Math.abs(player.vx)>0.3){
    if(Math.random()<0.3)snowTracks.push({x:player.x+5,y:player.y+14,life:3});
  }

  if(lavaZones.length){
    for(const lz of lavaZones){
      if(overlap(player,lz)){SFX.burn();die('Сгорел в лаве');return;}
    }
  }
  if(hotZones.length){
    let onHot=false;
    for(const hz of hotZones){
      if(player.x+player.w>hz.x&&player.x<hz.x+hz.w&&
         Math.abs(player.y+player.h-hz.y)<4)onHot=true;
    }
    if(onHot)player.light=Math.max(0,player.light-2.0*dt);
  }

  const walking=player.onGround&&Math.abs(player.vx)>0.5;
  if(walking){stepTimer-=dt;
    if(stepTimer<=0){SFX.footstep(stepSide,levelSurface);stepSide^=1;
      stepTimer=0.34-Math.abs(player.vx)*0.04;}}
  else stepTimer=0;

  // Компас заблудшего
  if(walking||Math.abs(player.vx)>0.1)idleTimer=0;
  else idleTimer+=dt;
  if(idleTimer>30&&!compassTarget){
    // Найти ближайший гриб, фонарь или выход
    let best=null,bestD=9999;
    const px=player.x+5,py=player.y+7;
    for(const s of shrooms){if(s.used)continue;const d=Math.hypot(s.x-px,s.y-py);if(d<bestD){bestD=d;best={x:s.x,y:s.y};}}
    for(const l of lanterns){if(l.lit)continue;const d=Math.hypot(l.x-px,l.y-py);if(d<bestD){bestD=d;best={x:l.x,y:l.y};}}
    if(EXIT){const d=Math.hypot(EXIT.x-px,EXIT.y-py);if(d<bestD){bestD=d;best={x:EXIT.x+17,y:EXIT.y+22};}}
    if(ALTAR){const d=Math.hypot(ALTAR.x-px,ALTAR.y-py);if(d<bestD){bestD=d;best={x:ALTAR.x+17,y:ALTAR.y+22};}}
    compassTarget=best;
  }
  if(compassTarget&&Math.hypot(compassTarget.x-player.x-5,compassTarget.y-player.y-7)<40)compassTarget=null;

  // Сердцебиение при низком свете
  if(player.light<25&&state==='play'){
    heartTimer-=dt;
    if(heartTimer<=0){
      const rate=1-player.light/25;
      SFX.heartbeat(rate);
      heartTimer=0.4+rate*0.6;
    }
  }

  let drainMult=1;
  if(difficulty==='easy')drainMult=0.7;
  else if(difficulty==='hard')drainMult=1.4;
  if(ngPlus)drainMult*=1.5;
  drainMult*=(1-upgrades.drain_res*0.15);
  if(levelTheme==='water')drainMult*=1.5;
  if(player.bubble>0){drainMult=0;player.bubble-=dt;}
  
  // Аура пиявок
  for(const e of enemies){
    if(e.dead||e.type!=='leech')continue;
    const d=Math.hypot(player.x+5-e.x-8,player.y+7-e.y-8);
    if(d<e.auraRad)drainMult*=3;
  }
  
  const drain=(focusing?8.0:2.6)*dt*drainMult;
  let nearShroom=false;
  for(const s of shrooms){
    if(s.used)continue;
    const d=Math.hypot(player.x-s.x,player.y-s.y);
    if(d<20&&d>10)nearShroom=true;
  }
  if(nearShroom&&player.bubble<=0)player.light=Math.min(player.maxLight,player.light+0.5*dt);
  player.light=Math.max(0,player.light-drain);

  if(levelTheme==='water'&&flashWave&&flashWave.life>flashWave.maxLife-0.05&&player.bubble<=0){
    player.bubble=1.5;
  }

  if(player.light<=0){
    if(levelTheme==='water')SFX.drown();
    die('Свет угас...');return;
  }
  if(player.y>VH+30){die('Ты растворился во тьме...');return;}

  whisperT-=dt;
  if(whisperT<=0){
    if(player.light<15){whisperMsg='«Не угасай...»';whisperT=4;SFX.whisper();}
    else if(player.light<30){whisperMsg='«Я ещё держусь...»';whisperT=6;}
    else whisperMsg='';
  }

  for(const s of shrooms){
    if(s.used){s.timer-=dt;if(s.timer<=0)s.used=false;continue;}
    if(overlap(player,{x:s.x-s.r,y:s.y-s.r,w:s.r*2,h:s.r*2})){
      s.used=true;s.timer=14;player.light=Math.min(player.maxLight,player.light+55);
      burst(s.x,s.y,'#7ff0d8',14,2.2);shake=Math.max(shake,2);SFX.shroom();}
  }
  for(const sh of shards){
    if(sh.taken)continue;
    if(overlap(player,{x:sh.x-5,y:sh.y-5,w:10,h:10})){
      sh.taken=true;player.light=Math.min(player.maxLight,player.light+12);
      currency++; // Осколки = валюта
      burst(sh.x,sh.y,'#ffd76a',9,1.8);SFX.shard();saveGame();}
  }
  for(const l of lanterns){
    if(l.lit)continue;
    if(overlap(player,{x:l.x-8,y:l.y-8,w:16,h:16})&&player.light>10){
      l.lit=true;player.light=Math.max(0,player.light-10);
      burst(l.x,l.y,'#ffdd88',20,2.5);SFX.shroom();}
  }

  for(const d of DIARIES){
    if(diariesFound.includes(d.id))continue;
    if(d.level!=String(level)&&d.level!==level)continue;
    if(overlap(player,{x:d.x-8,y:d.y-8,w:16,h:16})){
      diariesFound.push(d.id);SFX.diary();
      if(diariesFound.length>=12)unlockAch('all_diaries');
      saveGame();showDiary(d);
    }
  }

  if(trialZone&&!trialZone.triggered&&!trialActive){
    if(player.x>trialZone.x&&player.x<trialZone.x+trialZone.w&&
       player.y>trialZone.y-60&&player.y<trialZone.y+trialZone.h+20){
      trialZone.triggered=true;trialActive=true;trialTimer=20;
      SFX.trialStart();shake=Math.max(shake,6);
      for(const s of trialZone.spawns){
        enemies.push(makeEnemy({x:s.x,y:s.y,type:s.type}));
        burst(s.x+4,s.y+4,'#ff5c7a',10,2);}
    }
  }
  if(trialActive){
    trialTimer-=dt;
    let near=0;
    for(const e of enemies){if(e.dead)continue;
      if(Math.hypot(e.x-player.x,e.y-player.y)<200)near++;}
    if(trialTimer<=0){
      trialActive=false;
      if(near===0){player.light=player.maxLight;SFX.trialEnd();unlockAch('trial_master');
        burst(player.x,player.y,'#ffe9a3',30,3);}
    }
  }

  for(const p of pistons){
    p.phase+=dt*p.speed;
    p.y=p.y0+Math.sin(p.phase)*p.amp;
    const hitBox={x:p.x,y:p.y,w:p.w,h:p.h};
    if(overlap(player,hitBox)&&player.inv<=0&&player.dashing<=0){
      player.light=Math.max(0,player.light-15);
      player.inv=0.8;SFX.piston();
      player.vx=player.x<p.x?-4:4;player.vy=-3;
      damagedThisLevel=true;
    }
  }

  for(const ic of icicles){
    if(ic.respawn>0){ic.respawn-=dt;
      if(ic.respawn<=0){ic.falling=true;ic.vy=0;ic.y=ic.y0||ic.y;}}
    else if(!ic.falling){
      if(Math.abs(player.x-ic.x)<15){ic.falling=true;ic.vy=0;}
    }
    if(ic.falling){
      ic.vy+=0.6;ic.y+=ic.vy;
      if(overlap(player,{x:ic.x-4,y:ic.y-8,w:8,h:12})&&player.inv<=0&&player.dashing<=0){
        player.light=Math.max(0,player.light-18);
        player.inv=1.0;SFX.freeze();damagedThisLevel=true;
        burst(ic.x,ic.y,'#a0e0ff',10,2);
        ic.falling=false;ic.respawn=4;
      }
      if(ic.y>VH){ic.falling=false;ic.respawn=4;}
    }
  }

  const R=lightRadius();
  const px=player.x+player.w/2,py=player.y+player.h/2;
  let nearD=9999;
  for(const e of enemies){
    if(e.dead)continue;
    e.t+=dt*2.4;
    if(e.frozen>0){
      e.frozen-=dt;
      e.flash=Math.max(e.flash,e.frozen>0?0.3:0);
      if(overlap(player,e)&&player.inv<=0&&player.dashing<=0){
        player.light=Math.max(0,player.light-8);
        player.inv=1.0;SFX.hurt();damagedThisLevel=true;}
      continue;
    }
    const ex=e.x+e.w/2,ey=e.y+e.h/2;
    const dx=px-ex,dy=py-ey;
    const d=Math.hypot(dx,dy)||1;
    if(d<nearD)nearD=d;
    const lit=d<R;

    if(e.type==='stalker'){
      e.visible=player.light<40||lit;
      if(e.visible){
        e.x+=(dx/d)*e.speed;e.y+=(dy/d)*e.speed;
        e.hp-=e.hitDmg*dt*0.3;e.flash=1;
        if(Math.random()<0.2)SFX.hitEnemy();
        if(e.hp<=0){e.dead=true;stalkerKills++;if(stalkerKills>=10)unlockAch('shadow_dancer');
          burst(ex,ey,'#6a4a8a',14,2.4);player.light=Math.min(player.maxLight,player.light+5);SFX.killEnemy();saveGame();}
      } else {
        // Медленно крадётся в темноте
        e.x+=(dx/d)*0.3;e.y+=(dy/d)*0.3;
      }
    }
    else if(e.type==='leech'){
      e.x+=(dx/d)*e.speed;e.y+=(dy/d)*e.speed;
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;
        if(e.hp<=0){e.dead=true;burst(ex,ey,'#4a8a4a',10,2);SFX.killEnemy();}}
    }
    else if(e.type==='mimic'){
      if(!e.revealed){e.x=e.baseX;e.y=e.baseY;if(d<50)e.revealed=true;}
      else{
        if(lit){e.hp-=e.hitDmg*dt;e.flash=1;
          if(Math.random()<0.3)SFX.hitEnemy();
          e.x-=(dx/d)*e.retreat;e.y-=(dy/d)*e.retreat;}
        else{e.x+=(dx/d)*e.speed;e.y+=(dy/d)*e.speed;}
        if(e.hp<=0){e.dead=true;burst(ex,ey,'#ff8c5a',12,2.2);
          player.light=Math.min(player.maxLight,player.light+3);SFX.killEnemy();}
      }
    }
    else if(e.type==='swarm'){
      e.swarmPhase+=dt*6;
      const w=Math.sin(e.swarmPhase)*0.4;
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;
        e.x-=(dx/d)*e.retreat;e.y-=(dy/d)*e.retreat;
        if(e.hp<=0){e.dead=true;burst(ex,ey,'#ff6a8a',8,1.8);
          player.light=Math.min(player.maxLight,player.light+1);SFX.hitEnemy();}}
      else{const a=Math.atan2(dy,dx)+w;
        e.x+=Math.cos(a)*e.speed;e.y+=Math.sin(a)*e.speed;}
    }
    else if(e.type==='hopper'){
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;SFX.hitEnemy();
        e.x-=(dx/d)*e.retreat;e.y-=(dy/d)*e.retreat;
        e.jumpVx*=0.8;e.jumpVy*=0.8;}
      else{e.jumpTimer-=dt;
        if(e.jumpTimer<=0){e.jumpTimer=rnd(0.9,1.4);
          e.jumpVx=(dx/d)*3.2;e.jumpVy=(dy/d)*3.2-2.2;
          burst(ex,ey,'#7c5cff',5,1.2);}
        e.x+=e.jumpVx*dt*30;e.y+=e.jumpVy*dt*30;
        e.jumpVx*=Math.pow(0.92,dt*60);e.jumpVy+=6*dt;}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#c08cff',12,2.2);
        player.light=Math.min(player.maxLight,player.light+4);SFX.killEnemy();}
    }
    else if(e.type==='sentinel'){
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;SFX.hitEnemy();}
      else{e.shootTimer-=dt;
        if(e.shootTimer<=0&&d<220){e.shootTimer=1.8;
          projectiles.push({x:ex,y:ey,vx:(dx/d)*1.6,vy:(dy/d)*1.6,life:3,from:'sentinel'});
          burst(ex,ey,'#ff8c5a',4,1.2);}}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#ff9a5c',16,2.6);
        player.light=Math.min(player.maxLight,player.light+6);SFX.killEnemy();}
    }
    else if(e.type==='murena'){
      e.zig+=dt*8;
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;SFX.hitEnemy();
        e.x-=(dx/d)*e.retreat;e.y-=(dy/d)*e.retreat;}
      else{const a=Math.atan2(dy,dx)+Math.sin(e.zig)*0.7;
        e.x+=Math.cos(a)*e.speed;e.y+=Math.sin(a)*e.speed;}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#5c9aaa',10,2);
        player.light=Math.min(player.maxLight,player.light+4);SFX.killEnemy();}
    }
    else if(e.type==='jellyfish'){
      if(d<e.pullRad){
        const pull=1-d/e.pullRad;
        player.vx+=(dx/d)*pull*0.15;
        player.vy+=(dy/d)*pull*0.15;
      }
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;
        if(Math.random()<0.2)SFX.hitEnemy();}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#a0e0ff',12,2.2);
        player.light=Math.min(player.maxLight,player.light+5);SFX.killEnemy();}
    }
    else if(e.type==='iceWolf'){
      if(lit){e.hp-=e.hitDmg*dt*2;e.flash=1;
        if(Math.random()<0.25)SFX.hitEnemy();
        e.x-=(dx/d)*e.retreat;e.y-=(dy/d)*e.retreat;}
      else{e.x+=(dx/d)*e.speed;e.y+=(dy/d)*e.speed;}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#a0e0ff',10,2);
        player.light=Math.min(player.maxLight,player.light+3);SFX.killEnemy();}
    }
    else if(e.type==='smith'){
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;SFX.hitEnemy();}
      else{e.throwTimer-=dt;
        if(e.throwTimer<=0&&d<250){e.throwTimer=rnd(2,3);
          hammers.push({x:ex,y:ey,vx:(dx/d)*2,vy:(dy/d)*2-1.5,life:4,rot:0});
          SFX.bossHit();}}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#ff9a5c',14,2.4);
        player.light=Math.min(player.maxLight,player.light+7);SFX.killEnemy();}
    }
    else if(e.type==='spark'){
      e.orb+=dt*1.5;
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;
        if(Math.random()<0.25)SFX.hitEnemy();}
      else{
        const tx=e.baseX+Math.cos(e.orb)*e.orbR;
        const ty=e.baseY+Math.sin(e.orb)*e.orbR;
        e.x+=(tx-e.x)*0.05;e.y+=(ty-e.y)*0.05;
        if(Math.random()<0.3)burst(e.x+5,e.y+5,'#ff6a2a',1,0.5);}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#ff6a2a',10,2);
        player.light=Math.min(player.maxLight,player.light+4);SFX.killEnemy();}
    }
    else{
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;SFX.hitEnemy();
        e.x-=(dx/d)*e.retreat;e.y-=(dy/d)*e.retreat;
        if(e.hp<=0){e.dead=true;
          burst(ex,ey,e.type==='devourer'?'#7c5cff':'#8ea2ff',
            e.type==='devourer'?20:14,2.4);
          player.light=Math.min(player.maxLight,player.light+(e.type==='devourer'?8:4));
          SFX.killEnemy();}}
      else{e.x+=(dx/d)*e.speed;e.y+=(dy/d)*e.speed;}
    }
    if(player.inv<=0&&player.dashing<=0&&overlap(player,e)&&e.frozen<=0&&e.type!=='leech'){
      const loss=11*e.dmgMult*(ngPlus?1.5:1);
      player.light=Math.max(0,player.light-loss);
      player.inv=1.1;
      const k=px<ex?-1:1;player.vx=k*4.2;player.vy=-3.6;
      shake=Math.max(shake,5);hitFlash=0.25;
      burst(px,py,e.dmgMult>1?'#ff4d7a':'#ff6b8a',10,2);
      SFX.hurt();damagedThisLevel=true;}
  }
  if(boss&&!boss.dead)updateBoss(dt);

  for(let i=projectiles.length-1;i>=0;i--){
    const p=projectiles[i];
    p.x+=p.vx;p.y+=p.vy;p.life-=dt;
    let hw=false;
    for(const pl of platforms){
      if(p.x>pl.x&&p.x<pl.x+pl.w&&p.y>pl.y&&p.y<pl.y+pl.h){hw=true;break;}}
    if(!hw&&player.inv<=0&&player.dashing<=0&&p.x>player.x&&p.x<player.x+player.w&&
       p.y>player.y&&p.y<player.y+player.h){
      player.light=Math.max(0,player.light-(p.from==='boss'?12:8)*(ngPlus?1.5:1));
      player.inv=1.1;burst(p.x,p.y,'#ff5c8a',8,1.8);
      SFX.hurt();shake=Math.max(shake,4);hitFlash=0.25;damagedThisLevel=true;
      projectiles.splice(i,1);continue;
    }
    if(p.life<=0||hw){burst(p.x,p.y,'#8f6aff',4,1.0);projectiles.splice(i,1);}
  }
  for(let i=hammers.length-1;i>=0;i--){
    const h=hammers[i];
    h.x+=h.vx;h.y+=h.vy;h.vy+=0.15;h.life-=dt;h.rot+=dt*10;
    if(player.inv<=0&&player.dashing<=0&&h.x>player.x&&h.x<player.x+player.w&&
       h.y>player.y&&h.y<player.y+player.h){
      player.light=Math.max(0,player.light-14*(ngPlus?1.5:1));player.inv=1.1;
      burst(h.x,h.y,'#ff8c5a',10,2);SFX.hurt();hitFlash=0.25;damagedThisLevel=true;
      hammers.splice(i,1);continue;
    }
    if(h.life<=0||h.y>VH+20){burst(h.x,h.y,'#ff9a5c',6,1.5);hammers.splice(i,1);}
  }

  if(!silhouette&&Math.random()<0.001&&state==='play'){
    const sx=player.x+(Math.random()<0.5?-140:140);
    if(sx>0&&sx<LEVEL_W)silhouette={x:sx,y:GROUND_Y-20,t:4,alpha:0};
  }
  if(silhouette){silhouette.t-=dt;
    const d=Math.abs(player.x-silhouette.x);
    if(d<60){silhouette.t=0;burst(silhouette.x,silhouette.y,'#8fa4ff',12,2);}
    silhouette.alpha=Math.min(0.5,silhouette.t*0.5);
    if(silhouette.t<=0)silhouette=null;}

  const prg=clamp(player.x/LEVEL_W,0,1);
  const dng=clamp(1-nearD/150,0,1);
  let inten=prg*0.5+dng*0.7;
  if(boss&&!boss.dead)inten=Math.max(inten,0.85);
  SFX.setIntensity(clamp(inten,0,1));

  if(ALTAR&&overlap(player,ALTAR)){
    burst(ALTAR.x+17,ALTAR.y+10,'#ffe9a3',40,3.5);
    shake=8;SFX.win();unlockAch('first_light');
    if(!damagedThisLevel&&deathsThisLevel===0)unlockAch('no_damage');
    if(runTime<180)unlockAch('fast');
    if(deaths===0)unlockAch('true_sun');
    if(!bestTimes[1]||runTime<bestTimes[1])bestTimes[1]=runTime;
    saveGame();
    storyPhase='interlude';storySlide=0;storyTimer=0;storyDone=false;state='story';
    return;
  }
  if(EXIT&&overlap(player,EXIT)){
    burst(EXIT.x+17,EXIT.y+10,'#ffe9a3',30,3);SFX.win();
    if(!bestTimes[level]||runTime<bestTimes[level])bestTimes[level]=runTime;
    saveGame();
    if(level==='1.5'){loadLevel(4);state='play';}
    else if(level===4){loadLevel(5);state='play';}
    else if(level===5){loadLevel(6);bossIntroT=2.0;state='play';SFX.bossRoar();}
    return;
  }

  const maxCamX=Math.max(0,LEVEL_W-VW);
  const tX=clamp(player.x+player.w/2-VW/2,0,maxCamX);
  cam.x+=(tX-cam.x)*0.12;
  if(shake>0)shake=Math.max(0,shake-dt*18);
}

function showDiary(d){
  document.querySelectorAll('.diary-popup').forEach(el => el.remove());
  const el=document.createElement('div');
  el.className='diary-popup';
  el.style.cssText=`position:fixed;bottom:140px;left:50%;transform:translateX(-50%);
    background:rgba(4,6,16,0.95);color:#ffd76a;padding:12px 18px;
    border:2px solid rgba(255,215,106,0.6);border-radius:6px;
    font:11px "Courier New",monospace;max-width:80%;z-index:100;
    text-align:center;pointer-events:none;transition:opacity 0.5s`;
  el.innerHTML=`<b>${d.title}</b><br><br><span style="color:#c9d4e8">${d.text}</span>`;
  document.body.appendChild(el);
  setTimeout(()=>{el.style.opacity='0';setTimeout(()=>el.remove(),500);},5000);
}

function goToInterlude2(){
  storyPhase='interlude2';storySlide=0;storyTimer=0;storyDone=false;state='story';
  if(deaths===0&&!skins.includes('gold'))skins.push('gold');
  if(deaths>=5&&!skins.includes('shadow'))skins.push('shadow');
  if(diariesFound.length>=12&&!skins.includes('phantom'))skins.push('phantom');
  saveGame();
}
function winLevel2(){
  storyPhase='ending';storySlide=0;storyTimer=0;storyDone=false;state='story';
  if(deaths===0&&!skins.includes('gold'))skins.push('gold');
  if(deaths>=5&&!skins.includes('shadow'))skins.push('shadow');
  if(diariesFound.length>=12&&!skins.includes('phantom'))skins.push('phantom');
  if(ngPlus&&!skins.includes('eclipse'))skins.push('eclipse');
  if(ngPlus){ngPlusComplete=true;unlockAch('ng_plus');}
  saveGame();
}
function die(msg){
  state='dead';player.dead=true;paused=false;shake=9;
  burst(player.x+5,player.y+7,'#ffd76a',30,3);
  SFX.death();SFX.setIntensity(1);
  document.title=msg;
}

/* ==================== ОТРИСОВКА ==================== */
function drawBackground(){
  const g=ctx.createLinearGradient(0,0,0,VH);
  if(levelTheme==='water'){g.addColorStop(0,'#02101e');g.addColorStop(1,'#010510');}
  else if(levelTheme==='ice'){g.addColorStop(0,'#0a1220');g.addColorStop(1,'#02030a');}
  else if(levelTheme==='forge'){g.addColorStop(0,'#1e0a0a');g.addColorStop(1,'#0a0305');}
  else if(levelTheme==='boss'){g.addColorStop(0,'#1a0410');g.addColorStop(1,'#08020a');}
  else if(levelTheme==='forest'){g.addColorStop(0,'#08140a');g.addColorStop(1,'#02060a');}
  else {g.addColorStop(0,'#0a0d1e');g.addColorStop(1,'#04050c');}
  ctx.fillStyle=g;ctx.fillRect(0,0,VW,VH);

  if(levelTheme==='water'&&state==='play'){
    for(let i=0;i<20;i++){
      const t=(VH+time*20+i*40)%VH;
      const x=(i*47+Math.sin(time*1.5+i)*8)%VW;
      ctx.fillStyle='rgba(120,180,240,0.25)';ctx.fillRect(x,VH-t,1,2);}
  } else if(levelTheme==='forest'&&state==='play'){
    for(let i=0;i<15;i++){
      const t=(time*30+i*40)%VH;
      const x=(i*53+Math.sin(time+i)*10)%VW;
      ctx.fillStyle='rgba(90,140,90,0.35)';ctx.fillRect(x,t,1,2);}
  } else if(levelTheme==='ice'&&state==='play'){
    for(let i=0;i<20;i++){
      const t=(time*40+i*50)%VH;
      const x=(i*43+Math.sin(time*0.5+i)*20)%VW;
      ctx.fillStyle='rgba(200,230,255,0.5)';ctx.fillRect(x,t,1,1);}
  } else if(levelTheme==='forge'&&state==='play'){
    for(let i=0;i<15;i++){
      const t=(VH-time*30-i*60+VH*2)%VH;
      const x=(i*67+Math.cos(time*2+i)*10)%VW;
      ctx.fillStyle='rgba(255,140,60,0.6)';ctx.fillRect(x,t,1,2);}
  }

  if(level===5&&windActive>0){
    ctx.strokeStyle=`rgba(200,230,255,${Math.min(0.5,windActive)})`;
    ctx.lineWidth=1;
    for(let i=0;i<8;i++){
      const y=(i*35+time*80)%VH;
      ctx.beginPath();
      ctx.moveTo(0,y);ctx.lineTo(VW,y+windDir*3);ctx.stroke();
    }
  }

  ctx.fillStyle='#0d1122';
  for(let i=0;i<30;i++){let bx=((i*180-cam.x*0.25)%(VW+400)+VW+400)%(VW+400)-200;
    ctx.fillRect(Math.round(bx),VH-(95+(i%4)*38)-45,32+(i%3)*20,95+(i%4)*38);}
  ctx.fillStyle='#131a2e';
  for(let i=0;i<34;i++){let bx=((i*140-cam.x*0.5)%(VW+320)+VW+320)%(VW+320)-160;
    ctx.fillRect(Math.round(bx),VH-(55+(i%5)*32)-22,22+(i%4)*15,55+(i%5)*32);}
  const f=ctx.createLinearGradient(0,VH-70,0,VH);
  f.addColorStop(0,'rgba(30,40,80,0)');
  const fogCol=levelTheme==='forge'?'rgba(120,40,20,0.25)':
    levelTheme==='water'?'rgba(30,80,140,0.20)':
    levelTheme==='ice'?'rgba(100,140,200,0.18)':
    levelTheme==='boss'?'rgba(120,40,60,0.22)':
    levelTheme==='forest'?'rgba(30,80,50,0.16)':'rgba(40,55,110,0.16)';
  f.addColorStop(1,fogCol);
  ctx.fillStyle=f;ctx.fillRect(0,VH-70,VW,70);
}
function drawPlatforms(){
  for(const p of platforms){
    const x=Math.round(p.x-cam.x),y=Math.round(p.y);
    if(x+p.w<-10||x>VW+10)continue;
    if(levelTheme==='water'){
      ctx.fillStyle='#0a2535';ctx.fillRect(x,y,p.w,p.h);
      ctx.fillStyle='#1a5070';ctx.fillRect(x,y,p.w,2);
      ctx.fillStyle='#2a80a0';ctx.fillRect(x,y,p.w,1);
    } else if(levelTheme==='ice'){
      ctx.fillStyle='#1a2a40';ctx.fillRect(x,y,p.w,p.h);
      ctx.fillStyle='#4a7aaa';ctx.fillRect(x,y,p.w,2);
      ctx.fillStyle='#a0d0ff';ctx.fillRect(x,y,p.w,1);
    } else if(levelTheme==='forge'){
      ctx.fillStyle='#2a1a15';ctx.fillRect(x,y,p.w,p.h);
      ctx.fillStyle='#5a2a1a';ctx.fillRect(x,y,p.w,2);
      ctx.fillStyle='#ff6a2a';ctx.fillRect(x,y,p.w,1);
    } else if(levelTheme==='boss'){
      ctx.fillStyle='#2a1020';ctx.fillRect(x,y,p.w,p.h);
      ctx.fillStyle='#5a2040';ctx.fillRect(x,y,p.w,2);
    } else {
      ctx.fillStyle='#221d33';ctx.fillRect(x,y,p.w,p.h);
      ctx.fillStyle='#3d3560';ctx.fillRect(x,y,p.w,2);
      ctx.fillStyle='#514679';ctx.fillRect(x,y,p.w,1);
    }
    ctx.fillStyle='#171327';ctx.fillRect(x,y+p.h-3,p.w,3);
  }
  for(const ip of icePlatforms){
    const x=Math.round(ip.x-cam.x),y=Math.round(ip.y);
    const a=Math.min(1,ip.life/1.0);
    ctx.globalAlpha=a;
    ctx.fillStyle='#a0e0ff';ctx.fillRect(x,y,ip.w,ip.h);
    ctx.fillStyle='#e0f5ff';ctx.fillRect(x,y,ip.w,1);
    ctx.globalAlpha=1;
  }
  for(const s of secrets){
    if(s.done)continue;
    const x=Math.round(s.x-cam.x),y=Math.round(s.y);
    if(x+s.w<-10||x>VW+10)continue;
    ctx.fillStyle='#221d33';ctx.fillRect(x,y,s.w,s.h);
    ctx.fillStyle='#3d3560';ctx.fillRect(x,y,s.w,2);
  }
  // Разрушаемые стены
  for(const b of breakables){
    if(b.broken)continue;
    const x=Math.round(b.x-cam.x),y=Math.round(b.y);
    if(x+b.w<-10||x>VW+10)continue;
    ctx.fillStyle='#3a3050';ctx.fillRect(x,y,b.w,b.h);
    ctx.strokeStyle='#5a4a70';ctx.lineWidth=1;
    ctx.beginPath();
    ctx.moveTo(x+5,y+5);ctx.lineTo(x+b.w/2,y+b.h/2);
    ctx.lineTo(x+b.w-5,y+8);ctx.moveTo(x+b.w/2,y+b.h/2);
    ctx.lineTo(x+8,y+b.h-5);ctx.stroke();
    ctx.fillStyle='#4a3a60';ctx.fillRect(x,y,b.w,2);
  }
}
function drawLava(){
  for(const l of lavaZones){
    const x=Math.round(l.x-cam.x),y=l.y;
    if(x+l.w<-10||x>VW+10)continue;
    const p=0.7+Math.sin(time*2+l.x)*0.3;
    ctx.fillStyle=`rgba(255,${60+80*p},20,0.9)`;
    ctx.fillRect(x,y,l.w,l.h);
    ctx.fillStyle=`rgba(255,200,80,${0.5*p})`;
    ctx.fillRect(x,y,l.w,3);
  }
}
function drawPistons(){
  for(const p of pistons){
    const x=Math.round(p.x-cam.x),y=Math.round(p.y);
    ctx.fillStyle='#5a2a1a';ctx.fillRect(x,y,p.w,p.h);
    ctx.fillStyle='#8a3a2a';ctx.fillRect(x,y,p.w,3);
    ctx.fillStyle='#ff6a2a';ctx.fillRect(x+2,y+2,p.w-4,1);
  }
}
function drawIcicles(){
  for(const ic of icicles){
    if(!ic.falling)continue;
    const x=Math.round(ic.x-cam.x),y=Math.round(ic.y);
    ctx.fillStyle='#a0e0ff';
    ctx.beginPath();
    ctx.moveTo(x-4,y-8);ctx.lineTo(x+4,y-8);ctx.lineTo(x,y+8);ctx.closePath();ctx.fill();
  }
}
function drawShrooms(){
  for(const s of shrooms){
    const x=Math.round(s.x-cam.x),y=Math.round(s.y);
    if(x<-30||x>VW+30)continue;
    if(s.used){ctx.fillStyle='#2a3d3d';ctx.fillRect(x-2,y-2,5,4);continue;}
    ctx.fillStyle='#4d7f74';ctx.fillRect(x-1,y-1,3,5);
    ctx.fillStyle='#5fe0c4';ctx.fillRect(x-4,y-5,9,4);
    ctx.fillStyle='#a6fff0';ctx.fillRect(x-4,y-5,9,1);
    const pulse=0.6+Math.sin(time*4+s.x)*0.4;
    ctx.fillStyle=`rgba(120,255,225,${0.10*pulse})`;
    ctx.beginPath();ctx.arc(x,y-3,14+pulse*3,0,6.283);ctx.fill();
  }
}
function drawShards(){
  for(const s of shards){
    if(s.taken)continue;
    const x=Math.round(s.x-cam.x);
    const y=Math.round(s.y+Math.sin(time*2.5+s.t)*3);
    if(x<-20||x>VW+20)continue;
    ctx.fillStyle='#ffd76a';ctx.fillRect(x-1,y-3,2,6);ctx.fillRect(x-3,y-1,6,2);
    ctx.fillStyle='#fff3c4';ctx.fillRect(x-1,y-1,2,2);
  }
}
function drawLanterns(){
  for(const l of lanterns){
    const x=Math.round(l.x-cam.x),y=l.y;
    if(x<-30||x>VW+30)continue;
    ctx.fillStyle='#2a2440';ctx.fillRect(x-1,y-6,3,14);
    if(l.lit){ctx.fillStyle='#ffdd88';ctx.fillRect(x-3,y-10,7,6);
      const p=0.7+Math.sin(time*3+l.x)*0.3;
      ctx.fillStyle=`rgba(255,220,140,${0.4*p})`;
      ctx.beginPath();ctx.arc(x,y-8,22,0,6.283);ctx.fill();
    }else{ctx.fillStyle='#3a2e4a';ctx.fillRect(x-3,y-10,7,6);}
  }
}
function drawDiaries(){
  for(const d of DIARIES){
    if(diariesFound.includes(d.id))continue;
    if(d.level!=String(level)&&d.level!==level)continue;
    const x=Math.round(d.x-cam.x),y=d.y;
    if(x<-30||x>VW+30)continue;
    const p=0.6+Math.sin(time*3+d.x)*0.4;
    ctx.fillStyle='#c9a05a';ctx.fillRect(x-3,y-3,7,7);
    ctx.fillStyle='#e0b86a';ctx.fillRect(x-3,y-3,7,1);
    ctx.fillStyle=`rgba(255,215,150,${0.3*p})`;
    ctx.beginPath();ctx.arc(x,y,14,0,6.283);ctx.fill();
  }
}
function drawAltar(){
  if(!ALTAR)return;
  const x=Math.round(ALTAR.x-cam.x),y=ALTAR.y;
  if(x<-60||x>VW+60)return;
  ctx.fillStyle='#2a2440';ctx.fillRect(x-8,y+34,50,8);ctx.fillRect(x-4,y+28,42,6);
  ctx.fillStyle='#37304f';ctx.fillRect(x,y,6,34);ctx.fillRect(x+28,y,6,34);
  ctx.fillStyle='#4b4270';ctx.fillRect(x,y,2,34);ctx.fillRect(x+28,y,2,34);
  ctx.fillStyle='#37304f';ctx.fillRect(x,y-4,34,5);
  const p=0.7+Math.sin(time*3)*0.3;
  ctx.fillStyle='#ffe9a3';ctx.fillRect(x+14,y+10,6,10);
  ctx.fillStyle=`rgba(255,220,140,${0.5*p})`;ctx.fillRect(x+11,y+7,12,16);
  ctx.fillStyle='#fffbe0';ctx.fillRect(x+15,y+12,4,5);
}
function drawExit(){
  if(!EXIT)return;
  const x=Math.round(EXIT.x-cam.x),y=EXIT.y;
  if(x<-60||x>VW+60)return;
  ctx.fillStyle='#2a2440';ctx.fillRect(x-4,y+22,42,18);
  const p=0.7+Math.sin(time*3)*0.3;
  ctx.fillStyle=`rgba(180,255,200,${0.6*p})`;ctx.fillRect(x,y-4,34,30);
  ctx.fillStyle='#a6fff0';ctx.fillRect(x+4,y,26,22);
  ctx.fillStyle='#ffffff';ctx.fillRect(x+13,y+8,8,6);
}
function drawSilhouette(){
  if(!silhouette)return;
  const x=Math.round(silhouette.x-cam.x),y=silhouette.y;
  if(x<-30||x>VW+30)return;
  ctx.globalAlpha=silhouette.alpha;
  ctx.fillStyle='#0a0a14';ctx.fillRect(x,y,10,14);
  ctx.fillStyle='#8fa4ff';ctx.fillRect(x+3,y+2,2,2);ctx.fillRect(x+6,y+2,2,2);
  ctx.globalAlpha=1;
}
function drawBoss(){
  if(!boss||boss.dead)return;
  const x=Math.round(boss.x-cam.x),y=Math.round(boss.y);
  if(x<-100||x>VW+100)return;
  const fl=boss.hitFlash>0?1:boss.flash;
  const fr=boss.frozen>0;
  ctx.fillStyle='rgba(0,0,0,0.45)';
  ctx.beginPath();ctx.ellipse(x+boss.w/2,y+boss.h,boss.w*0.55,6,0,0,6.283);ctx.fill();
  const tint=fr?'#4a80b0':null;
  if(boss.type==='smith'){
    const body=fl?'#8a3030':(tint||'#2a1a15');
    ctx.fillStyle=body;ctx.fillRect(x+6,y+8,36,42);ctx.fillRect(x+2,y+16,44,28);
    ctx.fillStyle=fl?'#b04040':(tint||'#4a2a1a');
    ctx.fillRect(x,y+24,48,18);
    ctx.fillStyle=fl?'#ffcc66':(tint||'#ff6a2a');
    ctx.fillRect(x+12,y+18,6,4);ctx.fillRect(x+30,y+18,6,4);
    ctx.fillStyle='#0a0000';ctx.fillRect(x+14,y+34,20,5);
    ctx.fillStyle=fl?'#ffffff':'#ff6a2a';
    for(let i=0;i<4;i++)ctx.fillRect(x+16+i*5,y+34,2,3);
    const swing=Math.sin(time*2)*0.3;
    ctx.fillStyle='#7a5c34';ctx.fillRect(x+24,y-14+swing*4,4,14);
    ctx.fillStyle='#5a3a1a';ctx.fillRect(x+16,y-18+swing*4,20,6);
  } else {
    const body=fl?'#8a3030':(tint||'#1a0a14');
    ctx.fillStyle=body;ctx.fillRect(x+8,y+10,40,44);ctx.fillRect(x+4,y+18,48,30);ctx.fillRect(x,y+28,56,18);
    ctx.fillStyle=fl?'#b04040':(tint||'#2a0f0f');
    ctx.fillRect(x+2,y-6,8,18);ctx.fillRect(x+46,y-6,8,18);
    ctx.fillRect(x-2,y-14,8,12);ctx.fillRect(x+50,y-14,8,12);
    const crown=0.5+0.5*Math.sin(time*4);
    ctx.fillStyle=`rgba(255,90,40,${0.6*crown})`;ctx.fillRect(x+16,y-4,24,6);
    const eyeCol=boss.phase===2?'#ff2020':'#ff6030';
    ctx.fillStyle=fl?'#ffffff':(tint||eyeCol);
    ctx.fillRect(x+14,y+22,8,6);ctx.fillRect(x+34,y+22,8,6);
    ctx.fillStyle='#ffe9a3';ctx.fillRect(x+17,y+24,3,2);ctx.fillRect(x+37,y+24,3,2);
    ctx.fillStyle='#0a0000';ctx.fillRect(x+18,y+38,20,6);
    ctx.fillStyle=fl?'#ffffff':'#ff5c2a';
    for(let i=0;i<5;i++){ctx.fillRect(x+19+i*4,y+38,2,3);ctx.fillRect(x+19+i*4,y+41,2,3);}
  }
  if(boss.hitFlash>0){ctx.fillStyle=`rgba(255,220,180,${boss.hitFlash*0.4})`;
    ctx.beginPath();ctx.arc(x+boss.w/2,y+boss.h/2,boss.w*0.8,0,6.283);ctx.fill();}
}
function drawBossHP(){
  if(!boss||boss.dead)return;
  const w=Math.min(300,VW-60),h=8,x=(VW-w)/2,y=14;
  ctx.fillStyle='rgba(6,8,18,0.85)';ctx.fillRect(x-3,y-3,w+6,h+6);
  ctx.fillStyle='#2a0f18';ctx.fillRect(x,y,w,h);
  const pct=Math.max(0,boss.hp/boss.maxHp);
  const col=boss.phase===2?'#ff3a3a':'#c04a2a';
  ctx.fillStyle=col;ctx.fillRect(x,y,Math.round(w*pct),h);
  ctx.fillStyle='rgba(255,180,100,0.35)';ctx.fillRect(x,y,Math.round(w*pct),2);
  ctx.fillStyle='#ffd76a';ctx.font='bold 8px "Courier New",monospace';
  ctx.textAlign='center';
  ctx.fillText(boss.type==='smith'?'КУЗНЕЦ ТЬМЫ':'ХРАНИТЕЛЬ ПЕПЛА',VW/2,y-6);
  ctx.textAlign='left';
}
function drawEnemies(){
  for(const e of enemies){
    if(e.dead)continue;
    const bob=e.type==='hopper'?0:Math.sin(e.t)*2;
    const x=Math.round(e.x-cam.x),y=Math.round(e.y+bob);
    if(x<-50||x>VW+50)continue;
    const frz=e.frozen>0;
    const iceTint=frz?'#a0e0ff':null;

    if(e.type==='stalker'){
      if(!e.visible){
        // Призрак в темноте
        ctx.globalAlpha=0.15;
        ctx.fillStyle='#2a1a3a';ctx.fillRect(x+2,y+2,10,12);
        ctx.globalAlpha=1;
        continue;
      }
      const c=e.flash>0?'#8a5aaa':(iceTint||'#3a1a5a');
      ctx.fillStyle=c;ctx.fillRect(x+2,y,10,16);
      ctx.fillStyle=e.flash>0?'#ffffff':'#ff2040';
      ctx.fillRect(x+4,y+4,2,3);ctx.fillRect(x+8,y+4,2,3);
      // Шлейф
      ctx.globalAlpha=0.3;ctx.fillStyle='#6a3a8a';
      ctx.fillRect(x+2-e.vx*3,y+2,10,12);
      ctx.globalAlpha=1;
    }
    else if(e.type==='leech'){
      const c=e.flash>0?'#6aaa6a':(iceTint||'#2a4a2a');
      ctx.fillStyle=c;
      ctx.beginPath();ctx.arc(x+8,y+8,8,0,6.283);ctx.fill();
      // Аура
      ctx.globalAlpha=0.08+Math.sin(time*3)*0.04;
      ctx.fillStyle='#4a8a4a';
      ctx.beginPath();ctx.arc(x+8,y+8,e.auraRad,0,6.283);ctx.fill();
      ctx.globalAlpha=1;
      ctx.fillStyle=e.flash>0?'#ffffff':'#8aff8a';
      ctx.fillRect(x+5,y+5,2,2);ctx.fillRect(x+9,y+5,2,2);
    }
    else if(e.type==='crawler'){
      const c=e.flash>0?'#3a3f78':(iceTint||'#0e0e1a');
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+7,y+7,7,0,6.283);ctx.fill();
      ctx.fillRect(x+1,y+9,12,5);ctx.fillRect(x-1,y+7,3,4);ctx.fillRect(x+12,y+8,3,4);
      ctx.fillStyle=e.flash>0?'#ffffff':'#ff4d6d';
      ctx.fillRect(x+3,y+5,2,2);ctx.fillRect(x+9,y+5,2,2);
    }
    else if(e.type==='hopper'){
      const c=e.flash>0?'#7c6aff':(iceTint||'#1a0f2e');
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+6,y+6,6,0,6.283);ctx.fill();
      ctx.fillRect(x+1,y-2,3,4);ctx.fillRect(x+8,y-2,3,4);
      ctx.fillStyle=e.flash>0?'#ffffff':'#c08cff';
      ctx.fillRect(x+2,y+4,2,2);ctx.fillRect(x+8,y+4,2,2);
    }
    else if(e.type==='devourer'){
      const c=e.flash>0?'#5c4aaa':(iceTint||'#0a0a18');
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+11,y+11,11,0,6.283);ctx.fill();
      ctx.fillRect(x-2,y+14,26,8);ctx.fillRect(x+1,y+20,4,5);
      ctx.fillRect(x+17,y+20,4,5);ctx.fillRect(x+7,y+22,4,4);
      ctx.fillStyle=e.flash>0?'#ffffff':'#b060ff';
      ctx.fillRect(x+4,y+8,3,3);ctx.fillRect(x+15,y+8,3,3);ctx.fillRect(x+9,y+13,3,3);
    }
    else if(e.type==='sentinel'){
      const c=e.flash>0?'#ff8060':(iceTint||'#1a0f0a');
      ctx.fillStyle=c;ctx.fillRect(x+3,y,8,20);ctx.fillRect(x+2,y+2,10,4);ctx.fillRect(x+2,y+16,10,4);
      const glow=e.shootTimer<0.4?'#ffdd66':'#ff5c2a';
      ctx.fillStyle=e.flash>0?'#ffffff':(iceTint||glow);
      ctx.fillRect(x+5,y+8,4,4);
    }
    else if(e.type==='mimic'){
      if(!e.revealed){
        ctx.fillStyle='#4d7f74';ctx.fillRect(x-1,y-1,3,5);
        ctx.fillStyle='#5fe0c4';ctx.fillRect(x-4,y-5,9,4);
        ctx.fillStyle='#a6fff0';ctx.fillRect(x-4,y-5,9,1);
      }else{
        const c=e.flash>0?'#8a3a3a':(iceTint||'#3a0a0a');
        ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+6,y+6,6,0,6.283);ctx.fill();
        ctx.fillStyle=e.flash>0?'#ffffff':'#ff8c5a';
        ctx.fillRect(x+2,y+4,2,2);ctx.fillRect(x+7,y+4,2,2);
        ctx.fillStyle='#0a0000';ctx.fillRect(x+2,y+8,7,2);
      }
    }
    else if(e.type==='swarm'){
      const c=e.flash>0?'#ff8aa0':(iceTint||'#4a0a1a');
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+4,y+4,4,0,6.283);ctx.fill();
      ctx.fillStyle=e.flash>0?'#ffffff':'#ff5c7a';
      ctx.fillRect(x+2,y+2,2,2);
    }
    else if(e.type==='murena'){
      const c=e.flash>0?'#7fb0d0':(iceTint||'#1a3a4a');
      ctx.fillStyle=c;ctx.fillRect(x,y+2,e.w,e.h-4);
      ctx.fillRect(x-2,y+4,4,4);ctx.fillRect(x+e.w-2,y+4,4,4);
      ctx.fillStyle=e.flash>0?'#ffffff':(iceTint||'#5c9aaa');
      ctx.fillRect(x+3,y+4,2,2);ctx.fillRect(x+e.w-6,y+4,2,2);
    }
    else if(e.type==='jellyfish'){
      const c=e.flash>0?'#a0e0ff':(iceTint||'#2a4a6a');
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+7,y+6,7,3.14,0);ctx.fill();
      ctx.fillRect(x+2,y+6,10,3);
      for(let i=0;i<3;i++){
        const tx=x+2+i*3;
        const ty=y+9+Math.sin(time*3+i)*3;
        ctx.fillRect(tx,ty,1,7);
      }
      ctx.fillStyle=e.flash>0?'#ffffff':'#a0e0ff';
      ctx.fillRect(x+4,y+4,2,2);ctx.fillRect(x+8,y+4,2,2);
    }
    else if(e.type==='iceWolf'){
      const c=e.flash>0?'#c0f0ff':(iceTint||'#1a3a55');
      ctx.fillStyle=c;ctx.fillRect(x+1,y+3,14,7);
      ctx.fillRect(x-2,y+5,4,3);ctx.fillRect(x+13,y+5,4,3);
      ctx.fillRect(x+2,y,3,4);ctx.fillRect(x+11,y,3,4);
      ctx.fillStyle=e.flash>0?'#ffffff':(iceTint||'#a0e0ff');
      ctx.fillRect(x+4,y+5,2,2);ctx.fillRect(x+10,y+5,2,2);
    }
    else if(e.type==='smith'){
      const c=e.flash>0?'#ff8060':(iceTint||'#2a1a15');
      ctx.fillStyle=c;ctx.fillRect(x+2,y+4,12,14);
      ctx.fillRect(x+1,y+18,14,4);
      ctx.fillStyle=e.flash>0?'#ffffff':(iceTint||'#ff6a2a');
      ctx.fillRect(x+5,y+8,2,2);ctx.fillRect(x+9,y+8,2,2);
      ctx.fillStyle='#5a3a1a';ctx.fillRect(x+14,y+8,3,8);
    }
    else if(e.type==='spark'){
      const c=e.flash>0?'#ffe0a0':(iceTint||'#3a1a0a');
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+5,y+5,5,0,6.283);ctx.fill();
      ctx.fillStyle=e.flash>0?'#ffffff':(iceTint||'#ffaa3a');
      ctx.fillRect(x+3,y+3,2,2);ctx.fillRect(x+6,y+3,2,2);
      const a=0.2+Math.sin(time*8+e.t)*0.1;
      ctx.fillStyle=`rgba(255,140,60,${a})`;
      ctx.beginPath();ctx.arc(x+5,y+5,10,0,6.283);ctx.fill();
    }

    if(e.flash>0){
      ctx.fillStyle=`rgba(160,180,255,${e.flash*0.35})`;
      ctx.beginPath();ctx.arc(x+e.w/2,y+e.h/2,e.w,0,6.283);ctx.fill();
    }
  }
}
function getSkinColors(){
  switch(currentSkin){
    case 'shadow':return{cloak:'#1a1a2e',hood:'#2a2a4a',glow:'#ff5c5c',eyes:'#ff2020'};
    case 'gold':return{cloak:'#6a4a10',hood:'#8a6a20',glow:'#ffdd88',eyes:'#fffbe0'};
    case 'phantom':return{cloak:'#2e2748',hood:'#3a3170',glow:'#a0e0ff',eyes:'#e0f0ff'};
    case 'eclipse':return{cloak:'#0a0a14',hood:'#1a1a2e',glow:'#8a4aff',eyes:'#c080ff'};
    default:return{cloak:'#2e2748',hood:'#4a3d78',glow:'#ffd76a',eyes:'#ffd76a'};
  }
}
function getStaffGlow(){
  if(staffType==='ice')return'#a0e0ff';
  if(staffType==='lightning')return'#ffe9a3';
  return getSkinColors().glow;
}
function drawPlayer(){
  if(state==='dead')return;
  const x=Math.round(player.x-cam.x),y=Math.round(player.y);
  if(player.inv>0&&player.dashing<=0&&Math.floor(time*20)%2===0)return;
  const f=player.face;
  const sc=getSkinColors();
  const sg=getStaffGlow();
  const sway=Math.sin(time*8)*Math.min(2,Math.abs(player.vx)*1.5);
  
  // Эффект рывка
  if(player.dashing>0){
    ctx.globalAlpha=0.4;
    ctx.fillStyle=sc.cloak;ctx.fillRect(x-f*6+sway*0.3,y+4,10,10);
    ctx.globalAlpha=1;
  }
  
  ctx.fillStyle=sc.cloak;ctx.fillRect(x+sway*0.3,y+4,10,10);
  ctx.fillStyle='#3b3160';ctx.fillRect(x+1,y+2,8,8);
  ctx.fillStyle=sc.hood;ctx.fillRect(x+1,y,8,5);
  ctx.fillStyle='#5a4a8e';ctx.fillRect(x+1,y,8,1);
  ctx.fillStyle='#0a0a14';ctx.fillRect(x+2,y+2,6,3);
  ctx.fillStyle=sc.eyes;ctx.fillRect(x+3,y+2,2,2);ctx.fillRect(x+6,y+2,2,2);
  ctx.fillStyle='#211c36';ctx.fillRect(x+1,y+13,3,2);ctx.fillRect(x+6,y+13,3,2);
  const sx=f>0?x+11:x-3;
  ctx.fillStyle='#7a5c34';ctx.fillRect(sx,y-6,2,16);
  ctx.fillStyle='#9a7648';ctx.fillRect(sx,y-6,1,16);
  const p2=0.7+Math.sin(time*7)*0.3;
  ctx.fillStyle=sg;ctx.fillRect(sx-1,y-9,4,4);
  ctx.fillStyle=`rgba(255,235,170,${0.55*p2})`;ctx.fillRect(sx-3,y-11,8,8);
  
  // Индикатор заряда
  if(focusHoldTime>0.3&&player.dashing<=0){
    const chg=Math.min(1,(focusHoldTime-0.3)/0.3);
    ctx.fillStyle=`rgba(255,255,200,${chg*0.8})`;
    ctx.fillRect(x-2,y-16,14*chg,2);
    if(chg>=1){
      ctx.fillStyle=`rgba(255,255,200,${0.3+Math.sin(time*15)*0.2})`;
      ctx.beginPath();ctx.arc(x+5,y-12,8,0,6.283);ctx.fill();
    }
  }
  
  if(player.bubble>0){
    ctx.strokeStyle=`rgba(160,220,255,${player.bubble})`;
    ctx.lineWidth=1;
    ctx.beginPath();ctx.arc(x+5,y+7,20,0,6.283);ctx.stroke();
  }
}
function drawProjectiles(){
  for(const p of projectiles){
    const x=Math.round(p.x-cam.x),y=Math.round(p.y);
    if(x<-20||x>VW+20)continue;
    const col=p.from==='boss'?'rgba(255,90,40,0.4)':'rgba(150,110,255,0.35)';
    const core=p.from==='boss'?'#ff8c5a':'#c8a4ff';
    ctx.fillStyle=col;ctx.fillRect(x-4,y-4,8,8);
    ctx.fillStyle=core;ctx.fillRect(x-2,y-2,4,4);
    ctx.fillStyle='#ffffff';ctx.fillRect(x-1,y-1,2,2);
  }
  for(const h of hammers){
    const x=Math.round(h.x-cam.x),y=Math.round(h.y);
    ctx.save();
    ctx.translate(x,y);
    ctx.rotate(h.rot);
    ctx.fillStyle='#5a3a1a';ctx.fillRect(-8,-3,16,6);
    ctx.fillStyle='#7a5c34';ctx.fillRect(-4,-8,8,16);
    ctx.restore();
  }
}
function drawChainBolts(){
  if(!chainBolts.length)return;
  ctx.globalCompositeOperation='lighter';
  for(const b of chainBolts){
    const a=Math.min(1,b.life/0.3);
    ctx.strokeStyle=`rgba(255,235,150,${a})`;
    ctx.lineWidth=2;
    ctx.beginPath();
    ctx.moveTo(b.x1-cam.x,b.y1);
    const segs=4;
    for(let i=1;i<=segs;i++){
      const t=i/segs;
      const jx=(Math.random()-0.5)*6;
      const jy=(Math.random()-0.5)*6;
      ctx.lineTo(b.x1+(b.x2-b.x1)*t+jx-cam.x,b.y1+(b.y2-b.y1)*t+jy);
    }
    ctx.stroke();
  }
  ctx.globalCompositeOperation='source-over';
}
function drawFlashWave(){
  if(!flashWave)return;
  const x=flashWave.x-cam.x,y=flashWave.y;
  const a=flashWave.life/flashWave.maxLife;
  ctx.globalCompositeOperation='lighter';
  let c1='rgba(255,230,160,',c2='rgba(255,240,190,',c3='rgba(255,200,120,';
  if(staffType==='ice'){c1='rgba(160,220,255,';c2='rgba(200,240,255,';c3='rgba(100,180,255,';}
  if(staffType==='lightning'){c1='rgba(255,255,180,';c2='rgba(255,255,220,';c3='rgba(200,200,100,';}
  ctx.strokeStyle=`${c1}${a*0.9})`;ctx.lineWidth=3;
  ctx.beginPath();ctx.arc(x,y,flashWave.r,0,6.283);ctx.stroke();
  ctx.strokeStyle=`${c1}${a*0.55})`;ctx.lineWidth=1;
  ctx.beginPath();ctx.arc(x,y,flashWave.r*0.72,0,6.283);ctx.stroke();
  const g=ctx.createRadialGradient(x,y,0,x,y,flashWave.r);
  g.addColorStop(0,`${c2}${a*0.35})`);
  g.addColorStop(0.6,`${c3}${a*0.12})`);
  g.addColorStop(1,'rgba(255,180,80,0)');
  ctx.fillStyle=g;ctx.beginPath();ctx.arc(x,y,flashWave.r,0,6.283);ctx.fill();
  ctx.globalCompositeOperation='source-over';
}
function drawParticles(){
  for(const p of particles){
    ctx.globalAlpha=clamp(p.life/p.max,0,1);
    ctx.fillStyle=p.color;
    ctx.fillRect(Math.round(p.x-cam.x),Math.round(p.y),2,2);
  }
  ctx.globalAlpha=1;
  // Следы на снегу
  for(const t of snowTracks){
    const x=Math.round(t.x-cam.x),y=Math.round(t.y);
    ctx.globalAlpha=t.life/3*0.3;
    ctx.fillStyle='#c0d8f0';ctx.fillRect(x-1,y,3,1);
  }
  ctx.globalAlpha=1;
  if(state==='play'&&player.onGround&&Math.abs(player.vx)>0.5){
    const sx=Math.round(player.x-cam.x+5),sy=Math.round(player.y+14);
    if(Math.floor(time*8)%2===0){
      ctx.globalAlpha=0.25;ctx.fillStyle=getStaffGlow();
      ctx.fillRect(sx-1,sy,2,1);ctx.globalAlpha=1;
    }
  }
}
function drawMotes(){
  for(const m of motes){
    const x=Math.round(m.x-cam.x*0.8);
    const y=Math.round(m.y+Math.sin(time+m.x*0.01)*4);
    if(x<-5||x>VW+5)continue;
    ctx.globalAlpha=m.a*(0.5+0.5*Math.sin(time*1.5+m.x));
    let c='#8fa4ff';
    if(levelTheme==='water')c='#5cc8ff';
    else if(levelTheme==='ice')c='#c0e8ff';
    else if(levelTheme==='forge')c='#ffaa5c';
    else if(levelTheme==='forest')c='#8fff9a';
    else if(levelTheme==='boss')c='#ff9a5c';
    ctx.fillStyle=c;ctx.fillRect(x,y,m.s,m.s);
  }
  ctx.globalAlpha=1;
}
function drawRain(){
  if(!LEVELS[level]?.rain)return;
  ctx.strokeStyle='rgba(140,180,220,0.3)';ctx.lineWidth=1;
  for(const r of rain){
    r.y+=r.sp;if(r.y>VH){r.y=-10;r.x=rnd(0,VW);}
    const x=((r.x-cam.x*0.3)%VW+VW)%VW;
    ctx.beginPath();ctx.moveTo(x,r.y);ctx.lineTo(x-1,r.y+r.s);ctx.stroke();
  }
}
function drawDarkness(){
  dctx.globalCompositeOperation='source-over';
  dctx.clearRect(0,0,VW,VH);
  dctx.fillStyle='rgba(2,3,9,0.965)';dctx.fillRect(0,0,VW,VH);
  dctx.globalCompositeOperation='destination-out';
  if(state!=='dead'){
    const px=player.x+player.w/2-cam.x,py=player.y+player.h/2;
    const R=lightRadius();
    const g=dctx.createRadialGradient(px,py,0,px,py,R);
    g.addColorStop(0,'rgba(0,0,0,1)');g.addColorStop(0.45,'rgba(0,0,0,0.92)');
    g.addColorStop(0.75,'rgba(0,0,0,0.45)');g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(px,py,R,0,6.283);dctx.fill();
  }
  for(const s of shrooms){if(s.used)continue;
    const x=s.x-cam.x,y=s.y;
    if(x<-60||x>VW+60)continue;
    const g=dctx.createRadialGradient(x,y-3,0,x,y-3,28);
    g.addColorStop(0,'rgba(0,0,0,0.85)');g.addColorStop(0.6,'rgba(0,0,0,0.30)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(x,y-3,28,0,6.283);dctx.fill();}
  for(const l of lanterns){if(!l.lit)continue;
    const x=l.x-cam.x,y=l.y;
    if(x<-80||x>VW+80)continue;
    const g=dctx.createRadialGradient(x,y-8,0,x,y-8,42);
    g.addColorStop(0,'rgba(0,0,0,0.85)');g.addColorStop(0.6,'rgba(0,0,0,0.30)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(x,y-8,42,0,6.283);dctx.fill();}
  for(const l of lavaZones){
    const cx=l.x+l.w/2-cam.x,cy=l.y;
    const g=dctx.createRadialGradient(cx,cy,0,cx,cy,50);
    g.addColorStop(0,'rgba(0,0,0,0.7)');g.addColorStop(0.5,'rgba(0,0,0,0.3)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(cx,cy,50,0,6.283);dctx.fill();
  }
  if(ALTAR){const x=ALTAR.x+17-cam.x,y=ALTAR.y+12;
    const g=dctx.createRadialGradient(x,y,0,x,y,45);
    g.addColorStop(0,'rgba(0,0,0,0.9)');g.addColorStop(0.5,'rgba(0,0,0,0.35)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(x,y,45,0,6.283);dctx.fill();}
  if(EXIT){const x=EXIT.x+17-cam.x,y=EXIT.y+10;
    const g=dctx.createRadialGradient(x,y,0,x,y,50);
    g.addColorStop(0,'rgba(0,0,0,0.9)');g.addColorStop(0.5,'rgba(0,0,0,0.35)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(x,y,50,0,6.283);dctx.fill();}
  if(boss&&!boss.dead){const x=boss.x+boss.w/2-cam.x,y=boss.y+boss.h/2;
    const R2=90;const g=dctx.createRadialGradient(x,y,0,x,y,R2);
    g.addColorStop(0,'rgba(0,0,0,0.6)');g.addColorStop(0.7,'rgba(0,0,0,0.2)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(x,y,R2,0,6.283);dctx.fill();}
  dctx.globalCompositeOperation='source-over';
  ctx.drawImage(darkCv,0,0);
  if(state!=='dead'){
    const px=player.x+player.w/2-cam.x,py=player.y+player.h/2;
    const R=lightRadius();
    ctx.globalCompositeOperation='lighter';
    const g=ctx.createRadialGradient(px,py,0,px,py,R*0.95);
    let c='rgba(255,205,110,0.16)';
    if(staffType==='ice')c='rgba(160,220,255,0.16)';
    if(staffType==='lightning')c='rgba(255,255,180,0.16)';
    g.addColorStop(0,c);
    g.addColorStop(0.5,'rgba(255,175,70,0.06)');
    g.addColorStop(1,'rgba(255,150,50,0)');
    ctx.fillStyle=g;ctx.fillRect(px-R,py-R,R*2,R*2);
    ctx.globalCompositeOperation='source-over';
  }
}
function drawHUD(){
  if(state==='title'||state==='story')return;
  const w=92,h=7,x=10,y=10;
  ctx.fillStyle='rgba(6,8,18,0.75)';ctx.fillRect(x-2,y-2,w+4,h+4);
  ctx.fillStyle='#1b2138';ctx.fillRect(x,y,w,h);
  const pct=player.light/player.maxLight;
  const col=pct>0.5?'#ffd76a':pct>0.22?'#ff9a3c':'#ff4d5e';
  ctx.fillStyle=col;ctx.fillRect(x,y,Math.round(w*pct),h);
  ctx.fillStyle='rgba(255,255,255,0.25)';ctx.fillRect(x,y,Math.round(w*pct),2);
  ctx.fillStyle='#7f8bb0';ctx.font='8px "Courier New",monospace';
  ctx.fillText('СВЕТ',x,y+h+11);
  const mm=Math.floor(runTime/60),ss=Math.floor(runTime%60);
  ctx.fillStyle='#8fa0c8';
  ctx.fillText(`⏱ ${mm}:${ss<10?'0':''}${ss}  💀 ${deaths}  💎 ${currency}`,x,y+h+22);
  const stName=staffType==='fire'?'🔥':staffType==='ice'?'❄':'⚡';
  ctx.fillStyle=getStaffGlow();ctx.font='bold 12px "Courier New",monospace';
  ctx.fillText(stName,x,y+h+38);
  if(LEVELS[level].staff==='all'){
    ctx.font='7px "Courier New",monospace';ctx.fillStyle='#7f8bb0';
    ctx.fillText('1 2 3',x+18,y+h+38);
  }
  // Dash CD indicator
  if(player.dashCd>0){
    ctx.fillStyle='rgba(255,255,255,0.3)';
    ctx.fillText('💨···',x+50,y+h+38);
  } else {
    ctx.fillStyle='#8fa0c8';
    ctx.fillText('💨 OK',x+50,y+h+38);
  }
  
  if(player.light<25&&Math.floor(time*3)%2===0&&state==='play'&&!paused){
    ctx.fillStyle='#ff8b9c';ctx.fillText('НАЙДИ ГРИБ',VW/2-30,24);
  }
  const ix=10,iy=64,iw=60,ih=4;
  ctx.fillStyle='rgba(6,8,18,0.65)';ctx.fillRect(ix-1,iy-1,iw+2,ih+2);
  ctx.fillStyle='#1b2138';ctx.fillRect(ix,iy,iw,ih);
  ctx.fillStyle='#8f7bff';ctx.fillRect(ix,iy,Math.round(iw*SFX.getIntensity()),ih);
  ctx.fillStyle='#c9a05a';
  ctx.fillText(`📖 ${diariesFound.length}/12`,VW-64,32);
  if(secretsFound>0){ctx.fillStyle='#8fa0c8';ctx.fillText(`🗝 ${secretsFound}`,VW-64,44);}
  if(ngPlus){ctx.fillStyle='#8a4aff';ctx.fillText('NG+',VW-64,56);}
  if(trialActive){
    ctx.fillStyle='rgba(120,10,30,0.75)';ctx.fillRect(VW/2-80,40,160,32);
    ctx.strokeStyle='#ffd76a';ctx.lineWidth=1;ctx.strokeRect(VW/2-80,40,160,32);
    ctx.fillStyle='#ffd76a';ctx.font='bold 10px "Courier New",monospace';
    ctx.textAlign='center';ctx.fillText('⚔ ИСПЫТАНИЕ ⚔',VW/2,54);
    ctx.fillStyle='#ff8b9c';ctx.font='9px "Courier New",monospace';
    ctx.fillText(`Выжить: ${Math.ceil(trialTimer)}с`,VW/2,68);
    ctx.textAlign='left';
  }
  if(whisperMsg){
    ctx.globalAlpha=0.8+0.2*Math.sin(time*2);
    ctx.fillStyle='#9aa8d8';ctx.font='italic 9px "Courier New",monospace';
    ctx.textAlign='center';ctx.fillText(whisperMsg,VW/2,VH-40);
    ctx.textAlign='left';ctx.globalAlpha=1;
  }
  if(achievementPopup){
    ctx.globalAlpha=Math.min(1,achievementPopup.t);
    ctx.fillStyle='rgba(6,8,18,0.9)';ctx.fillRect(VW-160,60,152,32);
    ctx.strokeStyle='#ffd76a';ctx.lineWidth=1;ctx.strokeRect(VW-160,60,152,32);
    ctx.fillStyle='#ffd76a';ctx.font='bold 9px "Courier New",monospace';
    ctx.fillText('★ '+achievementPopup.name,VW-153,74);
    ctx.fillStyle='#8fa0c8';ctx.font='8px "Courier New",monospace';
    ctx.fillText(achievementPopup.desc,VW-153,86);
    ctx.globalAlpha=1;
  }
  if(mBannerT>0){
    ctx.globalAlpha=Math.min(1,mBannerT*2);
    ctx.fillStyle='rgba(6,8,18,0.85)';
    ctx.fillRect(VW/2-42,VH-30,84,16);
    ctx.fillStyle='#ffd76a';ctx.textAlign='center';
    ctx.font='9px "Courier New",monospace';
    ctx.fillText(SFX.isMuted()?'ЗВУК ВЫКЛ':'ЗВУК ВКЛ',VW/2,VH-19);
    ctx.textAlign='left';ctx.globalAlpha=1;
  }
  // Пульсация при низком свете
  if(player.light<25&&state==='play'){
    const a=1-player.light/25;
    const pulse=0.5+0.5*Math.sin(time*4+a*6);
    const g=ctx.createRadialGradient(VW/2,VH/2,VH*0.2,VW/2,VH/2,VH*0.8);
    g.addColorStop(0,'rgba(0,0,0,0)');
    g.addColorStop(1,`rgba(60,0,20,${a*0.7*pulse})`);
    ctx.fillStyle=g;ctx.fillRect(0,0,VW,VH);
  }
}
function drawCompass(){
  if(!compassTarget||state!=='play')return;
  const px=player.x+5-cam.x,py=player.y+7;
  const tx=compassTarget.x-cam.x,ty=compassTarget.y;
  const a=Math.atan2(ty-py,tx-px);
  const dist=40;
  const cx=px+Math.cos(a)*dist,cy=py+Math.sin(a)*dist;
  const blink=0.4+0.6*Math.abs(Math.sin(time*3));
  ctx.globalAlpha=blink;
  ctx.fillStyle='#ffd76a';
  ctx.beginPath();
  ctx.moveTo(cx+Math.cos(a)*6,cy+Math.sin(a)*6);
  ctx.lineTo(cx+Math.cos(a+2.5)*4,cy+Math.sin(a+2.5)*4);
  ctx.lineTo(cx+Math.cos(a-2.5)*4,cy+Math.sin(a-2.5)*4);
  ctx.closePath();ctx.fill();
  ctx.globalAlpha=1;
}
function drawPauseOverlay(){
  ctx.fillStyle='rgba(2,4,12,0.72)';ctx.fillRect(0,0,VW,VH);
  ctx.fillStyle='rgba(255,215,106,0.15)';
  ctx.fillRect(0,VH/2-50,VW,1);ctx.fillRect(0,VH/2+50,VW,1);
  ctx.textAlign='center';
  const p=0.75+0.25*Math.sin(time*3);
  ctx.fillStyle=`rgba(255,215,106,${p})`;
  ctx.font='bold 24px "Courier New",monospace';
  ctx.fillText('ПАУЗА',VW/2,VH/2-24);
  ctx.fillStyle='#8fa0c8';ctx.font='9px "Courier New",monospace';
  let task='';
  if(level===1)task='Найди Алтарь';
  else if(level==='1.5')task='Пройди Лес Шепотов';
  else if(level===4)task='Пройди Затонувший Город';
  else if(level===5)task='Пройди Ледяные Пики';
  else if(level===6)task='Убей Кузнеца Тьмы';
  else if(level===2)task='Убей Хранителя Пепла';
  ctx.fillText('Задача: '+task,VW/2,VH/2-2);
  ctx.fillStyle='#7f8bb0';ctx.font='8px "Courier New",monospace';
  ctx.fillText(`📖 ${diariesFound.length}/12  ·  🗝 ${secretsFound}  ·  💀 ${deaths}  ·  💎 ${currency}`,VW/2,VH/2+16);
  ctx.fillText(`Посох: ${staffType==='fire'?'Огонь':staffType==='ice'?'Лёд':'Молния'}`,VW/2,VH/2+30);
  const a=0.5+0.5*Math.sin(time*3);
  ctx.fillStyle=`rgba(200,215,255,${a})`;
  ctx.font='bold 9px "Courier New",monospace';
  ctx.fillText('[ ПРОДОЛЖИТЬ: ⏸ ]',VW/2,VH/2+46);
  ctx.textAlign='left';
}
function drawStoryArt(art){
  if(art==='stars'||art==='sun'){
    const count=art==='sun'?5:40;
    for(let i=0;i<count;i++){
      const sx=(i*137)%VW,sy=(i*71)%120;
      const b=0.3+0.7*Math.abs(Math.sin(time*0.5+i));
      ctx.fillStyle=`rgba(200,220,255,${b*0.7})`;ctx.fillRect(sx,sy,1,1);
    }
  }
  if(art==='sun'){
    const cx=VW/2,cy=90,p=0.7+0.3*Math.sin(time*1.5);
    const g=ctx.createRadialGradient(cx,cy,0,cx,cy,70);
    g.addColorStop(0,`rgba(255,240,180,${0.9*p})`);
    g.addColorStop(0.3,`rgba(255,200,100,${0.6*p})`);
    g.addColorStop(1,'rgba(255,150,50,0)');
    ctx.fillStyle=g;ctx.beginPath();ctx.arc(cx,cy,70,0,6.283);ctx.fill();
    ctx.fillStyle='#fff3c4';ctx.beginPath();ctx.arc(cx,cy,16,0,6.283);ctx.fill();
  }
  if(art==='hero'||art==='staff'){
    const hx=VW/2-5,hy=VH-60;
    ctx.fillStyle='#0a0a14';ctx.fillRect(hx,hy,10,20);ctx.fillRect(hx+1,hy-6,8,7);
    ctx.fillStyle='#2e2748';ctx.fillRect(hx,hy+4,10,12);
    const sx=hx+12;ctx.fillStyle='#7a5c34';ctx.fillRect(sx,hy-12,2,32);
    const p2=0.6+0.4*Math.sin(time*3);
    ctx.fillStyle=`rgba(255,235,170,${p2})`;ctx.fillRect(sx-2,hy-16,6,6);
  }
  if(art==='altar'){
    const ax=VW/2-20,ay=VH-110;
    ctx.fillStyle='#2a2440';ctx.fillRect(ax-4,ay+60,48,8);
    ctx.fillStyle='#37304f';ctx.fillRect(ax,ay,8,60);ctx.fillRect(ax+32,ay,8,60);
    ctx.fillStyle='#4b4270';ctx.fillRect(ax,ay,2,60);ctx.fillRect(ax+32,ay,2,60);
    ctx.fillStyle='#37304f';ctx.fillRect(ax,ay-6,40,6);
    const p=0.7+0.3*Math.sin(time*3);
    const g=ctx.createRadialGradient(ax+20,ay+22,0,ax+20,ay+22,50);
    g.addColorStop(0,`rgba(255,230,150,${0.5*p})`);
    g.addColorStop(1,'rgba(255,200,80,0)');
    ctx.fillStyle=g;ctx.beginPath();ctx.arc(ax+20,ay+22,50,0,6.283);ctx.fill();
    ctx.fillStyle='#ffe9a3';ctx.fillRect(ax+16,ay+14,8,12);
  }
}
function drawStory(){
  ctx.fillStyle='#02030a';ctx.fillRect(0,0,VW,VH);
  const slides=STORY[storyPhase],cur=slides[storySlide];
  if(!cur)return;
  drawStoryArt(cur.art);
  const tT=VH-100;
  ctx.fillStyle='rgba(4,6,16,0.85)';ctx.fillRect(0,tT-6,VW,106);
  ctx.strokeStyle='rgba(120,140,200,0.4)';ctx.lineWidth=1;
  ctx.beginPath();
  ctx.moveTo(40,tT-6);ctx.lineTo(VW-40,tT-6);
  ctx.moveTo(40,tT+94);ctx.lineTo(VW-40,tT+94);ctx.stroke();
  const rev=Math.floor(storyTimer/0.035);let idx=0;
  ctx.font='11px "Courier New",monospace';ctx.fillStyle='#c9d4e8';ctx.textAlign='center';
  for(let li=0;li<cur.lines.length;li++){
    const line=cur.lines[li];let shown='';
    for(let ci=0;ci<line.length;ci++){if(idx<rev)shown+=line[ci];idx++;}
    ctx.fillText(shown,VW/2,tT+22+li*20);
  }
  ctx.textAlign='left';
  if(storyDone){
    const a=0.4+0.6*Math.abs(Math.sin(time*3));
    ctx.fillStyle=`rgba(255,215,106,${a})`;
    ctx.font='bold 9px "Courier New",monospace';ctx.textAlign='center';
    ctx.fillText('[ ПРОБЕЛ ]',VW/2,VH-12);ctx.textAlign='left';
  }
  ctx.fillStyle='rgba(120,140,200,0.35)';ctx.font='8px "Courier New",monospace';
  ctx.textAlign='right';ctx.fillText(`${storySlide+1} / ${slides.length}`,VW-12,VH-8);
  ctx.textAlign='left';
}
function drawTitle(){
  ctx.fillStyle='rgba(3,4,12,0.86)';ctx.fillRect(0,0,VW,VH);
  for(let i=0;i<60;i++){
    const sx=(i*137)%VW,sy=(i*71)%VH;
    const b=0.2+0.5*Math.abs(Math.sin(time*0.7+i*0.9));
    ctx.fillStyle=`rgba(160,180,255,${b*0.5})`;ctx.fillRect(sx,sy,1,1);
  }
  ctx.textAlign='center';
  if(titleMenu==='main'){
    ctx.fillStyle='#ffd76a';ctx.font='bold 20px "Courier New",monospace';
    ctx.fillText('ХРОНИКИ',VW/2,40);
    ctx.fillText('ЗАБЫТОГО СВЕТА',VW/2,64);
    ctx.fillStyle='#5a6a94';ctx.font='9px "Courier New",monospace';
    ctx.fillText('Солнце погасло. Ты — последняя искра.',VW/2,84);
    
    const opts=['Продолжить','Новая игра'];
    if(ngPlus||ngPlusComplete)opts.push('Угасающее Солнце');
    opts.push('Костер','Скины','Настройки','Сброс');
    
    ctx.font='11px "Courier New",monospace';
    for(let i=0;i<opts.length;i++){
      const y=108+i*16;
      if(i===titleCursor){ctx.fillStyle='#ffd76a';ctx.fillText('▸ '+opts[i]+' ◂',VW/2,y);}
      else{ctx.fillStyle='#8fa0c8';ctx.fillText(opts[i],VW/2,y);}
    }
    if(deaths>0||diariesFound.length>0||currency>0){
      ctx.fillStyle='#7f8bb0';ctx.font='8px "Courier New",monospace';
      const info=[];
      if(diariesFound.length)info.push(`📖${diariesFound.length}/12`);
      if(secretsFound)info.push(`🗝${secretsFound}`);
      if(achievements.length)info.push(`★${achievements.length}/${ACHS.length}`);
      if(currency)info.push(`💎${currency}`);
      if(deaths)info.push(`💀${deaths}`);
      ctx.fillText(info.join(' · '),VW/2,VH-28);
    }
    const a=0.4+0.6*Math.abs(Math.sin(time*2.5));
    ctx.fillStyle=`rgba(255,215,106,${a})`;
    ctx.font='8px "Courier New",monospace';
    ctx.fillText('↑↓ выбор · ПРОБЕЛ ок',VW/2,VH-12);
  } else if(titleMenu==='bonfire'){
    ctx.fillStyle='#ff9a3c';ctx.font='bold 16px "Courier New",monospace';
    ctx.fillText('🔥 КОСТЕР 🔥',VW/2,30);
    ctx.fillStyle='#ffd76a';ctx.font='10px "Courier New",monospace';
    ctx.fillText(`Осколки: 💎 ${currency}`,VW/2,50);
    ctx.font='9px "Courier New",monospace';
    for(let i=0;i<UPGRADES.length;i++){
      const u=UPGRADES[i];
      const y=75+i*28;
      const lvl=upgrades[u.id];
      const cost=u.cost*(lvl+1);
      const maxed=lvl>=u.max;
      const canBuy=!maxed&&currency>=cost;
      if(i===titleCursor){ctx.fillStyle=canBuy?'#ffd76a':'#ff6b6b';}
      else{ctx.fillStyle=maxed?'#4a6a4a':'#8fa0c8';}
      const label=`${u.icon} ${u.name} [${lvl}/${u.max}] ${maxed?'МАКС':cost+'💎'}`;
      ctx.fillText((i===titleCursor?'▸ ':'  ')+label,VW/2,y);
      ctx.fillStyle='#5a6a84';ctx.font='7px "Courier New",monospace';
      ctx.fillText(u.desc,VW/2,y+10);
      ctx.font='9px "Courier New",monospace';
    }
    ctx.fillStyle='#7f8bb0';ctx.font='8px "Courier New",monospace';
    ctx.fillText('↑↓ выбор · ПРОБЕЛ купить · ESC назад',VW/2,VH-12);
  } else if(titleMenu==='skins'){
    ctx.fillStyle='#ffd76a';ctx.font='bold 16px "Courier New",monospace';
    ctx.fillText('СКИНЫ',VW/2,40);
    if(skins.length===1){
      ctx.fillStyle='#8fa0c8';ctx.font='10px "Courier New",monospace';
      ctx.fillText('Пока открыт только один',VW/2,90);
      ctx.fillText('Пройди игру разными способами',VW/2,108);
      ctx.fillText('чтобы открыть остальные.',VW/2,126);
    } else {
      ctx.font='11px "Courier New",monospace';
      for(let i=0;i<skins.length;i++){
        const y=80+i*22;
        const label=skinName(skins[i])+(skins[i]===currentSkin?' ✓':'');
        if(i===titleCursor){ctx.fillStyle='#ffd76a';ctx.fillText('▸ '+label+' ◂',VW/2,y);}
        else{ctx.fillStyle='#8fa0c8';ctx.fillText(label,VW/2,y);}
      }
    }
    ctx.fillStyle='#7f8bb0';ctx.font='9px "Courier New",monospace';
    ctx.fillText('[ ESC — назад ]',VW/2,VH-12);
  } else if(titleMenu==='settings'){
    ctx.fillStyle='#ffd76a';ctx.font='bold 16px "Courier New",monospace';
    ctx.fillText('НАСТРОЙКИ',VW/2,40);
    ctx.font='11px "Courier New",monospace';
    const dl='Сложность: '+diffName(difficulty);
    if(titleCursor===0)ctx.fillStyle='#ffd76a';else ctx.fillStyle='#8fa0c8';
    ctx.fillText((titleCursor===0?'▸ ':'  ')+dl+(titleCursor===0?' ◂':''),VW/2,90);
    const mv=Math.round(SFX.getMusicVolume()*100);
    const bar='█'.repeat(Math.round(mv/10))+'░'.repeat(10-Math.round(mv/10));
    const vl='Музыка: '+bar+' '+mv+'%';
    if(titleCursor===1)ctx.fillStyle='#ffd76a';else ctx.fillStyle='#8fa0c8';
    ctx.fillText((titleCursor===1?'▸ ':'  ')+vl+(titleCursor===1?' ◂':''),VW/2,110);
    if(titleCursor===2)ctx.fillStyle='#ffd76a';else ctx.fillStyle='#8fa0c8';
    ctx.fillText((titleCursor===2?'▸ ':'  ')+'Клавиши'+(titleCursor===2?' ◂':''),VW/2,130);
    if(titleCursor===3)ctx.fillStyle='#ffd76a';else ctx.fillStyle='#8fa0c8';
    ctx.fillText((titleCursor===3?'▸ ':'  ')+'Назад'+(titleCursor===3?' ◂':''),VW/2,150);
    ctx.fillStyle='#7f8bb0';ctx.font='9px "Courier New",monospace';
    ctx.fillText('↑↓ выбор · ←→ изменить · ПРОБЕЛ ок',VW/2,VH-12);
  } else if(titleMenu==='keys'){
    ctx.fillStyle='#ffd76a';ctx.font='bold 16px "Courier New",monospace';
    ctx.fillText('КЛАВИШИ',VW/2,40);
    ctx.font='10px "Courier New",monospace';
    const items=[`Прыжок: ${keyMap.jump}`,`Фокус/Рывок: ${keyMap.focus}`,'Сброс клавиш','Назад'];
    for(let i=0;i<items.length;i++){
      const y=80+i*24;
      if(i===titleCursor)ctx.fillStyle='#ffd76a';else ctx.fillStyle='#8fa0c8';
      ctx.fillText((i===titleCursor?'▸ ':'  ')+items[i],VW/2,y);
    }
    ctx.fillStyle='#5a6a84';ctx.font='8px "Courier New",monospace';
    ctx.fillText('Нажмите ПРОБЕЛ на пункте,',VW/2,VH-30);
    ctx.fillText('затем нажмите новую клавишу',VW/2,VH-18);
  }
  ctx.textAlign='left';
}
function drawDead(){
  ctx.fillStyle='rgba(20,0,10,0.72)';ctx.fillRect(0,0,VW,VH);
  ctx.textAlign='center';
  ctx.fillStyle='#ff6b8a';ctx.font='bold 18px "Courier New",monospace';
  ctx.fillText('СВЕТ УГАС',VW/2,118);
  ctx.fillStyle='#8fa0c8';ctx.font='9px "Courier New",monospace';
  ctx.fillText('Тьма поглотила странника...',VW/2,142);
  const mm=Math.floor(runTime/60),ss=Math.floor(runTime%60);
  ctx.fillText(`Время: ${mm}:${ss<10?'0':''}${ss}  ·  Смертей: ${deaths}`,VW/2,160);
  const a=0.5+0.5*Math.sin(time*4);
  ctx.globalAlpha=a;
  ctx.fillStyle='#ffd76a';ctx.font='bold 10px "Courier New",monospace';
  ctx.fillText('R — ПОПРОБОВАТЬ СНОВА',VW/2,190);
  ctx.globalAlpha=1;ctx.textAlign='left';
}

function render(){
  if(state==='story'){drawStory();return;}
  ctx.save();
  if(shake>0.2)ctx.translate(Math.round(rnd(-shake,shake)),Math.round(rnd(-shake,shake)));
  drawBackground();drawLava();drawPlatforms();drawPistons();drawIcicles();
  drawShrooms();drawShards();drawLanterns();drawDiaries();
  drawAltar();drawExit();drawEnemies();drawMotes();drawSilhouette();
  drawBoss();drawPlayer();drawProjectiles();drawChainBolts();drawParticles();
  drawFlashWave();drawRain();drawDarkness();drawHUD();drawCompass();
  if(boss&&!boss.dead&&state==='play'&&!paused)drawBossHP();
  if(bossIntroT>0&&!paused){
    ctx.globalAlpha=Math.min(1,bossIntroT);
    ctx.fillStyle='rgba(40,0,0,0.55)';
    ctx.fillRect(0,VH/2-24,VW,48);
    ctx.fillStyle='#ff5c2a';ctx.font='bold 14px "Courier New",monospace';
    ctx.textAlign='center';
    const txt=boss&&boss.type==='smith'?'КУЗНЕЦ ТЬМЫ ПРОБУДИЛСЯ':'ХРАНИТЕЛЬ ПЕПЛА ПРОБУДИЛСЯ';
    ctx.fillText(txt,VW/2,VH/2+6);
    ctx.textAlign='left';ctx.globalAlpha=1;
  }
  ctx.restore();
  if(hitFlash>0){
    ctx.globalCompositeOperation='screen';
    ctx.globalAlpha=hitFlash*0.6;
    ctx.fillStyle='#ff2040';ctx.fillRect(-2,0,VW,VH);
    ctx.fillStyle='#2040ff';ctx.fillRect(2,0,VW,VH);
    ctx.globalCompositeOperation='source-over';ctx.globalAlpha=1;
  }
  if(state==='title')drawTitle();
  if(state==='dead')drawDead();
  if(state==='play'&&paused)drawPauseOverlay();
}

/* ==================== ЦИКЛ ==================== */
let last=performance.now(),acc=0;
const STEP=1/60;
function loop(now){
  requestAnimationFrame(loop);
  let dt=(now-last)/1000;last=now;
  if(dt>0.1)dt=0.1;
  acc+=dt;
  let g=0;
  while(acc>=STEP&&g++<5){update(STEP);acc-=STEP;}
  render();
}
loadLevel(1);state='title';titleMenu='main';titleCursor=0;
requestAnimationFrame(loop);
})();
</script>
</body>
</html>
