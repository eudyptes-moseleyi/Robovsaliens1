<!DOCTYPE html>
<html lang="pt">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>Destroy Aliens.exe</title>
<style>
*{box-sizing:border-box}
html,body{margin:0;width:100%;height:100%;overflow:hidden;background:#090b10;font-family:Arial,sans-serif}
body{display:flex;align-items:center;justify-content:center}
#wrap{position:relative;width:min(100vw,177.78vh);height:min(100vh,56.25vw);max-width:1200px;max-height:675px}
canvas{display:block;width:100%;height:100%;background:#12141c;image-rendering:auto;touch-action:none}
#touch{position:absolute;inset:0;pointer-events:none}
.tbtn{position:absolute;pointer-events:auto;width:64px;height:64px;border-radius:16px;border:2px solid #ffffff55;background:#111b;color:white;font-size:26px;font-weight:bold;user-select:none;-webkit-user-select:none;touch-action:none}
#left{left:16px;bottom:20px} #right{left:90px;bottom:20px}
#jump{right:96px;bottom:20px} #interact{right:20px;bottom:88px}
#shoot{right:20px;bottom:20px;background:#5a0f0fbb;border-color:#ff6b6b88}
#pauseTouch{right:16px;top:16px;width:48px;height:48px;font-size:18px}
@media(min-width:900px){#touch{display:none}}
</style>
</head>
<body>
<div id="wrap">
<canvas id="game" width="900" height="500"></canvas>
<div id="touch">
<button class="tbtn" id="left">◀</button>
<button class="tbtn" id="right">▶</button>
<button class="tbtn" id="jump">▲</button>
<button class="tbtn" id="interact">E</button>
<button class="tbtn" id="shoot">●</button>
<button class="tbtn" id="pauseTouch">Ⅱ</button>
</div>
</div>

<script>
"use strict";

const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");
const W=900,H=500;
const WORLD_GAME=2000;
const WORLD_S2=1000;

const keys={left:false,right:false,jump:false,interact:false,shoot:false};
let state="menu";
let camera=0;
let cutFrame=0;
let cutTimer=0;
let last=performance.now();
let score=0;
let lives=3;
let shootCooldown=0;
let doorMsg=0;
let hitFlash=0;
let pausedFrom="game";

const images={};
const imageNames=[
 "logo.png","robo_p.png","robo_p2.png","robo_p3.png","robo_p4.png",
 "robo_run1.png","robo_run2.png","robo_run3.png","robo_run4.png","robo_run5.png",
 "fundo_tutorial.webp","fundo2_tutorial.webp","porta2.png","caixa.png",
 "cutscene1.png","cutscene2.png","cutscene3.png","cutscene4.png","cutscene5.png",
 "cutscene6.png","cutscene7.png","cutscene8.png","cutscene9.png","cutscene10.png",
 "cutscene11.png","cutscene12.png","cutscene13.png","cutscene14.png","cutscene15.png",
 "cutscene16.png","cutscene17.png","cutscene18.png",
 "seta_para_direita.png","seta_para_esquerda.png",
 "passadeira1.png","passadeira2.png","garra_fechada.png","garra_aberta.png",
 "robo_construçao.png","cabeca.png","cabeca_feieta.png"
];

for(const n of imageNames){
  const im=new Image();
  im.src=n;
  images[n]=im;
}

function imgReady(n){return images[n] && images[n].complete && images[n].naturalWidth>0}
function drawImg(n,x,y,w,h,flip=false){
  if(!imgReady(n)) return false;
  ctx.save();
  if(flip){ctx.translate(x+w,y);ctx.scale(-1,1);ctx.drawImage(images[n],0,0,w,h)}
  else ctx.drawImage(images[n],x,y,w,h);
  ctx.restore();
  return true;
}
function rect(x,y,w,h,c){ctx.fillStyle=c;ctx.fillRect(x,y,w,h)}
function text(t,x,y,size=20,c="#fff",align="left"){
  ctx.fillStyle=c;ctx.font=`${size}px Arial`;ctx.textAlign=align;ctx.fillText(t,x,y);
}
function button(x,y,w,h,label,c="#009b60"){
  ctx.fillStyle=c;ctx.fillRect(x,y,w,h);
  ctx.strokeStyle="#ffffff55";ctx.lineWidth=2;ctx.strokeRect(x,y,w,h);
  text(label,x+w/2,y+h/2+8,20,"#fff","center");
}
function clamp(v,a,b){return Math.max(a,Math.min(b,v))}
function worldWidth(){return state==="scenario2"?WORLD_S2:WORLD_GAME}

/* ---------- ÁUDIO (sintetizado, sem ficheiros externos) ---------- */
let audioCtx=null;
function ensureAudio(){
 if(!audioCtx){
   try{ audioCtx=new (window.AudioContext||window.webkitAudioContext)(); }catch(e){}
 }
 if(audioCtx && audioCtx.state==="suspended") audioCtx.resume();
}
function beep(freq,dur,type="square",vol=.08){
 if(!audioCtx) return;
 const osc=audioCtx.createOscillator();
 const gain=audioCtx.createGain();
 osc.type=type; osc.frequency.value=freq;
 gain.gain.value=vol;
 gain.gain.exponentialRampToValueAtTime(.001,audioCtx.currentTime+dur);
 osc.connect(gain); gain.connect(audioCtx.destination);
 osc.start(); osc.stop(audioCtx.currentTime+dur);
}
const sfx={
 shoot:()=>beep(880,.08,"square",.05),
 hit:()=>beep(140,.25,"sawtooth",.09),
 kill:()=>beep(520,.12,"triangle",.07),
 jump:()=>beep(400,.1,"sine",.04),
 door:()=>beep(200,.15,"sine",.06),
 win:()=>{beep(523,.15);setTimeout(()=>beep(659,.15),120);setTimeout(()=>beep(784,.25),240);}
};
window.addEventListener("pointerdown",ensureAudio,{once:true});
window.addEventListener("keydown",ensureAudio,{once:true});
function intersects(a,b){
 return a.x<b.x+b.w && a.x+a.w>b.x && a.y<b.y+b.h && a.y+a.h>b.y;
}

const player={
 x:100,y:350,w:60,h:70,vy:0,
 gravity:.7,jump:-14,speed:4,onGround:true,facing:1,
 walkFrame:0,walkTimer:0,idleFrame:0,idleTimer:0,
 invincible:0
};

const box={x:1500,y:295,w:100,h:100};
const door={x:1921,y:287,w:100,h:150};
const exitDoor={x:WORLD_S2-90,y:287,w:80,h:150};

let aliens=[];
let bullets=[];

function spawnAliens(){
 aliens=[
  {x:520, y:365,w:44,h:55,minX:460,maxX:700,dir:1,speed:1.4,alive:true},
  {x:900, y:365,w:44,h:55,minX:850,maxX:1050,dir:-1,speed:1.6,alive:true},
  {x:1200,y:365,w:44,h:55,minX:1130,maxX:1400,dir:1,speed:1.2,alive:true},
  {x:1700,y:365,w:44,h:55,minX:1650,maxX:1880,dir:-1,speed:1.8,alive:true}
 ];
}

function aliensRemaining(){return aliens.filter(a=>a.alive).length}

function resetPlayer(){
 player.x=100;player.y=350;player.vy=0;player.onGround=true;player.invincible=0;
 camera=0;
 bullets=[];
}

function startGame(){
 score=0;lives=3;
 resetPlayer();
 state="cutscene";
 cutFrame=0;cutTimer=0;
}

function startWorld(){
 resetPlayer();
 spawnAliens();
 state="game";
}

function updateCamera(){
 const left=260,right=640;
 let target=camera;
 if(player.x-camera<left) target=player.x-left;
 else if(player.x+player.w-camera>right) target=player.x+player.w-right;
 camera+=(target-camera)*.12;
 camera=clamp(camera,0,Math.max(0,worldWidth()-W));
}

function playerRect(){return {x:player.x,y:player.y,w:player.w,h:player.h}}

function fireBullet(){
 if(shootCooldown>0) return;
 shootCooldown=14;
 sfx.shoot();
 bullets.push({
  x: player.facing>0 ? player.x+player.w : player.x-10,
  y: player.y+28,
  w:10,h:6,
  vx: player.facing*9,
  alive:true
 });
}

function updateBullets(){
 for(const b of bullets){
   if(!b.alive) continue;
   b.x+=b.vx;
   if(b.x<-50 || b.x>worldWidth()+50) b.alive=false;
 }
 for(const b of bullets){
   if(!b.alive) continue;
   for(const a of aliens){
     if(!a.alive) continue;
     if(intersects({x:b.x,y:b.y,w:b.w,h:b.h},a)){
       a.alive=false;b.alive=false;score+=100;
       sfx.kill();
       break;
     }
   }
 }
 bullets=bullets.filter(b=>b.alive);
}

function updateAliens(){
 for(const a of aliens){
   if(!a.alive) continue;
   a.x+=a.dir*a.speed;
   if(a.x<a.minX){a.x=a.minX;a.dir=1}
   if(a.x+a.w>a.maxX){a.x=a.maxX-a.w;a.dir=-1}
 }
}

function handleAlienCollisions(){
 const pr=playerRect();
 for(const a of aliens){
   if(!a.alive) continue;
   if(!intersects(pr,a)) continue;
   const stomping = player.vy>0 && (pr.y+pr.h-player.vy) <= a.y+16;
   if(stomping){
     a.alive=false;score+=150;
     player.vy=player.jump*0.55;
     sfx.kill();
   }else if(player.invincible<=0){
     lives--;
     player.invincible=70;
     player.x += (pr.x+pr.w/2 < a.x+a.w/2) ? -30 : 30;
     hitFlash=14;
     sfx.hit();
     if(lives<=0){ state="gameover"; }
   }
 }
}

function updateGame(dt){
 let moving=false;
 if(keys.left){player.x-=player.speed;moving=true;player.facing=-1}
 if(keys.right){player.x+=player.speed;moving=true;player.facing=1}
 player.x=clamp(player.x,0,worldWidth()-player.w);

 if(keys.jump && player.onGround){
   player.vy=player.jump;player.onGround=false;
   keys.jump=false;
   sfx.jump();
 }

 player.vy+=player.gravity;
 player.y+=player.vy;

 const ground=420;
 if(player.y+player.h>=ground){
   player.y=ground-player.h;player.vy=0;player.onGround=true;
 }

 if(state==="game"){
   const pr=playerRect(), br=box;
   if(intersects(pr,br)){
     if(player.vy>=0 && pr.y+pr.h-player.vy<=br.y+20){
       player.y=br.y-player.h;player.vy=0;player.onGround=true;
     }else if(pr.x+pr.w/2<br.x+br.w/2){
       player.x=br.x-player.w;
     }else{
       player.x=br.x+br.w;
     }
   }
 }

 if(moving){
   player.walkTimer++;
   if(player.walkTimer>=5){
     player.walkTimer=0;
     player.walkFrame=(player.walkFrame+1)%5;
   }
   player.idleFrame=0;
 }else{
   player.walkFrame=0;player.walkTimer=0;
   if(player.onGround){
     player.idleTimer++;
     if(player.idleTimer>=12){
       player.idleTimer=0;
       player.idleFrame=(player.idleFrame+1)%8;
     }
   }
 }

 if(player.invincible>0) player.invincible--;
 if(hitFlash>0) hitFlash--;
 if(shootCooldown>0) shootCooldown--;
 if(doorMsg>0) doorMsg--;

 if(state==="game"){
   updateAliens();
   updateBullets();
   handleAlienCollisions();

   if(keys.interact && intersects(playerRect(),door)){
     keys.interact=false;
     if(aliensRemaining()===0){
       sfx.door();
       state="scenario2";
       resetPlayer();
     }else{
       doorMsg=90;
     }
   }
 }else if(state==="scenario2"){
   if(keys.interact && intersects(playerRect(),exitDoor)){
     keys.interact=false;
     sfx.win();
     state="win";
   }
 }

 updateCamera();
}

function drawBackground(){
 rect(0,0,W,H,"#12141c");

 let used1=drawImg("fundo_tutorial.webp",-camera,0,1000,500);
 let used2=drawImg("fundo2_tutorial.webp",1000-camera,0,1000,500);

 if(!used1 || !used2){
   let grad=ctx.createLinearGradient(0,0,0,H);
   grad.addColorStop(0,"#171b27");grad.addColorStop(1,"#292b35");
   ctx.fillStyle=grad;ctx.fillRect(0,0,W,H);
   for(let x=0;x<worldWidth();x+=100){
     const sx=x-camera;
     if(sx<-100||sx>W+100) continue;
     rect(sx,70,4,350,"#303441");
     rect(sx+20,110,55,8,"#3d414e");
   }
   for(let x=0;x<worldWidth();x+=180){
     const sx=x-camera;
     if(sx>-200&&sx<W+200){
       rect(sx,180,120,90,"#20242e");
       rect(sx+15,195,90,60,"#151820");
     }
   }
 }
 rect(-camera,420,worldWidth(),80,"#373943");
 for(let x=0;x<worldWidth();x+=40){
   const sx=x-camera;
   ctx.strokeStyle="#555866";ctx.lineWidth=2;
   ctx.beginPath();ctx.moveTo(sx,420);ctx.lineTo(sx,500);ctx.stroke();
 }
}

function drawPlayer(){
 let moving=keys.left||keys.right;
 let name;
 if(moving) name=`robo_run${player.walkFrame+1}.png`;
 else name=["robo_p.png","robo_p2.png","robo_p3.png","robo_p2.png","robo_p.png","robo_p2.png","robo_p4.png","robo_p2.png"][player.idleFrame];

 const x=player.x-camera,y=player.y;
 const blink = player.invincible>0 && Math.floor(player.invincible/6)%2===0;
 if(blink) ctx.globalAlpha=0.35;
 if(!drawImg(name,x,y,60,70,player.facing<0)){
   rect(x+8,y+10,44,55,"#26d34f");
   rect(x+16,y+18,28,18,"#b8e7ff");
   rect(x+23,y+23,5,5,"#111");
   rect(x+34,y+23,5,5,"#111");
 }
 ctx.globalAlpha=1;
}

function drawAliens(){
 for(const a of aliens){
   if(!a.alive) continue;
   const x=a.x-camera,y=a.y;
   if(x<-60||x>W+60) continue;
   rect(x,y+10,a.w,a.h-10,"#7a2fd6");
   rect(x+8,y,a.w-16,18,"#a05aff");
   rect(x+10,y+6,6,6,"#0ff");
   rect(x+a.w-16,y+6,6,6,"#0ff");
   rect(x-4,y+18,6,20,"#7a2fd6");
   rect(x+a.w-2,y+18,6,20,"#7a2fd6");
 }
}

function drawBullets(){
 for(const b of bullets){
   rect(b.x-camera,b.y,b.w,b.h,"#00ffc8");
 }
}

function drawWorld(){
 drawBackground();

 if(!drawImg("caixa.png",box.x-camera,box.y,100,100)){
   rect(box.x-camera,box.y,100,100,"#9b6635");
   rect(box.x-camera+12,box.y+12,76,76,"#70451f");
 }

 const doorLocked = aliensRemaining()>0;
 if(!drawImg("porta2.png",door.x-camera,door.y,100,150)){
   rect(door.x-camera,door.y,100,150, doorLocked?"#5a2222":"#8a4f18");
   rect(door.x-camera+15,door.y+15,70,120,"#171a20");
   text("E",door.x-camera+50,door.y+78,28,"#fff","center");
 }

 drawAliens();
 drawBullets();
 drawPlayer();

 if(hitFlash>0){ rect(0,0,W,H,`rgba(255,0,0,${hitFlash/40})`); }

 text(`Pontos: ${score}`,20,32,20,"#00ffc8");
 text(`Vidas: ${"♥".repeat(Math.max(0,lives))}`,20,58,20,"#ff6b6b");
 text(`Aliens: ${aliensRemaining()}/${aliens.length}`,W-20,32,18,"#a05aff","right");
 text("A/D mover • ESPAÇO saltar • F atira • E entra",W/2,32,16,"#bbb","center");

 if(doorMsg>0){
   ctx.fillStyle="rgba(0,0,0,.7)";
   ctx.fillRect(W/2-190,H/2-24,380,48);
   text("Destrói todos os aliens primeiro!",W/2,H/2+7,18,"#ff6b6b","center");
 }
}

function drawScenario2(){
 rect(0,0,W,H,"#141824");
 drawImg("fundo2_tutorial.webp",-camera,0,1000,500);
 rect(0,420,WORLD_S2,80,"#373943");

 const ex=exitDoor.x-camera;
 if(!drawImg("porta2.png",ex,exitDoor.y,exitDoor.w,exitDoor.h)){
   rect(ex,exitDoor.y,exitDoor.w,exitDoor.h,"#1f8a4f");
   rect(ex+10,exitDoor.y+12,exitDoor.w-20,exitDoor.h-24,"#0d3b21");
   text("SAÍDA",ex+exitDoor.w/2,exitDoor.y-10,15,"#1f8a4f","center");
 }

 drawPlayer();
 text("CENÁRIO 2",20,32,22,"#00ffc8");
 text(`Pontos: ${score}`,W-20,32,18,"#00ffc8","right");
 text("A/D mover • E na saída • ESC pausa",W/2,32,15,"#bbb","center");
}

function drawMenu(){
 rect(0,0,W,H,"#0f1119");
 if(imgReady("fundo_tutorial.webp")) drawImg("fundo_tutorial.webp",0,0,W,H);
 rect(0,0,W,H,"#00000077");
 text("DESTROY ALIENS",W/2,150,48,"#00ffc8","center");
 text("Uma aventura no laboratório",W/2,185,20,"#fff","center");
 button(350,225,200,60,"JOGAR","#009b60");
 text("PC: teclado (A/D, ESPAÇO, F, E)  |  Telemóvel: botões no ecrã",W/2,340,15,"#bbb","center");
 if(score>0 || lives<3){
   text(`Última partida: ${score} pontos`,W/2,375,15,"#888","center");
 }
}

function drawPause(){
 if(pausedFrom==="scenario2") drawScenario2(); else drawWorld();
 rect(0,0,W,H,"#000000aa");
 text("JOGO PAUSADO",W/2,120,40,"#fff","center");
 button(330,180,240,50,"CONTINUAR","#009b60");
 button(330,250,240,50,"VOLTAR AO MENU","#8b6400");
 button(330,320,240,50,"SAIR","#9b2020");
}

function drawGameOver(){
 rect(0,0,W,H,"#1a0808");
 text("GAME OVER",W/2,180,50,"#ff4444","center");
 text(`Pontuação final: ${score}`,W/2,230,24,"#fff","center");
 button(330,280,240,55,"TENTAR NOVAMENTE","#009b60");
}

function drawWin(){
 rect(0,0,W,H,"#08150e");
 text("MISSÃO CUMPRIDA!",W/2,180,44,"#00ffc8","center");
 text(`Pontuação final: ${score}`,W/2,230,24,"#fff","center");
 text(`Vidas restantes: ${lives}`,W/2,262,18,"#ff6b6b","center");
 button(330,300,240,55,"JOGAR NOVAMENTE","#009b60");
}

function drawCutscene(){
 ctx.fillStyle="rgba(0,0,0,.65)";
 ctx.fillRect(650,425,230,55);
 ctx.fillStyle="#fff";
 ctx.font="bold 22px Arial";
 ctx.textAlign="center";
 ctx.fillText(cutFrame>=17?"CONTINUAR JOGO":"CONTINUAR",765,460);
 ctx.textAlign="left";
 rect(0,0,W,H,"#11131b");
 const n=`cutscene${cutFrame+1}.png`;
 if(!drawImg(n,0,0,W,H)){
   text("CUTSCENE",W/2,180,42,"#00ffc8","center");
   text(`Cena ${cutFrame+1}/18`,W/2,225,22,"#fff","center");
   text("← / → para avançar",W/2,270,20,"#bbb","center");
 }
 text("→",850,455,42,"#fff","center");
 if(cutFrame>0) text("←",50,455,42,"#fff","center");
}

function updateCutscene(){
 cutTimer++;
}

function drawConstruction(){
 rect(0,0,W,H,"#12121a");
 text("A preparar o laboratório...",W/2,70,30,"#00ffc8","center");
 let p=Math.min(1,cutTimer/240);
 rect(150,220,600,24,"#333");
 rect(150,220,600*p,24,"#00ffc8");
 text(Math.floor(p*100)+"%",W/2,280,22,"#fff","center");
}

function updateConstruction(){
 cutTimer++;
 if(cutTimer>250){startWorld()}
}

function handlePointer(x,y){
 if(state==="menu"){
   if(x>=350&&x<=550&&y>=225&&y<=285) startGame();
 }else if(state==="pause"){
   if(x>=330&&x<=570&&y>=180&&y<=230) state=pausedFrom;
   else if(x>=330&&x<=570&&y>=250&&y<=300) {state="menu";resetPlayer()}
   else if(x>=330&&x<=570&&y>=320&&y<=370) state="menu";
 }else if(state==="gameover"){
   if(x>=330&&x<=570&&y>=280&&y<=335) state="menu";
 }else if(state==="win"){
   if(x>=330&&x<=570&&y>=300&&y<=355) state="menu";
 }
}

canvas.addEventListener("pointerdown",e=>{
 const r=canvas.getBoundingClientRect();
 const x=(e.clientX-r.left)*W/r.width;
 const y=(e.clientY-r.top)*H/r.height;
 if(state==="cutscene"){
   advanceCutscene();
   return;
 }
 handlePointer(x,y);
});

function keyDown(k){
 if(state==="cutscene") return advanceCutscene();
 if(k==="a"||k==="arrowleft")keys.left=true;
 if(k==="d"||k==="arrowright")keys.right=true;
 if(k===" "||k==="w"||k==="arrowup")keys.jump=true;
 if(k==="e")keys.interact=true;
 if(k==="f" && state==="game") fireBullet();
 if(k==="escape"){
   if(state==="game"||state==="scenario2"){ pausedFrom=state; state="pause"; }
   else if(state==="pause") state=pausedFrom;
 }
}
function advanceCutscene(){
 cutFrame++;
 if(cutFrame>=18){state="construction";cutTimer=0;cutFrame=0;}
}

function keyUp(k){
 if(k==="a"||k==="arrowleft")keys.left=false;
 if(k==="d"||k==="arrowright")keys.right=false;
 if(k==="e")keys.interact=false;
}
window.addEventListener("keydown",e=>{
 if([" ","ArrowUp","ArrowDown","ArrowLeft","ArrowRight"].includes(e.key))e.preventDefault();
 keyDown(e.key.toLowerCase());
});
window.addEventListener("keyup",e=>keyUp(e.key.toLowerCase()));

function bindButton(id,key){
 const b=document.getElementById(id);
 const down=e=>{e.preventDefault();keys[key]=true};
 const up=e=>{e.preventDefault();keys[key]=false};
 b.addEventListener("pointerdown",down);
 b.addEventListener("pointerup",up);
 b.addEventListener("pointercancel",up);
 b.addEventListener("pointerleave",up);
}
bindButton("left","left");
bindButton("right","right");
bindButton("jump","jump");
bindButton("interact","interact");
document.getElementById("shoot").addEventListener("pointerdown",e=>{
 e.preventDefault();
 if(state==="game") fireBullet();
});
document.getElementById("pauseTouch").addEventListener("pointerdown",e=>{
 e.preventDefault();
 if(state==="game"||state==="scenario2"){ pausedFrom=state; state="pause"; }
 else if(state==="pause") state=pausedFrom;
});

function loop(now){
 const dt=Math.min(.033,(now-last)/1000);last=now;

 if(state==="game"||state==="scenario2")updateGame(dt);
 if(state==="cutscene")updateCutscene();
 if(state==="construction")updateConstruction();

 if(state==="menu")drawMenu();
 else if(state==="game")drawWorld();
 else if(state==="scenario2")drawScenario2();
 else if(state==="pause")drawPause();
 else if(state==="gameover")drawGameOver();
 else if(state==="win")drawWin();
 else if(state==="cutscene")drawCutscene();
 else if(state==="construction")drawConstruction();

 requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
</script>
</body>
</html>
