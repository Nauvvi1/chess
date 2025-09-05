<script>
  import { onMount, tick } from 'svelte';
  import 'chessboard-element';
  import { Chess } from 'chess.js';

  let boardEl;
  let mounted = false;
  let game = new Chess();

  let historySAN = [];
  let status = 'Ваш ход (белыми)';
  let thinking = false;
  let depth = 12;

  // ---------- движок (asm.js воркер из /public/stockfish-asm.js) ----------
  let engine;
  let engineReady = false;
  let engineFailed = false;

  function bootEngine() {
    try {
      engine = new Worker('/stockfish-asm.js');
    } catch (e) {
      console.error('Не удалось создать WebWorker stockfish-asm.js', e);
      engineFailed = true;
      return;
    }

    engine.addEventListener('error', (err) => {
      console.error('Stockfish worker error:', err);
      engineFailed = true;
    });

    engine.addEventListener('message', (e) => {
      const line = typeof e.data === 'string' ? e.data : e.data?.data;
      if (!line) return;

      if (line === 'uciok') { engine.postMessage('isready'); return; }
      if (line === 'readyok') { engineReady = true; engineGo(); return; }
      if (line.startsWith('bestmove')) {
        thinking = false;
        const [, move] = line.split(' ');
        applyEngineMove(move);
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
    const from = uci.slice(0, 2);
    const to = uci.slice(2, 4);
    const promo = uci[4] || 'q';
    try {
      const move = game.move({ from, to, promotion: promo });
      if (move) {
        queueMicrotask(() => boardEl && boardEl.setPosition(game.fen()));
        historySAN = game.history();
        lastMove = { from, to };
        updateStatus();
        clearHighlights();                      // убираем старые хайлайты
        markLastMove();                         // подсветка последнего хода
        engineGo();
      }
    } catch {}
  }

  // ---------- подсветки + клик-клик ----------
  let supportsMarkers = false;

  // для фолбэка (если нет addMarker/removeMarkers)
  const styleEl = typeof document !== 'undefined' ? document.createElement('style') : null;
  if (styleEl) document.head.appendChild(styleEl);

  let selected = null;             // выбранная клетка (клик-клик)
  let legalTargets = [];           // массив целевых клеток для выбранной
  let lastMove = null;             // {from, to}

  function clearHighlights() {
    if (supportsMarkers) {
      try { boardEl.removeMarkers(undefined, 'legal'); } catch {}
      try { boardEl.removeMarkers(undefined, 'selected'); } catch {}
      if (!lastMove) { try { boardEl.removeMarkers(undefined, 'last'); } catch {} }
    } else if (styleEl) {
      styleEl.textContent = ''; // пересоздаём ниже
    }
  }

  function markSquares(sqs, kind) {
    if (supportsMarkers) {
      for (const s of sqs) boardEl.addMarker(s, kind);
    } else if (styleEl) {
      // разные цвета для разных видов
      const color =
        kind === 'selected' ? 'rgba(255,215,0,.45)' :
        kind === 'last'     ? 'rgba(125, 255, 125, .35)' :
                              'rgba(127,180,255,.35)';
      styleEl.textContent += sqs
        .map(sq => `chess-board::part(square-${sq}){ box-shadow: inset 0 0 0 9999px ${color}; }`)
        .join('\n') + '\n';
    }
  }

  function markLastMove() {
    if (!lastMove) return;
    if (supportsMarkers) {
      try { boardEl.removeMarkers(undefined, 'last'); } catch {}
    }
    markSquares([lastMove.from, lastMove.to], 'last');
  }

  function showLegal(square) {
    // показываем ходы только для белых фигур, в свой ход и без «раздумий» движка
    const piece = game.get(square);
    if (!piece || piece.color !== 'w' || thinking || game.turn() !== 'w' || game.isGameOver()) {
      selected = null; legalTargets = []; clearHighlights(); markLastMove(); return;
    }
    const moves = game.moves({ square, verbose: true });
    if (moves.length === 0) {
      selected = null; legalTargets = []; clearHighlights(); markLastMove(); return;
    }
    selected = square;
    legalTargets = moves.map(m => m.to);
    clearHighlights();
    // подсвечиваем выбранную и все доступные
    markSquares([square], 'selected');
    markSquares(legalTargets, 'legal');
    markLastMove();
  }

  // обработчик кликов по клеткам (клик-клик)
  function onClickSquare(e) {
    const { square } = e.detail;     // событие chessboard-element: click-square
    if (!square) return;

    // если ничего не выбрано — просто показать ходы этой фигуры
    if (!selected) {
      showLegal(square);
      return;
    }

    // если кликаем по той же клетке — снятие выбора
    if (selected === square) {
      selected = null; legalTargets = []; clearHighlights(); markLastMove(); return;
    }

    // если клик по одной из легальных — делаем ход
    if (legalTargets.includes(square)) {
      try {
        const move = game.move({ from: selected, to: square, promotion: 'q' });
        if (!move) {
          // на всякий случай — если move не прошёл (не должно)
          selected = null; legalTargets = []; clearHighlights(); markLastMove(); return;
        }
      } catch {
        selected = null; legalTargets = []; clearHighlights(); markLastMove(); return;
      }

      // доска сама переставила фигуру — просто обновляем состояние
      historySAN = game.history();
      lastMove = { from: selected, to: square };
      selected = null;
      legalTargets = [];
      clearHighlights();
      markLastMove();
      updateStatus();
      engineGo();
      return;
    }

    // если кликнули по другой своей фигуре — переключаем выбор и показываем её ходы
    const other = game.get(square);
    if (other && other.color === 'w') {
      showLegal(square);
      return;
    }

    // иначе — снимаем выбор
    selected = null; legalTargets = []; clearHighlights(); markLastMove();
  }

  // ---------- события доски (drag&drop остаётся рабочим) ----------
  function onDragStart(e) {
    const { piece } = e.detail;
    if (thinking || game.isGameOver()) { e.preventDefault(); return; }
    if (!piece || piece[0] !== 'w' || game.turn() !== 'w') { e.preventDefault(); }
    // при начале перетаскивания убираем клик-выбор
    selected = null; legalTargets = []; clearHighlights(); markLastMove();
  }

  function onDrop(e) {
    const { source, target, setAction } = e.detail;
    clearHighlights();
    try {
      const move = game.move({ from: source, to: target, promotion: 'q' });
      if (!move) { setAction('snapback'); return; }
    } catch {
      setAction('snapback'); return;
    }
    historySAN = game.history();
    lastMove = { from: source, to: target };
    updateStatus();
    markLastMove();
    engineGo();
  }

  function onMouseOver(e) {
    // лёгкая подсветка ходов при ховере (если нет выбранной)
    if (selected) return;
    const { square } = e.detail;
    if (thinking || game.turn() !== 'w' || game.isGameOver()) return;
    const piece = game.get(square);
    if (!piece || piece.color !== 'w') return;
    const moves = game.moves({ square, verbose: true });
    if (moves.length === 0) return;
    clearHighlights();
    markSquares([square], 'selected');
    markSquares(moves.map(m => m.to), 'legal');
    markLastMove();
  }
  function onMouseOut() {
    if (selected) return; // если есть явный выбор — не трогаем
    clearHighlights();
    markLastMove();
  }

  // ---------- сервис ----------
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
    lastMove = null;
    selected = null;
    legalTargets = [];
    clearHighlights();
    thinking = false;
    updateStatus();
    if (!engine || engineFailed) {
      engineFailed = false;
      engineReady = false;
      bootEngine();
    } else {
      engineGo();
    }
  }

  $: numberedHistory = (() => {
    const h = historySAN;
    const rows = [];
    for (let i = 0; i < h.length; i += 2) {
      rows.push({ move: i / 2 + 1, white: h[i], black: h[i + 1] || '' });
    }
    return rows;
  })();

  onMount(async () => {
    await tick();
    mounted = true;
    await tick();

    supportsMarkers = typeof boardEl?.addMarker === 'function';
    updateStatus();
    bootEngine();
  });
</script>

<style>
  :global(body) {
    margin: 0;
    font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, "Noto Sans", "Liberation Sans", sans-serif;
    background: #0b1220;
    color: #e9eef8;
  }
  .app {
    display: grid;
    grid-template-columns: 1fr 320px;
    gap: 16px;
    max-width: 1000px;
    margin: 24px auto;
    padding: 0 16px;
  }
  .board-box { background: #121a2b; border-radius: 16px; padding: 16px; box-shadow: 0 10px 30px rgba(0,0,0,.35); }
  .panel { background: #121a2b; border-radius: 16px; padding: 16px; box-shadow: 0 10px 30px rgba(0,0,0,.35);
    display: flex; flex-direction: column; gap: 12px; max-height: 720px; }
  .status { font-size: 14px; opacity: .85; margin-bottom: 8px; }
  .controls { display: flex; align-items: center; gap: 12px; margin-top: 10px; flex-wrap: wrap; }
  .controls label { font-size: 14px; opacity: .85; }
  .history { overflow: auto; border-radius: 10px; border: 1px solid rgba(255,255,255,.08); padding: 8px; background: rgba(255,255,255,.02); flex: 1; }
  .history table { width: 100%; border-collapse: collapse; font-size: 14px; }
  .history td { padding: 6px 8px; border-bottom: 1px dashed rgba(255,255,255,.06); vertical-align: top; }
  .history tr:last-child td { border-bottom: none; }
  .topbar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; gap: 12px; }
  .btn { appearance: none; border: none; background: linear-gradient(135deg, #7cc0ff, #8c88ff 40%, #ff758c 100%);
    color: white; border-radius: 10px; padding: 8px 12px; cursor: pointer; font-weight: 600; box-shadow: 0 4px 16px rgba(0,0,0,.3); }
  .btn:disabled { opacity: .7; cursor: not-allowed; }

  /* стили маркеров (для новых версий chessboard-element) */
  chess-board::part(marker-legal)    { background: rgba(127,180,255,.35); border-radius: 4px; inset: 0; }
  chess-board::part(marker-selected) { background: rgba(255,215,0,.45);   border-radius: 4px; inset: 0; }
  chess-board::part(marker-last)     { background: rgba(125,255,125,.35); border-radius: 4px; inset: 0; }
</style>

<div class="app">
  <div class="board-box">
    <div class="topbar">
      <div class="status">{status}</div>
      <button class="btn" on:click={resetGame}>Новая партия</button>
    </div>

    {#if mounted}
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
        style="width: min(92vw, 640px); max-width: 640px; aspect-ratio: 1/1; display:block; margin:auto;"
      ></chess-board>
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
