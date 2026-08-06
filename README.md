<index.html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, user-scalable=no">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="24H">
<meta name="theme-color" content="#0B0E14">
<title>24H — Military Time</title>
<style>
  :root{
    --ink:      #0B0E14;
    --panel:    #131824;
    --amber:    #FFA92B;
    --amber-dim:#5C3F14;
    --slate:    #6B7689;
    --bone:     #E8E2D6;
  }

  *{ margin:0; padding:0; box-sizing:border-box; }

  html,body{ height:100%; }

  body{
    background: var(--ink);
    color: var(--bone);
    font-family: ui-monospace, "SF Mono", Menlo, monospace;
    font-variant-numeric: tabular-nums;
    -webkit-user-select:none; user-select:none;
    -webkit-touch-callout:none;
    -webkit-tap-highlight-color:transparent;
    overflow:hidden;
    display:flex;
  }

  /* faint sodium-vapor glow behind the readout */
  body::before{
    content:"";
    position:fixed; inset:0;
    background: radial-gradient(60% 40% at 50% 42%, rgba(255,169,43,.10), transparent 70%);
    pointer-events:none;
  }

  .shell{
    position:relative;
    flex:1;
    display:flex;
    flex-direction:column;
    justify-content:space-between;
    padding: calc(env(safe-area-inset-top) + 4vh) calc(env(safe-area-inset-right) + 20px)
             calc(env(safe-area-inset-bottom) + 3vh) calc(env(safe-area-inset-left) + 20px);
  }

  /* ── header ─────────────────────────────── */
  .head{
    display:flex; justify-content:space-between; align-items:baseline;
    font-size: clamp(11px, 3vw, 15px);
    letter-spacing:.22em;
    color: var(--slate);
    text-transform:uppercase;
  }
  .head .date{ color: var(--bone); }

  /* ── main readout ───────────────────────── */
  .readout{
    display:flex; align-items:baseline; justify-content:center;
    gap:.06em;
    line-height:.8;
  }
  .hm{
    font-size: clamp(78px, 27vw, 260px);
    font-weight:600;
    letter-spacing:-.02em;
    color: var(--amber);
    text-shadow: 0 0 34px rgba(255,169,43,.28);
  }
  .colon{
    display:inline-block;
    transition: opacity .12s linear;
  }
  .colon.off{ opacity:.18; }
  .sec{
    font-size: clamp(26px, 8vw, 76px);
    font-weight:500;
    color: var(--slate);
    padding-left:.12em;
  }

  /* ── 60-tick minute ruler (the signature) ─ */
  .ruler{
    display:flex; align-items:flex-end; gap:2px;
    height:26px;
    margin: 5vh auto 0;
    width:min(100%, 620px);
  }
  .tick{
    flex:1;
    height:7px;
    background: var(--amber-dim);
    opacity:.5;
    border-radius:1px;
    transition: height .2s ease, opacity .2s ease, background .2s ease;
  }
  .tick.past{ opacity:.85; }
  .tick.now{
    height:22px;
    background: var(--amber);
    opacity:1;
  }
  .tick.five{ height:12px; }
  .tick.five.now{ height:22px; }

  /* ── footer ─────────────────────────────── */
  .foot{
    display:flex; justify-content:space-between; align-items:flex-end;
    font-size: clamp(11px, 3vw, 15px);
    letter-spacing:.18em;
    color: var(--slate);
    text-transform:uppercase;
  }
  .foot b{ color: var(--bone); font-weight:500; }

  button{
    font:inherit;
    font-size: clamp(11px, 3vw, 15px);
    letter-spacing:.18em;
    text-transform:uppercase;
    color: var(--slate);
    background:transparent;
    border:1px solid #232B3A;
    border-radius:2px;
    padding:8px 12px;
  }
  button[aria-pressed="true"]{
    color: var(--ink);
    background: var(--amber);
    border-color: var(--amber);
  }
  button:focus-visible{ outline:2px solid var(--amber); outline-offset:3px; }

  @media (orientation: landscape) and (max-height: 500px){
    .hm{ font-size: clamp(70px, 20vh, 190px); }
    .ruler{ margin-top:2.5vh; height:18px; }
    .tick.now{ height:16px; }
  }

  @media (prefers-reduced-motion: reduce){
    .colon, .tick{ transition:none; }
    .colon.off{ opacity:1; }
  }
</style>
</head>
<body>
<div class="shell">

  <div class="head">
    <span id="weekday">—</span>
    <span class="date" id="date">—</span>
  </div>

  <div>
    <div class="readout">
      <span class="hm"><span id="hh">00</span><span class="colon" id="colon">:</span><span id="mm">00</span></span>
      <span class="sec" id="ss">00</span>
    </div>
    <div class="ruler" id="ruler" aria-hidden="true"></div>
  </div>

  <div class="foot">
    <span>Zulu <b id="zulu">0000</b></span>
    <button id="wake" aria-pressed="false">Stay awake</button>
  </div>

</div>

<script>
(function(){
  const pad = n => String(n).padStart(2,'0');
  const el  = id => document.getElementById(id);
  const DAYS   = ['SUN','MON','TUE','WED','THU','FRI','SAT'];
  const MONTHS = ['JAN','FEB','MAR','APR','MAY','JUN','JUL','AUG','SEP','OCT','NOV','DEC'];

  // build the 60-tick ruler once
  const ruler = el('ruler');
  const ticks = [];
  for(let i=0;i<60;i++){
    const t = document.createElement('span');
    t.className = 'tick' + (i % 5 === 0 ? ' five' : '');
    ruler.appendChild(t);
    ticks.push(t);
  }

  let lastSec = -1;
  function tick(){
    const d = new Date();
    const s = d.getSeconds();

    el('hh').textContent = pad(d.getHours());
    el('mm').textContent = pad(d.getMinutes());
    el('ss').textContent = pad(s);
    el('colon').classList.toggle('off', s % 2 === 1);

    el('weekday').textContent = DAYS[d.getDay()];
    el('date').textContent = pad(d.getDate()) + ' ' + MONTHS[d.getMonth()] + ' ' + String(d.getFullYear()).slice(2);
    el('zulu').textContent = pad(d.getUTCHours()) + pad(d.getUTCMinutes());

    if(s !== lastSec){
      for(let i=0;i<60;i++){
        const t = ticks[i];
        t.classList.toggle('past', i < s);
        t.classList.toggle('now',  i === s);
      }
      lastSec = s;
    }
    requestAnimationFrame(tick);
  }
  tick();

  // keep the screen on while the clock is open
  const btn = el('wake');
  let lock = null;

  if(!('wakeLock' in navigator)){
    btn.disabled = true;
    btn.textContent = 'Screen sleeps';
  }

  btn.addEventListener('click', async () => {
    if(lock){
      await lock.release();
      lock = null;
      btn.setAttribute('aria-pressed','false');
      return;
    }
    try{
      lock = await navigator.wakeLock.request('screen');
      lock.addEventListener('release', () => {
        lock = null;
        btn.setAttribute('aria-pressed','false');
      });
      btn.setAttribute('aria-pressed','true');
    }catch(e){
      btn.textContent = 'Screen sleeps';
      btn.disabled = true;
    }
  });

  // re-acquire the lock when coming back from the app switcher
  document.addEventListener('visibilitychange', async () => {
    if(document.visibilityState === 'visible' && btn.getAttribute('aria-pressed') === 'true' && !lock){
      try{ lock = await navigator.wakeLock.request('screen'); }catch(e){}
    }
  });
})();
</script>
</body>
</html>
