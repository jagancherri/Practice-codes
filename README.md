<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Card Swipe (HTML / CSS / JS)</title>
  <style>
    :root{--card-w:320px;--card-h:480px}
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:Inter,ui-sans-serif,system-ui,Segoe UI,Roboto}
    body{display:grid;place-items:center;background:linear-gradient(180deg,#eef2ff,#fff)}

    .stage{width:var(--card-w);height:var(--card-h);position:relative}

    .card{position:absolute;inset:0;width:100%;height:100%;border-radius:18px;display:flex;align-items:flex-end;justify-content:flex-start;padding:20px;box-shadow:0 12px 30px rgba(20,20,40,0.12),0 6px 12px rgba(20,20,40,0.06);user-select:none;touch-action:none;transition:box-shadow .2s,transform .3s}

    /* stacked layout - top card later in DOM should be visually on top, but we set z-index in JS */
    .card .meta{color:#fff;font-weight:700;text-shadow:0 2px 6px rgba(0,0,0,0.4)}

    /* sample backgrounds */
    .card[data-bg="1"]{background:linear-gradient(135deg,#ff758c 0%,#ff7eb3 100%)}
    .card[data-bg="2"]{background:linear-gradient(135deg,#43cea2 0%,#185a9d 100%)}
    .card[data-bg="3"]{background:linear-gradient(135deg,#667eea 0%,#764ba2 100%)}
    .card[data-bg="4"]{background:linear-gradient(135deg,#f7971e 0%,#ffd200 100%)}
    .card[data-bg="5"]{background:linear-gradient(135deg,#89f7fe 0%,#66a6ff 100%)}

    /* like / nope badges */
    .badge{position:absolute;top:20px;padding:8px 12px;border-radius:8px;font-weight:800;letter-spacing:.06em;transform:rotate(-12deg);opacity:0;transition:opacity .12s,transform .12s}
    .badge.like{left:20px;background:rgba(255,255,255,0.12);color:#e8ffed;border:2px solid rgba(255,255,255,0.28)}
    .badge.nope{right:20px;background:rgba(0,0,0,0.12);color:#ffe8e8;border:2px solid rgba(255,255,255,0.12);transform:rotate(12deg)}

    .controls{margin-top:18px;display:flex;gap:12px;justify-content:center}
    .btn{padding:10px 14px;border-radius:10px;border:0;background:#fff;box-shadow:0 6px 18px rgba(20,20,40,0.08);cursor:pointer}

    /* subtle guidance text */
    .hint{position:fixed;left:50%;transform:translateX(-50%);bottom:26px;color:#333;font-size:13px;opacity:.9}

    @media (max-width:420px){:root{--card-w:92vw;--card-h:72vh}}
  </style>
</head>
<body>
  <div class="stage" id="stage">
    <!-- Cards (topmost last) -->
    <div class="card" data-bg="1" data-index="0">
      <div class="badge like">LIKE</div>
      <div class="badge nope">NOPE</div>
      <div class="meta">Emma, 26</div>
    </div>

    <div class="card" data-bg="2" data-index="1">
      <div class="badge like">LIKE</div>
      <div class="badge nope">NOPE</div>
      <div class="meta">Sam, 29</div>
    </div>

    <div class="card" data-bg="3" data-index="2">
      <div class="badge like">LIKE</div>
      <div class="badge nope">NOPE</div>
      <div class="meta">Priya, 24</div>
    </div>

    <div class="card" data-bg="4" data-index="3">
      <div class="badge like">LIKE</div>
      <div class="badge nope">NOPE</div>
      <div class="meta">Arjun, 27</div>
    </div>

    <div class="card" data-bg="5" data-index="4">
      <div class="badge like">LIKE</div>
      <div class="badge nope">NOPE</div>
      <div class="meta">Maya, 25</div>
    </div>
  </div>

  <div class="hint">Drag/touch a card left or right to swipe • or use the buttons below</div>
  <div class="controls" style="position:fixed;bottom:80px;left:50%;transform:translateX(-50%)">
    <button class="btn" id="nopeBtn">Nope</button>
    <button class="btn" id="rewindBtn">Undo</button>
    <button class="btn" id="likeBtn">Like</button>
  </div>

  <script>
    // Simple card swipe implementation using Pointer Events
    (function(){
      const stage = document.getElementById('stage');
      let cards = Array.from(stage.querySelectorAll('.card'));
      let topIndex = cards.length - 1; // index of topmost card in array
      const SWIPE_THRESHOLD = 120; // px
      const VELOCITY_THRESHOLD = 0.3; // px/ms (not used in this simple version)
      let runningStack = [];

      // set initial stacking (z-index & small scale offset)
      function layoutCards(){
        cards = Array.from(stage.querySelectorAll('.card'));
        cards.forEach((c,i)=>{
          const order = i; // lower index is bottom
          c.style.zIndex = i;
          const offset = Math.min(3, cards.length - 1 - i); // top few smaller offset
          c.style.transform = `translateY(${offset*10}px) scale(${1 - offset*0.02})`;
          c.style.transition = 'transform .25s ease, box-shadow .2s';
        });
        topIndex = cards.length - 1;
      }

      layoutCards();

      let active = null;
      let startX=0,startY=0,lastX=0,lastY=0,startTime=0;

      function onPointerDown(e){
        if (topIndex < 0) return;
        const topCard = cards[topIndex];
        if (!topCard) return;
        active = topCard;
        active.setPointerCapture(e.pointerId);
        startX = e.clientX; startY = e.clientY;
        lastX = startX; lastY = startY;
        startTime = performance.now();
        active.style.transition = 'transform 0s';
        active.style.boxShadow = '0 18px 40px rgba(20,20,40,0.18)';
      }

      function onPointerMove(e){
        if (!active) return;
        const dx = e.clientX - startX;
        const dy = e.clientY - startY;
        const rot = dx / 12; // rotate by horizontal movement
        active.style.transform = `translate(${dx}px, ${dy}px) rotate(${rot}deg)`;

        // show badges
        const likeBadge = active.querySelector('.badge.like');
        const nopeBadge = active.querySelector('.badge.nope');
        const alpha = Math.min(1, Math.abs(dx) / SWIPE_THRESHOLD);
        if(dx > 0){ likeBadge.style.opacity = alpha; likeBadge.style.transform = 'rotate(-12deg) scale(1.02)'; nopeBadge.style.opacity=0; }
        else{ nopeBadge.style.opacity = alpha; nopeBadge.style.transform = 'rotate(12deg) scale(1.02)'; likeBadge.style.opacity=0; }

        lastX = e.clientX; lastY = e.clientY;
      }

      function onPointerUp(e){
        if (!active) return;
        const dx = e.clientX - startX;
        const dt = performance.now() - startTime;
        const vx = dx / dt; // px per ms

        // decide swipe
        if (Math.abs(dx) > SWIPE_THRESHOLD){
          // swipe out
          const dir = dx > 0 ? 1 : -1;
          const flyX = dir * (window.innerWidth + 400);
          active.style.transition = 'transform 400ms cubic-bezier(.22,.9,.35,1)';
          active.style.transform = `translate(${flyX}px, ${lastY - startY}px) rotate(${dir*30}deg)`;
          active.style.opacity = 0.98;

          // remove after animation
          setTimeout(()=>{
            active.remove();
            layoutCards();
          },420);
        } else {
          // return to center
          active.style.transition = 'transform 300ms cubic-bezier(.2,.8,.3,1), box-shadow .2s';
          active.style.transform = '';
          active.style.boxShadow = '';
          const likeBadge = active.querySelector('.badge.like');
          const nopeBadge = active.querySelector('.badge.nope');
          likeBadge.style.opacity = 0; nopeBadge.style.opacity = 0;
        }

        try{ active.releasePointerCapture(e.pointerId);}catch(e){}
        active = null;
      }

      // attach pointer events to the stage but only let top card react
      stage.addEventListener('pointerdown', function(e){
        // find top card element under pointer (topmost in DOM with pointer target)
        const el = cards[topIndex];
        if(!el) return;
        // ensure pointerdown happened on that card (or inside it)
        if(el.contains(e.target)) onPointerDown(e);
      });
      window.addEventListener('pointermove', onPointerMove);
      window.addEventListener('pointerup', onPointerUp);

      // Buttons
      document.getElementById('likeBtn').addEventListener('click', ()=>programmaticSwipe(1));
      document.getElementById('nopeBtn').addEventListener('click', ()=>programmaticSwipe(-1));

      let history = []; // store removed cards for undo
      function programmaticSwipe(direction){
        if (topIndex < 0) return;
        const card = cards[topIndex];
        // animate
        card.style.transition = 'transform 450ms cubic-bezier(.22,.9,.35,1)';
        const flyX = direction * (window.innerWidth + 400);
        card.style.transform = `translate(${flyX}px, 0px) rotate(${direction*28}deg)`;
        setTimeout(()=>{
          // keep a lightweight record for rewind
          history.push({html: card.outerHTML});
          card.remove(); layoutCards();
        },460);
      }

      // Rewind (undo last swipe)
      document.getElementById('rewindBtn').addEventListener('click', ()=>{
        if(!history.length) return;
        const last = history.pop();
        // re-insert card at end (top)
        stage.insertAdjacentHTML('beforeend', last.html);
        layoutCards();
      });

      // small enhancement: allow keyboard arrows
      window.addEventListener('keydown', (e)=>{
        if(e.key === 'ArrowRight') programmaticSwipe(1);
        if(e.key === 'ArrowLeft') programmaticSwipe(-1);
        if(e.key === 'z' && (e.ctrlKey || e.metaKey)) document.getElementById('rewindBtn').click();
      });

      // expose few helpers (for debugging)
      window._cardStack = {layoutCards};

    })();
  </script>
</body>
</html>
