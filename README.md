<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<title>Christmas at Mom’s Bingo</title>

<style>
:root { --border: 1px solid #222; --gap: 8px; --cellMinH: 72px; }
body { margin:0; font-family: system-ui, -apple-system, Arial, sans-serif; background:#fff; }
.top{
  position: sticky; top: 0; z-index: 20;
  padding: 10px 12px calc(10px + env(safe-area-inset-top));
  background:#fff; border-bottom: var(--border);
}
.row{ display:flex; gap:10px; flex-wrap:wrap; align-items:center; justify-content:space-between; }
button{
  padding: 12px 14px; font-size: 16px;
  border: var(--border); border-radius: 12px; background:#fff;
}
button:active{ transform: scale(0.98); }
.pill{
  display:inline-flex; align-items:center; gap:8px;
  padding:10px 12px; border: var(--border); border-radius: 999px;
  font-size: 14px;
}
.big{ font-size:18px; font-weight:800; }

.content{ padding: 14px 12px 24px; }
.titleRow{ display:flex; justify-content:space-between; align-items:center; margin-bottom:10px; }
.status{ font-weight:800; }
.status.muted{ opacity:.65; font-weight:600; }

.grid{ display:grid; grid-template-columns: repeat(5, 1fr); gap: var(--gap); }
.cell{
  border: var(--border); border-radius: 14px;
  padding: 10px 8px; min-height: var(--cellMinH);
  display:flex; align-items:center; justify-content:center;
  text-align:center; font-weight:750; font-size:13px; line-height:1.15;
  user-select:none; -webkit-tap-highlight-color: transparent;
}
.cell.marked{ background:#d7ffd7; }
.cell.free{ background:#e8f0ff; font-weight:900; }
.cell.win{ outline: 4px solid gold; }

.hint{ font-size:12px; opacity:.75; margin-top:10px; }

@media (min-width: 390px){
  :root{ --cellMinH: 78px; }
  .cell{ font-size:14px; }
}
</style>
</head>

<body>
<div class="top">
  <div class="row">
    <div class="pill"><span id="who" class="big">Player</span></div>
    <div class="pill">Christmas at Mom’s</div>
  </div>

  <div class="row" style="margin-top:10px; justify-content:flex-start;">
    <button id="newRoundBtn">New Round</button>
    <button id="clearBtn">Clear Marks</button>
  </div>
</div>

<div class="content">
  <div class="titleRow">
    <div class="big" id="boardTitle">Card</div>
    <div id="status" class="status muted">No bingo</div>
  </div>

  <div id="grid" class="grid" aria-label="Bingo board"></div>
  <div class="hint" id="hint"></div>
</div>

<script>
/** ---------- Players (locked via ?p=...) ---------- **/
const VALID_PLAYERS = ["johnny","shauna","laura"];
const params = new URLSearchParams(location.search);
const p = (params.get("p") || "").toLowerCase();

if (!VALID_PLAYERS.includes(p)) {
  document.body.innerHTML =
    "<div style='padding:18px;font-family:system-ui,Arial,sans-serif'>" +
    "<h2>Missing player</h2>" +
    "<p>Add <code>?p=johnny</code>, <code>?p=shauna</code>, or <code>?p=laura</code> to the URL.</p>" +
    "</div>";
  throw new Error("Missing/invalid player param");
}

const player = p;

/** ---------- Shared “round” seed (same on all phones if URL matches) ---------- **/
const DEFAULT_GAME_CODE = "moms-christmas-default";
let gameCode = (params.get("game") || DEFAULT_GAME_CODE).trim();

/** ---------- Your squares (24) ---------- **/
const TERMS_24 = [
  "MEAN TO MERYN",
  "“GOD DAMMIT!”",
  "TALKS ABOUT THE VENUE",
  "MISPRONOUNCES SOMEONE’S NAME",
  "A CHILD CRIES",
  "BITCHES ABOUT AN IN-LAW",
  "BITCHES ABOUT BETH",
  "BITCHES ABOUT SUZIE",
  "BITCHES ABOUT TIMMY",
  "BITCHES ABOUT UNCLE TOM",
  "“GET A GRIP!’",
  "MEAN TO CAROLINE",
  "MEAN TO BRADY",
  "MEAN TO COOPER",
  "“JESUS CHRIST!”",
  "FORGETS A GIFT",
  "TALKS ABOUT RUTH’S CHRIS",
  "DRY MEATBALLS",
  "EYE ROLL",
  "MEAN TO SHAUNA",
  "MEAN TO JOHNNY",
  "MEAN TO MARIN",
  "DAVE GOES TO THE SUNROOM",
  "“I’M THE BOSS AROUND HERE”"
];

/** ---------- Deterministic PRNG + seeded shuffle ---------- **/
function xfnv1a(str){
  let h = 2166136261 >>> 0;
  for (let i=0; i<str.length; i++){
    h ^= str.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return h >>> 0;
}
function mulberry32(seed){
  return function(){
    let t = seed += 0x6D2B79F5;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
function seededShuffle(arr, rng){
  const a = arr.slice();
  for (let i=a.length-1; i>0; i--){
    const j = Math.floor(rng() * (i+1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

/** ---------- Card construction rules ---------- **
  - 24 items + FREE (center)
  - Three players’ cards are permutations of the same 24 items
  - At every position (excluding center), all 3 players have different text
**/
const POSITIONS = [...Array(25).keys()].filter(i => i !== 12); // exclude center
const PLAYER_ORDER = ["johnny","shauna","laura"];
const playerIndex = PLAYER_ORDER.indexOf(player);

// Build three permutations that avoid same-position matches between any pair.
function buildThreePermutations(items24, rng){
  const base = items24.slice(); // 24 symbols

  // p1: seeded shuffle
  const p1 = seededShuffle(base, rng);

  // p2: derangement vs p1
  let p2 = null;
  for (let tries=0; tries<5000; tries++){
    const cand = seededShuffle(base, rng);
    let ok = true;
    for (let i=0; i<24; i++){
      if (cand[i] === p1[i]) { ok = false; break; }
    }
    if (ok) { p2 = cand; break; }
  }
  if (!p2) throw new Error("Could not build p2 derangement");

  // p3: differs from both p1 and p2 at each position
  let p3 = null;
  for (let tries=0; tries<15000; tries++){
    const cand = seededShuffle(base, rng);
    let ok = true;
    for (let i=0; i<24; i++){
      if (cand[i] === p1[i] || cand[i] === p2[i]) { ok = false; break; }
    }
    if (ok) { p3 = cand; break; }
  }

  // If random tries fail (unlikely), do a simple greedy/backtracking build for p3
  if (!p3) {
    const used = new Set();
    const out = Array(24).fill(null);

    // Build availability per position
    const avail = Array.from({length:24}, (_,i)=> base.filter(x => x !== p1[i] && x !== p2[i]));
    // Order positions by fewest choices
    const order = [...Array(24).keys()].sort((a,b)=> avail[a].length - avail[b].length);

    function backtrack(k){
      if (k === order.length) return true;
      const pos = order[k];
      // try in seeded order for stability
      const choices = seededShuffle(avail[pos], rng);
      for (const choice of choices){
        if (used.has(choice)) continue;
        used.add(choice);
        out[pos] = choice;
        if (backtrack(k+1)) return true;
        used.delete(choice);
        out[pos] = null;
      }
      return false;
    }

    if (!backtrack(0)) throw new Error("Could not build p3 permutation");
    p3 = out;
  }

  return [p1, p2, p3];
}

function makeCardForPlayer(items24, rng, whichPlayerIdx){
  const [p1,p2,p3] = buildThreePermutations(items24, rng);
  const perms = [p1,p2,p3];
  const perm = perms[whichPlayerIdx]; // 24-length

  // Map 24 values into 25 grid with FREE at center
  const grid = Array(25).fill("");
  let k = 0;
  for (const pos of POSITIONS) grid[pos] = perm[k++];
  grid[12] = "FREE";

  const marked = Array(25).fill(false);
  marked[12] = true; // FREE marked
  return { grid, marked, win: [] };
}

/** ---------- Bingo detection ---------- **/
function findWin(marked){
  const lines = [
    [0,1,2,3,4],[5,6,7,8,9],[10,11,12,13,14],[15,16,17,18,19],[20,21,22,23,24],
    [0,5,10,15,20],[1,6,11,16,21],[2,7,12,17,22],[3,8,13,18,23],[4,9,14,19,24],
    [0,6,12,18,24],[4,8,12,16,20]
  ];
  for (const l of lines) if (l.every(i => marked[i])) return l;
  return [];
}

/** ---------- Persistence (per-device, per-player, per-gameCode) ---------- **/
function storageKey(){
  return `momsbingo:v1:${location.pathname}:${player}:${gameCode}`;
}
function saveState(state){
  try { localStorage.setItem(storageKey(), JSON.stringify(state)); } catch(e){}
}
function loadState(){
  try {
    const raw = localStorage.getItem(storageKey());
    if (!raw) return null;
    const parsed = JSON.parse(raw);
    if (!parsed || !parsed.card || !parsed.card.grid || !parsed.card.marked) return null;
    return parsed;
  } catch(e){ return null; }
}

/** ---------- UI ---------- **/
const whoEl = document.getElementById("who");
const titleEl = document.getElementById("boardTitle");
const statusEl = document.getElementById("status");
const gridEl = document.getElementById("grid");
const hintEl = document.getElementById("hint");

whoEl.textContent = player.charAt(0).toUpperCase() + player.slice(1);
titleEl.textContent = `${whoEl.textContent} Card`;
hintEl.textContent = `Locked to ${whoEl.textContent}. Auto-saves on this device. Use the same &game= code on all phones to keep everyone on the same round.`;

let state = null;

function render(){
  const card = state.card;
  const win = findWin(card.marked);
  card.win = win;

  statusEl.textContent = win.length ? "BINGO! 🎉" : "No bingo";
  statusEl.className = "status " + (win.length ? "" : "muted");

  gridEl.innerHTML = "";
  card.grid.forEach((txt, i) => {
    const d = document.createElement("div");
    d.className = "cell";
    if (i === 12) d.classList.add("free");
    if (card.marked[i]) d.classList.add("marked");
    if (win.includes(i)) d.classList.add("win");
    d.textContent = (i === 12) ? "★ FREE ★" : txt;

    d.addEventListener("click", () => {
      if (i === 12) return;
      card.marked[i] = !card.marked[i];
      // keep free marked
      card.marked[12] = true;
      saveState(state);
      render();
    }, {passive:true});

    gridEl.appendChild(d);
  });
}

function newRound(){
  // Generate a new code (optional) and rewrite URL so you can share it
  const buf = new Uint32Array(2);
  crypto.getRandomValues(buf);
  gameCode = `xmas-${buf[0].toString(16)}${buf[1].toString(16)}`.slice(0, 14);

  const url = new URL(location.href);
  url.searchParams.set("p", player);
  url.searchParams.set("game", gameCode);
  history.replaceState(null, "", url.toString());

  const rng = mulberry32(xfnv1a(gameCode));
  const card = makeCardForPlayer(TERMS_24, rng, playerIndex);
  state = { gameCode, card };
  saveState(state);

  hintEl.textContent =
    `Locked to ${whoEl.textContent}. Round code: ${gameCode}. (Use the same &game=${gameCode} on all phones.)`;

  render();
}

document.getElementById("clearBtn").addEventListener("click", () => {
  const card = state.card;
  card.marked = card.marked.map((_,i)=> i===12);
  saveState(state);
  render();
});

document.getElementById("newRoundBtn").addEventListener("click", () => {
  newRound();
});

/** ---------- Init ---------- **/
const saved = loadState();
if (saved) {
  state = saved;
  // ensure FREE marked
  state.card.marked[12] = true;
  render();
} else {
  const rng = mulberry32(xfnv1a(gameCode));
  const card = makeCardForPlayer(TERMS_24, rng, playerIndex);
  state = { gameCode, card };
  saveState(state);
  render();
}
</script>
</body>
</html>
