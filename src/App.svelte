<script>
  import { onMount, tick } from 'svelte';
  import 'chessboard-element';
  import { Chess } from 'chess.js';

  let boardEl;
  let overlayEl;
  let mounted = false;
  let game = new Chess();

  let historySAN = [];
  let status = 'Ваш ход (белыми)';
  let thinking = false;
  let depth = 12;

  let engine, engineReady=false, engineFailed=false;

  function bootEngine() {
    try { engine = new Worker('/stockfish-asm.js'); }
    catch (e) { console.error('[SF] worker error', e); engineFailed=true; return; }

    engine.addEventListener('error', e => { console.error('[SF] runtime error', e); engineFailed = true; });
    engine.addEventListener('message', (e) => {
      const line = typeof e.data === 'string' ? e.data : e.data?.data;
      if (!line) return;
      if (line === 'uciok') { engine.postMessage('isready'); return; }
      if (line === 'readyok') { engineReady = true; engineGo(); return; }
      if (line.startsWith('bestmove')) {
        thinking = false;
        const [, uci] = line.split(' ');
        console.log('[SF] bestmove', uci);
        applyEngineMove(uci);
      }
    });

    engine.postMessage('uci');
    engine.postMessage('setoption name Threads value 1');
    engine.postMessage('setoption name Hash value 32');
  }

  function engineGo() {
    if (!engine || !engineReady || game.isGameOver() || game.turn() !== 'b') return;
    thinking = true;
    status = `Компьютер думает (глубина ${depth})…`;
    engine.postMessage('position fen ' + game.fen());
    engine.postMessage('go depth ' + depth);
  }

  function applyEngineMove(uci) {
    if (!uci || uci === '(none)') return;
    const from = uci.slice(0,2), to = uci.slice(2,4), promo = uci[4] || 'q';
    try {
      const mv = game.move({ from, to, promotion: promo });
      console.log('[MOVE][engine]', mv);
      if (mv) {
        queueMicrotask(() => boardEl && boardEl.setPosition(game.fen()));
        historySAN = game.history();
        setLastMove(from, to);      // подсветим «откуда/куда»
        updateStatus();
        engineGo();
      }
    } catch (e) { console.error('[MOVE][engine] failed', e); }
  }

  // ====== overlay (независимый от ::part и markers) ======
  const ALL_SQUARES = [];
  for (const f of 'abcdefgh') for (let r=1; r<=8; r++) ALL_SQUARES.push(`${f}${r}`);

  // состояние подсветок
  let dragging = false;
  let selected = null;
  let legalTargets = [];
  let lastFrom = null;
  let lastTo = null;

  function buildOverlay() {
    if (!overlayEl) return;
    overlayEl.innerHTML = '';
    // grid-area: col / row — у chessboard первый индекс это файл (a..h) → колонка
    // делаем a-h = 1-8, 1-8 = ряды
    const fileIndex = f => f.charCodeAt(0) - 96; // 'a'->1
    for (const sq of ALL_SQUARES) {
      const col = fileIndex(sq[0]);
      const row = Number(sq[1]);
      const cell = document.createElement('div');
      cell.className = 'hl-cell';
      cell.dataset.sq = sq;
      cell.style.gridArea = `${9 - row} / ${col}`;
      overlayEl.appendChild(cell);
    }
    console.log('[HL] overlay built (64 cells)');
  }

  function setClasses() {
    if (!overlayEl) return;
    const all = overlayEl.children;
    for (const el of all) el.className = 'hl-cell'; // reset
    if (lastFrom) getCell(lastFrom)?.classList.add('last-from');
    if (lastTo)   getCell(lastTo)?.classList.add('last-to');
    if (selected) {
      getCell(selected)?.classList.add('sel');
      for (const t of legalTargets) getCell(t)?.classList.add('legal');
    }
    // логи для наглядности
    console.log('[HL] paint',
      { selected, legalTargets, lastFrom, lastTo, dragging }
    );
  }

  function getCell(sq) {
    return overlayEl?.querySelector(`.hl-cell[data-sq="${sq}"]`);
  }

  function clearSelect() {
    selected = null;
    legalTargets = [];
  }

  function setLastMove(from, to) {
    lastFrom = from;
    lastTo   = to;
    setClasses();
  }

  // показать легальные для клетки
  function showLegal(square) {
    if (thinking || game.turn() !== 'w' || game.isGameOver()) {
      clearSelect(); setClasses(); return;
    }
    const piece = game.get(square);
    if (!piece || piece.color !== 'w') {
      clearSelect(); setClasses(); return;
    }
    const moves = game.moves({ square, verbose: true });
    if (!moves.length) { clearSelect(); setClasses(); return; }

    selected = square;
    legalTargets = moves.map(m => m.to);
    console.log('[LEGAL] from', square, '->', legalTargets);
    setClasses();
  }

  // ====== события ======
  function onClickSquare(e) {
    const { square } = e.detail;
    if (!square) return;
    console.log('[CLICK]', square, 'selected=', selected);

    if (!selected) { showLegal(square); return; }
    if (selected === square) { clearSelect(); setClasses(); return; }

    if (legalTargets.includes(square)) {
      try {
        const mv = game.move({ from: selected, to: square, promotion: 'q' });
        console.log('[MOVE][user]', mv);
        if (!mv) { clearSelect(); setClasses(); return; }
      } catch (err) { console.error('[MOVE][user] failed', err); clearSelect(); setClasses(); return; }

      historySAN = game.history();
      queueMicrotask(() => boardEl && boardEl.setPosition(game.fen()));
      setLastMove(selected, square);
      clearSelect();
      updateStatus();
      engineGo();
      return;
    }

    const other = game.get(square);
    if (other && other.color === 'w') { showLegal(square); return; }
    clearSelect(); setClasses();
  }

  function patchPieceStyle(retries = 20) {
    const sr = boardEl?.shadowRoot;
    if (!sr) {
      // ждём, пока chess-board создаст shadowRoot
      if (retries > 0) setTimeout(() => patchPieceStyle(retries - 1), 16);
      return;
    }
    if (sr.getElementById('user-piece-style')) return; // уже впрыснули

    const st = document.createElement('style');
    st.id = 'user-piece-style';
    st.textContent = `
      /* именно это тебе и нужно */
      .piece-image {
        position: relative !important;
      }
    `;
    sr.appendChild(st);
    console.log('[PATCH] injected .piece-image style into shadow root');
  }

  function onMouseOver(e) {
    if (selected || dragging) return;
    const { square } = e.detail;
    if (thinking || game.turn() !== 'w' || game.isGameOver()) return;
    const piece = game.get(square);
    if (!piece || piece.color !== 'w') return;
    const moves = game.moves({ square, verbose: true });
    if (!moves.length) return;

    selected = square;
    legalTargets = moves.map(m => m.to);
    setClasses();
  }
  function onMouseOut() {
    if (dragging) return;
    if (!selected) return;
    // убираем подсказку наведения, но не lastFrom/To
    clearSelect(); setClasses();
  }

  // drag&drop — подсветка держится пока тащим
  function onDragStart(e) {
    const { piece, source } = e.detail;
    if (thinking || game.isGameOver()) { e.preventDefault(); return; }
    if (!piece || piece[0] !== 'w' || game.turn() !== 'w') { e.preventDefault(); return; }
    dragging = true;
    console.log('[DRAG] start', source);
    showLegal(source);
  }
  function onDrop(e) {
    const { source, target, setAction } = e.detail;
    dragging = false;
    console.log('[DRAG] drop', source, '->', target);
    try {
      const mv = game.move({ from: source, to: target, promotion: 'q' });
      console.log('[MOVE][drag]', mv);
      if (!mv) { setAction('snapback'); return; }
    } catch (err) { console.error('[MOVE][drag] failed', err); setAction('snapback'); return; }

    historySAN = game.history();
    queueMicrotask(() => boardEl && boardEl.setPosition(game.fen()));
    setLastMove(source, target);
    clearSelect();
    updateStatus();
    engineGo();
  }

  // ===== сервис =====
  function updateStatus() {
    if (game.isCheckmate()) {
      status = game.turn() === 'w' ? 'Мат! Победили чёрные.' : 'Мат! Победили белые.';
    } else if (game.isDraw()) {
      status = 'Ничья.';
    } else {
      status = game.turn() === 'w' ? 'Ваш ход (белыми)' : 'Ход компьютера (чёрными)';
    }
  }

  function resetGame() {
    game = new Chess();
    boardEl?.setAttribute('position', 'start');
    historySAN = [];
    dragging = false;
    clearSelect();
    lastFrom = lastTo = null;
    setClasses();
    thinking = false;
    updateStatus();
    if (!engine || engineFailed) { engineFailed=false; engineReady=false; bootEngine(); }
    else { engineGo(); }
  }

  $: numberedHistory = (() => {
    const h = historySAN, rows = [];
    for (let i=0;i<h.length;i+=2) rows.push({ move: i/2+1, white: h[i], black: h[i+1] || '' });
    return rows;
  })();

  onMount(async () => {
    await tick(); mounted = true; await tick();
    console.log('[INIT] boardEl=', !!boardEl);
    patchPieceStyle(); 
    buildOverlay();
    setClasses();
    updateStatus();
    bootEngine();
  });
</script>

<style>
  :global(body){
    margin:0;
    font-family:system-ui,-apple-system, Segoe UI, Roboto, Arial, "Noto Sans", "Liberation Sans", sans-serif;
    background:#0b1220; color:#e9eef8;
  }
  .app{display:grid;grid-template-columns:1fr 320px;gap:16px;max-width:1000px;margin:24px auto;padding:0 16px}
  .board-box{background:#121a2b;border-radius:16px;padding:16px;box-shadow:0 10px 30px rgba(0,0,0,.35)}
  .panel{background:#121a2b;border-radius:16px;padding:16px;box-shadow:0 10px 30px rgba(0,0,0,.35);display:flex;flex-direction:column;gap:12px;max-height:720px}
  .status{font-size:14px;opacity:.85;margin-bottom:8px}
  .controls{display:flex;align-items:center;gap:12px;margin-top:10px;flex-wrap:wrap}
  .controls label{font-size:14px;opacity:.85}
  .history{overflow:auto;border-radius:10px;border:1px solid rgba(255,255,255,.08);padding:8px;background:rgba(255,255,255,.02);flex:1}
  .history table{width:100%;border-collapse:collapse;font-size:14px}
  .history td{padding:6px 8px;border-bottom:1px dashed rgba(255,255,255,.06);vertical-align:top}
  .history tr:last-child td{border-bottom:none}
  .topbar{display:flex;justify-content:space-between;align-items:center;margin-bottom:10px;gap:12px}
  .btn{appearance:none;border:none;background:linear-gradient(135deg,#7cc0ff,#8c88ff 40%,#ff758c 100%);color:#fff;border-radius:10px;padding:8px 12px;cursor:pointer;font-weight:600;box-shadow:0 4px 16px rgba(0,0,0,.3)}
  .btn:disabled{opacity:.7;cursor:not-allowed}
/* Обёртка для оверлея */
.board-wrap{ position:relative; width:min(92vw,640px); max-width:640px; aspect-ratio:1/1; margin:auto; }

:global(.hl-overlay){
  position:absolute; inset:0;
  display:grid; grid-template-columns:repeat(8,1fr); grid-template-rows:repeat(8,1fr);
  z-index:9;               /* поверх доски; pointer-events:none оставляем, чтобы drag работал */
  pointer-events:none;
}
:global(.hl-cell){ position:relative; }

/* Мягкая палитра (чуть светлее) */
:root{
  --sel:       rgba(170,155,255,.28);
  --legal:     rgba(150,200,255,.14);
  --legal-dot: rgba(150,200,255,.75);
  --from:      rgba(160,150,255,.22);
  --to:        rgba(140,200,255,.24);
}
:global(.hl-cell.sel)       { box-shadow: inset 0 0 0 9999px var(--sel);   border-radius:8px; }
:global(.hl-cell.legal)     { box-shadow: inset 0 0 0 9999px var(--legal); border-radius:8px; }
:global(.hl-cell.last-from) { box-shadow: inset 0 0 0 9999px var(--from);  border-radius:8px; }
:global(.hl-cell.last-to)   { box-shadow: inset 0 0 0 9999px var(--to);    border-radius:8px; }

/* «точка» по центру легальных */
:global(.hl-cell.legal)::after{
  content:""; position:absolute; left:50%; top:50%; transform:translate(-50%,-50%);
  width:26%; height:26%; border-radius:50%;
  background: radial-gradient(circle at 50% 50%, var(--legal-dot), rgba(150,200,255,.08) 70%, transparent 71%);
  filter: drop-shadow(0 0 6px rgba(150,200,255,.26));
}
</style>

<div class="app">
  <div class="board-box">
    <div class="topbar">
      <div class="status">{status}</div>
      <button class="btn" on:click={resetGame}>Новая партия</button>
    </div>

    {#if mounted}
      <div class="board-wrap">
        <chess-board
          bind:this={boardEl}
          draggable-pieces
          notation
          role="application"
          aria-label="Шахматная доска"
          position="start"
          move-input-mode="drag-click"
          on:drag-start={onDragStart}
          on:drop={onDrop}
          on:mouseover-square={onMouseOver}
          on:mouseout-square={onMouseOut}
          on:click-square={onClickSquare}
          style="display:block; width:100%; height:100%;"
        ></chess-board>

        <!-- наш независимый оверлей -->
        <div class="hl-overlay" bind:this={overlayEl}></div>
      </div>
    {/if}

    <div class="controls">
      <label for="depth">Сложность (глубина): {depth}</label>
      <input id="depth" type="range" min="1" max="30" bind:value={depth} />
      <button class="btn" on:click={() => boardEl?.flip()}>Повернуть доску</button>
    </div>
  </div>

  <aside class="panel">
    <div style="font-weight:700;">История ходов</div>
    <div class="history">
      <table>
        <tbody>
          {#each numberedHistory as row}
            <tr>
              <td style="width:36px;opacity:.7;">{row.move}.</td>
              <td>{row.white}</td>
              <td>{row.black}</td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
    <div style="font-size:12px;opacity:.7;">
      Игра против Stockfish (браузерная версия). Ваши ходы — белые. Промоция пешки автоматически — в ферзя.
    </div>
  </aside>
</div>
