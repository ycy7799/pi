<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1.0, maximum-scale=1.0, user-scalable=no"/>
    <title>🍓 炫彩水果消消乐 - Pi 版</title>
    
    <script src="https://sdk.minepi.com/pi-sdk.js"></script>
    <style>
        * { margin:0; padding:0; box-sizing:border-box; user-select:none; }
        body {
            background: radial-gradient(circle at top, #2b1055, #12002f 70%);
            min-height:100vh;
            overflow:hidden;
            font-family:"Arial", sans-serif;
            display:flex;
            justify-content:center;
            align-items:center;
            color:white;
        }
        .stars { position:fixed; inset:0; overflow:hidden; z-index:0; }
        .star {
            position:absolute;
            background:white;
            border-radius:50%;
            animation:twinkle 3s infinite alternate;
        }
        @keyframes twinkle { from{opacity:0.3;} to{opacity:1;} }

        .game {
            position:relative; z-index:2;
            width:95vw; max-width:520px;
            padding:18px;
            border-radius:25px;
            backdrop-filter:blur(12px);
            background:rgba(255,255,255,0.08);
            border:2px solid rgba(255,255,255,0.15);
            box-shadow:0 0 45px rgba(255,120,255,0.3);
        }

        .title { text-align:center; font-size:36px; margin-bottom:8px; text-shadow:0 0 20px #ff89ff; }

        .topbar {
            display:flex; justify-content:space-between; align-items:center;
            margin-bottom:12px; font-size:18px;
        }

        .timer { color:#ff6b6b; font-weight:bold; }

        .board {
            display:grid;
            grid-template-columns:repeat(8,1fr);
            gap:5px;
            background:rgba(0,0,0,0.3);
            padding:10px;
            border-radius:20px;
            position:relative;
        }

        .cell {
            aspect-ratio:1/1;
            border-radius:15px;
            display:flex;
            justify-content:center;
            align-items:center;
            font-size:38px;
            cursor:pointer;
            transition:all 0.2s;
            background:rgba(255,255,255,0.1);
            box-shadow:inset 0 3px 8px rgba(255,255,255,0.25), 0 4px 12px rgba(0,0,0,0.4);
            position:relative;
            overflow:hidden;
        }

        .cell:active { transform:scale(0.88); }
        .selected { transform:scale(1.22); box-shadow:0 0 30px #fff, 0 0 55px #ff77ff; }

        .clearing {
            animation: electric 0.45s forwards;
        }
        @keyframes electric {
            0% { transform:scale(1); filter:brightness(1); }
            30% { transform:scale(1.3); filter:brightness(3) hue-rotate(50deg); }
            100% { transform:scale(0); filter:brightness(1); }
        }

        #connection { position:absolute; top:0; left:0; width:100%; height:100%; pointer-events:none; z-index:5; }

        .float-score {
            position:absolute;
            font-size:22px;
            font-weight:bold;
            color:#ffe66d;
            text-shadow:0 0 10px #ff0;
            pointer-events:none;
            z-index:10;
            animation:floatUp 1.2s forwards;
        }
        @keyframes floatUp {
            0% { transform:translateY(0); opacity:1; }
            100% { transform:translateY(-80px); opacity:0; }
        }

        .combo { text-align:center; margin:10px 0; color:#ffe66d; font-size:23px; height:32px; text-shadow:0 0 12px #ff0; }
        .pi-info { text-align:center; margin:12px 0 8px; font-size:14px; color:#a0ffcc; }
    </style>
</head>
<body>

<div class="stars" id="stars"></div>

<div class="game">
    <div class="title">🍓 炫彩水果消消乐 🍇</div>
    
    <div class="topbar">
        <div>⭐ 分数：<span id="score">0</span></div>
        <div class="timer">⏱️ <span id="time">60</span></div>
        <div>🔥 连击：<span id="comboNum">0</span></div>
    </div>

    <div class="board" id="board">
        <svg id="connection" width="100%" height="100%"></svg>
    </div>

    <div class="combo" id="comboText"></div>
    <div class="pi-info">🌟 Pi 生态专版 • 自动开始</div>
</div>

<script>
// ==================== 变量 ====================
const fruits = ["🍎","🍇","🍓","🍊","🥝","🍋","🍉","🫐"];
let grid = [], selected = null, score = 0, combo = 0, timeLeft = 60, gameRunning = true;
const rows = 8, cols = 8;
let timerInterval = null;

// DOM 元素
const boardEl = document.getElementById("board");
const scoreEl = document.getElementById("score");
const timeEl = document.getElementById("time");
const comboNumEl = document.getElementById("comboNum");
const comboTextEl = document.getElementById("comboText");

// ==================== 星空背景 ====================
const starsContainer = document.getElementById("stars");
for(let i = 0; i < 200; i++) {
    const s = document.createElement("div");
    s.className = "star";
    s.style.width = s.style.height = (Math.random()*3 + 1) + "px";
    s.style.left = Math.random()*100 + "%";
    s.style.top = Math.random()*100 + "%";
    s.style.animationDelay = Math.random()*4 + "s";
    starsContainer.appendChild(s);
}

// ==================== 音效 ====================
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
function playSound(freq, duration, type="triangle", vol=0.12) {
    try {
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = type; 
        osc.frequency.value = freq;
        gain.gain.value = vol;
        osc.connect(gain).connect(audioCtx.destination);
        osc.start();
        setTimeout(() => osc.stop(), duration);
    } catch(e) {}
}

// ==================== 游戏核心 ====================
function createBoard() {
    grid = [];
    for(let r = 0; r < rows; r++) {
        grid[r] = [];
        for(let c = 0; c < cols; c++) {
            grid[r][c] = fruits[Math.floor(Math.random() * fruits.length)];
        }
    }
    ensureHasMatches();
}

function ensureHasMatches() {
    let attempts = 0;
    while (attempts < 50) {
        if (checkMatches().length > 0) return;
        const r1 = Math.floor(Math.random()*rows), c1 = Math.floor(Math.random()*cols);
        const r2 = Math.floor(Math.random()*rows), c2 = Math.floor(Math.random()*cols);
        [grid[r1][c1], grid[r2][c2]] = [grid[r2][c2], grid[r1][c1]];
        attempts++;
    }
}

function renderBoard() {
    boardEl.innerHTML = '<svg id="connection" width="100%" height="100%"></svg>';
    
    for(let r = 0; r < rows; r++) {
        for(let c = 0; c < cols; c++) {
            const cell = document.createElement("div");
            cell.className = "cell";
            cell.innerHTML = grid[r][c] || "";
            if(selected && selected.r === r && selected.c === c) {
                cell.classList.add("selected");
            }
            cell.onclick = () => handleClick(r, c);
            boardEl.appendChild(cell);
        }
    }
}

function handleClick(r, c) {
    if(!gameRunning) return;

    if(!selected) {
        selected = {r, c};
        renderBoard();
        return;
    }

    const sr = selected.r, sc = selected.c;
    selected = null;

    if(Math.abs(r - sr) + Math.abs(c - sc) === 1) {
        swap(sr, sc, r, c);
        renderBoard();

        const matches = checkMatches();
        if(matches.length > 0) {
            combo++;
            comboNumEl.innerText = combo;
            processMatches();
        } else {
            swap(sr, sc, r, c); // 换回
            combo = 0;
            comboNumEl.innerText = "0";
            renderBoard();
        }
    } else {
        renderBoard();
    }
}

function swap(r1,c1,r2,c2) {
    [grid[r1][c1], grid[r2][c2]] = [grid[r2][c2], grid[r1][c1]];
}

function checkMatches() {
    let matches = new Set();
    // 横向
    for(let r = 0; r < rows; r++) {
        for(let c = 0; c < cols - 2; c++) {
            if(grid[r][c] && grid[r][c] === grid[r][c+1] && grid[r][c] === grid[r][c+2]) {
                matches.add(`${r},${c}`);
                matches.add(`${r},${c+1}`);
                matches.add(`${r},${c+2}`);
            }
        }
    }
    // 纵向
    for(let c = 0; c < cols; c++) {
        for(let r = 0; r < rows - 2; r++) {
            if(grid[r][c] && grid[r][c] === grid[r+1][c] && grid[r][c] === grid[r+2][c]) {
                matches.add(`${r},${c}`);
                matches.add(`${r+1},${c}`);
                matches.add(`${r+2},${c}`);
            }
        }
    }
    return Array.from(matches).map(s => s.split(',').map(Number));
}

function processMatches() {
    const matches = checkMatches();
    if(matches.length === 0) return;

    comboTextEl.innerHTML = `✨ 连消 ×${matches.length}`;

    const cells = boardEl.querySelectorAll('.cell');
    let addScore = matches.length * 22 * Math.max(1, combo);

    matches.forEach(([r,c]) => {
        const idx = r * cols + c;
        if(cells[idx]) {
            cells[idx].classList.add("clearing");
            createParticles(cells[idx], grid[r][c]);
        }
        grid[r][c] = null;
    });

    score += addScore;
    scoreEl.innerText = score;

    setTimeout(() => {
        dropFruits();
        fillFruits();
        renderBoard();
        setTimeout(processMatches, 280);
    }, 380);
}

function createParticles(el, fruit) {
    for(let i = 0; i < 25; i++) {
        const p = document.createElement("div");
        p.style.position = "absolute";
        p.style.left = "50%";
        p.style.top = "50%";
        p.style.fontSize = "22px";
        p.textContent = fruit;
        el.appendChild(p);
        
        const angle = Math.random() * Math.PI * 2;
        const dist = 55;
        p.animate([
            {transform: `translate(-50%, -50%) scale(1)`, opacity: 1},
            {transform: `translate(${Math.cos(angle)*dist}px, ${Math.sin(angle)*dist}px) scale(0.2)`, opacity: 0}
        ], {duration: 650});
        
        setTimeout(() => p.remove(), 900);
    }
}

function dropFruits() {
    for(let c = 0; c < cols; c++) {
        let write = rows - 1;
        for(let r = rows - 1; r >= 0; r--) {
            if(grid[r][c] !== null) {
                grid[write][c] = grid[r][c];
                write--;
            }
        }
    }
}

function fillFruits() {
    for(let r = 0; r < rows; r++) {
        for(let c = 0; c < cols; c++) {
            if(!grid[r][c]) {
                grid[r][c] = fruits[Math.floor(Math.random() * fruits.length)];
            }
        }
    }
}

function startTimer() {
    if(timerInterval) clearInterval(timerInterval);
    timeLeft = 60;
    timeEl.innerText = timeLeft;
    gameRunning = true;

    timerInterval = setInterval(() => {
        timeLeft--;
        timeEl.innerText = timeLeft;
        if(timeLeft <= 10) timeEl.style.color = "#ff0000";

        if(timeLeft <= 0) {
            clearInterval(timerInterval);
            gameRunning = false;
            alert(`🎉 时间到！\n你的最终得分是 ${score} 分！`);
        }
    }, 1000);
}

// ==================== 初始化游戏 ====================
window.onload = () => {
    createBoard();
    renderBoard();
    startTimer();
};
</script>
</body>
</html>
