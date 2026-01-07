<!doctype html>
<html lang="zh">
<head>
<meta charset="utf-8"/><meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>斗地主（简化 单机 1v2）</title>
<style>
  body{margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,"PingFang SC","Microsoft YaHei";background:#0b1020;color:#e8ecff}
  .wrap{max-width:1100px;margin:0 auto;padding:16px;display:grid;grid-template-columns:1fr 320px;gap:12px}
  .panel{background:rgba(255,255,255,.05);border:1px solid rgba(255,255,255,.10);border-radius:14px;padding:12px}
  h1{font-size:18px;margin:0 0 8px}
  .row{display:flex;gap:10px;flex-wrap:wrap;align-items:center}
  .btn{cursor:pointer;border:1px solid rgba(255,255,255,.16);background:rgba(255,255,255,.06);color:#e8ecff;padding:10px 12px;border-radius:12px;font-weight:800}
  .btn.primary{background:rgba(122,162,247,.18);border-color:rgba(122,162,247,.45)}
  .btn:disabled{opacity:.45;cursor:not-allowed}
  .pill{padding:6px 10px;border-radius:999px;background:rgba(255,255,255,.06);border:1px solid rgba(255,255,255,.08);font-size:12px;color:#a9b1d6}
  .cards{display:flex;gap:6px;flex-wrap:wrap}
  .card{min-width:42px;padding:8px 10px;border-radius:12px;border:1px solid rgba(255,255,255,.10);
    background:linear-gradient(180deg,rgba(255,255,255,.08),rgba(255,255,255,.03));cursor:pointer;font-weight:900}
  .card.sel{outline:2px solid rgba(122,162,247,.55)}
  .log{height:360px;overflow:auto;background:rgba(0,0,0,.25);border:1px solid rgba(255,255,255,.08);border-radius:12px;padding:10px}
  .log p{margin:0 0 8px;color:#a9b1d6;font-size:12px}
  .muted{color:#a9b1d6;font-size:12px;line-height:1.35}
</style>
</head>
<body>
<div class="wrap">
  <div class="panel">
    <h1>斗地主（简化 单机 1v2）</h1>
    <div class="row">
      <span class="pill" id="rolePill">你：未定</span>
      <span class="pill" id="turnPill">回合：—</span>
      <button class="btn" id="restart">重新开始</button>
    </div>

    <div class="panel" style="margin-top:10px">
      <div class="row" style="justify-content:space-between">
        <b>桌面出牌</b>
        <span class="pill" id="lastInfo">无</span>
      </div>
      <div class="cards" id="lastCards"></div>
    </div>

    <div class="panel" style="margin-top:10px">
      <div class="row" style="justify-content:space-between">
        <b>你的手牌</b>
        <span class="pill">手牌: <span id="myN"></span></span>
        <span class="pill">AI1: <span id="a1N"></span> 张</span>
        <span class="pill">AI2: <span id="a2N"></span> 张</span>
      </div>
      <div class="cards" id="myHand"></div>
      <div class="row" style="margin-top:10px">
        <button class="btn primary" id="btnBid1">叫地主</button>
        <button class="btn" id="btnBid0">不叫</button>
        <button class="btn primary" id="btnPlay" disabled>出牌</button>
        <button class="btn" id="btnPass" disabled>不出</button>
      </div>
      <div class="muted" style="margin-top:8px">
        覆盖牌型：单/对/三/三带一/顺子(≥5)/炸弹/王炸。AI 是基础跟牌，够玩但不算聪明。
      </div>
    </div>
  </div>

  <div class="panel">
    <b>行动日志</b>
    <div class="log" id="log"></div>
  </div>
</div>

<script>
(() => {
  const $ = s => document.querySelector(s);
  const logEl = $("#log");
  const RANKS = ["3","4","5","6","7","8","9","10","J","Q","K","A","2","小王","大王"];
  const rankValue = r => RANKS.indexOf(r) + 3; // 3..17
  function pushLog(t){ const p=document.createElement("p"); p.textContent=t; logEl.prepend(p); }
  function shuffle(a){ for(let i=a.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [a[i],a[j]]=[a[j],a[i]]; } return a; }

  function buildDeck(){
    const deck=[];
    const base = RANKS.slice(0,13); // 3..2
    for(const r of base){
      for(let s=0;s<4;s++) deck.push({r, v:rankValue(r), id:r+"-"+s});
    }
    deck.push({r:"小王", v:rankValue("小王"), id:"SJ"});
    deck.push({r:"大王", v:rankValue("大王"), id:"BJ"});
    return shuffle(deck);
  }

  // 牌型解析
  function countMap(cards){
    const m=new Map();
    for(const c of cards) m.set(c.v, (m.get(c.v)||0)+1);
    return m;
  }
  function isStraight(vals){
    const v=[...vals].sort((a,b)=>a-b);
    if(v.length<5) return false;
    if(v[v.length-1] >= rankValue("2")) return false; // 2/王不能进顺子
    for(let i=1;i<v.length;i++) if(v[i]!==v[i-1]+1) return false;
    return true;
  }

  function analyze(cards){
    // cards: array of card objects
    if(!cards || cards.length===0) return null;
    const n=cards.length;
    const vals=cards.map(c=>c.v);
    const m=countMap(cards);
    const uniq=[...m.keys()].sort((a,b)=>a-b);
    const counts=[...m.values()].sort((a,b)=>b-a);

    // 王炸
    if(n===2 && vals.includes(rankValue("小王")) && vals.includes(rankValue("大王")))
      return {type:"ROCKET", main: 999, len:2};

    // 炸弹
    if(n===4 && m.size===1) return {type:"BOMB", main: uniq[0], len:4};

    // 单/对/三
    if(n===1) return {type:"SINGLE", main: vals[0], len:1};
    if(n===2 && m.size===1) return {type:"PAIR", main: uniq[0], len:2};
    if(n===3 && m.size===1) return {type:"TRIPLE", main: uniq[0], len:3};

    // 三带一
    if(n===4 && m.size===2 && counts[0]===3) {
      const main = [...m.entries()].find(([,c])=>c===3)[0];
      return {type:"TRIPLE1", main, len:4};
    }

    // 顺子（简化：全是单牌且连续）
    if(n>=5 && m.size===n && isStraight(uniq)) return {type:"STRAIGHT", main: uniq[uniq.length-1], len:n};

    return {type:"INVALID"};
  }

  function canBeat(play, last){
    if(!play || play.type==="INVALID") return false;
    if(!last) return true; // 任意先手
    if(play.type==="ROCKET") return true;
    if(play.type==="BOMB" && last.type!=="BOMB" && last.type!=="ROCKET") return true;
    if(play.type!==last.type) return false;
    if(play.len!==last.len) return false;
    return play.main > last.main;
  }

  function sortHand(hand){
    hand.sort((a,b)=>a.v-b.v);
  }

  // AI：找到“能压过 last 的最小出牌”，否则不出；若 last 为空，出最小单牌
  function aiPick(hand, last){
    sortHand(hand);
    const byVal = new Map();
    for(const c of hand){
      if(!byVal.has(c.v)) byVal.set(c.v, []);
      byVal.get(c.v).push(c);
    }
    const vals=[...byVal.keys()].sort((a,b)=>a-b);

    function pickSingle(minV){
      for(const v of vals) if(v>minV) return [byVal.get(v)[0]];
      return null;
    }
    function pickPair(minV){
      for(const v of vals) if(v>minV && byVal.get(v).length>=2) return byVal.get(v).slice(0,2);
      return null;
    }
    function pickTriple(minV){
      for(const v of vals) if(v>minV && byVal.get(v).length>=3) return byVal.get(v).slice(0,3);
      return null;
    }
    function pickTriple1(minV){
      for(const v of vals){
        if(v>minV && byVal.get(v).length>=3){
          // 找一张带牌（不与三同点）
          for(const u of vals){
            if(u!==v && byVal.get(u).length>=1){
              return [...byVal.get(v).slice(0,3), byVal.get(u)[0]];
            }
          }
        }
      }
      return null;
    }
    function pickStraight(len, minMain){
      // 找长度=len 的最小顺子，末端>minMain
      // 只在 3..A
      const cand = vals.filter(v => v < rankValue("2")); // 3..A
      for(let i=0;i<=cand.length-len;i++){
        const slice=cand.slice(i,i+len);
        if(!isStraight(slice)) continue;
        const main=slice[slice.length-1];
        if(main>minMain){
          // 取对应牌
          return slice.map(v=>byVal.get(v)[0]);
        }
      }
      return null;
    }
    function pickBomb(minV){
      for(const v of vals) if(v>minV && byVal.get(v).length>=4) return byVal.get(v).slice(0,4);
      return null;
    }
    function hasRocket(){
      const sj = hand.find(c=>c.r==="小王");
      const bj = hand.find(c=>c.r==="大王");
      return (sj && bj) ? [sj,bj] : null;
    }

    if(!last){
      // 先手：出最小单牌
      return [hand[0]];
    }

    // 同牌型应对
    const t = last.type;
    let play=null;
    if(t==="SINGLE") play = pickSingle(last.main);
    else if(t==="PAIR") play = pickPair(last.main);
    else if(t==="TRIPLE") play = pickTriple(last.main);
    else if(t==="TRIPLE1") play = pickTriple1(last.main);
    else if(t==="STRAIGHT") play = pickStraight(last.len, last.main);
    else if(t==="BOMB") play = pickBomb(last.main);

    if(play) return play;

    // 炸弹压制
    const bomb = pickBomb(-1);
    if(bomb){
      const a=analyze(bomb);
      if(canBeat(a,last)) return bomb;
    }
    const rocket = hasRocket();
    if(rocket){
      const a=analyze(rocket);
      if(canBeat(a,last)) return rocket;
    }
    return null;
  }

  const state = {
    deck: [],
    bottom: [],
    players: [
      { name:"你", hand:[], role:"?" },
      { name:"AI1", hand:[], role:"?" },
      { name:"AI2", hand:[], role:"?" },
    ],
    landlord: -1,
    turn: 0,
    lastPlay: null,
    lastPlayer: -1,
    selected: new Set(),
    phase: "BID", // BID or PLAY
    bidTurn: 0,
    bidCalled: false,
  };

  function render(){
    $("#rolePill").textContent = `你：${state.players[0].role==="L"?"地主":"农民"}`;
    $("#turnPill").textContent = `回合：${state.players[state.turn].name}`;
    $("#myN").textContent = state.players[0].hand.length;
    $("#a1N").textContent = state.players[1].hand.length;
    $("#a2N").textContent = state.players[2].hand.length;

    $("#btnBid1").disabled = state.phase!=="BID" || state.turn!==0;
    $("#btnBid0").disabled = state.phase!=="BID" || state.turn!==0;

    $("#btnPlay").disabled = !(state.phase==="PLAY" && state.turn===0 && state.selected.size>0 && isPlayableSelection());
    $("#btnPass").disabled = !(state.phase==="PLAY" && state.turn===0 && state.lastPlay && state.lastPlayer!==0);

    const lastInfo = $("#lastInfo");
    const lastCards = $("#lastCards");
    lastCards.innerHTML="";
    if(!state.lastPlay){
      lastInfo.textContent="无";
    }else{
      lastInfo.textContent = `${state.players[state.lastPlayer].name}：${formatPlay(state.lastPlay.cards)}（${state.lastPlay.an.type}）`;
      for(const c of state.lastPlay.cards){
        const el=document.createElement("div"); el.className="card"; el.textContent=label(c);
        lastCards.appendChild(el);
      }
    }

    const myHand=$("#myHand");
    myHand.innerHTML="";
    state.players[0].hand.forEach((c,idx)=>{
      const el=document.createElement("div");
      el.className="card"+(state.selected.has(idx)?" sel":"");
      el.textContent=label(c);
      el.onclick=()=>{
        if(state.turn!==0 || state.phase!=="PLAY") return;
        if(state.selected.has(idx)) state.selected.delete(idx); else state.selected.add(idx);
        render();
      };
      myHand.appendChild(el);
    });
  }

  function label(c){
    return c.r;
  }
  function formatPlay(cards){
    return cards.map(label).join(" ");
  }

  function selectionCards(){
    const hand=state.players[0].hand;
    return [...state.selected].sort((a,b)=>a-b).map(i=>hand[i]);
  }

  function isPlayableSelection(){
    const cards=selectionCards();
    const an=analyze(cards);
    if(!an || an.type==="INVALID") return false;
    if(!state.lastPlay || state.lastPlayer===0){
      // 若桌面无牌或你是上家（你刚出完，桌面被你控住），可任意有效牌型
      return true;
    }
    return canBeat(an, state.lastPlay.an);
  }

  function removeSelectedFromHand(){
    const idxs=[...state.selected].sort((a,b)=>b-a);
    const hand=state.players[0].hand;
    const cards=[];
    for(const i of idxs) cards.push(hand.splice(i,1)[0]);
    state.selected.clear();
    return cards.reverse();
  }

  function nextTurn(){
    state.turn = (state.turn+1)%3;
    render();
    if(state.phase==="PLAY" && state.players[state.turn].name!=="你") setTimeout(aiAct, 450);
  }

  function checkEnd(){
    for(let i=0;i<3;i++){
      if(state.players[i].hand.length===0){
        const winnerRole = state.players[i].role;
        alert(`游戏结束：${state.players[i].name} 出完牌！胜方：${winnerRole==="L"?"地主":"农民"}`);
        init();
        return true;
      }
    }
    return false;
  }

  function aiBid(i){
    // 简化：随机 40% 叫
    return Math.random() < 0.40;
  }

  function finishBid(landlord){
    state.landlord = landlord;
    state.players.forEach((p,idx)=> p.role = (idx===landlord) ? "L" : "F");
    // 地主拿底牌
    state.players[landlord].hand.push(...state.bottom);
    sortHand(state.players[landlord].hand);
    state.phase="PLAY";
    state.turn=landlord;
    state.lastPlay=null;
    state.lastPlayer=-1;
    pushLog(`确定地主：${state.players[landlord].name}，底牌加入其手牌。`);
    render();
    if(state.turn!==0) setTimeout(aiAct, 500);
  }

  function bidStep(){
    // 回合轮到谁叫
    if(state.turn===0) { render(); return; }
    const i=state.turn;
    const call = aiBid(i);
    pushLog(`${state.players[i].name}：${call?"叫地主":"不叫"}`);
    if(call){ state.bidCalled=true; finishBid(i); return; }
    // 都不叫则默认你为地主（简化）
    if(state.turn===2){
      if(!state.bidCalled){
        pushLog("无人叫地主（简化）：默认你为地主。");
        finishBid(0); return;
      }
    }
    state.turn=(state.turn+1)%3;
    bidStep();
  }

  function aiAct(){
    const i=state.turn;
    const me=state.players[i];
    const last = (state.lastPlay && state.lastPlayer!==i) ? state.lastPlay.an : null;
    const cards = aiPick(me.hand, (state.lastPlay && state.lastPlayer!==i) ? state.lastPlay.an : null);
    // 如果上家是自己（说明桌面轮空回到自己），last 视为 null
    const effectiveLast = (state.lastPlay && state.lastPlayer!==i) ? state.lastPlay : null;
    const picked = aiPick(me.hand, effectiveLast ? effectiveLast.an : null);

    if(!picked){
      pushLog(`${me.name}：不出`);
      // 若三家都不出，清空桌面（简化：当轮回到出牌者时清）
      // 处理：如果当前玩家不出且下一个轮到 lastPlayer，说明两家都PASS，桌面清空
      // 简化实现：当连续两次 PASS 且回到 lastPlayer 前，清空
      // 这里用 lastPlayer 逻辑：若当前玩家不出，且下一家就是 lastPlayer，则清空桌面
      const next = (state.turn+1)%3;
      if(state.lastPlay && next===state.lastPlayer){
        pushLog("两家都不出，桌面清空，由上家重新先手。");
        state.lastPlay=null; state.lastPlayer=-1;
      }
      nextTurn();
      return;
    }

    // 出牌
    // 从手牌移除 picked
    const set = new Set(picked.map(c=>c.id));
    me.hand = me.hand.filter(c=>!set.has(c.id));
    const an = analyze(picked);
    state.lastPlay={cards:picked, an};
    state.lastPlayer=i;
    pushLog(`${me.name}：出 ${formatPlay(picked)}（${an.type}）`);
    if(checkEnd()) return;
    nextTurn();
  }

  function init(){
    logEl.innerHTML="";
    state.deck = buildDeck();
    state.players.forEach(p=>p.hand=[]);
    state.bottom = state.deck.splice(0,3);
    // 发牌 17 张
    for(let r=0;r<17;r++){
      for(let i=0;i<3;i++) state.players[i].hand.push(state.deck.pop());
    }
    state.players.forEach(p=>sortHand(p.hand));
    state.players.forEach(p=>p.role="?");
    state.landlord=-1;
    state.turn=0;
    state.lastPlay=null;
    state.lastPlayer=-1;
    state.selected.clear();
    state.phase="BID";
    state.bidCalled=false;
    pushLog("发牌完成。你先选择：叫地主 / 不叫。");
    render();
  }

  $("#restart").onclick=init;

  $("#btnBid1").onclick=()=>{
    if(state.phase!=="BID" || state.turn!==0) return;
    pushLog("你：叫地主");
    state.bidCalled=true;
    finishBid(0);
  };
  $("#btnBid0").onclick=()=>{
    if(state.phase!=="BID" || state.turn!==0) return;
    pushLog("你：不叫");
    state.turn=1;
    bidStep();
  };

  $("#btnPlay").onclick=()=>{
    if(state.phase!=="PLAY" || state.turn!==0) return;
    const cards = selectionCards();
    const an = analyze(cards);
    const effectiveLast = (state.lastPlay && state.lastPlayer!==0) ? state.lastPlay : null;
    if(!canBeat(an, effectiveLast ? effectiveLast.an : null)){
      alert("这手牌无法压过桌面（或不是有效牌型）");
      return;
    }
    // 移除手牌
    const played = removeSelectedFromHand();
    state.lastPlay={cards:played, an};
    state.lastPlayer=0;
    pushLog(`你：出 ${formatPlay(played)}（${an.type}）`);
    render();
    if(checkEnd()) return;
    nextTurn();
  };

  $("#btnPass").onclick=()=>{
    if(state.phase!=="PLAY" || state.turn!==0) return;
    if(!state.lastPlay || state.lastPlayer===0) return;
    pushLog("你：不出");
    // 若下一家就是 lastPlayer，清空桌面
    const next=(state.turn+1)%3;
    if(state.lastPlay && next===state.lastPlayer){
      pushLog("两家都不出，桌面清空，由上家重新先手。");
      state.lastPlay=null; state.lastPlayer=-1;
    }
    nextTurn();
  };

  init();
})();
</script>
</body>
</html>
