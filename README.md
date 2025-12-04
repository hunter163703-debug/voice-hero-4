<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Voice Hero: Parkinson's Therapy</title>
  <style>
    body {
      margin: 0; padding: 0;
      background-color: #050510;
      color: white;
      font-family: 'Microsoft JhengHei', sans-serif;
      overflow: hidden;
      display: flex; justify-content: center; align-items: center;
      height: 100vh;
    }
    /* 預設載入畫面，不依賴 JS */
    #static-loading {
      position: absolute; top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      text-align: center; font-size: 24px; color: #0ff;
      z-index: 1000;
      background: rgba(0,0,0,0.9); padding: 40px; border-radius: 10px; border: 1px solid #444;
    }
    canvas {
      box-shadow: 0 0 50px rgba(0, 200, 255, 0.15);
      border: 2px solid #334; border-radius: 16px;
      display: none; /* JS 啟動後會開啟 */
    }
  </style>

  <!-- 改用 cdnjs 來源，最穩定 -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.0/p5.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.0/addons/p5.sound.min.js"></script>

</head>
<body>

  <!-- 靜態載入文字 (如果畫面卡在這，表示 Script 沒跑) -->
  <div id="static-loading">
    <div>⚡ 系統啟動中...</div>
    <div style="font-size:16px; color:#aaa; margin-top:15px;">如果卡住超過 10 秒，請檢查網路</div>
  </div>

  <script>
    // ==========================================
    // 1. 錯誤攔截與初始化
    // ==========================================
    window.onload = function() {
      // 檢查 P5 是否載入成功
      if (typeof p5 === 'undefined') {
        document.getElementById('static-loading').innerHTML = 
          "<span style='color:red'>❌ 錯誤：核心檔案載入失敗</span><br>請重新整理網頁";
      }
    };

    // ==========================================
    // 2. 全域變數
    // ==========================================
    let mic;
    let gameState = "START"; 
    const W = 1024;
    const H = 768;
    
    let micLevel = 0;
    let dbSimulated = 0;
    const THRESHOLD = 40;

    let particles = [];
    let stars = [];
    let gridOffset = 0;

    // 關卡變數
    let l1_phase = 0, l1_robots = [], l1_turretAngle = 0, l1_turretDir = 1;
    let l2_phase = 0, l2_timer = 0, l2_balloonSize = 50, l2_targetTime = 8, l2_silenceCount = 0;
    let l3_phase = 0, l3_bpm = 60, l3_startTime = 0, l3_lastBeatIdx = -1, l3_missCount = 0;
    let l3_lyrics = ["三輪車 跑得快", "上面坐個 老太太", "要五毛 給一塊", "你說奇怪 不奇怪"];
    let l3_isSpeaking = false, l3_feedback = "", l3_feedbackColor;

    // ==========================================
    // 3. P5.js Setup & Draw
    // ==========================================
    function setup() {
      // 隱藏載入畫面
      let loadDiv = document.getElementById('static-loading');
      if(loadDiv) loadDiv.style.display = 'none';

      let cnv = createCanvas(W, H);
      cnv.style('display', 'block'); // 顯示畫布
      
      mic = new p5.AudioIn();
      textAlign(CENTER, CENTER);
      rectMode(CENTER);
      
      for(let i=0; i<60; i++) stars.push({x:random(W), y:random(H), s:random(1,3)});
      l3_feedbackColor = color(255);
    }

    function draw() {
      // Audio Processing
      if (gameState !== "START" && mic) {
        try {
          let vol = mic.getLevel();
          micLevel = lerp(micLevel, vol, 0.1);
          if (vol > 0.001) {
            let rawDB = map(Math.log10(vol), -3, -0.5, 30, 100);
            dbSimulated = constrain(rawDB, 0, 110);
          } else {
            dbSimulated = lerp(dbSimulated, 10, 0.1);
          }
        } catch(e) { dbSimulated = 0; }
      }

      drawSciFiBackground();

      switch(gameState) {
        case "START": drawStartScreen(); break;
        case "MENU": drawMenuScreen(); break;
        case "L1": drawLevel1(); break;
        case "L2": drawLevel2(); break;
        case "L3": drawLevel3(); break;
        case "END": drawEndScreen(); break;
      }
      updateParticles();
    }

    // ==========================================
    // 4. 視覺函式
    // ==========================================
    function drawSciFiBackground() {
      background(10, 12, 20);
      noStroke(); fill(255, 255, 255, 180);
      for(let s of stars) ellipse(s.x, s.y, s.s);
      stroke(0, 255, 255, 40); strokeWeight(1);
      gridOffset = (gridOffset + 0.8) % 40;
      for(let x = 0; x <= W; x+=80) line(x, H/2, (x-W/2)*3 + W/2, H);
      for(let y = H/2; y < H; y+=40) {
        let py = y + gridOffset;
        if(py < H) line(0, py, W, py);
      }
    }

    function drawNeonGoal(txt, subTxt, phaseColor) {
      push(); translate(W/2, 60);
      drawingContext.shadowBlur = 20; drawingContext.shadowColor = phaseColor;
      stroke(phaseColor); strokeWeight(2); fill(0, 0, 0, 200);
      rect(0, 0, 500, 70, 10);
      noStroke(); fill(phaseColor); textSize(28); textStyle(BOLD); text(txt, 0, -10);
      fill(255); textSize(16); textStyle(NORMAL); drawingContext.shadowBlur = 0; 
      text(subTxt, 0, 20); pop();
    }

    function drawButton(label, x, y, w, h, c) {
      let isHover = mouseX > x-w/2 && mouseX < x+w/2 && mouseY > y-h/2 && mouseY < y+h/2;
      if(isHover) {
        stroke(c); strokeWeight(3); fill(c);
        drawingContext.shadowBlur = 20; drawingContext.shadowColor = c;
      } else {
        stroke(c); strokeWeight(2); fill(0,0,0, 150);
        drawingContext.shadowBlur = 0;
      }
      rect(x, y, w, h, 10);
      drawingContext.shadowBlur = 0; 
      fill(isHover ? 0 : 255); noStroke(); textSize(22); text(label, x, y);
      return isHover && mouseIsPressed;
    }

    function drawBackButton() {
      if(drawButton("EXIT", 60, 40, 80, 40, "#666")) gameState = "MENU";
    }
    
    function drawDBMeter(x, y) {
      textAlign(LEFT); textSize(14); fill(180);
      text(`VOICE: ${nf(dbSimulated, 2, 1)} dB`, x - 150, y - 20);
      noFill(); stroke(80); strokeWeight(2); rect(x, y, 300, 16, 4);
      noStroke(); let w = map(dbSimulated, 0, 100, 0, 296); w = constrain(w, 0, 296);
      if(dbSimulated < THRESHOLD) fill(100); else if(dbSimulated < 50) fill(0, 255, 0); 
      else if(dbSimulated < 60) fill(255, 255, 0); else fill(255, 0, 0); 
      rectMode(CORNER); rect(x - 148, y - 6, w, 12, 2);
      stroke(255); strokeWeight(2); let tx = map(THRESHOLD, 0, 100, 0, 300);
      line(x - 150 + tx, y - 10, x - 150 + tx, y + 10);
      rectMode(CENTER); textAlign(CENTER);
    }

    // ==========================================
    // 5. 遊戲流程
    // ==========================================
    function drawStartScreen() {
      fill(0, 255, 255); textSize(60); textStyle(BOLD);
      let floatY = sin(frameCount * 0.05) * 5;
      text("VOICE HERO", W/2, H/3 + floatY);
      fill(255); textSize(26); textStyle(NORMAL);
      text("Parkinson's Therapy Edition", W/2, H/3 + 60 + floatY);
      fill(150); textSize(16); text("請確認麥克風已開啟", W/2, H - 150);
      
      if (drawButton("START GAME", W/2, H/2 + 50, 250, 60, "#0ff")) {
        userStartAudio().then(() => { mic.start(); gameState = "MENU"; }).catch(() => gameState = "MENU"); 
      }
    }

    function drawMenuScreen() {
      fill(255); textSize(40); text("訓練任務選擇", W/2, 100);
      drawDBMeter(W/2, 160);
      if(drawButton("L1: Loud (音量)", W/2, 280, 400, 80, "#0f0")) { initLevel1(); gameState = "L1"; }
      if(drawButton("L2: Long (長音)", W/2, 400, 400, 80, "#ff0")) { initLevel2(); gameState = "L2"; }
      if(drawButton("L3: Speed (語速)", W/2, 520, 400, 80, "#f0f")) { initLevel3(); gameState = "L3"; }
    }

    function drawEndScreen() {
      if(frameCount%5==0) createParticle(random(W), random(H), color(random(255),255,255), 10);
      textSize(50); fill(0, 255, 0); text("🎉 訓練完成 🎉", W/2, H/3);
      if(drawButton("返回主選單", W/2, H-150, 200, 60, "#fff")) gameState = "MENU";
    }

    // --- Level 1 ---
    function initLevel1() { l1_phase = 0; setupL1Robots(); }
    function setupL1Robots() {
      l1_robots = [];
      let cVal = (l1_phase==0)?color(0,255,0):(l1_phase==1)?color(255,255,0):color(255,50,50);
      let yPos = H/2 - (l1_phase*100);
      for(let i=0; i<3; i++) l1_robots.push({x: W/2-150+i*150, y: yPos, c: cVal, active: true});
    }
    function drawLevel1() {
      let tMin=40, tMax=50, c="#0f0", txt="40-50 dB";
      if(l1_phase==1){ tMin=50; tMax=60; c="#ff0"; txt="50-60 dB"; }
      if(l1_phase==2){ tMin=60; tMax=110; c="#f00"; txt="60+ dB"; }
      drawNeonGoal(`目標：${txt}`, "請維持音量擊倒機器人", c);
      
      let activeCount = 0;
      for(let r of l1_robots) {
        if(r.active) {
          activeCount++;
          push(); translate(r.x, r.y + sin(frameCount*0.05)*8);
          fill(r.c); noStroke(); rect(0,0,60,60,8); fill(0); rect(-15,-10,10,10); rect(15,-10,10,10);
          pop();
        }
      }
      if(activeCount === 0) {
        l1_phase++;
        if(l1_phase > 2) { textSize(60); fill(0,255,0); text("CLEAR!", W/2, H/2); if(drawButton("NEXT >>", W/2, H/2+100, 200, 60, "#0f0")) { initLevel2(); gameState="L2"; } return; }
        setupL1Robots();
      }
      push(); translate(W/2, H-80);
      l1_turretAngle += 0.02 * l1_turretDir; if(abs(l1_turretAngle)>0.4) l1_turretDir*=-1;
      rotate(l1_turretAngle); fill(50,100,200); rect(0,0,80,80,15); fill(100,200,255); rect(0,-40,30,60);
      if(dbSimulated >= tMin) {
        fill(255,200,50,200); ellipse(0,-80,60,60);
        if(frameCount % 15 === 0 && activeCount > 0) {
           stroke(255,255,0); strokeWeight(8); line(0,-80,0,-600);
           let idx = l1_robots.findIndex(r => r.active);
           if(idx !== -1) { l1_robots[idx].active = false; for(let k=0;k<15;k++) createParticle(l1_robots[idx].x, l1_robots[idx].y, l1_robots[idx].c, 8); }
        }
      }
      pop(); drawBackButton();
    }

    // --- Level 2 ---
    function initLevel2() { l2_phase=0; setupL2(); }
    function setupL2() { l2_timer=0; l2_balloonSize=50; l2_silenceCount=0; l2_targetTime = (l2_phase==0)?8:(l2_phase==1)?10:15; }
    function drawLevel2() {
      let c = (l2_phase==0)?"#f00":(l2_phase==1)?"#ff0":"#0ff";
      drawNeonGoal(`維持長音：${l2_targetTime} 秒`, (dbSimulated>=THRESHOLD)?"很好！":"請持續發聲", c);
      fill(255); textSize(24); textAlign(LEFT); text(`TIME: ${nf(l2_timer, 1, 2)}s`, 20, H-40);
      
      if(dbSimulated >= THRESHOLD) {
        l2_silenceCount = 0; l2_timer += deltaTime/1000;
        l2_balloonSize = map(l2_timer/l2_targetTime, 0, 1, 50, 350);
        if(frameCount%5==0) createParticle(W/2, H/2+180, color(255,100), 3);
      } else if(l2_timer > 0) {
        l2_silenceCount += deltaTime/1000;
        if(l2_silenceCount > 0.1) { // 100ms
           l2_balloonSize = max(50, l2_balloonSize - 2);
           fill(255,50,50); textSize(20); textAlign(CENTER); text("聲音中斷！", W/2, H/2+220);
        }
      }
      let bc = (l2_phase==0)?color(255,100,100):(l2_phase==1)?color(255,200,0):color(0,150,255);
      noStroke(); fill(bc); let s = (dbSimulated>=THRESHOLD)?random(-2,2):0;
      ellipse(W/2+s, H/2+s, l2_balloonSize, l2_balloonSize*1.2);
      if(l2_timer >= l2_targetTime) {
         for(let i=0; i<50; i++) createParticle(W/2, H/2, bc, 12);
         l2_phase++; if(l2_phase>2){ textSize(60); fill(0,255,0); text("CLEAR!", W/2, H/2); if(drawButton("NEXT >>", W/2, H/2+100, 200, 60, "#0f0")) { initLevel3(); gameState="L3"; } return; }
         setupL2();
      }
      drawBackButton();
    }

    // --- Level 3 ---
    function initLevel3() { l3_phase=0; setupL3(); }
    function setupL3() { l3_bpm=(l3_phase==0)?60:(l3_phase==1)?80:100; l3_startTime=millis(); l3_missCount=0; l3_feedback=""; }
    function drawLevel3() {
      drawNeonGoal(`語速挑戰：${l3_bpm} BPM`, (l3_missCount>=5)?"⚠️ 節奏混亂":"請在綠燈時朗讀", (l3_missCount>=5)?"#f00":"#0ff");
      let interval = 60000/l3_bpm;
      let elapsed = millis() - l3_startTime;
      let beatIdx = floor(elapsed/interval);
      let slot = beatIdx % 6;
      
      for(let i=0; i<6; i++) {
        let cx = W/2 - 100 + i*40;
        if(i===slot) { fill(0,255,0); drawingContext.shadowBlur=15; drawingContext.shadowColor="#0f0"; ellipse(cx, 200, 30, 30); drawingContext.shadowBlur=0; }
        else { fill(50); noStroke(); ellipse(cx, 200, 20, 20); }
      }
      
      let lineIdx = floor(elapsed / (4*interval)) % l3_lyrics.length;
      let totalLines = floor(elapsed / (4*interval));
      textSize(60); textAlign(CENTER); fill(255); text(l3_lyrics[lineIdx], W/2, H/2+50);

      if(dbSimulated >= THRESHOLD) {
        if(!l3_isSpeaking) {
          l3_isSpeaking = true;
          let progress = (elapsed%interval)/interval;
          let diff = min(progress*interval, (1-progress)*interval);
          if(diff < 150) { l3_feedback="✨ Perfect!"; l3_feedbackColor=color(0,255,0); l3_missCount=max(0, l3_missCount-1); }
          else if(diff > 200) { l3_feedback="❌ Miss"; l3_feedbackColor=color(255,100,100); l3_missCount++; }
        }
      } else { l3_isSpeaking = false; }
      
      textSize(30); fill(l3_feedbackColor); text(l3_feedback, W/2, H/2+150);
      if(totalLines >= 8) { l3_phase++; if(l3_phase>2) gameState="END"; else setupL3(); }
      drawBackButton();
    }

    function createParticle(x, y, c, s) { particles.push({x:x, y:y, vx:random(-4,4), vy:random(-4,4), life:255, c:c, size:s}); }
    function updateParticles() {
      for(let i=particles.length-1; i>=0; i--) {
        let p = particles[i]; p.x+=p.vx; p.y+=p.vy; p.life-=8;
        if(p.life<=0) particles.splice(i,1); else { fill(red(p.c), green(p.c), blue(p.c), p.life); noStroke(); ellipse(p.x, p.y, p.size, p.size); }
      }
    }
  </script>
</body>
</html>
