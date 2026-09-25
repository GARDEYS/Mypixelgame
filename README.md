<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="theme-color" content="#03040a">
<title>Хроники Забытого Света</title>
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
    #bL{left:calc(12px + env(safe-area-inset-left,0));bottom:calc(80px + env(safe-area-inset-bottom,0))}
    #bR{left:calc(78px + env(safe-area-inset-left,0));bottom:calc(80px + env(safe-area-inset-bottom,0))}
    #bJ{right:calc(12px + env(safe-area-inset-right,0));bottom:calc(90px + env(safe-area-inset-bottom,0))}
    #bF{right:calc(92px + env(safe-area-inset-right,0));bottom:calc(50px + env(safe-area-inset-bottom,0))}
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
(() => {
"use strict";

/* ====================== АУДИО ====================== */
const SFX = (() => {
  let ac=null,master=null,input=null,reverb=null,drySend=null,wetSend=null;
  let ambientGain=null,musicGain=null;
  let layerBase=null,layerPad=null,layerArp=null,layerPerc=null;
  let enabled=true,muted=false,noiseBuf=null,lastHitT=0;
  let musicIntensity=0,musicIntensitySm=0,musicTimer=null,nextNoteTime=0,step=0;
  const BPM=72,STEP_DUR=60/BPM/2,LOOKAHEAD=0.12,INTERVAL=25,TOTAL_STEPS=32;
  let ambientLayer='forest';
  const N={'A2':110,'F2':87.31,'C3':130.81,'G2':98,'F3':174.61,'A3':220,'B3':246.94,
    'C4':261.63,'D4':293.66,'E4':329.63,'F4':349.23,'G4':392,'A4':440,'B4':493.88,
    'C5':523.25,'D5':587.33,'E5':659.25,'G5':783.99,'A5':880};
  const MELODY=['A4',null,'E5',null,'C5',null,'A4',null,'F4',null,'C5',null,'A4',null,'F4',null,
    'C4',null,'G4',null,'E5',null,'C5',null,'G3',null,'D4',null,'B4',null,'G4',null];
  const BASS=['A2',null,null,null,'A2',null,null,null,'F2',null,null,null,'F2',null,null,null,
    'C3',null,null,null,'C3',null,null,null,'G2',null,null,null,'G2',null,null,null];
  const PADS=[['A3','C4','E4'],['F3','A3','C4'],['C4','E4','G4'],['G3','B3','D4']];
  const ARP=['A5','C5','E5','G5'];
  const MOTIF=['A4','C5','E5','D5','C5'];

  function ensure(){
    if(!enabled) return null;
    if(ac){if(ac.state==='suspended')ac.resume();return ac;}
    try{
      ac=new (window.AudioContext||window.webkitAudioContext)();
      master=ac.createGain();master.gain.value=0.34;master.connect(ac.destination);
      reverb=ac.createConvolver();reverb.buffer=makeImpulse(ac,2.6,3.2);
      input=ac.createGain();
      drySend=ac.createGain();drySend.gain.value=0.78;
      wetSend=ac.createGain();wetSend.gain.value=0.62;
      input.connect(drySend);drySend.connect(master);
      input.connect(wetSend);wetSend.connect(reverb);reverb.connect(master);
      musicGain=ac.createGain();musicGain.gain.value=0.0001;musicGain.connect(input);
      layerBase=ac.createGain();layerBase.gain.value=1;
      layerPad=ac.createGain();layerPad.gain.value=0;
      layerArp=ac.createGain();layerArp.gain.value=0;
      layerPerc=ac.createGain();layerPerc.gain.value=0;
      layerBase.connect(musicGain);layerPad.connect(musicGain);
      layerArp.connect(musicGain);layerPerc.connect(musicGain);
    }catch(e){enabled=false;}
    return ac;
  }
  function makeImpulse(c,dur,decay){
    const rate=c.sampleRate,len=Math.floor(rate*dur);
    const buf=c.createBuffer(2,len,rate);
    for(let ch=0;ch<2;ch++){const d=buf.getChannelData(ch);
      for(let i=0;i<len;i++){const t=i/len;let v=(Math.random()*2-1)*Math.pow(1-t,decay);
        if(i<rate*0.04)v*=1.7;d[i]=v;}}
    return buf;
  }
  function getNoise(c){
    if(noiseBuf)return noiseBuf;
    const len=Math.floor(c.sampleRate*0.6);
    noiseBuf=c.createBuffer(1,len,c.sampleRate);
    const d=noiseBuf.getChannelData(0);
    for(let i=0;i<len;i++)d[i]=Math.random()*2-1;
    return noiseBuf;
  }
  function env(g,t0,a,pk,dc){
    g.gain.cancelScheduledValues(t0);
    g.gain.setValueAtTime(0.0001,t0);
    g.gain.exponentialRampToValueAtTime(Math.max(0.0002,pk),t0+a);
    g.gain.exponentialRampToValueAtTime(0.0001,t0+a+dc);
  }
  function tone(f,dur,type,vol,slide,delay){
    const c=ensure();if(!c||muted)return;
    const t=c.currentTime+(delay||0);
    const o=c.createOscillator(),g=c.createGain();
    o.type=type||'square';o.frequency.setValueAtTime(f,t);
    if(slide)o.frequency.exponentialRampToValueAtTime(Math.max(20,slide),t+dur);
    env(g,t,0.005,vol||0.25,dur);
    o.connect(g);g.connect(input);o.start(t);o.stop(t+dur+0.06);
  }
  function noise(dur,vol,from,to,q,delay){
    const c=ensure();if(!c||muted)return;
    const t=c.currentTime+(delay||0);
    const src=c.createBufferSource();src.buffer=getNoise(c);
    const flt=c.createBiquadFilter();flt.type='lowpass';flt.Q.value=q||1.2;
    flt.frequency.setValueAtTime(from||3000,t);
    flt.frequency.exponentialRampToValueAtTime(Math.max(60,to||180),t+dur);
    const g=c.createGain();env(g,t,0.004,vol||0.18,dur);
    src.connect(flt);flt.connect(g);g.connect(input);
    src.start(t);src.stop(t+dur+0.05);
  }
  function startAmbient(){
    const c=ensure();if(!c||ambientGain)return;
    const t=c.currentTime;
    ambientGain=c.createGain();
    ambientGain.gain.setValueAtTime(0.0001,t);
    ambientGain.gain.exponentialRampToValueAtTime(0.055,t+4);
    ambientGain.connect(input);
    const lp=c.createBiquadFilter();lp.type='lowpass';lp.frequency.value=320;
    lp.connect(ambientGain);
    [110,110.7,164.8].forEach((f,i)=>{
      const o=c.createOscillator();o.type=i===2?'triangle':'sine';o.frequency.value=f;
      const g=c.createGain();g.gain.value=i===2?0.35:0.55;
      o.connect(g);g.connect(lp);o.start(t);
    });
    const lfo=c.createOscillator();lfo.frequency.value=0.06;
    const lfoG=c.createGain();lfoG.gain.value=0.018;
    lfo.connect(lfoG);lfoG.connect(ambientGain.gain);lfo.start(t);
  }
  function playNote(f,t,dur,type,vol,dest){
    const o=ac.createOscillator();o.type=type;o.frequency.value=f;
    const g=ac.createGain();
    g.gain.setValueAtTime(0.0001,t);
    g.gain.exponentialRampToValueAtTime(vol,t+0.015);
    g.gain.setValueAtTime(vol,t+dur*0.55);
    g.gain.exponentialRampToValueAtTime(0.0001,t+dur);
    o.connect(g);g.connect(dest);o.start(t);o.stop(t+dur+0.05);
  }
  function kickPerc(t){
    const o=ac.createOscillator(),g=ac.createGain();
    o.frequency.setValueAtTime(120,t);
    o.frequency.exponentialRampToValueAtTime(40,t+0.12);
    g.gain.setValueAtTime(0.0001,t);
    g.gain.exponentialRampToValueAtTime(0.20,t+0.005);
    g.gain.exponentialRampToValueAtTime(0.0001,t+0.18);
    o.connect(g);g.connect(layerPerc);o.start(t);o.stop(t+0.22);
  }
  function hatPerc(t){
    const src=ac.createBufferSource();src.buffer=getNoise(ac);
    const flt=ac.createBiquadFilter();flt.type='highpass';flt.frequency.value=6500;
    const g=ac.createGain();
    g.gain.setValueAtTime(0.0001,t);
    g.gain.exponentialRampToValueAtTime(0.055,t+0.003);
    g.gain.exponentialRampToValueAtTime(0.0001,t+0.05);
    src.connect(flt);flt.connect(g);g.connect(layerPerc);
    src.start(t);src.stop(t+0.1);
  }
  function scheduleStep(i,t){
    const mel=MELODY[i];
    if(mel){playNote(N[mel],t,STEP_DUR*1.7,'triangle',0.16,layerBase);
      playNote(N[mel]*2,t,STEP_DUR*0.9,'sine',0.045,layerBase);}
    const bass=BASS[i];
    if(bass){playNote(N[bass],t,STEP_DUR*3.2,'sawtooth',0.10,layerBase);
      playNote(N[bass]/2,t,STEP_DUR*3.2,'sine',0.08,layerBase);}
    if(i%8===0){const ch=PADS[Math.floor(i/8)];
      if(ch)ch.forEach(f=>playNote(N[f],t,STEP_DUR*7.6,'sine',0.06,layerPad));}
    if(i%2===1){const n=ARP[Math.floor(i/2)%ARP.length];
      playNote(N[n],t,STEP_DUR*0.75,'square',0.038,layerArp);}
    if(i%4===0)kickPerc(t);
    if(i%4===2)hatPerc(t);
  }
  function updateLayers(){
    musicIntensitySm+=(musicIntensity-musicIntensitySm)*0.06;
    const I=musicIntensitySm;
    layerPad.gain.value=Math.min(1,I*1.1);
    layerArp.gain.value=Math.max(0,(I-0.35)/0.65);
    layerPerc.gain.value=Math.max(0,(I-0.55)/0.45);
  }
  function scheduler(){
    if(!ac||muted)return;
    updateLayers();
    if(nextNoteTime<ac.currentTime)nextNoteTime=ac.currentTime+0.05;
    let guard=0;
    while(nextNoteTime<ac.currentTime+LOOKAHEAD&&guard++<32){
      scheduleStep(step,nextNoteTime);
      nextNoteTime+=STEP_DUR;step=(step+1)%TOTAL_STEPS;
    }
  }
  function startMusic(){
    const c=ensure();if(!c||musicTimer)return;
    step=0;nextNoteTime=c.currentTime+0.15;
    musicTimer=setInterval(scheduler,INTERVAL);
    const t=c.currentTime;
    musicGain.gain.cancelScheduledValues(t);
    musicGain.gain.setValueAtTime(Math.max(0.0001,musicGain.gain.value),t);
    musicGain.gain.exponentialRampToValueAtTime(0.85,t+3.5);
  }

  return {
    init(){ensure();startAmbient();startMusic();},
    setIntensity(v){musicIntensity=v<0?0:v>1?1:v;},
    getIntensity(){return musicIntensitySm;},
    toggleMute(){muted=!muted;if(master)master.gain.value=muted?0:0.34;return muted;},
    isMuted(){return muted;},
    setDifficulty(d){/* не влияет на звук */},
    motif(){
      MOTIF.forEach((f,i)=>{tone(N[f],0.35,'triangle',0.14,null,i*0.14);
        tone(N[f]*2,0.25,'sine',0.04,null,i*0.14+0.02);});
    },
    choir(){
      [523.25,659.25,783.99].forEach((f,i)=>{
        tone(f,0.5,'sine',0.09,null,i*0.03);
        tone(f*1.5,0.4,'triangle',0.03,null,i*0.05);
      });
    },
    jump(){tone(320,0.10,'square',0.16,520);},
    djump(){tone(480,0.12,'square',0.15,760);tone(720,0.09,'triangle',0.09,900,0.03);},
    land(){noise(0.10,0.16,900,90,1.0);},
    footstep(side,surface){
      const c=ensure();if(!c||muted)return;
      const t=c.currentTime;
      const src=c.createBufferSource();src.buffer=getNoise(c);
      const flt=c.createBiquadFilter();flt.type='lowpass';
      let f0=side?780:1000;
      if(surface==='water')f0=1400;else if(surface==='ice')f0=1800;
      else if(surface==='moss')f0=500;
      flt.frequency.setValueAtTime(f0,t);
      flt.frequency.exponentialRampToValueAtTime(surface==='water'?300:150,t+0.06);
      const g=c.createGain();
      g.gain.setValueAtTime(0.0001,t);
      g.gain.exponentialRampToValueAtTime(0.055,t+0.004);
      g.gain.exponentialRampToValueAtTime(0.0001,t+0.07);
      src.connect(flt);flt.connect(g);g.connect(input);
      src.start(t);src.stop(t+0.1);
    },
    flash(){noise(0.28,0.20,5000,400,1.5);
      tone(880,0.22,'triangle',0.10,1600,0.01);
      tone(1320,0.18,'sine',0.06,2200,0.03);
      [523.25,659.25,783.99].forEach((f,i)=>tone(f,0.4,'sine',0.06,null,i*0.02));},
    hitEnemy(){const now=performance.now();
      if(now-lastHitT<55)return;lastHitT=now;
      noise(0.06,0.13,2400,500);tone(420,0.05,'square',0.08,260);},
    killEnemy(){tone(260,0.18,'sawtooth',0.18,90);noise(0.16,0.14,1800,200,1.1);},
    hurt(){tone(180,0.30,'sawtooth',0.22,70);noise(0.22,0.18,1400,120,1.0);},
    shroom(){[523.25,659.25,783.99,1046.5].forEach((f,i)=>tone(f,0.22,'triangle',0.14,null,i*0.05));},
    shard(){tone(880,0.07,'triangle',0.14,1320);tone(1320,0.12,'sine',0.10,null,0.05);},
    death(){tone(220,0.9,'sawtooth',0.22,40);noise(0.8,0.16,1200,60,0.9);tone(110,1.2,'sine',0.14,30,0.1);},
    bossRoar(){tone(80,1.4,'sawtooth',0.28,50);tone(60,1.6,'sine',0.18,40,0.05);noise(0.9,0.20,900,80,0.9);},
    bossHit(){noise(0.12,0.14,3000,400);tone(300,0.10,'square',0.12,180);},
    bossPhase(){tone(120,0.6,'sawtooth',0.24,60);tone(180,0.4,'square',0.14,90,0.1);noise(0.5,0.18,2200,200,1.2);},
    storyBlip(){tone(660,0.05,'triangle',0.10,720);},
    storyNext(){tone(440,0.10,'triangle',0.12,660);},
    achievement(){[880,1175,1568].forEach((f,i)=>tone(f,0.3,'triangle',0.15,null,i*0.08));},
    whisper(){noise(0.6,0.05,600,200,0.8);},
    diary(){tone(440,0.4,'triangle',0.15,550);tone(660,0.5,'sine',0.08,null,0.1);},
    trial(){[523.25,587.33,659.25,783.99].forEach((f,i)=>tone(f,0.2,'square',0.12,null,i*0.06));},
    win(){[523.25,659.25,783.99,1046.5,1318.5].forEach((f,i)=>{
      tone(f,0.4,'triangle',0.16,null,i*0.11);
      tone(f*2,0.3,'sine',0.05,null,i*0.11+0.02);});
      noise(0.6,0.10,6000,800,1.4,0.05);}
  };
})();

/* ====================== КАНВАС ====================== */
const BASE_H=270,MIN_VW=320,MAX_VW=720,MAX_SCALE=4.5;
let VW=480,VH=BASE_H;
const cvs=document.getElementById('c');
const ctx=cvs.getContext('2d');
const darkCv=document.createElement('canvas');
const dctx=darkCv.getContext('2d');

function resize(){
  const cssW=Math.max(1,innerWidth),cssH=Math.max(1,innerHeight);
  let scale=cssH/BASE_H;
  if(scale>MAX_SCALE)scale=MAX_SCALE;if(scale<1)scale=1;
  let virtW=Math.ceil(cssW/scale);
  if(virtW<MIN_VW)virtW=MIN_VW;if(virtW>MAX_VW)virtW=MAX_VW;
  VW=virtW;VH=BASE_H;
  cvs.width=VW;cvs.height=VH;ctx.imageSmoothingEnabled=false;
  const ds=Math.min(cssW/VW,cssH/VH);
  cvs.style.width=Math.floor(VW*ds)+'px';
  cvs.style.height=Math.floor(VH*ds)+'px';
  darkCv.width=VW;darkCv.height=VH;dctx.imageSmoothingEnabled=false;
}
addEventListener('resize',resize);
addEventListener('orientationchange',()=>{setTimeout(resize,150);setTimeout(resize,450);});
if(window.visualViewport)window.visualViewport.addEventListener('resize',resize);
resize();

/* ====================== ВВОД ====================== */
const keys={};
let jumpBuffer=0,restartBuffer=0,confirmBuffer=0,pauseBuffer=0,paused=false;
let prevFocus=false,mutedBannerT=0;

addEventListener('keydown',e=>{
  if(['ArrowLeft','ArrowRight','ArrowUp','ArrowDown','Space'].includes(e.code))e.preventDefault();
  SFX.init();
  if(keys[e.code])return;keys[e.code]=true;
  if(e.code==='KeyW'||e.code==='ArrowUp'||e.code==='Space'){jumpBuffer=0.14;confirmBuffer=0.14;}
  if(e.code==='KeyR'||e.code==='Enter')restartBuffer=0.14;
  if(e.code==='KeyP'||e.code==='Escape')pauseBuffer=0.14;
  if(e.code==='KeyM'){SFX.toggleMute();mutedBannerT=1.6;}
});
addEventListener('keyup',e=>{keys[e.code]=false;});
const left=()=>keys['KeyA']||keys['ArrowLeft'];
const right=()=>keys['KeyD']||keys['ArrowRight'];
const focus=()=>keys['ShiftLeft']||keys['ShiftRight'];

(function setupTouch(){
  const isTouch=('ontouchstart' in window)||(navigator.maxTouchPoints>0);
  if(!isTouch)return;
  document.getElementById('touch').classList.add('on');
  function bind(id,onDown,onUp){
    const el=document.getElementById(id);
    const press=e=>{e.preventDefault();SFX.init();el.classList.add('press');onDown&&onDown();};
    const release=e=>{e.preventDefault();el.classList.remove('press');onUp&&onUp();};
    el.addEventListener('touchstart',press,{passive:false});
    el.addEventListener('touchend',release,{passive:false});
    el.addEventListener('touchcancel',release,{passive:false});
    el.addEventListener('mousedown',press);el.addEventListener('mouseup',release);
    el.addEventListener('mouseleave',release);
  }
  bind('bL',()=>{keys['KeyA']=true;},()=>{keys['KeyA']=false;});
  bind('bR',()=>{keys['KeyD']=true;},()=>{keys['KeyD']=false;});
  bind('bJ',()=>{keys['Space']=true;jumpBuffer=0.14;confirmBuffer=0.14;},()=>{keys['Space']=false;});
  bind('bF',()=>{keys['ShiftLeft']=true;},()=>{keys['ShiftLeft']=false;});
  bind('bRst',()=>{restartBuffer=0.14;confirmBuffer=0.14;},null);
  bind('bMute',()=>{SFX.toggleMute();mutedBannerT=1.6;},null);
  bind('bP',()=>{pauseBuffer=0.14;},null);
})();

/* ====================== ДНЕВНИКИ ====================== */
const DIARIES=[
  {id:'d1',level:1,x:320,y:150,title:'Запись I — Начало',
   text:'«Солнце угасло не за один день. Оно уходило медленно, как свеча в сырой комнате. Мы думали — вернётся. Не вернулось.»'},
  {id:'d2',level:1,x:1010,y:120,title:'Запись II — Иссохший колодец',
   text:'«Вода в колодцах застыла. Хлеб не поднимался. Я видел, как люди ели землю.»'},
  {id:'d3',level:1,x:1800,y:110,title:'Запись III — Искра',
   text:'«Говорят, Искра Первого Пламени передаётся по крови. Но у меня не было детей. Только посох. И я его отдал.»'},
  {id:'d4',level:'1.5',x:400,y:150,title:'Запись IV — Лес',
   text:'«Деревья здесь шепчут. Они помнят свет. И они голодны.»'},
  {id:'d5',level:'1.5',x:1200,y:120,title:'Запись V — Хранитель',
   text:'«Я видел его. Он был как мы — когда-то. Но свет сжёг его изнутри. Теперь он только пепел.»'},
  {id:'d6',level:'1.5',x:1550,y:150,title:'Запись VI — Молитва',
   text:'«Если ты читаешь это — значит дошёл дальше меня. Не отдавай Искру. Она — последнее, что осталось.»'},
  {id:'d7',level:2,x:200,y:150,title:'Запись VII — Тень',
   text:'«Ты видел её? Фигуру вдалеке? Это я. Из будущего. Я не смог. Может, ты сможешь.»'},
  {id:'d8',level:2,x:820,y:150,title:'Запись VIII — Последний',
   text:'«Когда я умру, свет не угаснет. Он будет жить, пока ты идёшь. Спасибо, Люмен.»'}
];

/* ====================== ДОСТИЖЕНИЯ ====================== */
const ACHS=[
  {id:'first_light',name:'Первый свет',desc:'Активируй Алтарь'},
  {id:'all_diaries',name:'Хронист',desc:'Собери все дневники'},
  {id:'no_damage',name:'Не тронь меня',desc:'Пройди уровень без урона'},
  {id:'fast',name:'Скороход',desc:'Уровень 1 за 3 минуты'},
  {id:'pyromancer',name:'Пироман',desc:'20 вспышек за уровень'},
  {id:'trial_master',name:'Испытание',desc:'Пройди комнату-испытание'},
  {id:'boss_slayer',name:'Убийца Хранителя',desc:'Убей Хранителя Пепла'},
  {id:'true_sun',name:'Истинный Свет',desc:'Собери всё с первого раза'}
];

/* ====================== УРОВНИ ====================== */
const LEVELS={
  1:{
    width:2600,groundY:230,spawn:{x:40,y:180},surface:'stone',
    platforms:[
      {x:0,y:230,w:400,h:40},{x:455,y:230,w:250,h:40},{x:760,y:230,w:170,h:40},
      {x:985,y:230,w:400,h:40},{x:1440,y:230,w:300,h:40},{x:1795,y:230,w:805,h:40},
      {x:250,y:180,w:70,h:10},{x:390,y:148,w:60,h:10},{x:545,y:175,w:80,h:10},
      {x:695,y:140,w:70,h:10},{x:845,y:178,w:90,h:10},{x:990,y:140,w:80,h:10},
      {x:1140,y:168,w:90,h:10},{x:1300,y:132,w:80,h:10},{x:1440,y:162,w:90,h:10},
      {x:1600,y:178,w:90,h:10},{x:1760,y:138,w:100,h:10},{x:1950,y:178,w:90,h:10},
      {x:2100,y:135,w:110,h:10},{x:2280,y:172,w:100,h:10}
    ],
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
      {x:2300,y:170,type:'devourer'}
    ],
    altar:{x:2500,y:186,w:34,h:44},hasBoss:false
  },
  '1.5':{
    width:1900,groundY:230,spawn:{x:40,y:180},surface:'moss',
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
    shrooms:[{x:100,y:222},{x:440,y:222},{x:740,y:222},{x:1100,y:222},
      {x:1500,y:222},{x:1800,y:222},{x:580,y:142},{x:860,y:132},{x:1570,y:112}],
    shards:[{x:200,y:160},{x:480,y:170},{x:1000,y:160},{x:1440,y:140},{x:1740,y:150}],
    lanterns:[{x:900,y:180},{x:1400,y:150}],
    enemies:[
      {x:250,y:190,type:'crawler'},{x:420,y:190,type:'hopper'},
      {x:600,y:140,type:'mimic'},{x:760,y:190,type:'crawler'},
      {x:920,y:150,type:'swarm'},{x:1080,y:190,type:'hopper'},
      {x:1260,y:180,type:'sentinel'},{x:1480,y:140,type:'crawler'},
      {x:1650,y:170,type:'swarm'},{x:1780,y:190,type:'devourer'}
    ],
    altar:null,hasBoss:false,
    exit:{x:1850,y:186,w:34,h:44}
  },
  2:{
    width:900,groundY:230,spawn:{x:60,y:180},surface:'stone',
    platforms:[
      {x:0,y:230,w:900,h:40},{x:80,y:170,w:90,h:10},
      {x:730,y:170,w:90,h:10},{x:340,y:130,w:70,h:10},{x:490,y:130,w:70,h:10}
    ],
    shrooms:[{x:130,y:222},{x:770,y:222},{x:280,y:162},{x:620,y:162},{x:450,y:122}],
    shards:[{x:375,y:105},{x:525,y:105}],
    lanterns:[{x:400,y:180}],
    enemies:[],altar:null,hasBoss:true,boss:{x:720,y:166}
  }
};

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
    {art:'hero',lines:['Он ждёт тебя в Сердце Тьмы.','Не дай ему поглотить Искру.']}
  ],
  ending:[
    {art:'altar',lines:['Хранитель пал.','Его тело рассыпалось искрами, и тьма отступила.']},
    {art:'staff',lines:['Ты поднял посох. Последний луч','вырвался из него и коснулся неба.']},
    {art:'sun',lines:['Солнце вернулось.','Дети впервые увидели свет.']},
    {art:'hero',lines:['Но ты знал: где-то в глубине, во тьме,','оно снова ждёт своего часа.']},
    {art:'sun',lines:['КОНЕЦ','','Но свет всегда возвращается.']}
  ]
};

/* ====================== СОСТОЯНИЕ ====================== */
let state='title',storyPhase='intro',storySlide=0,storyTimer=0,storyDone=false;
let level=1,time=0,cam={x:0},shake=0;
let deaths=0,runTime=0,deathsThisLevel=0,damagedThisLevel=false,flashesThisLevel=0;
let bestTimes={},bestDeaths={},diariesFound=[],achievements=[],skins=['default'],
  currentSkin='default',difficulty='normal';

const player={x:40,y:180,w:10,h:14,vx:0,vy:0,onGround:false,jumps:0,face:1,
  light:70,inv:0,dead:false,coyote:0};

let enemies=[],shrooms=[],shards=[],lanterns=[],particles=[],motes=[],
  projectiles=[],silhouette=null;
let platforms=[],LEVEL_W=2600,GROUND_Y=230,ALTAR=null,EXIT=null;
let boss=null,bossIntroT=0,stepTimer=0,stepSide=0;
let flashWave=null,flashCooldown=0;
let whisperT=0,whisperMsg='';
let trialActive=null,trialTimer=0,trialKills=0,trialNeed=0;
let achievementPopup=null;
let levelSurface='stone';
let hitFlash=0;

/* ====================== УТИЛИТЫ ====================== */
const rand=(a,b)=>a+Math.random()*(b-a);
const clamp=(v,a,b)=>v<a?a:v>b?b:v;
function overlap(a,b){return a.x<b.x+b.w&&a.x+a.w>b.x&&a.y<b.y+b.h&&a.y+a.h>b.y;}
function burst(x,y,c,n,spd){
  for(let i=0;i<n;i++){const a=Math.random()*Math.PI*2,s=rand(0.4,1)*(spd||2);
    particles.push({x,y,vx:Math.cos(a)*s,vy:Math.sin(a)*s-0.4,
      life:rand(0.3,0.8),max:0.8,color:c});}
}
function unlockAch(id){
  if(achievements.includes(id))return;
  achievements.push(id);
  const a=ACHS.find(x=>x.id===id);
  if(a){achievementPopup={name:a.name,desc:a.desc,t:3};SFX.achievement();}
  saveGame();
}

/* ====================== СОХРАНЕНИЕ ====================== */
function saveGame(){
  try{
    localStorage.setItem('fl_save',JSON.stringify({
      level,deaths,bestTimes,bestDeaths,diariesFound,achievements,skins,currentSkin,difficulty
    }));
  }catch(e){}
}
function loadGame(){
  try{
    const s=JSON.parse(localStorage.getItem('fl_save')||'{}');
    if(s.level)level=s.level;
    if(s.deaths)deaths=s.deaths;
    if(s.bestTimes)bestTimes=s.bestTimes;
    if(s.bestDeaths)bestDeaths=s.bestDeaths;
    if(s.diariesFound)diariesFound=s.diariesFound;
    if(s.achievements)achievements=s.achievements;
    if(s.skins)skins=s.skins;
    if(s.currentSkin)currentSkin=s.currentSkin;
    if(s.difficulty)difficulty=s.difficulty;
  }catch(e){}
}
loadGame();

/* ====================== ФАБРИКИ ====================== */
function makeEnemy(spawn){
  const base={x:spawn.x,y:spawn.y,type:spawn.type,t:Math.random()*6.28,dead:false,flash:0};
  switch(spawn.type){
    case 'crawler':return{...base,w:14,h:14,hp:2.0,dmgMult:1.0,hitDmg:2.2,speed:0.62,retreat:1.5};
    case 'hopper':return{...base,w:12,h:12,hp:1.6,dmgMult:1.0,hitDmg:2.6,speed:0.35,retreat:1.8,
      jumpTimer:rand(0.4,1.0),jumpVx:0,jumpVy:0};
    case 'devourer':return{...base,w:22,h:22,hp:5.5,dmgMult:2.0,hitDmg:1.5,speed:0.32,retreat:0.9};
    case 'sentinel':return{...base,w:14,h:20,hp:3.5,dmgMult:1.0,hitDmg:2.0,speed:0,retreat:0,
      shootTimer:rand(0.8,1.8)};
    case 'mimic':return{...base,w:12,h:12,hp:1.5,dmgMult:1.0,hitDmg:2.4,speed:0.5,retreat:1.6,
      revealed:false,revealT:0,baseX:spawn.x,baseY:spawn.y};
    case 'swarm':return{...base,w:8,h:8,hp:0.9,dmgMult:0.5,hitDmg:2.5,speed:0.75,retreat:1.2,
      swarmPhase:Math.random()*6.28};
    default:return{...base,w:14,h:14,hp:2.0,dmgMult:1.0,hitDmg:2.2,speed:0.62,retreat:1.5};
  }
}
function makeBoss(spawn){
  return{x:spawn.x,y:spawn.y,w:56,h:64,hp:60,maxHp:60,phase:1,
    state:'idle',stateT:0,idleDur:1.4,flash:0,vx:0,vy:0,dead:false,
    facing:-1,attackCount:0,hitFlash:0,flashDone:false};
}

/* ====================== УРОВЕНЬ ====================== */
function loadLevel(n){
  level=n;
  const L=LEVELS[n];
  if(!L)return;
  LEVEL_W=L.width;GROUND_Y=L.groundY;levelSurface=L.surface||'stone';
  platforms=L.platforms.slice();ALTAR=L.altar;EXIT=L.exit||null;
  player.x=L.spawn.x;player.y=L.spawn.y;
  player.vx=0;player.vy=0;player.onGround=false;player.jumps=0;
  player.light=n===2?100:70;player.inv=0;player.face=1;player.dead=false;player.coyote=0;
  enemies=L.enemies.map(s=>makeEnemy(s));
  projectiles=[];
  shrooms=L.shrooms.map(s=>({x:s.x,y:s.y,r:9,used:false,timer:0}));
  shards=L.shards.map(s=>({x:s.x,y:s.y,taken:false,t:Math.random()*6.28}));
  lanterns=(L.lanterns||[]).map(l=>({x:l.x,y:l.y,lit:false}));
  boss=L.hasBoss?makeBoss(L.boss):null;bossIntroT=0;
  particles=[];motes=[];
  const mc=Math.min(90,Math.max(40,Math.floor(VW*0.18)));
  for(let i=0;i<mc;i++)motes.push({x:rand(0,Math.max(LEVEL_W,VW*2)),y:rand(0,VH),
    vx:rand(-0.12,0.12),vy:rand(-0.06,0.06),s:Math.random()<0.3?2:1,a:rand(0.15,0.5)});
  cam.x=0;shake=0;stepTimer=0;stepSide=0;
  flashWave=null;flashCooldown=0;paused=false;hitFlash=0;
  trialActive=null;silhouette=null;whisperT=0;
  deathsThisLevel=0;damagedThisLevel=false;flashesThisLevel=0;
  SFX.setIntensity(0);
}

/* ====================== ФИЗИКА ====================== */
function moveAndCollide(e){
  e.x+=e.vx;
  for(const p of platforms){if(overlap(e,p)){
    if(e.vx>0)e.x=p.x-e.w;else if(e.vx<0)e.x=p.x+p.w;e.vx=0;}}
  e.y+=e.vy;e.onGround=false;
  for(const p of platforms){if(overlap(e,p)){
    if(e.vy>0){e.y=p.y-e.h;e.onGround=true;}
    else if(e.vy<0){e.y=p.y+p.h;}e.vy=0;}}
}
function lightRadius(){
  const base=14+player.light*0.34;
  const flick=1+Math.sin(time*9)*0.02+Math.sin(time*23)*0.012;
  let mult=1;
  if(focus()&&player.light>0)mult=1.5;
  if(flashWave){const t=flashWave.life/flashWave.maxLife;mult+=t*1.8;}
  return base*mult*flick;
}

/* ====================== БОСС ====================== */
function bossSpread(){
  const cx=boss.x+boss.w/2,cy=boss.y+boss.h/2;
  const px=player.x+player.w/2,py=player.y+player.h/2;
  const baseAng=Math.atan2(py-cy,px-cx);
  const count=boss.phase===1?5:7,spread=0.6;
  for(let i=0;i<count;i++){const a=baseAng+(i/(count-1)-0.5)*spread;
    projectiles.push({x:cx,y:cy,vx:Math.cos(a)*1.7,vy:Math.sin(a)*1.7,life:4,from:'boss'});}
  burst(cx,cy,'#ff8c5a',8,1.6);SFX.bossHit();
}
function bossSummon(){
  const count=boss.phase===1?2:3;
  for(let i=0;i<count;i++){const sx=boss.x+rand(-60,60),sy=boss.y+30;
    const type=Math.random()<0.6?'crawler':'hopper';
    enemies.push(makeEnemy({x:sx,y:sy,type}));
    burst(sx+7,sy+7,'#c08cff',10,1.8);}
  SFX.bossRoar();
}
function bossSlam(){boss.vy=-6.5;boss.vx=0;burst(boss.x+boss.w/2,boss.y+boss.h,'#ff6a2a',12,2.2);}
function bossSlamImpact(){
  const cx=boss.x+boss.w/2,cy=boss.y+boss.h;
  const count=boss.phase===1?8:12;
  for(let i=0;i<count;i++){const a=(i/count)*Math.PI*2;
    projectiles.push({x:cx,y:cy,vx:Math.cos(a)*1.5,vy:Math.sin(a)*1.5,life:3,from:'boss'});}
  shake=Math.max(shake,8);SFX.bossPhase();
  for(let i=0;i<24;i++)burst(cx,cy,'#ff9a5c',1,3);
}
function bossCharge(){
  const px=player.x+player.w/2,cx=boss.x+boss.w/2;
  boss.vx=(px<cx?-1:1)*3.4;boss.facing=px<cx?-1:1;
  burst(boss.x+boss.w/2,boss.y+boss.h-4,'#ff6a2a',14,2);SFX.bossRoar();
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
  const dist=Math.hypot(px-cx,py-cy)||1;
  const R=lightRadius();
  if(dist<R*1.15){boss.hp-=3.0*dt;boss.flash=1;if(Math.random()<0.15)SFX.hitEnemy();}
  if(flashWave&&!boss.flashDone){
    const d=Math.hypot(cx-flashWave.x,cy-flashWave.y);
    if(d<180&&flashWave.life>flashWave.maxLife-0.1){
      const power=1-d/180;boss.hp-=4+8*power;boss.hitFlash=1;shake=Math.max(shake,6);}
  }
  if(boss.hp<=0){
    boss.dead=true;shake=16;SFX.killEnemy();SFX.bossRoar();
    for(let i=0;i<60;i++)burst(cx,cy,i%2?'#ff9a5c':'#ffe9a3',1,4);
    unlockAch('boss_slayer');
    setTimeout(()=>{if(state==='play'&&!paused)winLevel2();},1500);
    return;
  }
  boss.stateT-=dt;
  if(boss.state==='idle'){boss.vx*=0.85;
    if(boss.stateT<=0){boss.attackCount++;const roll=Math.random();
      if(boss.phase===1){
        if(roll<0.55)boss.state='spread';else if(roll<0.85)boss.state='summon';else boss.state='charge';
      }else{
        if(roll<0.35)boss.state='spread';else if(roll<0.55)boss.state='summon';
        else if(roll<0.75)boss.state='charge';else boss.state='slam';}
      boss.stateT=boss.state==='idle'?boss.idleDur:0.6;}}
  else if(boss.state==='spread'){
    if(boss.stateT<0.55&&!boss._fired){bossSpread();boss._fired=true;}
    if(boss.stateT<=0){boss.state='idle';boss.stateT=boss.idleDur;boss._fired=false;}}
  else if(boss.state==='summon'){
    if(boss.stateT<0.4&&!boss._fired){bossSummon();boss._fired=true;}
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
  if(player.inv<=0&&overlap(player,boss)){
    player.light=Math.max(0,player.light-20);player.inv=1.3;
    const k=px<cx?-1:1;player.vx=k*5;player.vy=-4;
    shake=Math.max(shake,7);burst(px,py,'#ff4d7a',14,2.4);SFX.hurt();
    damagedThisLevel=true;}
}

/* ====================== ОБНОВЛЕНИЕ ====================== */
function update(dt){
  time+=dt;
  if(mutedBannerT>0)mutedBannerT-=dt;
  if(hitFlash>0)hitFlash-=dt;
  if(flashCooldown>0)flashCooldown-=dt;
  if(flashWave){flashWave.life-=dt;
    flashWave.r=flashWave.maxR*(1-flashWave.life/flashWave.maxLife);
    if(flashWave.life<=0)flashWave=null;}
  if(bossIntroT>0)bossIntroT-=dt;
  if(achievementPopup){achievementPopup.t-=dt;if(achievementPopup.t<=0)achievementPopup=null;}
  if(paused&&state!=='play')paused=false;
  pauseBuffer-=dt;
  if(pauseBuffer>0){pauseBuffer=0;
    if(state==='play'){paused=!paused;SFX.storyNext();}}

  if(state==='play')runTime+=dt;

  for(const m of motes){m.x+=m.vx;m.y+=m.vy;
    if(m.y<-5)m.y=VH+5;if(m.y>VH+5)m.y=-5;}
  for(let i=particles.length-1;i>=0;i--){const p=particles[i];
    p.x+=p.vx;p.y+=p.vy;p.vy+=0.06;p.life-=dt;
    if(p.life<=0)particles.splice(i,1);}

  /* === TITLE === */
  if(state==='title'){
    if(confirmBuffer>0){confirmBuffer=0;storyPhase='intro';storySlide=0;
      storyTimer=0;storyDone=false;state='story';SFX.storyNext();}
    return;
  }

  /* === STORY === */
  if(state==='story'){
    const slides=STORY[storyPhase],cur=slides[storySlide];
    if(!cur)return;
    const totalChars=cur.lines.join('').length,charTime=0.035;
    if(!storyDone){storyTimer+=dt;
      const revealed=Math.floor(storyTimer/charTime);
      const prevRevealed=Math.floor((storyTimer-dt)/charTime);
      if(revealed>prevRevealed&&revealed<totalChars&&revealed%3===0)SFX.storyBlip();
      if(revealed>=totalChars)storyDone=true;}
    if(confirmBuffer>0){
      confirmBuffer=0;
      if(!storyDone){storyDone=true;storyTimer=totalChars*charTime+0.1;}
      else{storySlide++;storyTimer=0;storyDone=false;
        if(storySlide>=slides.length){
          if(storyPhase==='intro'){loadLevel(1);state='play';SFX.motif();}
          else if(storyPhase==='interlude'){loadLevel(2);bossIntroT=2.5;state='play';SFX.bossRoar();}
          else if(storyPhase==='ending'){state='title';storySlide=0;}
        }else SFX.storyNext();}
    }
    return;
  }

  /* === DEAD / WIN === */
  restartBuffer-=dt;
  if(restartBuffer>0){
    restartBuffer=0;
    if(state==='dead'){deaths++;deathsThisLevel++;saveGame();loadLevel(level);state='play';}
    else if(state==='win')state='title';
    return;
  }
  if(paused)return;
  if(state!=='play')return;

  /* === УПРАВЛЕНИЕ === */
  const speed=2.0,accel=0.35;
  if(left()){player.vx-=accel;player.face=-1;}
  if(right()){player.vx+=accel;player.face=1;}
  if(!left()&&!right())player.vx*=player.onGround?0.72:0.90;
  player.vx=clamp(player.vx,-speed,speed);

  const focusing=focus()&&player.light>0;

  /* === ВСПЫШКА === */
  if(focusing&&!prevFocus&&flashCooldown<=0&&player.light>15){
    SFX.flash();SFX.choir();shake=Math.max(shake,5);flashCooldown=0.55;
    flashesThisLevel++;
    const cx=player.x+player.w/2,cy=player.y+player.h/2;
    flashWave={x:cx,y:cy,r:20,maxR:180,life:0.55,maxLife:0.55};
    if(boss)boss.flashDone=false;
    player.light=Math.max(0,player.light-18);
    for(const e of enemies){if(e.dead)continue;
      const dx=(e.x+e.w/2)-cx,dy=(e.y+e.h/2)-cy,d=Math.hypot(dx,dy)||1;
      if(d<180){const power=1-d/180;e.hp-=1.5+3.5*power;e.flash=1;
        e.x+=(dx/d)*(3+5*power);e.y+=(dy/d)*(3+5*power);
        if(e.hp<=0){e.dead=true;burst(e.x+e.w/2,e.y+e.h/2,'#8ea2ff',14,2.4);
          player.light=Math.min(100,player.light+4);}}}
    if(boss&&!boss.dead){
      const bd=Math.hypot(boss.x+boss.w/2-cx,boss.y+boss.h/2-cy);
      if(bd<180){const power=1-bd/180;boss.hp-=4+8*power;boss.hitFlash=1;}}
    if(flashesThisLevel>=20)unlockAch('pyromancer');
  }
  prevFocus=focusing;

  /* === COYOTE-TIME + ПРЫЖОК === */
  if(player.onGround)player.coyote=0.1;
  else player.coyote=Math.max(0,player.coyote-dt);

  jumpBuffer-=dt;
  if(jumpBuffer>0){
    if(player.onGround){
      player.vy=-7.8;player.jumps=1;jumpBuffer=0;player.coyote=0;
      burst(player.x+5,player.y+14,'#6f7fd6',5,1.2);SFX.jump();
    }else if(player.coyote>0){
      player.vy=-7.8;player.jumps=1;jumpBuffer=0;player.coyote=0;
      burst(player.x+5,player.y+14,'#7f8fe6',6,1.3);SFX.jump();
    }else if(player.jumps<2){
      player.vy=-7.0;player.jumps=2;jumpBuffer=0;
      burst(player.x+5,player.y+14,'#9aa8ff',7,1.6);SFX.djump();}
  }

  player.vy+=0.5;if(player.vy>11)player.vy=11;
  const wasGround=player.onGround,fallSpeed=player.vy;
  moveAndCollide(player);
  if(player.onGround&&!wasGround&&fallSpeed>4){
    SFX.land();burst(player.x+5,player.y+14,'#4a5070',4,1.0);stepTimer=0;}
  if(player.onGround)player.jumps=0;
  if(player.inv>0)player.inv-=dt;

  /* === ШАГИ === */
  const walking=player.onGround&&Math.abs(player.vx)>0.5;
  if(walking){stepTimer-=dt;
    if(stepTimer<=0){SFX.footstep(stepSide,levelSurface);stepSide^=1;
      stepTimer=0.34-Math.abs(player.vx)*0.04;}}
  else stepTimer=0;

  /* === СВЕТ === */
  let drainMult=1;
  if(difficulty==='easy')drainMult=0.7;
  else if(difficulty==='hard')drainMult=1.4;
  const drain=(focusing?8.0:2.6)*dt*drainMult;

  // 2.6: стойка рядом с неиспользованным грибом восстанавливает свет
  let nearShroom=false;
  for(const s of shrooms){
    if(s.used)continue;
    const d=Math.hypot(player.x-s.x,player.y-s.y);
    if(d<20&&d>10)nearShroom=true; // рядом, но не касается
  }
  if(nearShroom)player.light=Math.min(100,player.light+0.5*dt);

  player.light=Math.max(0,player.light-drain);

  if(player.light<=0){die('Свет угас...');return;}
  if(player.y>VH+30){die('Ты растворился во тьме...');return;}

  /* === 9.1: голос посоха === */
  whisperT-=dt;
  if(whisperT<=0){
    if(player.light<15){whisperMsg='«Не угасай...»';whisperT=4;SFX.whisper();}
    else if(player.light<30){whisperMsg='«Я ещё держусь...»';whisperT=6;}
    else whisperMsg='';
  }

  /* === ГРИБЫ === */
  for(const s of shrooms){
    if(s.used){s.timer-=dt;if(s.timer<=0)s.used=false;continue;}
    if(overlap(player,{x:s.x-s.r,y:s.y-s.r,w:s.r*2,h:s.r*2})){
      s.used=true;s.timer=14;player.light=Math.min(100,player.light+55);
      burst(s.x,s.y,'#7ff0d8',14,2.2);shake=Math.max(shake,2);SFX.shroom();}
  }

  /* === ОСКОЛКИ === */
  for(const sh of shards){
    if(sh.taken)continue;
    if(overlap(player,{x:sh.x-5,y:sh.y-5,w:10,h:10})){
      sh.taken=true;player.light=Math.min(100,player.light+12);
      burst(sh.x,sh.y,'#ffd76a',9,1.8);SFX.shard();}
  }

  /* === ФОНАРИ === */
  for(const l of lanterns){
    if(l.lit)continue;
    if(overlap(player,{x:l.x-8,y:l.y-8,w:16,h:16})&&player.light>10){
      l.lit=true;player.light=Math.max(0,player.light-10);
      burst(l.x,l.y,'#ffdd88',20,2.5);SFX.shroom();}
  }

  /* === 4.2: ДНЕВНИКИ === */
  for(const d of DIARIES){
    if(diariesFound.includes(d.id))continue;
    if(d.level!=String(level)&&d.level!==level)continue;
    if(overlap(player,{x:d.x-8,y:d.y-8,w:16,h:16})){
      diariesFound.push(d.id);SFX.diary();
      unlockAch('first_diary');
      if(diariesFound.length>=8)unlockAch('all_diaries');
      saveGame();
      showDiary(d);
    }
  }

  /* === ВРАГИ === */
  const R=lightRadius();
  const px=player.x+player.w/2,py=player.y+player.h/2;
  let nearestDist=9999;
  let aliveEnemies=0;

  for(const e of enemies){
    if(e.dead)continue;
    aliveEnemies++;
    e.t+=dt*2.4;
    const ex=e.x+e.w/2,ey=e.y+e.h/2;
    const dx=px-ex,dy=py-ey;
    const dist=Math.hypot(dx,dy)||1;
    if(dist<nearestDist)nearestDist=dist;
    const lit=dist<R;

    if(e.type==='mimic'){
      if(!e.revealed){
        e.x=e.baseX;e.y=e.baseY;
        if(dist<50)e.revealed=true;
      }else{
        e.revealT+=dt;
        if(lit){e.hp-=e.hitDmg*dt;e.flash=1;
          if(Math.random()<0.3)SFX.hitEnemy();
          e.x-=(dx/dist)*e.retreat;e.y-=(dy/dist)*e.retreat;}
        else{e.flash=Math.max(0,e.flash-dt*2);
          e.x+=(dx/dist)*e.speed;e.y+=(dy/dist)*e.speed;}
        if(e.hp<=0){e.dead=true;burst(ex,ey,'#ff8c5a',12,2.2);
          player.light=Math.min(100,player.light+3);SFX.killEnemy();}
      }
    }
    else if(e.type==='swarm'){
      e.swarmPhase+=dt*6;
      const wobble=Math.sin(e.swarmPhase)*0.4;
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;
        e.x-=(dx/dist)*e.retreat;e.y-=(dy/dist)*e.retreat;
        if(e.hp<=0){e.dead=true;burst(ex,ey,'#ff6a8a',8,1.8);
          player.light=Math.min(100,player.light+1);SFX.hitEnemy();}}
      else{e.flash=Math.max(0,e.flash-dt*2);
        const ang=Math.atan2(dy,dx)+wobble;
        e.x+=Math.cos(ang)*e.speed;e.y+=Math.sin(ang)*e.speed;}
    }
    else if(e.type==='hopper'){
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;SFX.hitEnemy();
        e.x-=(dx/dist)*e.retreat;e.y-=(dy/dist)*e.retreat;
        e.jumpVx*=0.8;e.jumpVy*=0.8;}
      else{e.flash=Math.max(0,e.flash-dt*2);e.jumpTimer-=dt;
        if(e.jumpTimer<=0){e.jumpTimer=rand(0.9,1.4);
          e.jumpVx=(dx/dist)*3.2;e.jumpVy=(dy/dist)*3.2-2.2;
          burst(ex,ey,'#7c5cff',5,1.2);}
        e.x+=e.jumpVx*dt*30;e.y+=e.jumpVy*dt*30;
        e.jumpVx*=Math.pow(0.92,dt*60);e.jumpVy+=6*dt;}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#c08cff',12,2.2);
        player.light=Math.min(100,player.light+4);SFX.killEnemy();}
    }
    else if(e.type==='sentinel'){
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;SFX.hitEnemy();}
      else{e.flash=Math.max(0,e.flash-dt*2);e.shootTimer-=dt;
        if(e.shootTimer<=0&&dist<220){e.shootTimer=1.8;
          projectiles.push({x:ex,y:ey,vx:(dx/dist)*1.6,vy:(dy/dist)*1.6,life:3,from:'sentinel'});
          burst(ex,ey,'#ff8c5a',4,1.2);}}
      if(e.hp<=0){e.dead=true;burst(ex,ey,'#ff9a5c',16,2.6);
        player.light=Math.min(100,player.light+6);SFX.killEnemy();}
    }
    else{
      if(lit){e.hp-=e.hitDmg*dt;e.flash=1;SFX.hitEnemy();
        e.x-=(dx/dist)*e.retreat;e.y-=(dy/dist)*e.retreat;
        if(e.hp<=0){e.dead=true;
          burst(ex,ey,e.type==='devourer'?'#7c5cff':'#8ea2ff',
            e.type==='devourer'?20:14,2.4);
          player.light=Math.min(100,player.light+(e.type==='devourer'?8:4));
          SFX.killEnemy();}}
      else{e.flash=Math.max(0,e.flash-dt*2);
        e.x+=(dx/dist)*e.speed;e.y+=(dy/dist)*e.speed;}
    }

    if(player.inv<=0&&overlap(player,e)){
      const loss=11*e.dmgMult;
      player.light=Math.max(0,player.light-loss);
      player.inv=1.1;
      const k=px<ex?-1:1;player.vx=k*4.2;player.vy=-3.6;
      shake=Math.max(shake,5);hitFlash=0.25;
      burst(px,py,e.dmgMult>1?'#ff4d7a':'#ff6b8a',10,2);
      SFX.hurt();damagedThisLevel=true;}
  }

  /* === КОМНАТА-ИСПЫТАНИЕ 4.3 === */
  if(trialActive){
    trialTimer-=dt;
    if(trialTimer<=0||aliveEnemies===0){
      const survived=trialTimer>0;
      trialActive=null;
      if(survived||aliveEnemies===0){
        player.light=100;
        SFX.win();
        unlockAch('trial_master');
        // Постоянный бонус
        player.light=Math.min(120,player.light+20);
      }
    }
  } else {
    // Проверка входа в испытание — по координатам
    if(level===1&&!achievements.includes('trial_master')&&player.x>1400&&player.x<1480){
      if(aliveEnemies<6){trialActive='forest';trialTimer=20;}
    }
  }

  if(boss&&!boss.dead)updateBoss(dt);

  /* === СНАРЯДЫ === */
  for(let i=projectiles.length-1;i>=0;i--){
    const p=projectiles[i];
    p.x+=p.vx;p.y+=p.vy;p.life-=dt;
    let hitWall=false;
    for(const pl of platforms){
      if(p.x>pl.x&&p.x<pl.x+pl.w&&p.y>pl.y&&p.y<pl.y+pl.h){hitWall=true;break;}}
    if(!hitWall&&player.inv<=0&&p.x>player.x&&p.x<player.x+player.w&&
       p.y>player.y&&p.y<player.y+player.h){
      player.light=Math.max(0,player.light-(p.from==='boss'?12:8));
      player.inv=1.1;burst(p.x,p.y,'#ff5c8a',8,1.8);
      SFX.hurt();shake=Math.max(shake,4);hitFlash=0.25;
      damagedThisLevel=true;
      projectiles.splice(i,1);continue;
    }
    if(p.life<=0||hitWall){burst(p.x,p.y,'#8f6aff',4,1.0);projectiles.splice(i,1);}
  }

  /* === 9.2: Скрытый силуэт === */
  if(!silhouette&&Math.random()<0.001&&state==='play'){
    const sx=player.x+(Math.random()<0.5?-140:140);
    if(sx>0&&sx<LEVEL_W)
      silhouette={x:sx,y:GROUND_Y-20,t:4,alpha:0};
  }
  if(silhouette){
    silhouette.t-=dt;
    const d=Math.abs(player.x-silhouette.x);
    if(d<60){silhouette.t=0;burst(silhouette.x,silhouette.y,'#8fa4ff',12,2);}
    silhouette.alpha=Math.min(0.5,silhouette.t*0.5);
    if(silhouette.t<=0)silhouette=null;
  }

  /* === МУЗЫКА === */
  const progress=clamp(player.x/LEVEL_W,0,1);
  const danger=clamp(1-nearestDist/150,0,1);
  let intensity=progress*0.5+danger*0.7;
  if(boss&&!boss.dead)intensity=Math.max(intensity,0.85);
  SFX.setIntensity(clamp(intensity,0,1));

  /* === АЛТАРЬ (Ур.1) === */
  if(ALTAR&&overlap(player,ALTAR)){
    burst(ALTAR.x+17,ALTAR.y+10,'#ffe9a3',40,3.5);
    shake=8;SFX.win();unlockAch('first_light');
    // Достижения по уровню
    if(!damagedThisLevel&&deathsThisLevel===0)unlockAch('no_damage');
    if(runTime<180)unlockAch('fast');
    if(!bestTimes[1]||runTime<bestTimes[1])bestTimes[1]=runTime;
    saveGame();
    if(level===1){
      storyPhase='interlude';storySlide=0;storyTimer=0;storyDone=false;state='story';
    }
    return;
  }
  /* === ВЫХОД (Ур.1.5) === */
  if(EXIT&&overlap(player,EXIT)){
    burst(EXIT.x+17,EXIT.y+10,'#ffe9a3',30,3);
    SFX.win();
    if(!bestTimes['1.5']||runTime<bestTimes['1.5'])bestTimes['1.5']=runTime;
    saveGame();
    storyPhase='interlude';storySlide=0;storyTimer=0;storyDone=false;state='story';
    return;
  }

  /* === КАМЕРА === */
  const maxCamX=Math.max(0,LEVEL_W-VW);
  const targetX=clamp(player.x+player.w/2-VW/2,0,maxCamX);
  cam.x+=(targetX-cam.x)*0.12;
  if(shake>0)shake=Math.max(0,shake-dt*18);
}

function showDiary(d){
  // Простое всплытие внизу
  const el=document.createElement('div');
  el.style.cssText=`position:fixed;bottom:140px;left:50%;transform:translateX(-50%);
    background:rgba(4,6,16,0.95);color:#ffd76a;padding:12px 18px;
    border:2px solid rgba(255,215,106,0.6);border-radius:6px;
    font:11px "Courier New",monospace;max-width:80%;z-index:100;
    text-align:center;pointer-events:none;transition:opacity 0.5s`;
  el.innerHTML=`<b>${d.title}</b><br><br><span style="color:#c9d4e8">${d.text}</span>`;
  document.body.appendChild(el);
  setTimeout(()=>{el.style.opacity='0';setTimeout(()=>el.remove(),500);},5000);
}

function winLevel2(){
  storyPhase='ending';storySlide=0;storyTimer=0;storyDone=false;state='story';
  // Скины в зависимости от статистики
  if(deaths===0&&!skins.includes('gold'))skins.push('gold');
  if(deaths>=5&&!skins.includes('shadow'))skins.push('shadow');
  if(diariesFound.length>=8&&!skins.includes('phantom'))skins.push('phantom');
  saveGame();
}

function die(msg){
  state='dead';player.dead=true;paused=false;shake=9;
  burst(player.x+5,player.y+7,'#ffd76a',30,3);
  SFX.death();SFX.setIntensity(1);
  document.title=msg;
}

/* ====================== ОТРИСОВКА ====================== */
function drawBackground(){
  const g=ctx.createLinearGradient(0,0,0,VH);
  g.addColorStop(0,'#0a0d1e');g.addColorStop(1,'#04050c');
  ctx.fillStyle=g;ctx.fillRect(0,0,VW,VH);
  if(level===2&&state==='play'){ctx.fillStyle='rgba(40,10,20,0.25)';ctx.fillRect(0,0,VW,VH);}
  else if(level==='1.5'){ctx.fillStyle='rgba(10,30,15,0.20)';ctx.fillRect(0,0,VW,VH);}
  ctx.fillStyle='#0d1122';
  for(let i=0;i<30;i++){let bx=((i*180-cam.x*0.25)%(VW+400)+VW+400)%(VW+400)-200;
    ctx.fillRect(Math.round(bx),VH-(95+(i%4)*38)-45,32+(i%3)*20,95+(i%4)*38);}
  ctx.fillStyle='#131a2e';
  for(let i=0;i<34;i++){let bx=((i*140-cam.x*0.5)%(VW+320)+VW+320)%(VW+320)-160;
    ctx.fillRect(Math.round(bx),VH-(55+(i%5)*32)-22,22+(i%4)*15,55+(i%5)*32);}
  // Погодные эффекты 6.7
  if(level==='1.5'&&state==='play'){
    for(let i=0;i<15;i++){
      const t=(time*30+i*40)%VH;
      const x=(i*53+Math.sin(time+i)*10)%VW;
      ctx.fillStyle='rgba(90,140,90,0.35)';
      ctx.fillRect(x,t,1,2);}}
  else if(level===2&&state==='play'){
    for(let i=0;i<10;i++){
      const t=(time*40+i*60)%VH;
      const x=(i*71+Math.cos(time+i)*8)%VW;
      ctx.fillStyle='rgba(255,120,60,0.25)';
      ctx.fillRect(x,t,1,1);}}
  const f=ctx.createLinearGradient(0,VH-70,0,VH);
  f.addColorStop(0,'rgba(30,40,80,0)');
  f.addColorStop(1,level===2?'rgba(120,40,50,0.20)':
    level==='1.5'?'rgba(30,80,50,0.16)':'rgba(40,55,110,0.16)');
  ctx.fillStyle=f;ctx.fillRect(0,VH-70,VW,70);
}
function drawPlatforms(){
  for(const p of platforms){
    const x=Math.round(p.x-cam.x),y=Math.round(p.y);
    if(x+p.w<-10||x>VW+10)continue;
    ctx.fillStyle='#221d33';ctx.fillRect(x,y,p.w,p.h);
    ctx.fillStyle='#3d3560';ctx.fillRect(x,y,p.w,2);
    ctx.fillStyle='#514679';ctx.fillRect(x,y,p.w,1);
    ctx.fillStyle='#2c2542';
    for(let i=0;i<p.w;i+=6){const h=((p.x+i)*7919)%11;if(h<4)ctx.fillRect(x+i,y+4+(h*2),2,2);}
    ctx.fillStyle='#171327';ctx.fillRect(x,y+p.h-3,p.w,3);
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
    if(l.lit){
      ctx.fillStyle='#ffdd88';ctx.fillRect(x-3,y-10,7,6);
      const pulse=0.7+Math.sin(time*3+l.x)*0.3;
      ctx.fillStyle=`rgba(255,220,140,${0.4*pulse})`;
      ctx.beginPath();ctx.arc(x,y-8,22,0,6.283);ctx.fill();
    }else{
      ctx.fillStyle='#3a2e4a';ctx.fillRect(x-3,y-10,7,6);
      ctx.fillStyle='#5a4a70';ctx.fillRect(x-2,y-9,5,4);
    }
  }
}
function drawDiaries(){
  for(const d of DIARIES){
    if(diariesFound.includes(d.id))continue;
    if(d.level!=String(level)&&d.level!==level)continue;
    const x=Math.round(d.x-cam.x),y=d.y;
    if(x<-30||x>VW+30)continue;
    const pulse=0.6+Math.sin(time*3+d.x)*0.4;
    ctx.fillStyle='#c9a05a';ctx.fillRect(x-3,y-3,7,7);
    ctx.fillStyle='#e0b86a';ctx.fillRect(x-3,y-3,7,1);
    ctx.fillStyle='#8a6a30';ctx.fillRect(x-2,y+1,5,1);
    ctx.fillStyle=`rgba(255,215,150,${0.3*pulse})`;
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
  const pulse=0.7+Math.sin(time*3)*0.3;
  ctx.fillStyle='#ffe9a3';ctx.fillRect(x+14,y+10,6,10);
  ctx.fillStyle=`rgba(255,220,140,${0.5*pulse})`;ctx.fillRect(x+11,y+7,12,16);
  ctx.fillStyle='#fffbe0';ctx.fillRect(x+15,y+12,4,5);
}
function drawExit(){
  if(!EXIT)return;
  const x=Math.round(EXIT.x-cam.x),y=EXIT.y;
  if(x<-60||x>VW+60)return;
  ctx.fillStyle='#2a2440';ctx.fillRect(x-4,y+22,42,18);
  const pulse=0.7+Math.sin(time*3)*0.3;
  ctx.fillStyle=`rgba(180,255,200,${0.6*pulse})`;ctx.fillRect(x,y-4,34,30);
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
  ctx.fillStyle='rgba(0,0,0,0.45)';
  ctx.beginPath();ctx.ellipse(x+28,y+64,30,6,0,0,6.283);ctx.fill();
  const body=fl?'#8a3030':'#1a0a14';
  ctx.fillStyle=body;ctx.fillRect(x+8,y+10,40,44);ctx.fillRect(x+4,y+18,48,30);ctx.fillRect(x,y+28,56,18);
  ctx.fillStyle=fl?'#b04040':'#2a0f0f';
  ctx.fillRect(x+2,y-6,8,18);ctx.fillRect(x+46,y-6,8,18);
  ctx.fillRect(x-2,y-14,8,12);ctx.fillRect(x+50,y-14,8,12);
  const crown=0.5+0.5*Math.sin(time*4);
  ctx.fillStyle=`rgba(255,90,40,${0.6*crown})`;ctx.fillRect(x+16,y-4,24,6);
  ctx.fillStyle=`rgba(255,180,80,${0.5*crown})`;ctx.fillRect(x+20,y-8,16,5);
  const eyeCol=boss.phase===2?'#ff2020':'#ff6030';
  ctx.fillStyle=fl?'#ffffff':eyeCol;ctx.fillRect(x+14,y+22,8,6);ctx.fillRect(x+34,y+22,8,6);
  ctx.fillStyle='#ffe9a3';ctx.fillRect(x+17,y+24,3,2);ctx.fillRect(x+37,y+24,3,2);
  ctx.fillStyle='#0a0000';ctx.fillRect(x+18,y+38,20,6);
  ctx.fillStyle=fl?'#ffffff':'#ff5c2a';
  for(let i=0;i<5;i++){ctx.fillRect(x+19+i*4,y+38,2,3);ctx.fillRect(x+19+i*4,y+41,2,3);}
  const auraA=0.10+0.05*Math.sin(time*3);
  const auraCol=boss.phase===2?`rgba(255,80,40,${auraA*1.5})`:`rgba(180,60,120,${auraA})`;
  ctx.fillStyle=auraCol;ctx.beginPath();ctx.arc(x+28,y+32,44,0,6.283);ctx.fill();
  if(boss.hitFlash>0){ctx.fillStyle=`rgba(255,220,180,${boss.hitFlash*0.4})`;
    ctx.beginPath();ctx.arc(x+28,y+32,44,0,6.283);ctx.fill();}
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
  ctx.textAlign='center';ctx.fillText('ХРАНИТЕЛЬ ПЕПЛА',VW/2,y-6);ctx.textAlign='left';
}
function drawEnemies(){
  for(const e of enemies){
    if(e.dead)continue;
    const bob=e.type==='hopper'?0:Math.sin(e.t)*2;
    const x=Math.round(e.x-cam.x),y=Math.round(e.y+bob);
    if(x<-50||x>VW+50)continue;
    if(e.type==='crawler'){
      const c=e.flash>0?'#3a3f78':'#0e0e1a';
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+7,y+7,7,0,6.283);ctx.fill();
      ctx.fillRect(x+1,y+9,12,5);ctx.fillRect(x-1,y+7,3,4);ctx.fillRect(x+12,y+8,3,4);
      ctx.fillStyle=e.flash>0?'#ffffff':'#ff4d6d';
      ctx.fillRect(x+3,y+5,2,2);ctx.fillRect(x+9,y+5,2,2);
    }
    else if(e.type==='hopper'){
      const c=e.flash>0?'#7c6aff':'#1a0f2e';
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+6,y+6,6,0,6.283);ctx.fill();
      ctx.fillRect(x+1,y-2,3,4);ctx.fillRect(x+8,y-2,3,4);
      ctx.fillStyle=e.flash>0?'#ffffff':'#c08cff';
      ctx.fillRect(x+2,y+4,2,2);ctx.fillRect(x+8,y+4,2,2);
    }
    else if(e.type==='devourer'){
      const c=e.flash>0?'#5c4aaa':'#0a0a18';
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+11,y+11,11,0,6.283);ctx.fill();
      ctx.fillRect(x-2,y+14,26,8);ctx.fillRect(x+1,y+20,4,5);
      ctx.fillRect(x+17,y+20,4,5);ctx.fillRect(x+7,y+22,4,4);
      ctx.fillStyle=e.flash>0?'#ffffff':'#b060ff';
      ctx.fillRect(x+4,y+8,3,3);ctx.fillRect(x+15,y+8,3,3);ctx.fillRect(x+9,y+13,3,3);
      ctx.fillStyle=`rgba(120,80,200,${0.10+0.05*Math.sin(e.t*3)})`;
      ctx.beginPath();ctx.arc(x+11,y+11,18,0,6.283);ctx.fill();
    }
    else if(e.type==='sentinel'){
      const c=e.flash>0?'#ff8060':'#1a0f0a';
      ctx.fillStyle=c;ctx.fillRect(x+3,y,8,20);ctx.fillRect(x+2,y+2,10,4);ctx.fillRect(x+2,y+16,10,4);
      const glow=e.shootTimer<0.4?'#ffdd66':'#ff5c2a';
      ctx.fillStyle=e.flash>0?'#ffffff':glow;ctx.fillRect(x+5,y+8,4,4);
      ctx.fillStyle='rgba(255,100,60,0.25)';ctx.fillRect(x+3,y+6,8,8);
    }
    else if(e.type==='mimic'){
      if(!e.revealed){
        ctx.fillStyle='#4d7f74';ctx.fillRect(x-1,y-1,3,5);
        ctx.fillStyle='#5fe0c4';ctx.fillRect(x-4,y-5,9,4);
        ctx.fillStyle='#a6fff0';ctx.fillRect(x-4,y-5,9,1);
      }else{
        const c=e.flash>0?'#8a3a3a':'#3a0a0a';
        ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+6,y+6,6,0,6.283);ctx.fill();
        ctx.fillStyle=e.flash>0?'#ffffff':'#ff8c5a';
        ctx.fillRect(x+2,y+4,2,2);ctx.fillRect(x+7,y+4,2,2);
        ctx.fillStyle='#0a0000';ctx.fillRect(x+2,y+8,7,2);
      }
    }
    else if(e.type==='swarm'){
      const c=e.flash>0?'#ff8aa0':'#4a0a1a';
      ctx.fillStyle=c;ctx.beginPath();ctx.arc(x+4,y+4,4,0,6.283);ctx.fill();
      ctx.fillStyle=e.flash>0?'#ffffff':'#ff5c7a';
      ctx.fillRect(x+2,y+2,2,2);
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
    default:return{cloak:'#2e2748',hood:'#4a3d78',glow:'#ffd76a',eyes:'#ffd76a'};
  }
}
function drawPlayer(){
  if(state==='dead')return;
  const x=Math.round(player.x-cam.x),y=Math.round(player.y);
  if(player.inv>0&&Math.floor(time*20)%2===0)return;
  const f=player.face;
  const sc=getSkinColors();
  // Тряпичная физика плаща 6.2
  const cloakSway=Math.sin(time*8)*Math.min(2,Math.abs(player.vx)*1.5);
  ctx.fillStyle=sc.cloak;ctx.fillRect(x+cloakSway*0.3,y+4,10,10);
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
  ctx.fillStyle=sc.glow;ctx.fillRect(sx-1,y-9,4,4);
  ctx.fillStyle=`rgba(255,235,170,${0.55*p2})`;ctx.fillRect(sx-3,y-11,8,8);
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
}
function drawFlashWave(){
  if(!flashWave)return;
  const x=flashWave.x-cam.x,y=flashWave.y;
  const a=flashWave.life/flashWave.maxLife;
  ctx.globalCompositeOperation='lighter';
  ctx.strokeStyle=`rgba(255,230,160,${a*0.9})`;ctx.lineWidth=3;
  ctx.beginPath();ctx.arc(x,y,flashWave.r,0,6.283);ctx.stroke();
  ctx.strokeStyle=`rgba(170,200,255,${a*0.55})`;ctx.lineWidth=1;
  ctx.beginPath();ctx.arc(x,y,flashWave.r*0.72,0,6.283);ctx.stroke();
  const g=ctx.createRadialGradient(x,y,0,x,y,flashWave.r);
  g.addColorStop(0,`rgba(255,240,190,${a*0.35})`);
  g.addColorStop(0.6,`rgba(255,200,120,${a*0.12})`);
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
  // Следы 6.3
  if(state==='play'&&player.onGround&&Math.abs(player.vx)>0.5){
    const sx=Math.round(player.x-cam.x+5),sy=Math.round(player.y+14);
    if(Math.floor(time*8)%2===0){
      ctx.globalAlpha=0.25;
      ctx.fillStyle=getSkinColors().glow;
      ctx.fillRect(sx-1,sy,2,1);
      ctx.globalAlpha=1;
    }
  }
}
function drawMotes(){
  for(const m of motes){
    const x=Math.round(m.x-cam.x*0.8);
    const y=Math.round(m.y+Math.sin(time+m.x*0.01)*4);
    if(x<-5||x>VW+5)continue;
    ctx.globalAlpha=m.a*(0.5+0.5*Math.sin(time*1.5+m.x));
    ctx.fillStyle=level===2?'#ff9a5c':level==='1.5'?'#8fff9a':'#8fa4ff';
    ctx.fillRect(x,y,m.s,m.s);
  }
  ctx.globalAlpha=1;
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
  for(const s of shrooms){
    if(s.used)continue;
    const x=s.x-cam.x,y=s.y;
    if(x<-60||x>VW+60)continue;
    const g=dctx.createRadialGradient(x,y-3,0,x,y-3,28);
    g.addColorStop(0,'rgba(0,0,0,0.85)');g.addColorStop(0.6,'rgba(0,0,0,0.30)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(x,y-3,28,0,6.283);dctx.fill();
  }
  for(const l of lanterns){
    if(!l.lit)continue;
    const x=l.x-cam.x,y=l.y;
    if(x<-80||x>VW+80)continue;
    const g=dctx.createRadialGradient(x,y-8,0,x,y-8,42);
    g.addColorStop(0,'rgba(0,0,0,0.85)');g.addColorStop(0.6,'rgba(0,0,0,0.30)');
    g.addColorStop(1,'rgba(0,0,0,0)');
    dctx.fillStyle=g;dctx.beginPath();dctx.arc(x,y-8,42,0,6.283);dctx.fill();
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
    g.addColorStop(0,'rgba(255,205,110,0.16)');
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
  const pct=player.light/100;
  const col=pct>0.5?'#ffd76a':pct>0.22?'#ff9a3c':'#ff4d5e';
  ctx.fillStyle=col;ctx.fillRect(x,y,Math.round(w*pct),h);
  ctx.fillStyle='rgba(255,255,255,0.25)';ctx.fillRect(x,y,Math.round(w*pct),2);
  ctx.fillStyle='#7f8bb0';ctx.font='8px "Courier New",monospace';
  ctx.fillText('СВЕТ',x,y+h+11);
  // Таймер + смерти 1.2
  const mm=Math.floor(runTime/60),ss=Math.floor(runTime%60);
  ctx.fillStyle='#8fa0c8';
  ctx.fillText(`⏱ ${mm}:${ss<10?'0':''}${ss}  💀 ${deaths}`,x,y+h+22);
  // Индикатор гриба рядом
  if(player.light<25&&Math.floor(time*3)%2===0&&state==='play'&&!paused){
    ctx.fillStyle='#ff8b9c';ctx.fillText('НАЙДИ ГРИБ',VW/2-30,24);
  }
  const ix=10,iy=48,iw=60,ih=4;
  ctx.fillStyle='rgba(6,8,18,0.65)';ctx.fillRect(ix-1,iy-1,iw+2,ih+2);
  ctx.fillStyle='#1b2138';ctx.fillRect(ix,iy,iw,ih);
  ctx.fillStyle='#8f7bff';ctx.fillRect(ix,iy,Math.round(iw*SFX.getIntensity()),ih);
  // Дневники счётчик
  ctx.fillStyle='#c9a05a';
  ctx.fillText(`📖 ${diariesFound.length}/8`,VW-56,32);
  // Голос посоха 9.1
  if(whisperMsg){
    ctx.globalAlpha=0.8+0.2*Math.sin(time*2);
    ctx.fillStyle='#9aa8d8';ctx.font='italic 9px "Courier New",monospace';
    ctx.textAlign='center';ctx.fillText(whisperMsg,VW/2,VH-40);
    ctx.textAlign='left';ctx.globalAlpha=1;
  }
  // Достижение 8.5
  if(achievementPopup){
    ctx.globalAlpha=Math.min(1,achievementPopup.t);
    ctx.fillStyle='rgba(6,8,18,0.9)';ctx.fillRect(VW-160,50,152,32);
    ctx.strokeStyle='#ffd76a';ctx.lineWidth=1;ctx.strokeRect(VW-160,50,152,32);
    ctx.fillStyle='#ffd76a';ctx.font='bold 9px "Courier New",monospace';
    ctx.fillText('★ '+achievementPopup.name,VW-153,64);
    ctx.fillStyle='#8fa0c8';ctx.font='8px "Courier New",monospace';
    ctx.fillText(achievementPopup.desc,VW-153,76);
    ctx.globalAlpha=1;
  }
  if(mutedBannerT>0){
    ctx.globalAlpha=Math.min(1,mutedBannerT*2);
    ctx.fillStyle='rgba(6,8,18,0.85)';
    ctx.fillRect(VW/2-42,VH-30,84,16);
    ctx.fillStyle='#ffd76a';ctx.textAlign='center';
    ctx.font='9px "Courier New",monospace';
    ctx.fillText(SFX.isMuted()?'ЗВУК ВЫКЛ':'ЗВУК ВКЛ',VW/2,VH-19);
    ctx.textAlign='left';ctx.globalAlpha=1;
  }
  // Виньетка при низком свете 6.4
  if(player.light<25&&state==='play'){
    const a=1-player.light/25;
    const g=ctx.createRadialGradient(VW/2,VH/2,VH*0.2,VW/2,VH/2,VH*0.8);
    g.addColorStop(0,'rgba(0,0,0,0)');
    g.addColorStop(1,`rgba(60,0,20,${a*0.7})`);
    ctx.fillStyle=g;ctx.fillRect(0,0,VW,VH);
  }
  // Slow-motion tint 6.6
  if(state==='dead'&&time%1<0.3){
    ctx.fillStyle='rgba(80,10,30,0.3)';ctx.fillRect(0,0,VW,VH);
  }
}
function drawPauseOverlay(){
  ctx.fillStyle='rgba(2,4,12,0.72)';ctx.fillRect(0,0,VW,VH);
  ctx.fillStyle='rgba(255,215,106,0.15)';
  ctx.fillRect(0,VH/2-46,VW,1);ctx.fillRect(0,VH/2+46,VW,1);
  ctx.textAlign='center';
  const pulse=0.75+0.25*Math.sin(time*3);
  ctx.fillStyle=`rgba(255,215,106,${pulse})`;
  ctx.font='bold 26px "Courier New",monospace';
  ctx.fillText('ПАУЗА',VW/2,VH/2-20);
  // Задачи 8.4
  ctx.fillStyle='#8fa0c8';ctx.font='9px "Courier New",monospace';
  let task='';
  if(level===1)task='Задача: Найди Алтарь Древних';
  else if(level==='1.5')task='Задача: Пройди Лес Шепотов';
  else if(level===2)task='Задача: Убей Хранителя Пепла';
  ctx.fillText(task,VW/2,VH/2+2);
  ctx.fillStyle='#7f8bb0';ctx.font='8px "Courier New",monospace';
  ctx.fillText(`Дневники: ${diariesFound.length}/8  ·  Смертей: ${deaths}`,VW/2,VH/2+18);
  const a=0.5+0.5*Math.sin(time*3);
  ctx.fillStyle=`rgba(200,215,255,${a})`;
  ctx.font='bold 9px "Courier New",monospace';
  ctx.fillText('[ ПРОДОЛЖИТЬ: ⏸ ]',VW/2,VH/2+38);
  ctx.textAlign='left';
}
function drawStoryArt(art){
  if(art==='stars'||art==='sun'){
    const count=art==='sun'?5:40;
    for(let i=0;i<count;i++){
      const sx=(i*137)%VW,sy=(i*71)%120;
      const b=0.3+0.7*Math.abs(Math.sin(time*0.5+i));
      ctx.fillStyle=`rgba(200,220,255,${b*0.7})`;
      ctx.fillRect(sx,sy,1,1);
    }
  }
  if(art==='sun'){
    const cx=VW/2,cy=90,pulse=0.7+0.3*Math.sin(time*1.5);
    const g=ctx.createRadialGradient(cx,cy,0,cx,cy,70);
    g.addColorStop(0,`rgba(255,240,180,${0.9*pulse})`);
    g.addColorStop(0.3,`rgba(255,200,100,${0.6*pulse})`);
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
    const pulse=0.7+0.3*Math.sin(time*3);
    const g=ctx.createRadialGradient(ax+20,ay+22,0,ax+20,ay+22,50);
    g.addColorStop(0,`rgba(255,230,150,${0.5*pulse})`);
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
  const textTop=VH-100;
  ctx.fillStyle='rgba(4,6,16,0.85)';ctx.fillRect(0,textTop-6,VW,106);
  ctx.strokeStyle='rgba(120,140,200,0.4)';ctx.lineWidth=1;
  ctx.beginPath();
  ctx.moveTo(40,textTop-6);ctx.lineTo(VW-40,textTop-6);
  ctx.moveTo(40,textTop+94);ctx.lineTo(VW-40,textTop+94);ctx.stroke();
  const revealed=Math.floor(storyTimer/0.035);let idx=0;
  ctx.font='11px "Courier New",monospace';ctx.fillStyle='#c9d4e8';ctx.textAlign='center';
  for(let li=0;li<cur.lines.length;li++){
    const line=cur.lines[li];let shown='';
    for(let ci=0;ci<line.length;ci++){if(idx<revealed)shown+=line[ci];idx++;}
    ctx.fillText(shown,VW/2,textTop+22+li*20);
  }
  ctx.textAlign='left';
  if(storyDone){
    const a=0.4+0.6*Math.abs(Math.sin(time*3));
    ctx.fillStyle=`rgba(255,215,106,${a})`;
    ctx.font='bold 9px "Courier New",monospace';ctx.textAlign='center';
    ctx.fillText('[ ПРОБЕЛ / ⤒ ]',VW/2,VH-12);ctx.textAlign='left';
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
  ctx.fillStyle='#ffd76a';ctx.font='bold 20px "Courier New",monospace';
  ctx.fillText('ХРОНИКИ',VW/2,60);
  ctx.fillText('ЗАБЫТОГО СВЕТА',VW/2,86);
  ctx.fillStyle='#5a6a94';ctx.font='9px "Courier New",monospace';
  ctx.fillText('Солнце погасло. Ты — последняя искра.',VW/2,110);
  ctx.fillStyle='#8fa0c8';
  ctx.fillText('A/D · ПРОБЕЛ · SHIFT · P',VW/2,140);
  ctx.fillText('R — заново     M — звук',VW/2,154);
  if(achievements.length>0){
    ctx.fillStyle='#7f8bb0';ctx.fillText(`★ Достижений: ${achievements.length}/${ACHS.length}`,VW/2,172);
  }
  if(diariesFound.length>0){
    ctx.fillStyle='#c9a05a';ctx.fillText(`📖 Дневники: ${diariesFound.length}/8`,VW/2,186);
  }
  if(deaths>0){
    ctx.fillStyle='#8fa0c8';ctx.fillText(`Смертей всего: ${deaths}`,VW/2,200);
  }
  const a=0.5+0.5*Math.sin(time*3);
  ctx.globalAlpha=a;
  ctx.fillStyle='#ffd76a';ctx.font='bold 11px "Courier New",monospace';
  ctx.fillText('НАЖМИ ПРОБЕЛ',VW/2,240);
  ctx.globalAlpha=1;ctx.textAlign='left';
}
function drawDead(){
  ctx.fillStyle='rgba(20,0,10,0.72)';ctx.fillRect(0,0,VW,VH);
  ctx.textAlign='center';
  ctx.fillStyle='#ff6b8a';ctx.font='bold 18px "Courier New",monospace';
  ctx.fillText('СВЕТ УГАС',VW/2,118);
  ctx.fillStyle='#8fa0c8';ctx.font='9px "Courier New",monospace';
  ctx.fillText('Тьма поглотила странника...',VW/2,142);
  ctx.fillStyle='#7f8bb0';
  const mm=Math.floor(runTime/60),ss=Math.floor(runTime%60);
  ctx.fillText(`Время: ${mm}:${ss<10?'0':''}${ss}  ·  Смертей в забеге: ${deaths}`,VW/2,160);
  const a=0.5+0.5*Math.sin(time*4);
  ctx.globalAlpha=a;
  ctx.fillStyle='#ffd76a';ctx.font='bold 10px "Courier New",monospace';
  ctx.fillText('R / ⤒ — ПОПРОБОВАТЬ СНОВА',VW/2,190);
  ctx.globalAlpha=1;ctx.textAlign='left';
}
function drawWinScreen(){
  // 1.4: Экран победы с очками
  ctx.fillStyle='rgba(6,10,20,0.92)';ctx.fillRect(0,0,VW,VH);
  ctx.textAlign='center';
  ctx.fillStyle='#ffe9a3';ctx.font='bold 20px "Courier New",monospace';
  ctx.fillText('★ ИСКРА ВОЗВРАЩЕНА ★',VW/2,50);

  // Рейтинг
  let rank='C';
  if(deaths===0&&runTime<300)rank='S';
  else if(deaths<=2&&runTime<420)rank='A';
  else if(deaths<=6)rank='B';
  const rankCol=rank==='S'?'#ffd76a':rank==='A'?'#8fe09a':rank==='B'?'#8fa0c8':'#a08080';
  ctx.fillStyle=rankCol;ctx.font='bold 40px "Courier New",monospace';
  ctx.fillText(rank,VW/2,105);

  const mm=Math.floor(runTime/60),ss=Math.floor(runTime%60);
  ctx.fillStyle='#c9d4e8';ctx.font='11px "Courier New",monospace';
  ctx.fillText(`Время: ${mm}:${ss<10?'0':''}${ss}`,VW/2,140);
  ctx.fillText(`Смертей: ${deaths}`,VW/2,158);
  ctx.fillText(`Дневников: ${diariesFound.length}/8`,VW/2,176);
  ctx.fillText(`Достижений: ${achievements.length}/${ACHS.length}`,VW/2,194);

  // Открытые скины
  if(skins.length>1){
    ctx.fillStyle='#ffd76a';ctx.font='bold 9px "Courier New",monospace';
    ctx.fillText(`Открыты скины: ${skins.join(', ')}`,VW/2,216);
  }

  const a=0.5+0.5*Math.sin(time*3);
  ctx.globalAlpha=a;
  ctx.fillStyle='#8fa0c8';ctx.font='bold 10px "Courier New",monospace';
  ctx.fillText('[ ПРОБЕЛ ] — на титул',VW/2,244);
  ctx.globalAlpha=1;ctx.textAlign='left';
}
function drawTrialHUD(){
  if(!trialActive)return;
  ctx.fillStyle='rgba(120,10,30,0.75)';ctx.fillRect(VW/2-80,40,160,32);
  ctx.strokeStyle='#ffd76a';ctx.lineWidth=1;ctx.strokeRect(VW/2-80,40,160,32);
  ctx.fillStyle='#ffd76a';ctx.font='bold 10px "Courier New",monospace';
  ctx.textAlign='center';
  ctx.fillText('⚔ ИСПЫТАНИЕ ⚔',VW/2,54);
  ctx.fillStyle='#ff8b9c';ctx.font='9px "Courier New",monospace';
  ctx.fillText(`Выжить: ${Math.ceil(trialTimer)}с`,VW/2,68);
  ctx.textAlign='left';
}

function render(){
  if(state==='story'){drawStory();return;}
  if(state==='win'){drawWinScreen();return;}

  ctx.save();
  if(shake>0.2){
    ctx.translate(Math.round(rand(-shake,shake)),Math.round(rand(-shake,shake)));
  }
  drawBackground();drawPlatforms();drawShrooms();drawShards();
  drawLanterns();drawDiaries();drawAltar();drawExit();
  drawEnemies();drawMotes();drawSilhouette();
  drawBoss();drawPlayer();drawProjectiles();drawParticles();
  drawFlashWave();drawDarkness();drawHUD();drawTrialHUD();
  if(boss&&!boss.dead&&state==='play'&&!paused)drawBossHP();
  if(bossIntroT>0&&!paused){
    ctx.globalAlpha=Math.min(1,bossIntroT);
    ctx.fillStyle='rgba(40,0,0,0.55)';
    ctx.fillRect(0,VH/2-24,VW,48);
    ctx.fillStyle='#ff5c2a';ctx.font='bold 16px "Courier New",monospace';
    ctx.textAlign='center';
    ctx.fillText('ХРАНИТЕЛЬ ПЕПЛА ПРОБУДИЛСЯ',VW/2,VH/2+6);
    ctx.textAlign='left';ctx.globalAlpha=1;
  }
  ctx.restore();

  // Chromatic aberration при уроне 6.5
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

/* ====================== ЦИКЛ ====================== */
let last=performance.now(),acc=0;
const STEP=1/60;
function loop(now){
  requestAnimationFrame(loop);
  let dt=(now-last)/1000;last=now;
  if(dt>0.1)dt=0.1;
  acc+=dt;
  let guard=0;
  while(acc>=STEP&&guard++<5){update(STEP);acc-=STEP;}
  render();
}

loadLevel(1);
requestAnimationFrame(loop);

})();
</script>
</body>
</html>
