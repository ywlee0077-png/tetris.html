<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" />
  <title>Mini Tetris Minimal (Responsive + Drag Drop)</title>
  <style>
    :root { color-scheme: dark; }
    body{
      margin:0; background:#0b0f14; color:#e8eef6;
      font-family:system-ui, -apple-system, Segoe UI, Roboto, Arial;
      display:flex; align-items:center; justify-content:center; min-height:100vh;
      overflow:hidden;
    }
    .app{
      width:100vw; height:100vh;
      display:flex; flex-direction:column; align-items:center;
      padding:12px 12px 16px;
      box-sizing:border-box;
      gap:10px;
    }
    .top{
      width:min(520px, 100%);
      display:flex; justify-content:space-between; align-items:center;
      font-size:14px; opacity:.95;
      padding:0 2px;
      box-sizing:border-box;
    }
    .badge{
      font-size:12px; padding:2px 8px; border-radius:999px;
      border:1px solid rgba(255,255,255,.14); opacity:.9;
    }
    canvas{
      display:block;
      width:min(520px, 100%);
      height:calc(100vh - 64px);
      background:#071018; border-radius:14px;
      border:1px solid rgba(255,255,255,.08);
      box-shadow:0 10px 28px rgba(0,0,0,.35);
      touch-action:none; user-select:none; -webkit-user-select:none;
    }
  </style>
</head>
<body>
  <div class="app">
    <div class="top">
      <div>Score: <b id="score">0</b> · Lines: <b id="lines">0</b></div>
      <div id="state" class="badge">RUN</div>
    </div>

    <canvas id="game"></canvas>
  </div>

<script>
(() => {
  const COLS=10, ROWS=20;

  const canvas=document.getElementById("game");
  const ctx=canvas.getContext("2d", { alpha: false });

  const elScore=document.getElementById("score");
  const elLines=document.getElementById("lines");
  const elState=document.getElementById("state");

  const COLORS={
    I:"#57d6ff", O:"#ffd057", T:"#b88cff",
    S:"#62f29b", Z:"#ff6b6b", J:"#6ea0ff", L:"#ffb36e",
    GRID:"rgba(255,255,255,0.06)",
    BG:"#0a0f15"
  };

  const SHAPES={
    I:[[0,0,0,0],[1,1,1,1],[0,0,0,0],[0,0,0,0]],
    O:[[0,1,1,0],[0,1,1,0],[0,0,0,0],[0,0,0,0]],
    T:[[0,1,0,0],[1,1,1,0],[0,0,0,0],[0,0,0,0]],
    S:[[0,1,1,0],[1,1,0,0],[0,0,0,0],[0,0,0,0]],
    Z:[[1,1,0,0],[0,1,1,0],[0,0,0,0],[0,0,0,0]],
    J:[[1,0,0,0],[1,1,1,0],[0,0,0,0],[0,0,0,0]],
    L:[[0,0,1,0],[1,1,1,0],[0,0,0,0],[0,0,0,0]],
  };

  const clone=(m)=>m.map(r=>r.slice());
  const rotateCW=(m)=>{
    const n=m.length;
    const r=Array.from({length:n},()=>Array(n).fill(0));
    for(let y=0;y<n;y++) for(let x=0;x<n;x++) r[x][n-1-y]=m[y][x];
    return r;
  };

  // ===== 반응형 =====
  let BLOCK=32;
  let boardPXW=COLS*32, boardPXH=ROWS*32;

  function resizeCanvas(){
    const dpr = Math.max(1, Math.min(3, window.devicePixelRatio || 1));
    const rect = canvas.getBoundingClientRect();
    const cssW = Math.floor(rect.width);
    const cssH = Math.floor(rect.height);

    canvas.width  = Math.floor(cssW * dpr);
    canvas.height = Math.floor(cssH * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

    const maxBlockW = Math.floor(cssW / COLS);
    const maxBlockH = Math.floor(cssH / ROWS);
    BLOCK = Math.max(12, Math.min(maxBlockW, maxBlockH));

    boardPXW = COLS * BLOCK;
    boardPXH = ROWS * BLOCK;
  }
  function requestResize(){ requestAnimationFrame(resizeCanvas); }

  // ===== 게임 상태 =====
  let board, cur, nextType, score, lines;
  let paused=false, over=false;
  let last=0, acc=0, interval=650;

  function randType(){
    const t=["I","O","T","S","Z","J","L"];
    return t[Math.floor(Math.random()*t.length)];
  }
  function updateUI(){
    elScore.textContent=score;
    elLines.textContent=lines;
    elState.textContent = over ? "OVER" : (paused ? "PAUSE" : "RUN");
  }

  function reset(){
    board=Array.from({length:ROWS},()=>Array(COLS).fill(null));
    score=0; lines=0; paused=false; over=false;
    interval=650;
    nextType=randType();
    cur=spawn();
    updateUI();
  }

  function spawn(){
    const type=nextType;
    nextType=randType();
    const p={ type, matrix:clone(SHAPES[type]), x:Math.floor(COLS/2)-2, y:-1 };
    if(collides(p)) over=true;
    return p;
  }

  function collides(p){
    const m=p.matrix;
    for(let y=0;y<4;y++){
      for(let x=0;x<4;x++){
        if(!m[y][x]) continue;
        const bx=p.x+x, by=p.y+y;
        if(bx<0||bx>=COLS) return true;
        if(by>=ROWS) return true;
        if(by>=0 && board[by][bx]) return true;
      }
    }
    return false;
  }

  function merge(){
    const m=cur.matrix;
    for(let y=0;y<4;y++){
      for(let x=0;x<4;x++){
        if(!m[y][x]) continue;
        const bx=cur.x+x, by=cur.y+y;
        if(by>=0) board[by][bx]=cur.type;
      }
    }
  }

  function clearLines(){
    let cleared=0;
    outer: for(let y=ROWS-1;y>=0;y--){
      for(let x=0;x<COLS;x++){
        if(!board[y][x]) continue outer;
      }
      board.splice(y,1);
      board.unshift(Array(COLS).fill(null));
      cleared++; y++;
    }
    if(cleared){
      lines += cleared;
      score += cleared*100;
      interval = Math.max(120, 650 - Math.floor(lines/5)*60);
    }
  }

  function move(dx){
    if(paused||over) return;
    const p={...cur, x:cur.x+dx};
    if(!collides(p)) cur.x+=dx;
  }

  function drop(){
    if(paused||over) return;
    const p={...cur, y:cur.y+1};
    if(!collides(p)){
      cur.y++;
    }else{
      merge();
      clearLines();
      cur=spawn();
      updateUI();
    }
  }

  function hardDrop(){
    if(paused||over) return;
    while(true){
      const p={...cur, y:cur.y+1};
      if(collides(p)) break;
      cur.y++;
      score += 2;
    }
    merge();
    clearLines();
    cur=spawn();
    updateUI();
  }

  function rotate(){
    if(paused||over) return;
    const rotated=rotateCW(cur.matrix);
    const kicks=[0,-1,1,-2,2];
    for(const k of kicks){
      const p={...cur, matrix:rotated, x:cur.x+k};
      if(!collides(p)){ cur.matrix=rotated; cur.x=p.x; return; }
    }
  }

  // ===== 그리기 =====
  function drawCell(color,x,y,ox,oy){
    const px = ox + x*BLOCK;
    const py = oy + y*BLOCK;
    ctx.fillStyle=color;
    ctx.fillRect(px, py, BLOCK, BLOCK);
    ctx.strokeStyle="rgba(0,0,0,0.35)";
    ctx.lineWidth=2;
    ctx.strokeRect(px+1, py+1, BLOCK-2, BLOCK-2);
  }
  function drawGrid(ox,oy){
    ctx.strokeStyle=COLORS.GRID;
    ctx.lineWidth=1;
    for(let x=1;x<COLS;x++){
      ctx.beginPath();
      ctx.moveTo(ox + x*BLOCK, oy);
      ctx.lineTo(ox + x*BLOCK, oy + boardPXH);
      ctx.stroke();
    }
    for(let y=1;y<ROWS;y++){
      ctx.beginPath();
      ctx.moveTo(ox, oy + y*BLOCK);
      ctx.lineTo(ox + boardPXW, oy + y*BLOCK);
      ctx.stroke();
    }
  }

  function render(){
    const cssW = canvas.getBoundingClientRect().width;
    const cssH = canvas.getBoundingClientRect().height;

    ctx.fillStyle=COLORS.BG;
    ctx.fillRect(0,0,cssW,cssH);

    const ox = Math.floor((cssW - boardPXW)/2);
    const oy = Math.floor((cssH - boardPXH)/2);

    for(let y=0;y<ROWS;y++){
      for(let x=0;x<COLS;x++){
        const t=board[y][x];
        if(t) drawCell(COLORS[t], x, y, ox, oy);
      }
    }

    if(!over){
      const m=cur.matrix;
      for(let y=0;y<4;y++){
        for(let x=0;x<4;x++){
          if(!m[y][x]) continue;
          const bx=cur.x+x, by=cur.y+y;
          if(by<0) continue;
          drawCell(COLORS[cur.type], bx, by, ox, oy);
        }
      }
    }

    drawGrid(ox,oy);

    if(over){
      ctx.save();
      ctx.fillStyle="rgba(0,0,0,0.55)";
      ctx.fillRect(0,0,cssW,cssH);
      ctx.fillStyle="#e8eef6";
      ctx.textAlign="center";
      ctx.font="800 28px system-ui";
      ctx.fillText("GAME OVER", cssW/2, cssH/2);
      ctx.font="14px system-ui";
      ctx.fillStyle="rgba(232,238,246,0.85)";
      ctx.fillText("tap", cssW/2, cssH/2 + 24);
      ctx.restore();
    }
  }

  function loop(t=0){
    const dt=t-last; last=t;
    if(!paused && !over){
      acc += dt;
      if(acc>=interval){ drop(); acc=0; }
    }
    render();
    requestAnimationFrame(loop);
  }

  // ===== 입력: PC =====
  window.addEventListener("keydown",(e)=>{
    const k=e.key.toLowerCase();
    if(["arrowleft","arrowright","arrowdown","arrowup"," ","p","r"].includes(k) || e.key===" ") e.preventDefault();

    if(k==="p"){ if(!over){ paused=!paused; updateUI(); } return; }
    if(k==="r"){ reset(); return; }

    if(paused||over) return;
    if(k==="arrowleft") move(-1);
    else if(k==="arrowright") move(1);
    else if(k==="arrowdown") drop();
    else if(k==="arrowup") rotate();
    else if(e.key===" " || k===" ") hardDrop();
    updateUI();
  });

  // ===== 입력: 모바일 제스처 (연속 드롭 추가) =====
  const G={
    id:null, sx:0, sy:0, st:0,
    dragDropping:false,
    dropTimer:null
  };

  // 감도/속도 튜닝 (원하면 여기만 조절)
  const TH={
    tapMove:10,
    swipe:22,
    dragStart:18,      // 아래로 이 정도 이상 끌면 연속 드롭 시작
    dragRepeatMs:55    // 연속 드롭 속도(작을수록 더 빠름)
  };

  function startDragDrop(){
    if(G.dropTimer) return;
    G.dragDropping = true;
    G.dropTimer = setInterval(()=>{
      if(paused || over) return;
      drop();
      updateUI();
    }, TH.dragRepeatMs);
  }
  function stopDragDrop(){
    G.dragDropping = false;
    if(G.dropTimer){
      clearInterval(G.dropTimer);
      G.dropTimer = null;
    }
  }

  canvas.addEventListener("pointerdown",(e)=>{
    if(G.id!==null) return;
    G.id=e.pointerId;
    canvas.setPointerCapture(G.id);
    G.sx=e.clientX; G.sy=e.clientY; G.st=performance.now();
    stopDragDrop();
  });

  canvas.addEventListener("pointermove",(e)=>{
    if(e.pointerId!==G.id) return;

    // OVER/PAUSE에서는 연속드롭 시작하지 않음
    if(paused || over) return;

    const dx=e.clientX-G.sx;
    const dy=e.clientY-G.sy;

    // 세로 이동이 더 크고, 아래 방향이면 연속드롭 발동
    if(!G.dragDropping && dy > TH.dragStart && Math.abs(dy) > Math.abs(dx)){
      startDragDrop();
    }
  });

  canvas.addEventListener("pointerup",(e)=>{
    if(e.pointerId!==G.id) return;

    // 손 떼면 연속 드롭 종료
    stopDragDrop();

    const dx=e.clientX-G.sx, dy=e.clientY-G.sy;
    const dist=Math.hypot(dx,dy);
    const dt=performance.now()-G.st;
    const isTap=(dist<=TH.tapMove && dt<=250);

    // OVER면 탭으로 즉시 재시작
    if(over && isTap){
      reset();
      G.id=null;
      return;
    }

    // 게임 중
    if(!paused && !over){
      if(isTap){
        rotate();
      }else{
        // 연속드롭이 아닌 "짧은 스와이프"는 기존처럼 1회 액션
        if(Math.abs(dx)>Math.abs(dy)){
          if(Math.abs(dx)>TH.swipe) move(dx>0?1:-1);
        }else{
          // 아래 스와이프는 1칸 드롭
          if(dy>TH.swipe) drop();
        }
      }
      updateUI();
    }

    G.id=null;
  });

  canvas.addEventListener("pointercancel",()=>{
    stopDragDrop();
    G.id=null;
  });

  // ===== 리사이즈 훅 =====
  window.addEventListener("resize", requestResize);
  window.addEventListener("orientationchange", requestResize);
  if(window.visualViewport){
    window.visualViewport.addEventListener("resize", requestResize);
  }

  // Start
  requestResize();
  reset();
  updateUI();
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
