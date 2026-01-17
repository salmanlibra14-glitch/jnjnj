<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Ludo Pro Matte - Final Fix</title>
    <script src="https://cdn.ably.com/lib/ably.min-1.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>
    <style>
        :root { --red: #c62828; --green: #2e7d32; --blue: #1565c0; --yellow: #f9a825; --bg: #0f172a; }
        * { box-sizing: border-box; touch-action: manipulation; }
        body { background: var(--bg); display: flex; flex-direction: column; align-items: center; justify-content: center; font-family: 'Segoe UI', sans-serif; margin: 0; padding: 0; color: white; height: 100vh; overflow: hidden; }
        .setup-screen { background: #1e293b; padding: 25px; border-radius: 20px; border: 1px solid #334155; width: 90%; max-width: 350px; text-align: center; position: absolute; top: 50%; left: 50%; transform: translate(-50%,-50%); z-index: 1000; }
        input, select, button { width: 100%; padding: 12px; margin: 8px 0; border-radius: 10px; border: 1px solid #334155; background: #0f172a; color: white; outline: none; }
        button { background: #10b981; border: none; font-weight: bold; cursor: pointer; }
        .game-container { display: none; flex-direction: column; align-items: center; width: 100%; max-width: 450px; gap: 10px; padding: 10px; }
        .side-panels { display: flex; justify-content: space-between; width: 100%; gap: 10px; }
        .p-ui { background: #1e293b; padding: 12px; border-radius: 12px; border: 2px solid #334155; width: 48%; text-align: center; opacity: 0.4; transition: 0.4s; }
        .joined { opacity: 1; }
        .active { opacity: 1; transform: scale(1.05); }
        #ui-Red.active { border-color: var(--red); box-shadow: 0 0 20px var(--red); }
        #ui-Green.active { border-color: var(--green); box-shadow: 0 0 20px var(--green); }
        #ui-Blue.active { border-color: var(--blue); box-shadow: 0 0 20px var(--blue); }
        #ui-Yellow.active { border-color: var(--yellow); box-shadow: 0 0 20px var(--yellow); }
        .dice { font-size: 35px; color: #ffffff; cursor: pointer; margin-bottom: 5px; }
        .p-name { font-size: 10px; text-transform: uppercase; font-weight: bold; color: #94a3b8; }
        .board-box { border: 4px solid #334155; background: #fff; border-radius: 8px; width: 94vw; height: 94vw; max-width: 420px; max-height: 420px; }
        canvas { display: block; width: 100% !important; height: 100% !important; }
        .chat-area { width: 100%; background: #1e293b; border-radius: 15px; height: 60px; margin-top: 5px; overflow: hidden; display: flex; align-items: center; justify-content: center; }
        #chat-msgs { color: #3b82f6; font-weight: bold; font-size: 14px; }
    </style>
</head>
<body>

<div class="setup-screen" id="ui-overlay">
    <h2 style="color: #3b82f6;">LUDO PRO</h2>
    <input type="text" id="playerName" placeholder="Apna Naam">
    <input type="text" id="roomId" placeholder="Room ID">
    <select id="pickColor">
        <option value="Red">RED 🔴</option>
        <option value="Green">GREEN 🟢</option>
        <option value="Yellow">YELLOW 🟡</option>
        <option value="Blue">BLUE 🔵</option>
    </select>
    <button onclick="enterGame()">JOIN GAME</button>
</div>

<div class="game-container" id="game-main">
    <div class="side-panels">
        <div id="ui-Red" class="p-ui"><div id="dice-Red" class="dice" onclick="handleRoll('Red')">⚀</div><div class="p-name" id="name-Red">RED</div></div>
        <div id="ui-Green" class="p-ui"><div id="dice-Green" class="dice" onclick="handleRoll('Green')">⚀</div><div class="p-name" id="name-Green">GREEN</div></div>
    </div>
    <div class="board-box"><canvas id="ludoCanvas" width="600" height="600"></canvas></div>
    <div class="side-panels">
        <div id="ui-Blue" class="p-ui"><div id="dice-Blue" class="dice" onclick="handleRoll('Blue')">⚀</div><div class="p-name" id="name-Blue">BLUE</div></div>
        <div id="ui-Yellow" class="p-ui"><div id="dice-Yellow" class="dice" onclick="handleRoll('Yellow')">⚀</div><div class="p-name" id="name-Yellow">YELLOW</div></div>
    </div>
    <div class="chat-area"><div id="chat-msgs">Ready to Play!</div></div>
    <button id="adminStartBtn" onclick="startGame()" style="display:none; background:#059669; margin-top:5px;">🚀 START GAME</button>
</div>

<script>
const ably = new Ably.Realtime('qaEL9g.I7WsLQ:EvnF3CZWGEtuU0Wdo_V7kraWU9hNRl-fXnFdABniQY4');
firebase.initializeApp({ databaseURL: "https://ludo-deep-matte-default-rtdb.firebaseio.com" });
const db = firebase.database();

let channel, currentRoom, myName, myColor, gameStarted = false;
const canvas = document.getElementById('ludoCanvas'); const ctx = canvas.getContext('2d');
const S = 40; const COLORS = { Red: '#c62828', Green: '#2e7d32', Blue: '#1565c0', Yellow: '#f9a825' };
const icons = ['⚀','⚁','⚂','⚃','⚄','⚅'];

const mainPath = [[1,6],[2,6],[3,6],[4,6],[5,6],[6,5],[6,4],[6,3],[6,2],[6,1],[6,0],[7,0],[8,0],[8,1],[8,2],[8,3],[8,4],[8,5],[9,6],[10,6],[11,6],[12,6],[13,6],[14,6],[14,7],[14,8],[13,8],[12,8],[11,8],[10,8],[9,8],[8,9],[8,10],[8,11],[8,12],[8,13],[8,14],[7,14],[6,14],[6,13],[6,12],[6,11],[6,10],[6,9],[5,8],[4,8],[3,8],[2,8],[1,8],[0,8],[0,7],[0,6]];
const homePaths = { Red: [[1,7],[2,7],[3,7],[4,7],[5,7]], Green: [[7,1],[7,2],[7,3],[7,4],[7,5]], Yellow: [[13,7],[12,7],[11,7],[10,7],[9,7]], Blue: [[7,13],[7,12],[7,11],[7,10],[7,9]] };
const stars = [[1,6],[2,8],[6,1],[8,2],[13,8],[12,6],[8,13],[6,12]];

let tokens = {
    Red: { s: 0, p: [{pos:-1,hx:1.5,hy:1.5},{pos:-1,hx:4.5,hy:1.5},{pos:-1,hx:1.5,hy:4.5},{pos:-1,hx:4.5,hy:4.5}] },
    Green: { s: 13, p: [{pos:-1,hx:10.5,hy:1.5},{pos:-1,hx:13.5,hy:1.5},{pos:-1,hx:10.5,hy:4.5},{pos:-1,hx:13.5,hy:4.5}] },
    Yellow: { s: 26, p: [{pos:-1,hx:10.5,hy:10.5},{pos:-1,hx:13.5,hy:10.5},{pos:-1,hx:10.5,hy:13.5},{pos:-1,hx:13.5,hy:13.5}] },
    Blue: { s: 39, p: [{pos:-1,hx:1.5,hy:10.5},{pos:-1,hx:4.5,hy:10.5},{pos:-1,hx:1.5,hy:13.5},{pos:-1,hx:4.5,hy:13.5}] }
};
let dVal = 1, turn = 'Red', rolled = false;

function drawBoard() {
    ctx.fillStyle = "#fff"; ctx.fillRect(0,0,600,600);
    const box = (x,y,c) => { 
        ctx.fillStyle=c; ctx.fillRect(x*S,y*S,6*S,6*S);
        ctx.fillStyle="rgba(255,255,255,0.4)"; 
        [[1.5,1.5],[4.5,1.5],[1.5,4.5],[4.5,4.5]].forEach(p => { ctx.beginPath(); ctx.arc((x+p[0])*S,(y+p[1])*S,25,0,7); ctx.fill(); });
    };
    box(0,0,COLORS.Red); box(9,0,COLORS.Green); box(0,9,COLORS.Blue); box(9,9,COLORS.Yellow);
    ctx.fillStyle = COLORS.Red; ctx.fillRect(1*S, 7*S, 5*S, S); 
    ctx.fillStyle = COLORS.Green; ctx.fillRect(7*S, 1*S, S, 5*S);
    ctx.fillStyle = COLORS.Yellow; ctx.fillRect(9*S, 7*S, 5*S, S);
    ctx.fillStyle = COLORS.Blue; ctx.fillRect(7*S, 9*S, S, 5*S);
    const tri = (x1,y1,x2,y2,c) => { ctx.fillStyle=c; ctx.beginPath(); ctx.moveTo(300,300); ctx.lineTo(x1*S,y1*S); ctx.lineTo(x2*S,y2*S); ctx.fill(); };
    tri(6,6,6,9,COLORS.Red); tri(6,6,9,6,COLORS.Green); tri(9,6,9,9,COLORS.Yellow); tri(6,9,9,9,COLORS.Blue);
    ctx.strokeStyle = "#cbd5e1"; ctx.lineWidth = 1;
    for(let i=0; i<15; i++) for(let j=0; j<15; j++) { if(!((i<6&&j<6)||(i>8&&j<6)||(i<6&&j>8)||(i>8&&j>8))) ctx.strokeRect(i*S,j*S,S,S); }
    ctx.fillStyle = "#64748b"; ctx.font = "20px Arial";
    stars.forEach(s => ctx.fillText("★", s[0]*S+10, s[1]*S+28));
}

function enterGame() {
    myName = document.getElementById('playerName').value;
    currentRoom = document.getElementById('roomId').value;
    myColor = document.getElementById('pickColor').value;
    if(!myName || !currentRoom) return;
    channel = ably.channels.get('ludo-' + currentRoom);
    db.ref('rooms/'+currentRoom+'/players/'+myColor).set(myName);
    document.getElementById('ui-overlay').style.display = 'none';
    document.getElementById('game-main').style.display = 'flex';
    if(myColor === 'Red') document.getElementById('adminStartBtn').style.display = 'block';
    channel.subscribe('move', (msg) => {
        const d = msg.data; tokens = d.tokens; turn = d.turn; dVal = d.dVal; rolled = d.rolled; updateUI();
    });
    db.ref('rooms/'+currentRoom+'/players').on('value', snap => {
        const ps = snap.val() || {};
        for(let c in ps) { document.getElementById('name-'+c).innerText = ps[c]; document.getElementById('ui-'+c).classList.add('joined'); }
    });
    db.ref('rooms/'+currentRoom+'/gameStarted').on('value', s => { if(s.val()) gameStarted = true; document.getElementById('adminStartBtn').style.display = 'none'; });
}

function handleRoll(c) {
    if(!gameStarted || c !== myColor || c !== turn || rolled) return;
    dVal = Math.floor(Math.random()*6)+1;
    rolled = true;
    
    // Check if player can move
    const canMove = tokens[turn].p.some(tk => (tk.pos === -1 && dVal === 6) || (tk.pos >= 0 && tk.pos + dVal <= 56));
    
    if(!canMove) {
        setTimeout(() => {
            rolled = false;
            nextTurn();
            sync();
        }, 1000);
    } else {
        sync();
    }
}

function nextTurn() {
    const order = ['Red', 'Green', 'Yellow', 'Blue'];
    turn = order[(order.indexOf(turn) + 1) % 4];
}

function updateUI() {
    document.querySelectorAll('.p-ui').forEach(u => u.classList.remove('active'));
    document.getElementById('ui-'+turn).classList.add('active');
    document.getElementById('dice-'+turn).innerText = icons[dVal-1];
    document.getElementById('chat-msgs').innerText = turn + "'s Turn (Rolled: " + dVal + ")";
    draw();
}

function draw() {
    ctx.clearRect(0,0,600,600); drawBoard();
    for(let c in tokens) {
        tokens[c].p.forEach(tk => {
            let x, y;
            if(tk.pos===-1){ x=tk.hx*S; y=tk.hy*S; }
            else if(tk.pos < 51){ let idx = (tokens[c].s + tk.pos) % 52; x = mainPath[idx][0]*S + 20; y = mainPath[idx][1]*S + 20; }
            else if(tk.pos < 56) { let idx = tk.pos - 51; x = homePaths[c][idx][0]*S + 20; y = homePaths[c][idx][1]*S + 20; }
            else { x=300; y=300; }
            ctx.fillStyle = COLORS[c]; ctx.beginPath(); ctx.arc(x,y,16,0,7); ctx.fill();
            ctx.strokeStyle = "white"; ctx.lineWidth = 2; ctx.stroke();
        });
    }
}

function sync() { channel.publish('move', {tokens, turn, dVal, rolled}); }
function startGame() { db.ref('rooms/'+currentRoom+'/gameStarted').set(true); sync(); }

canvas.addEventListener('touchstart', e => {
    e.preventDefault();
    if(!gameStarted || !rolled || turn !== myColor) return;
    const r = canvas.getBoundingClientRect(); const t = e.touches[0];
    const mx = (t.clientX - r.left) * (600/r.width); const my = (t.clientY - r.top) * (600/r.height);
    
    tokens[turn].p.forEach((tk, i) => {
        let tx, ty;
        if(tk.pos===-1){ tx=tk.hx*S; ty=tk.hy*S; }
        else if(tk.pos < 51){ let idx = (tokens[turn].s + tk.pos) % 52; tx = mainPath[idx][0]*S + 20; ty = mainPath[idx][1]*S + 20; }
        else if(tk.pos < 56) { let idx = tk.pos - 51; tx = homePaths[turn][idx][0]*S + 20; ty = homePaths[turn][idx][1]*S + 20; }
        else { tx=300; ty=300; }
        
        if(Math.hypot(mx-tx, my-ty) < 40){
            let moved = false;
            if(dVal===6 && tk.pos===-1) { tk.pos=0; rolled=false; moved=true; }
            else if(tk.pos>=0 && tk.pos+dVal<=56) { 
                tk.pos+=dVal; 
                moved = true;
                if(dVal !== 6 && tk.pos !== 56) nextTurn();
                rolled = false;
            }
            if(moved) sync();
        }
    });
}, {passive: false});

drawBoard();
</script>
</body>
</html>
