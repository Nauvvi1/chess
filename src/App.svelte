<script>
  import { onMount } from "svelte";
  import "chessboard-element";
  import { Chess } from "chess.js";

  let boardEl;
  let game = new Chess();
  let historyVerbose = [];
  let thinking = false;
  let status = "Ваш ход (белыми)";
  let depth = 12;

  const highlightStyles = document.createElement("style");
  document.head.appendChild(highlightStyles);
  const whiteSquareGrey = "#a9a9a9";
  const blackSquareGrey = "#696969";

  function clearHighlights() {
    highlightStyles.textContent = "";
  }
  function highlightSquare(square) {
    const file = square.charCodeAt(0) - "a".charCodeAt(0);
    const rank = parseInt(square[1], 10) - 1;
    const isLight = (file + rank) % 2 === 0;
    const color = isLight ? whiteSquareGrey : blackSquareGrey;
    highlightStyles.textContent += `
      chess-board::part(${square}) { background-color: ${color}; }
    `;
  }

  let engine;
  let engineReady = false;

  function setupEngine() {
    const SF_URL =
      "https://cdn.jsdelivr.net/npm/stockfish.wasm@0.10.0/stockfish.js";
    engine = new Worker(SF_URL);
    engine.addEventListener("message", (e) => {
      const line = typeof e.data === "string" ? e.data : e.data?.data;
      if (!line) return;
      // console.debug('SF:', line);

      if (line === "uciok") {
        engine.postMessage("isready");
        return;
      }
      if (line === "readyok") {
        engineReady = true;
        return;
      }
      if (line.startsWith("bestmove")) {
        const [, move] = line.split(" ");
        thinking = false;
        applyEngineMove(move);
      }
    });
    engine.postMessage("uci");
    engine.postMessage("setoption name Threads value 1");
    engine.postMessage("setoption name Hash value 32");
  }

  function engineGo() {
    if (!engineReady || game.game_over()) return;
    thinking = true;
    engine.postMessage("position fen " + game.fen());
    engine.postMessage("go depth " + depth);
    status = `Компьютер думает (глубина ${depth})…`;
  }

  function applyEngineMove(uci) {
    if (!uci || uci === "(none)") return;
    const from = uci.slice(0, 2);
    const to = uci.slice(2, 4);
    const promo = uci[4] || "q";
    const move = game.move({ from, to, promotion: promo });
    if (move) {
      boardEl.setPosition(game.fen());
      historyVerbose = game.history({ verbose: true });
      updateStatus();
    }
  }

  function onDragStart(e) {
    if (game.game_over() || thinking || game.turn() !== "w") {
      e.preventDefault();
    }
  }

  function onDrop(e) {
    const { source, target, setAction } = e.detail;

    const move = game.move({ from: source, to: target, promotion: "q" });
    clearHighlights();

    if (move == null) {
      setAction("snapback");
      return;
    }

    boardEl.setPosition(game.fen());
    historyVerbose = game.history({ verbose: true });
    updateStatus();

    setTimeout(engineGo, 80);
  }

  function onMouseOver(e) {
    const { square } = e.detail;
    const moves = game.moves({ square, verbose: true });
    if (moves.length === 0) return;

    highlightSquare(square);
    for (const m of moves) highlightSquare(m.to);
  }
  function onMouseOut() {
    clearHighlights();
  }

  function updateStatus() {
    if (game.isCheckmate()) {
      status =
        game.turn() === "w" ? "Мат! Победили чёрные." : "Мат! Победили белые.";
      return;
    }
    if (game.isDraw()) {
      status = "Ничья.";
      return;
    }
    status =
      game.turn() === "w" ? "Ваш ход (белыми)" : "Ход компьютера (чёрными)";
  }

  function resetGame() {
    game = new Chess();
    boardEl.start();
    historyVerbose = [];
    thinking = false;
    updateStatus();
  }

  $: numberedHistory = (() => {
    const h = game.history();
    const rows = [];
    for (let i = 0; i < h.length; i += 2) {
      const white = h[i];
      const black = h[i + 1] || "";
      rows.push({ move: i / 2 + 1, white, black });
    }
    return rows;
  })();

  onMount(() => {
    setupEngine();
    boardEl.setPosition("start");
  });
</script>

<div class="app">
  <div class="board-box">
    <div class="topbar">
      <div class="status">{status}</div>
      <button class="btn" on:click={resetGame}>Новая партия</button>
    </div>

    <chess-board
      bind:this={boardEl}
      draggable-pieces
      notation
      on:drag-start={onDragStart}
      on:drop={onDrop}
      on:mouseover-square={onMouseOver}
      on:mouseout-square={onMouseOut}
      style="width: min(92vw, 640px); max-width: 640px; aspect-ratio: 1/1; display:block; margin:auto;"
    ></chess-board>

    <div class="controls">
      <label for="depth">Сложность (глубина): {depth}</label>
      <input id="depth" type="range" min="1" max="30" bind:value={depth} />
      <button class="btn" on:click={() => boardEl.flip()}
        >Повернуть доску</button
      >
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
      Игра против Stockfish (браузерная версия). Ваши ходы — белые. Промоция
      пешки автоматически — в ферзя.
    </div>
  </aside>
</div>

<style>
  :global(body) {
    margin: 0;
    font-family:
      system-ui,
      -apple-system,
      Segoe UI,
      Roboto,
      Arial,
      "Noto Sans",
      "Liberation Sans",
      sans-serif;
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
  .board-box {
    background: #121a2b;
    border-radius: 16px;
    padding: 16px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.35);
  }
  .panel {
    background: #121a2b;
    border-radius: 16px;
    padding: 16px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.35);
    display: flex;
    flex-direction: column;
    gap: 12px;
    max-height: 720px;
  }
  .status {
    font-size: 14px;
    opacity: 0.85;
    margin-bottom: 8px;
  }
  .controls {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-top: 10px;
    flex-wrap: wrap;
  }
  .controls label {
    font-size: 14px;
    opacity: 0.85;
  }
  .history {
    overflow: auto;
    border-radius: 10px;
    border: 1px solid rgba(255, 255, 255, 0.08);
    padding: 8px;
    background: rgba(255, 255, 255, 0.02);
    flex: 1;
  }
  .history table {
    width: 100%;
    border-collapse: collapse;
    font-size: 14px;
  }
  .history td {
    padding: 6px 8px;
    border-bottom: 1px dashed rgba(255, 255, 255, 0.06);
    vertical-align: top;
  }
  .history tr:last-child td {
    border-bottom: none;
  }
  .topbar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
    gap: 12px;
  }
  .btn {
    appearance: none;
    border: none;
    background: linear-gradient(135deg, #7cc0ff, #8c88ff 40%, #ff758c 100%);
    color: white;
    border-radius: 10px;
    padding: 8px 12px;
    cursor: pointer;
    font-weight: 600;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.3);
  }
  .btn:disabled {
    opacity: 0.7;
    cursor: not-allowed;
  }
  @media (max-width: 900px) {
    .app {
      grid-template-columns: 1fr;
    }
  }
</style>
