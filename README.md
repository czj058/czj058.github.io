<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>平台跑酷（Demo版）</title>
<style>
*{margin:0;padding:0;box-sizing:border-box}
html,body{
  width:100%;
  height:100%;
  overflow:hidden;
  touch-action:none;
  user-select:none;
}
canvas{
  display:block;
  width:100vw;
  height:100vh;
}
</style>
</head>
<body>
<canvas id="game"></canvas>

<script>
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

//游戏状态新增 loading 加载界面、fail失败界面
let gameState = "menu"; // menu / play / win / fail / loading / levelSelect
let currentLevel = 1;
let loadProgress = 0;
const LOAD_TARGET = 100;

// 随机提示列表
const tipList = [
  "小提示：可以同时按下两个按键",
  "小提示：记得躲避尖刺",
  "小提示：碰到出口就赢了",
  "小提示：返回主界面按钮，真的可以返回主界面",
  "小提示：跳下去会死哟"
];
let currentTip = "";

function resize(){
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
  loadLevel(currentLevel);
  layoutButtons();
  resetPlayer();
}
window.addEventListener('resize', resize);

const player = {
  x: 80,
  y: 200,
  w: 36,
  h: 48,
  vx: 0,
  vy: 0,
  speed: 6,
  jumpPower: -14,
  gravity: 0.6,
  onGround: false
};
const hudHeight = 150;

let platforms = [];
let gate = {};
let spikes = [];
let fallingSpikes = [];
let levelTimer = 0;

const SPIKE_W = 40;
const SPIKE_H = 35;
const SPIKE_INTERVAL = 10; // 第三关尖刺，每10帧生成一个，生成更快
const FALLING_SPEED = 7;

function loadLevel(level){
  const skyBottom = canvas.height - hudHeight;
  fallingSpikes = [];
  levelTimer = 0;

  if(level === 1){
    const groundY = skyBottom - 40;
    platforms = [
      { x: 60,  y: groundY,                 w: 220, h: 24 },
      { x: 340, y: groundY - 80,            w: 200, h: 24 },
      { x: 620, y: groundY - 160,           w: 200, h: 24 }
    ];
    spikes = [
      {x:260, y:groundY - SPIKE_H, w:SPIKE_W, h:SPIKE_H},
      {x:480, y:groundY - 80 - SPIKE_H, w:SPIKE_W, h:SPIKE_H},
    ];
    gate = {
      x: Math.min(platforms[2].x + platforms[2].w - 70, canvas.width - 100),
      y: platforms[2].y - 90,
      w: 60,
      h: 90
    };
  }else if(level === 2){
    const baseY = skyBottom - 30;
    platforms = [
      { x: 30,  y: baseY,                  w:110, h:24 },
      { x: 200, y: baseY - 80,              w:110, h:24 },
      { x: 370, y: baseY - 160,             w:110, h:24 },
      { x: 540, y: baseY - 240,             w:110, h:24 },
      { x: 710, y: baseY - 320,             w:120, h:24 }
    ];
    spikes = [
      {x:390, y: baseY - 160 - SPIKE_H, w:SPIKE_W, h:SPIKE_H}
    ];
    gate = {
      x: platforms[4].x + 30,
      y: platforms[4].y -90,
      w:60,
      h:90
    };
  }else if(level === 3){
    const groundY = skyBottom - 30;
    platforms = [
      {x:0, y:groundY, w:canvas.width + 120, h:24}
    ];
    spikes = [];
    gate = {
      x: canvas.width - 120,
      y: groundY -90,
      w:60,
      h:90
    };
  }
}

const buttons = {
  up:    { x:0, y:0, size:70, label:"↑", pressed:false },
  left:  { x:0, y:0, size:70, label:"←", pressed:false },
  right: { x:0, y:0, size:70, label:"→", pressed:false }
};
function layoutButtons(){
  const gap = 20;
  const s = buttons.up.size;
  const totalW = s * 3 + gap * 2;
  const startX = (canvas.width - totalW) / 2;
  const centerY = canvas.height - hudHeight / 2;
  buttons.left.x = startX;
  buttons.left.y = centerY - s / 2;
  buttons.up.x = buttons.left.x + s + gap;
  buttons.up.y = centerY - s / 2;
  buttons.right.x = buttons.up.x + s + gap;
  buttons.right.y = centerY - s / 2;
}

const keys = {};
document.addEventListener('keydown', e => {
  keys[e.code] = true;
  if(['ArrowUp','ArrowLeft','ArrowRight'].includes(e.code)) e.preventDefault();
});
document.addEventListener('keyup', e => keys[e.code] = false);
const touchesMap = new Map();

function pointInButton(px,py,btn){
  const cx = btn.x + btn.size/2;
  const cy = btn.y + btn.size/2;
  const r = btn.size/2;
  const dx = px - cx;
  const dy = py - cy;
  return dx*dx + dy*dy <= r*r;
}

function handleClickPos(mx, my){
  if(gameState === "menu"){
    //开始游戏
    if(mx > canvas.width/2 -120 && mx < canvas.width/2 +120 && my>canvas.height/2-20 && my<canvas.height/2+40){
      currentLevel = 1;
      startLoading();
    }
    //选择关卡
    if(mx > canvas.width/2 -120 && mx < canvas.width/2 +120 && my>canvas.height/2+60 && my<canvas.height/2+120){
      gameState = "levelSelect";
    }
  }else if(gameState === "levelSelect"){
    //关卡选择界面
    //第一关
    if(mx>canvas.width/2-100 && mx<canvas.width/2+100 && my>canvas.height/2-120 && my<canvas.height/2-60){
      currentLevel = 1;
      startLoading();
    }
    //第二关
    if(mx>canvas.width/2-100 && mx<canvas.width/2+100 && my>canvas.height/2-40 && my<canvas.height/2+20){
      currentLevel = 2;
      startLoading();
    }
    //第三关
    if(mx>canvas.width/2-100 && mx<canvas.width/2+100 && my>canvas.height/2+40 && my<canvas.height/2+100){
      currentLevel = 3;
      startLoading();
    }
    //返回主菜单按钮
    if(mx>canvas.width/2-100 && mx<canvas.width/2+100 && my>canvas.height/2+140 && my<canvas.height/2+190){
      gameState = "menu";
    }
  }else if(gameState === "fail"){
    //失败界面：重新玩 / 返回主菜单
    if(mx>canvas.width/2-140 && mx<canvas.width/2+140 && my>canvas.height/2 && my<canvas.height/2+50){
      //重新玩：重启本关
      startLoading();
    }
    if(mx>canvas.width/2-140 && mx<canvas.width/2+140 && my>canvas.height/2+70 && my<canvas.height/2+120){
      gameState = "menu";
    }
  }else if(gameState === "win"){
    if(currentLevel === 1){
      if(mx>canvas.width/2-140 && mx<canvas.width/2+140 && my>canvas.height/2 && my<canvas.height/2+50){
        currentLevel = 2;
        startLoading();
      }
      if(mx>canvas.width/2-140 && mx<canvas.width/2+140 && my>canvas.height/2+70 && my<canvas.height/2+120){
        gameState = "menu";
      }
    }else if(currentLevel === 2){
      if(mx>canvas.width/2-140 && mx<canvas.width/2+140 && my>canvas.height/2 && my<canvas.height/2+50){
        currentLevel = 3;
        startLoading();
      }
      if(mx>canvas.width/2-140 && mx<canvas.width/2+140 && my>canvas.height/2+70 && my<canvas.height/2+120){
        gameState = "menu";
      }
    }else{
      if(mx>canvas.width/2-140 && mx<canvas.width/2+140 && my>canvas.height/2 && my<canvas.height/2+50){
        gameState = "menu";
      }
    }
  }else if(gameState === "play"){
    if(pointInButton(mx,my,buttons.up)) buttons.up.pressed=true;
    if(pointInButton(mx,my,buttons.left)) buttons.left.pressed=true;
    if(pointInButton(mx,my,buttons.right)) buttons.right.pressed=true;
  }
}

canvas.addEventListener('touchstart', e=>{
  e.preventDefault();
  const rect = canvas.getBoundingClientRect();
  for(let t of e.touches) {
    touchesMap.set(t.identifier, {x:t.clientX, y:t.clientY});
    const tx = (t.clientX - rect.left) * (canvas.width / rect.width);
    const ty = (t.clientY - rect.top) * (canvas.height / rect.height);
    handleClickPos(tx, ty);
  }
  updateTouchButtons();
},{passive:false});
canvas.addEventListener('touchmove', e=>{
  e.preventDefault();
  for(let t of e.touches) touchesMap.set(t.identifier, {x:t.clientX, y:t.clientY});
  updateTouchButtons();
},{passive:false});
canvas.addEventListener('touchend', e=>{
  for(let t of e.changedTouches) touchesMap.delete(t.identifier);
  updateTouchButtons();
});
canvas.addEventListener('touchcancel', e=>{
  for(let t of e.changedTouches) touchesMap.delete(t.identifier);
  updateTouchButtons();
});
canvas.addEventListener('mousedown',e=>{
  const rect = canvas.getBoundingClientRect();
  const mx = (e.clientX - rect.left)*(canvas.width/rect.width);
  const my = (e.clientY - rect.top)*(canvas.height/rect.height);
  handleClickPos(mx, my);
});
canvas.addEventListener('mouseup',()=>{
  buttons.up.pressed=false;
  buttons.left.pressed=false;
  buttons.right.pressed=false;
});

function updateTouchButtons(){
  buttons.up.pressed = false;
  buttons.left.pressed = false;
  buttons.right.pressed = false;
  const rect = canvas.getBoundingClientRect();
  touchesMap.forEach(t=>{
    const x = (t.x - rect.left) * (canvas.width / rect.width);
    const y = (t.y - rect.top) * (canvas.height / rect.height);
    if(pointInButton(x,y,buttons.up)) buttons.up.pressed = true;
    if(pointInButton(x,y,buttons.left)) buttons.left.pressed = true;
    if(pointInButton(x,y,buttons.right)) buttons.right.pressed = true;
  });
}

function resetPlayer(){
  player.x = Math.max(20, platforms[0].x + 20);
  player.y = platforms[0].y - player.h - 20;
  player.vx = 0;
  player.vy = 0;
}

// 启动加载界面，随机选一条提示
function startLoading(){
  gameState = "loading";
  loadProgress = 0;
  //随机挑选提示
  currentTip = tipList[Math.floor(Math.random() * tipList.length)];
}

function update(){
  if(gameState === "loading"){
    loadProgress += 1.2;
    if(loadProgress >= LOAD_TARGET){
      loadLevel(currentLevel);
      resetPlayer();
      gameState = "play";
    }
    return;
  }
  if(gameState !== "play") return;

  let moveX = 0;
  if(keys['ArrowLeft'] || buttons.left.pressed) moveX -= player.speed;
  if(keys['ArrowRight'] || buttons.right.pressed) moveX += player.speed;
  player.vx = moveX;
  if((keys['ArrowUp'] || buttons.up.pressed) && player.onGround){
    player.vy = player.jumpPower;
    player.onGround = false;
  }
  player.vy += player.gravity;
  player.x += player.vx;
  player.y += player.vy;
  player.onGround = false;
  for(let p of platforms){
    if(
      player.x + player.w > p.x &&
      player.x < p.x + p.w &&
      player.y + player.h > p.y &&
      player.y + player.h < p.y + p.h + 12 &&
      player.vy >= 0
    ){
      player.y = p.y - player.h;
      player.vy = 0;
      player.onGround = true;
    }
  }

  // 普通固定尖刺碰撞
  for(let spike of spikes){
    if(
      player.x + player.w > spike.x &&
      player.x < spike.x + spike.w &&
      player.y + player.h > spike.y &&
      player.y < spike.y + spike.h
    ){
      gameState = "fail";
    }
  }

  // 第三关：每10帧生成下落尖刺
  if(currentLevel === 3){
    levelTimer++;
    if(levelTimer % SPIKE_INTERVAL === 0){
      fallingSpikes.push({
        x: 40 + (levelTimer / SPIKE_INTERVAL) * 110,
        y: -30,
        w: SPIKE_W,
        h: SPIKE_H,
        vy: FALLING_SPEED
      });
    }

    for(let i = fallingSpikes.length - 1; i >= 0; i--){
      const fs = fallingSpikes[i];
      fs.y += fs.vy;

      if(
        player.x + player.w > fs.x &&
        player.x < fs.x + fs.w &&
        player.y + player.h > fs.y &&
        player.y < fs.y + fs.h
      ){
        gameState = "fail";
      }

      if(fs.y > platforms[0].y){
        fallingSpikes.splice(i, 1);
      }
    }
  }

  if(player.y > canvas.height - hudHeight + 60){
    gameState = "fail";
  }

  if(
    player.x + player.w > gate.x &&
    player.x < gate.x + gate.w &&
    player.y + player.h > gate.y &&
    player.y < gate.y + gate.h
  ){
    gameState = "win";
  }
}

function drawSky(){
  const gradient = ctx.createLinearGradient(0,0,0,canvas.height - hudHeight);
  gradient.addColorStop(0,"#70c5ff");
  gradient.addColorStop(0.7,"#b8e6ff");
  gradient.addColorStop(1,"#ffffff");
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.width, canvas.height - hudHeight);
  ctx.fillStyle = "rgba(255,255,255,0.9)";
  drawCloud(120, 90, 40);
  drawCloud(380, 60, 50);
  drawCloud(640, 110, 35);
  drawCloud(900, 70, 45);
}
function drawCloud(x,y,r){
  ctx.beginPath();
  ctx.arc(x,y,r,0,Math.PI*2);
  ctx.arc(x+r*0.8,y-r*0.2,r*0.7,0,Math.PI*2);
  ctx.arc(x+r*1.4,y,r*0.8,0,Math.PI*2);
  ctx.fill();
}
function drawVoidHUD(){
  ctx.fillStyle = "#111827";
  ctx.fillRect(0, canvas.height - hudHeight, canvas.width, hudHeight);
  ctx.strokeStyle = "rgba(255,255,255,0.12)";
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.moveTo(0, canvas.height - hudHeight);
  ctx.lineTo(canvas.width, canvas.height - hudHeight);
  ctx.stroke();
}
function drawPlatforms(){
  for(let p of platforms){
    ctx.fillStyle = "rgba(0,0,0,0.25)";
    ctx.fillRect(p.x+4,p.y+6,p.w,p.h);
    ctx.fillStyle="#2ecc71";
    ctx.fillRect(p.x,p.y,p.w,p.h);
    ctx.fillStyle="#82e0aa";
    ctx.fillRect(p.x,p.y,p.w,6);
  }
}

function drawSpikes(){
  ctx.fillStyle = "#ff3333";
  ctx.strokeStyle = "#aa0000";
  ctx.lineWidth = 2;

  for(let s of spikes){
    ctx.beginPath();
    ctx.moveTo(s.x, s.y + s.h);
    ctx.lineTo(s.x + s.w / 2, s.y);
    ctx.lineTo(s.x + s.w, s.y + s.h);
    ctx.closePath();
    ctx.fill();
    ctx.stroke();
  }

  if(currentLevel === 3){
    for(let fs of fallingSpikes){
      ctx.beginPath();
      ctx.moveTo(fs.x, fs.y + fs.h);
      ctx.lineTo(fs.x + fs.w / 2, fs.y);
      ctx.lineTo(fs.x + fs.w, fs.y + fs.h);
      ctx.closePath();
      ctx.fill();
      ctx.stroke();
    }
  }
}

function drawGate(){
  ctx.fillStyle = "rgba(0,0,0,0.3)";
  ctx.fillRect(gate.x+6, gate.y+6, gate.w, gate.h);
  ctx.fillStyle = "#f1c40f";
  ctx.fillRect(gate.x, gate.y, gate.w, gate.h);
  ctx.fillStyle = "#000";
  ctx.fillRect(gate.x+8, gate.y+10, gate.w-16, gate.h-20);
  ctx.fillStyle = "#f1c40f";
  ctx.fillRect(gate.x+14, gate.y+18, gate.w-28, 10);
  ctx.fillStyle = "#fff";
  ctx.font = "bold 16px sans-serif";
  ctx.textAlign = "center";
  ctx.fillText("出口", gate.x + gate.w/2, gate.y - 12);
}
function drawPlayer(){
  ctx.fillStyle = "rgba(0,0,0,0.3)";
  ctx.fillRect(player.x+4, player.y+4, player.w, player.h);
  ctx.fillStyle="#3498db";
  ctx.fillRect(player.x,player.y,player.w,player.h);
  ctx.fillStyle="#74b9ff";
  ctx.fillRect(player.x,player.y,player.w,8);
  ctx.fillStyle="#fff";
  ctx.fillRect(player.x+8,player.y+12,6,6);
  ctx.fillRect(player.x+22,player.y+12,6,6);
}
function drawButton(btn){
  const cx = btn.x + btn.size/2;
  const cy = btn.y + btn.size/2;
  const r = btn.size/2;
  ctx.fillStyle = btn.pressed ? "rgba(255,255,255,0.92)" : "rgba(255,255,255,0.18)";
  ctx.beginPath();
  ctx.arc(cx, cy, r, 0, Math.PI*2);
  ctx.fill();
  ctx.strokeStyle = btn.pressed ? "#fff" : "rgba(255,255,255,0.5)";
  ctx.lineWidth = 3;
  ctx.stroke();
  ctx.fillStyle = btn.pressed ? "#000" : "#fff";
  ctx.font = "bold 32px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText(btn.label, cx, cy);
}

function drawMenu(){
  ctx.fillStyle="#87CEEB";
  ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle="#fff";
  ctx.font="bold 48px sans-serif";
  ctx.textAlign="center";
  ctx.fillText("平台跑酷（Demo版）", canvas.width/2, canvas.height/2 - 100);
  //开始游戏
  ctx.fillStyle="#27ae60";
  ctx.fillRect(canvas.width/2-120, canvas.height/2-20, 240,60);
  ctx.fillStyle="#fff";
  ctx.font="bold 28px sans-serif";
  ctx.fillText("开始游戏", canvas.width/2, canvas.height/2+15);
  //选择关卡
  ctx.fillStyle="#2980b9";
  ctx.fillRect(canvas.width/2-120, canvas.height/2+60, 240,60);
  ctx.fillStyle="#fff";
  ctx.fillText("选择关卡", canvas.width/2, canvas.height/2+95);
}

//关卡选择界面
function drawLevelSelect(){
  ctx.fillStyle="#87CEEB";
  ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle="#fff";
  ctx.font="bold 40px sans-serif";
  ctx.textAlign="center";
  ctx.fillText("选择关卡", canvas.width/2, canvas.height/2 -180);

  //第一关按钮
  ctx.fillStyle="#27ae60";
  ctx.fillRect(canvas.width/2-100, canvas.height/2-120,200,60);
  ctx.fillStyle="#fff";
  ctx.font="bold 26px sans-serif";
  ctx.fillText("第一关", canvas.width/2, canvas.height/2 -85);

  //第二关按钮
  ctx.fillStyle="#2980b9";
  ctx.fillRect(canvas.width/2-100, canvas.height/2-40,200,60);
  ctx.fillStyle="#fff";
  ctx.fillText("第二关", canvas.width/2, canvas.height/2-5);

  //第三关按钮
  ctx.fillStyle="#c0392b";
  ctx.fillRect(canvas.width/2-100, canvas.height/2+40,200,60);
  ctx.fillStyle="#fff";
  ctx.fillText("第三关", canvas.width/2, canvas.height/2+75);

  //返回主菜单
  ctx.fillStyle="#444444";
  ctx.fillRect(canvas.width/2-100, canvas.height/2+140,200,50);
  ctx.fillStyle="#fff";
  ctx.font="bold 22px sans-serif";
  ctx.fillText("返回主菜单", canvas.width/2, canvas.height/2+172);
}

//失败界面
function drawFail(){
  ctx.fillStyle="rgba(0,0,0,0.7)";
  ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle="#ff5555";
  ctx.font="bold 48px sans-serif";
  ctx.textAlign="center";
  ctx.fillText("💀 游戏失败", canvas.width/2, canvas.height/2 - 80);

  //重新玩按钮
  ctx.fillStyle="#2980b9";
  ctx.fillRect(canvas.width/2-140, canvas.height/2, 280,50);
  ctx.fillStyle="#fff";
  ctx.font="bold 24px sans-serif";
  ctx.fillText("重新玩", canvas.width/2, canvas.height/2+32);

  //返回主菜单
  ctx.fillStyle="#c0392b";
  ctx.fillRect(canvas.width/2-140, canvas.height/2+70, 280,50);
  ctx.fillStyle="#fff";
  ctx.fillText("退回主界面", canvas.width/2, canvas.height/2+102);
}

function drawWin(){
  ctx.fillStyle="rgba(0,0,0,0.65)";
  ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle="#fff";
  ctx.font="bold 48px sans-serif";
  ctx.textAlign="center";
  if(currentLevel===1){
    ctx.fillText("🎉第一关通关！", canvas.width/2, canvas.height/2 - 80);
    ctx.fillStyle="#2980b9";
    ctx.fillRect(canvas.width/2-140, canvas.height/2, 280,50);
    ctx.fillStyle="#fff";
    ctx.font="bold 24px sans-serif";
    ctx.fillText("进入第二关", canvas.width/2, canvas.height/2+32);
    ctx.fillStyle="#c0392b";
    ctx.fillRect(canvas.width/2-140, canvas.height/2+70, 280,50);
    ctx.fillStyle="#fff";
    ctx.fillText("返回主菜单", canvas.width/2, canvas.height/2+102);
  }else if(currentLevel===2){
    ctx.fillText("🎉第二关通关！", canvas.width/2, canvas.height/2 - 80);
    ctx.fillStyle="#2980b9";
    ctx.fillRect(canvas.width/2-140, canvas.height/2, 280,50);
    ctx.fillStyle="#fff";
    ctx.font="bold 24px sans-serif";
    ctx.fillText("进入第三关", canvas.width/2, canvas.height/2+32);
    ctx.fillStyle="#c0392b";
    ctx.fillRect(canvas.width/2-140, canvas.height/2+70, 280,50);
    ctx.fillStyle="#fff";
    ctx.fillText("返回主菜单", canvas.width/2, canvas.height/2+102);
  }else{
    ctx.fillText("🎉第三关通关！全部关卡完成", canvas.width/2, canvas.height/2 - 80);
    ctx.fillStyle="#2980b9";
    ctx.fillRect(canvas.width/2-140, canvas.height/2, 280,50);
    ctx.fillStyle="#fff";
    ctx.font="bold 24px sans-serif";
    ctx.fillText("返回主菜单", canvas.width/2, canvas.height/2+32);
  }
}

//加载界面
function drawLoading(){
  ctx.fillStyle="#111827";
  ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle="#ffffff";
  ctx.font="bold 32px sans-serif";
  ctx.textAlign="center";
  ctx.fillText("加载中", canvas.width/2, canvas.height/2 - 100);
  ctx.font="bold 26px sans-serif";
  ctx.fillStyle="#ffdd77";
  ctx.fillText(currentTip, canvas.width/2, canvas.height/2 - 30);

  //进度条
  const barW = 400;
  const barH = 24;
  const barX = canvas.width/2 - barW/2;
  const barY = canvas.height/2 + 40;
  ctx.fillStyle="#333";
  ctx.fillRect(barX, barY, barW, barH);
  ctx.fillStyle="#42a5f5";
  ctx.fillRect(barX, barY, barW * loadProgress / LOAD_TARGET, barH);
  ctx.strokeStyle="#fff";
  ctx.lineWidth=2;
  ctx.strokeRect(barX, barY, barW, barH);
}

function draw(){
  if(gameState === "menu"){
    drawMenu();
    return;
  }
  if(gameState === "levelSelect"){
    drawLevelSelect();
    return;
  }
  if(gameState === "loading"){
    drawLoading();
    return;
  }
  if(gameState === "fail"){
    drawSky();
    drawVoidHUD();
    drawPlatforms();
    drawSpikes();
    drawGate();
    drawPlayer();
    drawButton(buttons.up);
    drawButton(buttons.left);
    drawButton(buttons.right);
    drawFail();
    return;
  }
  if(gameState === "win"){
    drawSky();
    drawVoidHUD();
    drawPlatforms();
    drawSpikes();
    drawGate();
    drawPlayer();
    drawButton(buttons.up);
    drawButton(buttons.left);
    drawButton(buttons.right);
    drawWin();
    return;
  }
  drawSky();
  drawVoidHUD();
  drawPlatforms();
  drawSpikes();
  drawGate();
  drawPlayer();
  drawButton(buttons.up);
  drawButton(buttons.left);
  drawButton(buttons.right);

  ctx.fillStyle = "rgba(0,0,0,0.4)";
  ctx.fillRect(10,10,140,40);
  ctx.fillStyle="#ffffff";
  ctx.font="bold 22px sans-serif";
  ctx.textAlign="left";
  ctx.fillText("关卡："+currentLevel, 20,34);
}

function loop(){
  update();
  draw();
  requestAnimationFrame(loop);
}
resize();
loop();
</script>
</body>
</html>
