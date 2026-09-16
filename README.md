<!DOCTYPE html>
<html lang="he" dir="rtl">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Blackjack Club — Multiplayer</title>
<style>
*{box-sizing:border-box}
body{margin:0;font-family:Arial,sans-serif;background:radial-gradient(circle at top,#16452f,#07130e 60%);color:#fff;min-height:100vh}
.wrap{max-width:1150px;margin:auto;padding:24px}
.top{display:flex;justify-content:space-between;align-items:center;gap:15px;margin-bottom:18px}
h1{margin:0;font-size:30px}.sub{color:#b9d7c5}
.card{background:rgba(8,20,14,.88);border:1px solid #315a43;border-radius:20px;padding:20px;box-shadow:0 15px 50px #0006;margin-bottom:16px}
.grid{display:grid;grid-template-columns:1.1fr .9fr;gap:16px}
@media(max-width:800px){.grid{grid-template-columns:1fr}.top{align-items:flex-start;flex-direction:column}}
label{display:block;margin:10px 0 6px;color:#cfe7d7}
input{width:100%;padding:13px;border-radius:11px;border:1px solid #466b54;background:#10251a;color:#fff;font-size:16px}
button{border:0;border-radius:12px;padding:12px 17px;font-weight:700;cursor:pointer;background:#e7bd55;color:#171208;margin:4px}
button:hover{filter:brightness(1.08)}
button.secondary{background:#234e38;color:#fff}
button:disabled{opacity:.45;cursor:not-allowed}
.row{display:flex;flex-wrap:wrap;gap:7px;align-items:center}
.hidden{display:none!important}
.room{font-size:25px;letter-spacing:4px;background:#0a1b12;border:1px dashed #6e9b7e;border-radius:12px;padding:10px 14px;display:inline-block}
.status{padding:9px 12px;border-radius:10px;background:#183b29;color:#c9efd5;margin-top:10px}
.players{display:grid;gap:8px}
.player{display:flex;justify-content:space-between;padding:11px;border-radius:10px;background:#10251a;border:1px solid #284b37}
.table{min-height:390px;border-radius:28px;padding:25px;background:radial-gradient(ellipse,#197044,#0a432a 60%,#06301e);border:8px solid #553a22;box-shadow:inset 0 0 60px #0008,0 20px 60px #0007}
.hand{display:flex;justify-content:center;gap:8px;min-height:92px;flex-wrap:wrap}
.playing-card{width:58px;height:82px;border-radius:8px;background:#fff;color:#111;display:flex;flex-direction:column;justify-content:space-between;padding:6px;font-weight:800;box-shadow:0 5px 10px #0005}
.red{color:#c62828}
.back{background:repeating-linear-gradient(45deg,#1f4f9b,#1f4f9b 5px,#fff 5px,#fff 7px);color:transparent}
.center{text-align:center}.total{font-size:18px;margin:7px}
.actions{display:flex;justify-content:center;flex-wrap:wrap;margin-top:12px}
.small{font-size:13px;color:#9eb9a8}
.log{height:150px;overflow:auto;background:#07150d;border-radius:10px;padding:10px;font-size:13px}
hr{border:0;border-top:1px solid #315a43;margin:16px 0}
.ready{color:#9ff0b4;font-weight:700}
</style>
</head>
<body>
<div class="wrap">
  <div class="top">
    <div>
      <h1>♠ Blackjack Club</h1>
      <div class="sub">משחק קלפים רב־משתתפים — ללא כסף, הימורים או צ'יפים</div>
    </div>
    <div id="connection" class="status">מתחבר...</div>
  </div>

  <div id="setup" class="card">
    <h2>פתיחת שולחן חדש</h2>
    <label>השם שלך</label>
    <input id="hostName" maxlength="18" placeholder="למשל Amit">
    <button onclick="createRoom()">♠ צור שולחן ודילר</button>

    <hr>

    <h2>הצטרפות לשולחן</h2>
    <label>שם</label>
    <input id="joinName" maxlength="18" placeholder="השם שלך">
    <label>קוד חדר</label>
    <input id="roomCode" maxlength="8" placeholder="למשל A7K2P9">
    <button class="secondary" onclick="joinRoom()">הצטרף לחדר</button>
    <p class="small">שלחו לחברים את קוד החדר. מי שפתח את החדר הוא הדילר.</p>
  </div>

  <div id="game" class="hidden">
    <div class="grid">
      <div>
        <div class="card">
          <div class="row"><b>קוד חדר:</b><span id="code" class="room"></span><button class="secondary" onclick="copyRoom()">העתק</button></div>
          <div id="role" class="status"></div>
          <div id="gameStatus" class="status">ממתינים...</div>
        </div>
        <div class="card">
          <h3>שחקנים</h3>
          <div id="players" class="players"></div>
        </div>
      </div>

      <div class="card">
        <h3>לוג</h3>
        <div id="log" class="log"></div>
      </div>
    </div>

    <div class="table">
      <div class="center">
        <h2>בית / דילר</h2>
        <div id="dealerHand" class="hand"></div>
        <div id="dealerTotal" class="total"></div>
      </div>

      <hr>

      <div class="center">
        <h2>השחקן שלך</h2>
        <div id="myHand" class="hand"></div>
        <div id="myTotal" class="total"></div>
      </div>

      <div id="actions" class="actions"></div>
    </div>
  </div>
</div>

<script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
<script>
let peer, host=false, myId='', roomId='', myName='', conn=null, state=null, hostConns={};

const suits=['♠','♥','♦','♣'];
const ranks=['A','2','3','4','5','6','7','8','9','10','J','Q','K'];

function log(t){
  const e=document.getElementById('log');
  e.innerHTML='<div>'+new Date().toLocaleTimeString('he-IL')+' — '+esc(t)+'</div>'+e.innerHTML;
}

function rid(){
  return Math.random().toString(36).slice(2,8).toUpperCase();
}

function createRoom(){
  myName=document.getElementById('hostName').value.trim()||'הדילר';
  roomId=rid();
  host=true;

  state={
    phase:'lobby',
    deck:[],
    dealer:[],
    players:{},
    turn:null,
    round:0
  };

  peer=new Peer('bj-'+roomId,{debug:0});

  peer.on('open',()=>{
    myId=peer.id;
    state.players[myId]={
      name:myName,
      hand:[],
      done:false,
      ready:false
    };
    showGame();
    log('החדר נוצר. אתה הדילר.');
  });

  peer.on('connection',c=>{
    hostConns[c.peer]=c;

    c.on('open',()=>{
      state.players[c.peer]={
        name:'שחקן',
        hand:[],
        done:false,
        ready:false
      };
      c.send({type:'state',state});
      broadcast();
      log('שחקן חדש הצטרף');
    });

    c.on('close',()=>{
      delete hostConns[c.peer];
      delete state.players[c.peer];
      if(state.turn===c.peer) nextTurn();
      broadcast();
    });

    c.on('data',m=>handleHost(c,m));
  });

  peer.on('error',()=>alert('אירעה בעיית חיבור. נסה ליצור חדר חדש.'));
}

function joinRoom(){
  myName=document.getElementById('joinName').value.trim()||'שחקן';
  roomId=document.getElementById('roomCode').value.trim().toUpperCase();

  if(!roomId){
    alert('הכנס קוד חדר');
    return;
  }

  host=false;
  peer=new Peer({debug:0});

  peer.on('open',id=>{
    myId=id;
    conn=peer.connect('bj-'+roomId);

    conn.on('open',()=>{
      conn.send({type:'join',name:myName});
    });

    conn.on('data',m=>handleClient(m));
  });

  peer.on('error',()=>{
    alert('לא ניתן להתחבר לחדר. בדוק את הקוד ונסה שוב.');
  });
}

function showGame(){
  document.getElementById('setup').classList.add('hidden');
  document.getElementById('game').classList.remove('hidden');
  document.getElementById('code').textContent=roomId;
  document.getElementById('connection').textContent='מחובר';
  render();
}

function handleHost(c,m){
  if(m.type==='join'){
    state.players[c.peer]={
      name:String(m.name||'שחקן').slice(0,18),
      hand:[],
      done:false,
      ready:false
    };
    c.send({type:'state',state});
    broadcast();
    return;
  }

  if(m.type==='ready'){
    if(state.phase==='lobby'){
      const p=state.players[c.peer];
      if(p){
        p.ready=true;
        broadcast();
      }
    }
    return;
  }

  if(m.type==='startRound'){
    if(state.phase==='lobby' || state.phase==='finished'){
      startRound();
    }
    return;
  }

  if(m.type==='action'){
    if(state.phase==='playing' && state.turn===c.peer){
      doAction(c.peer,m.action);
    }
  }
}

function handleClient(m){
  if(m.type==='state'){
    state=m.state;
    showGame();
  }
  if(m.type==='room'){
    state=m.state;
    render();
  }
}

function broadcast(){
  Object.values(hostConns).forEach(c=>{
    try{
      if(c.open)c.send({type:'room',state});
    }catch(e){}
  });
  render();
}

function deck(){
  let d=[];
  for(const s of suits){
    for(const r of ranks)d.push({s,r});
  }
  return d.sort(()=>Math.random()-.5);
}

function val(hand){
  let n=0,a=0;
  hand.forEach(c=>{
    if(c.r==='A'){n+=11;a++}
    else n+=['K','Q','J'].includes(c.r)?10:+c.r;
  });
  while(n>21&&a--)n-=10;
  return n;
}

function allReady(){
  const ps=Object.values(state.players);
  return ps.length>0 && ps.every(p=>p.ready);
}

function startRound(){
  state.round++;
  state.phase='playing';
  state.deck=deck();
  state.dealer=[state.deck.pop(),state.deck.pop()];

  const ids=Object.keys(state.players);

  ids.forEach(id=>{
    const p=state.players[id];
    p.hand=[state.deck.pop(),state.deck.pop()];
    p.done=false;
    p.ready=false;
  });

  state.turn=ids[0];
  broadcast();
  checkTurn();
}

function doAction(id,a){
  const p=state.players[id];
  if(!p)return;

  if(a==='hit'){
    p.hand.push(state.deck.pop());
    if(val(p.hand)>=21)p.done=true;
  }

  if(a==='stand'){
    p.done=true;
  }

  nextTurn();
}

function nextTurn(){
  const ids=Object.keys(state.players);
  const i=ids.indexOf(state.turn);
  const next=ids.slice(i+1).find(id=>!state.players[id].done);

  if(next){
    state.turn=next;
  }else{
    state.turn=null;
    dealerPlay();
  }

  broadcast();
}

function checkTurn(){
  const p=state.players[state.turn];
  if(p && val(p.hand)>=21)nextTurn();
}

function dealerPlay(){
  state.phase='dealer';
  broadcast();

  while(val(state.dealer)<17){
    state.dealer.push(state.deck.pop());
  }

  state.phase='finished';
  Object.values(state.players).forEach(p=>p.done=true);
  broadcast();
}

function render(){
  if(!state)return;

  showGame();

  document.getElementById('role').textContent=
    host ? '👑 אתה הדילר (הבית)' : '🎮 אתה שחקן';

  let status='ממתינים...';

  if(state.phase==='lobby'){
    status=allReady() ? 'כולם מוכנים — הדילר יכול להתחיל' : 'מסמנים "מוכן" ומחכים לדילר';
  }else if(state.phase==='playing'){
    const turnPlayer=state.players[state.turn];
    status=turnPlayer ? `התור של ${turnPlayer.name}` : 'השחקנים משחקים';
  }else if(state.phase==='dealer'){
    status='הדילר פועל...';
  }else if(state.phase==='finished'){
    status='הסיבוב הסתיים';
  }

  document.getElementById('gameStatus').textContent=status;

  document.getElementById('players').innerHTML=
    Object.entries(state.players).map(([id,p])=>{
      const ready=p.ready && state.phase==='lobby';
      const turn=id===state.turn && state.phase==='playing';
      return `<div class="player">
        <span>${id===myId?'⭐ ':''}${esc(p.name)} ${id===myId?'(אתה)':''}</span>
        <span>${ready?'<span class="ready">✓ מוכן</span>':turn?'🎯 תור':''}</span>
      </div>`;
    }).join('');

  const me=state.players[myId];
  if(!me)return;

  document.getElementById('dealerHand').innerHTML=
    state.dealer.map((c,i)=>card(c,(i===1&&state.phase==='playing'))).join('');

  document.getElementById('dealerTotal').textContent=
    state.phase==='playing'
      ? 'קלף נסתר'
      : (state.dealer.length ? 'סה״כ: '+val(state.dealer) : '');

  document.getElementById('myHand').innerHTML=
    me.hand.map(c=>card(c,false)).join('');

  document.getElementById('myTotal').textContent=
    me.hand.length ? 'סה״כ: '+val(me.hand) : '';

  const a=document.getElementById('actions');
  a.innerHTML='';

  if(!host && state.phase==='lobby' && !me.ready){
    a.innerHTML='<button onclick="readyUp()">✓ מוכן</button>';
  }

  if(!host && state.phase==='playing' && state.turn===myId && !me.done){
    a.innerHTML=
      '<button onclick="act(\\'hit\\')">קלף</button>'+
      '<button onclick="act(\\'stand\\')" class="secondary">עמוד</button>';
  }

  if(host && state.phase==='lobby'){
    a.innerHTML=
      '<button '+(allReady()?'':'disabled')+' onclick="startHostRound()">▶ התחל סיבוב</button>';
  }

  if(host && state.phase==='finished'){
    a.innerHTML='<button onclick="startHostRound()">🔄 סיבוב חדש</button>';
  }

  if(!host && state.phase==='finished'){
    a.innerHTML='<span class="small">ממתינים לדילר לסיבוב הבא</span>';
  }
}

function readyUp(){
  if(conn)conn.send({type:'ready'});
}

function startHostRound(){
  if(host)startRound();
}

function card(c,hidden){
  if(hidden)return '<div class="playing-card back">?</div>';
  const red=(c.s==='♥'||c.s==='♦')?'red':'';
  return `<div class="playing-card ${red}"><span>${c.r}</span><span>${c.s}</span></div>`;
}

function act(x){
  if(conn)conn.send({type:'action',action:x});
}

function copyRoom(){
  navigator.clipboard?.writeText(roomId);
  alert('קוד החדר הועתק');
}

function esc(s){
  return String(s).replace(/[&<>"']/g,x=>({
    '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#039;'
  }[x]));
}
</script>
</body>
</html>
