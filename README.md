
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>THE CSO BOT ZOMBIE</title>

<style>
*{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
html,body{margin:0;width:100%;height:100%;overflow:hidden;background:#080b10;color:white;font-family:Arial,sans-serif;touch-action:none}
canvas{display:block}
#hud{position:absolute;inset:0;pointer-events:none}
#top{display:flex;justify-content:space-around;padding:10px 4px;font-size:12px;font-weight:bold;text-shadow:0 2px 4px black}
.info{background:#0009;padding:8px;border-radius:8px}
#hp{color:#ff5555}#ammo{color:#ffd45b}#kills{color:#67eaff}
#cross{position:absolute;left:50%;top:50%;transform:translate(-50%,-50%);font-size:24px;color:#fff}
#stick{position:absolute;left:20px;bottom:35px;width:140px;height:140px;border:2px solid #ffffff50;border-radius:50%;background:#ffffff10;pointer-events:auto}
#knob{position:absolute;left:40px;top:40px;width:56px;height:56px;border-radius:50%;background:#ffffff50}
.btn{position:absolute;pointer-events:auto;border-radius:50%;background:#15202bcc;border:2px solid #ffffff70;color:white;font-weight:bold;display:flex;align-items:center;justify-content:center}
#fire{right:25px;bottom:65px;width:90px;height:90px;background:#9e2222bb}
#reload{right:125px;bottom:40px;width:60px;height:60px;font-size:12px}
#jump{right:30px;bottom:175px;width:60px;height:60px;font-size:12px}
#message{position:absolute;top:35%;width:100%;text-align:center;font-size:24px;text-shadow:0 3px 6px black}
#start,#over{position:absolute;inset:0;background:#05080cee;display:flex;align-items:center;justify-content:center;flex-direction:column;text-align:center}
#over{display:none}
h1{color:#e22;font-size:38px;letter-spacing:4px}
button{background:#a21d1d;color:white;border:0;padding:16px 40px;border-radius:8px;font-size:20px}
</style>
</head>

<body>
<div id="hud">
 <div id="top">
  <div class="info" id="hp">HP 100</div>
  <div class="info" id="ammo">30/90</div>
  <div class="info" id="kills">KILL 0</div>
  <div class="info" id="round">ROUND 1</div>
 </div>
 <div id="cross">+</div>
 <div id="stick"><div id="knob"></div></div>
 <div class="btn" id="fire">FIRE</div>
 <div class="btn" id="reload">RELOAD</div>
 <div class="btn" id="jump">JUMP</div>
 <div id="message"></div>
</div>

<div id="start">
 <h1>THE CSO</h1>
 <h2>BOTいるゾンビ！</h2>
 <p>BOTと集団で籠り場を守れ！</p>
 <p>左：移動　右：視点　FIRE：射撃</p>
 <button id="startbtn">START</button>
</div>

<div id="over">
 <h1>INFECTED</h1>
 <p id="result"></p>
 <button id="retry">RETRY</button>
</div>

<script type="module">
import * as THREE from
'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';

const $=id=>document.getElementById(id);

let scene,camera,renderer,clock;
let running=false;
let hp=100,ammo=30,reserve=90,kills=0,round=1;
let yaw=0,pitch=0,velocityY=0,grounded=true;
let firing=false,reloading=false,fireTimer=0;
let moveX=0,moveY=0;
let bots=[],zombies=[],campObjects=[];
const ray=new THREE.Raycaster();
const forward=new THREE.Vector3(),right=new THREE.Vector3(),move=new THREE.Vector3();

const CAMP=new THREE.Vector3(0,0,8);

// ---------------- WORLD ----------------
function mat(c,r=.85){
 return new THREE.MeshStandardMaterial({color:c,roughness:r});
}
function cube(x,y,z,sx,sy,sz,m){
 const o=new THREE.Mesh(new THREE.BoxGeometry(sx,sy,sz),m);
 o.position.set(x,y,z);
 o.castShadow=true;o.receiveShadow=true;
 scene.add(o);return o;
}
function world(){
 scene=new THREE.Scene();
 scene.background=new THREE.Color(0x111923);
 scene.fog=new THREE.Fog(0x111923,25,75);

 camera=new THREE.PerspectiveCamera(75,innerWidth/innerHeight,.1,100);
 camera.position.set(0,1.7,12);
 camera.rotation.order="YXZ";
 scene.add(camera);

 scene.add(new THREE.HemisphereLight(0xb0c5df,0x252525,1.5));
 const sun=new THREE.DirectionalLight(0xffffff,2);
 sun.position.set(10,20,5);sun.castShadow=true;
 sun.shadow.mapSize.set(1024,1024);
 scene.add(sun);

 cube(0,-.25,0,70,.5,70,mat(0x383d42));

 const wall=mat(0x414951);
 cube(0,3,-30,60,6,1,wall);
 cube(0,3,30,60,6,1,wall);
 cube(-30,3,0,1,6,60,wall);
 cube(30,3,0,1,6,60,wall);

 for(let x=-25;x<=25;x+=10)
  for(let z=-25;z<=25;z+=10)
   cube(x,3,z,1,6,1,mat(0x59616a));

 // コンテナ
 for(let i=0;i<20;i++){
  let x=(Math.random()-.5)*45;
  let z=(Math.random()-.5)*45;
  if(Math.abs(x)<8&&z>0&&z<18)continue;
  cube(x,1,z,3,2,3,mat(0x45515b));
 }

 // 籠り場の目印
 cube(0,.05,8,7,.1,7,mat(0x283d48));
 for(let i=0;i<4;i++){
  const l=new THREE.PointLight(0x80caff,8,10);
  l.position.set((i%2?1:-1)*3,3,8+(i<2?-3:3));
  scene.add(l);
 }
}

// ---------------- CHARACTER ----------------
function makeCharacter(x,z,type){
 const g=new THREE.Group();
 g.position.set(x,0,z);
 scene.add(g);

 const skin=mat(type==="zombie"?0x83a77c:0xc69a78);
 const shirt=mat(type==="bot"?0x293e78:
                 type==="zombie"?0x526b48:0x343a40);
 const pants=mat(type==="zombie"?0x3c493b:0x24272c);

 const head=new THREE.Mesh(new THREE.BoxGeometry(.45,.45,.45),skin);
 head.position.y=1.8;g.add(head);

 const body=new THREE.Mesh(new THREE.BoxGeometry(.65,.8,.35),shirt);
 body.position.y=1.15;g.add(body);

 const legs=[],arms=[];
 for(let i=0;i<2;i++){
  const leg=new THREE.Mesh(new THREE.BoxGeometry(.22,.65,.25),pants);
  leg.position.set(i?.17:-.17,.4,0);
  g.add(leg);legs.push(leg);

  const arm=new THREE.Mesh(new THREE.BoxGeometry(.2,.65,.2),skin);
  arm.position.set(i?.43:-.43,1.2,0);
  g.add(arm);arms.push(arm);
 }

 // BOT用の簡易銃
 if(type==="bot"){
  const gun=new THREE.Mesh(new THREE.BoxGeometry(.12,.12,.65),mat(0x171717));
  gun.position.set(.35,1.25,-.35);g.add(gun);
 }

 // ゾンビのかわいい目
 if(type==="zombie"){
  const eyeMat=new THREE.MeshBasicMaterial({color:0xff4545});
  for(let i=0;i<2;i++){
   const eye=new THREE.Mesh(new THREE.SphereGeometry(.055,8,8),eyeMat);
   eye.position.set(i?.11:-.11,1.85,-.23);
   g.add(eye);
  }
  // 小さな口
  const mouth=new THREE.Mesh(new THREE.BoxGeometry(.15,.035,.03),mat(0x263126));
  mouth.position.set(0,1.68,-.23);g.add(mouth);
 }

 g.userData={
  type,hp:100,alive:true,legs,arms,head,body,
  phase:Math.random()*10,attack:0,
  speed:type==="zombie"?1.35:2.5,
  shoot:Math.random(),
  slot:new THREE.Vector3()
 };
 return g;
}

// ---------------- BOT GROUP ----------------
function spawnBots(){
 bots=[];
 const positions=[
  [-3,6],[-1,5],[1,5],[3,6],
  [-2,10],[2,10]
 ];
 positions.forEach(p=>{
  const b=makeCharacter(p[0],p[1],"bot");
  b.userData.slot.set(p[0],0,p[1]);
  bots.push(b);
 });
}

function spawnZombie(){
 const a=Math.random()*Math.PI*2;
 const r=20+Math.random()*8;
 zombies.push(makeCharacter(Math.cos(a)*r,Math.sin(a)*r,"zombie"));
}

// ---------------- RESET ----------------
function reset(){
 zombies.forEach(z=>scene.remove(z));
 bots.forEach(b=>scene.remove(b));
 zombies=[];bots=[];
 hp=100;ammo=30;reserve=90;kills=0;round=1;
 yaw=0;pitch=0;velocityY=0;
 camera.position.set(0,1.7,12);
 spawnBots();
 for(let i=0;i<8;i++)spawnZombie();
 updateHUD();
}

function updateHUD(){
 $("hp").textContent="HP "+Math.max(0,Math.floor(hp));
 $("ammo").textContent=ammo+"/"+reserve;
 $("kills").textContent="KILL "+kills;
 $("round").textContent="ROUND "+round;
}

// ---------------- SHOOT ----------------
function shoot(){
 if(!running||reloading||ammo<=0)return;
 ammo--;
 ray.setFromCamera(new THREE.Vector2(0,0),camera);
 const targets=[];
 zombies.forEach(z=>{
  if(z.userData.alive)z.traverse(o=>{if(o.isMesh)targets.push(o)});
 });
 const hits=ray.intersectObjects(targets);
 if(hits.length){
  let z=hits[0].object;
  while(z&&!z.userData.type)z=z.parent;
  if(z)damageZombie(z,40);
 }
 updateHUD();
}
function damageZombie(z,n){
 if(!z.userData.alive)return;
 z.userData.hp-=n;
 if(z.userData.hp<=0){
  z.userData.alive=false;
  scene.remove(z);kills++;updateHUD();
 }
}
function reload(){
 if(reloading||ammo===30||reserve<=0||!running)return;
 reloading=true;$("message").textContent="RELOADING";
 setTimeout(()=>{
  if(!running)return;
  const n=Math.min(30-ammo,reserve);
  ammo+=n;reserve-=n;reloading=false;
  $("message").textContent="";
  updateHUD();
 },1500);
}

// ---------------- AI ----------------
function moveToward(g,target,dt,speed){
 const dx=target.x-g.position.x,dz=target.z-g.position.z;
 const d=Math.hypot(dx,dz);
 if(d>0.1){
  g.position.x+=dx/d*speed*dt;
  g.position.z+=dz/d*speed*dt;
  g.rotation.y=Math.atan2(dx,dz);
 }
 return d;
}

function botAI(dt){
 bots.forEach(b=>{
  if(!b.userData.alive)return;
  const d=b.userData;

  // BOTは籠り場の自分の位置に集合
  const dist=moveToward(b,d.slot,dt,3.5);

  // 配置についたら周囲を警戒
  if(dist<.3){
   b.position.x=d.slot.x+Math.sin(clock.elapsedTime*.25+d.phase)*.08;
   b.position.z=d.slot.z+Math.cos(clock.elapsedTime*.25+d.phase)*.08;
  }

  // 籠り場から近いゾンビを優先射撃
  let target=null,best=18;
  zombies.forEach(z=>{
   if(!z.userData.alive)return;
   const zd=b.position.distanceTo(z.position);
   if(zd<best){best=zd;target=z}
  });

  if(target){
   b.rotation.y=Math.atan2(target.position.x-b.position.x,
                           target.position.z-b.position.z);
   d.shoot-=dt;
   if(d.shoot<=0){
    damageZombie(target,10);
    d.shoot=.5+Math.random()*.5;
   }
  }
 });
}

function zombieAI(dt){
 zombies.forEach(z=>{
  if(!z.userData.alive)return;
  const d=z.userData;

  // プレイヤーとBOTの中から近い相手を探す
  let target=camera;
  let best=z.position.distanceTo(camera.position);
  bots.forEach(b=>{
   if(!b.userData.alive)return;
   const dist=z.position.distanceTo(b.position);
   if(dist<best){best=dist;target=b}
  });

  const dist=moveToward(z,target.position,dt,d.speed);

  // かわいいゾンビのゆらゆら歩き
  d.phase+=dt*5;
  z.position.y=Math.abs(Math.sin(d.phase))*.06;
  z.rotation.z=Math.sin(d.phase*.5)*.06;
  d.legs[0].rotation.x=Math.sin(d.phase)*.45;
  d.legs[1].rotation.x=-Math.sin(d.phase)*.45;
  d.arms[0].rotation.x=-.8+Math.sin(d.phase)*.3;
  d.arms[1].rotation.x=-.8-Math.sin(d.phase)*.3;

  // 攻撃時は腕を振る
  d.attack-=dt;
  if(dist<1.5&&d.attack<=0){
   if(target===camera){
    hp-=8;updateHUD();
    if(hp<=0)gameOver();
   }else{
    target.userData.hp-=8;
   }
   d.attack=1;
   d.arms[0].rotation.x=-1.8;
   d.arms[1].rotation.x=-1.8;
  }
 });
}

// ---------------- GAME ----------------
function gameOver(){
 running=false;
 $("over").style.display="flex";
 $("result").textContent="KILLS "+kills;
}

function update(dt){
 if(!running)return;

 forward.set(-Math.sin(yaw),0,-Math.cos(yaw));
 right.set(Math.cos(yaw),0,-Math.sin(yaw);
 move.set(0,0,0);
 move.addScaledVector(forward,-moveY);
 move.addScaledVector(right,moveX);
 if(move.length()>1)move.normalize();

 camera.position.addScaledVector(move,5*dt);
 camera.position.x=Math.max(-28,Math.min(28,camera.position.x));
 camera.position.z=Math.max(-28,Math.min(28,camera.position.z));

 velocityY-=18*dt;
 camera.position.y+=velocityY*dt;
 if(camera.position.y<1.7){
  camera.position.y=1.7;velocityY=0;grounded=true;
 }
 camera.rotation.set(pitch,yaw,0,"YXZ");

 if(firing){
  fireTimer-=dt;
  if(fireTimer<=0){shoot();fireTimer=.12}
 }

 botAI(dt);
 zombieAI(dt);

 // 全滅したら次ラウンド
 if(zombies.every(z=>!z.userData.alive)){
  round++;
  for(let i=0;i<5+round;i++)spawnZombie();
  $("message").textContent="ROUND "+round;
  setTimeout(()=>{$("message").textContent=""},1500);
 }
}

function animate(){
 requestAnimationFrame(animate);
 const dt=Math.min(clock.getDelta(),.05);
 update(dt);
 renderer.render(scene,camera);
}

// ---------------- MOBILE CONTROLS ----------------
let look=null,lx=0,ly=0;
document.addEventListener("pointerdown",e=>{
 if(e.target.closest(".btn")||e.target.closest("#stick"))return;
 if(!running)return;
 look=e.pointerId;lx=e.clientX;ly=e.clientY;
});
document.addEventListener("pointermove",e=>{
 if(e.pointerId!==look)return;
 yaw-=(e.clientX-lx)*.004;
 pitch-=(e.clientY-ly)*.004;
 lx=e.clientX;ly=e.clientY;
 pitch=Math.max(-1.3,Math.min(1.3,pitch));
});
document.addEventListener("pointerup",e=>{if(e.pointerId===look)look=null});
document.addEventListener("pointercancel",()=>look=null);

// スティック
const stick=$("stick"),knob=$("knob");
let sp=null;
stick.onpointerdown=e=>{sp=e.pointerId;stick.setPointerCapture(sp);moveStick(e)};
stick.onpointermove=e=>{if(e.pointerId===sp)moveStick(e)};
function moveStick(e){
 const r=stick.getBoundingClientRect();
 let x=e.clientX-r.left-r.width/2;
 let y=e.clientY-r.top-r.height/2;
 const len=Math.hypot(x,y),max=45;
 if(len>max){x=x/len*max;y=y/len*max}
 knob.style.transform=`translate(${x}px,${y}px)`;
 moveX=x/max;moveY=y/max;
}
function resetStick(){sp=null;moveX=moveY=0;knob.style.transform="translate(0,0)"}
stick.onpointerup=resetStick;stick.onpointercancel=resetStick;

// 射撃
$("fire").onpointerdown=e=>{e.preventDefault();firing=true;shoot();fireTimer=.12};
$("fire").onpointerup=()=>firing=false;
$("fire").onpointercancel=()=>firing=false;
$("reload").onpointerdown=reload;
$("jump").onpointerdown=()=>{
 if(grounded){velocityY=7;grounded=false}
};

$("startbtn").onclick=()=>{
 $("start").style.display="none";
 $("over").style.display="none";
 reset();running=true;
};
$("retry").onclick=()=>{
 $("over").style.display="none";
 reset();running=true;
};

addEventListener("resize",()=>{
 if(!camera||!renderer)return;
 camera.aspect=innerWidth/innerHeight;
 camera.updateProjectionMatrix();
 renderer.setSize(innerWidth,innerHeight);
});

renderer=new THREE.WebGLRenderer({antialias:true});
renderer.setPixelRatio(Math.min(devicePixelRatio,1.5));
renderer.setSize(innerWidth,innerHeight);
renderer.shadowMap.enabled=true;
renderer.shadowMap.type=THREE.PCFSoftShadowMap;
$("game").appendChild(renderer.domElement);

clock=new THREE.Clock();
world();reset();animate();
</script>
</body>
</html>
