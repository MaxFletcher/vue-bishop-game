<script setup lang="ts">
// This starter template is using Vue 3 <script setup> SFCs
// Check out https://vuejs.org/api/sfc-script-setup.html#script-setup
import {
  ref, reactive, computed, watch, onMounted,
} from 'vue';
// import bishopData from './assets/bishop-image';
// import bishopUrl from './assets/bishop-chess.svg';

enum BishopColour {
  White = 0,
  Black = 1,
}

interface CellDefinition {
  row: number;
  column: number;
  state: -1 | BishopColour;
}

interface BoardState {
  cells: number[][];
  selectedCell: CellDefinition | null;
}

const boardState: BoardState = reactive({
  cells: [],
  selectedCell: null,
});

const boardColumns = 5;
const boardRows = 4;

const canvas = ref<HTMLCanvasElement>();
const canvasContext = ref<CanvasRenderingContext2D>();
const canvasWidth = 640;
const canvasHeight = 480;

const tileWidth = canvasWidth / boardColumns;
const tileHeight = canvasHeight / boardRows;

onMounted(() => {
  // Create board
  for (let r = 0; r < boardRows; r += 1) {
    boardState.cells[r] = [];

    for (let c = 0; c < boardColumns; c += 1) {
      boardState.cells[r][c] = -1;
    }
  }

  // Assign initial bishop positions on the board
  boardState.cells[0][0] = BishopColour.White;
  boardState.cells[1][0] = BishopColour.White;
  boardState.cells[2][0] = BishopColour.White;
  boardState.cells[3][0] = BishopColour.White;

  boardState.cells[0][4] = BishopColour.Black;
  boardState.cells[1][4] = BishopColour.Black;
  boardState.cells[2][4] = BishopColour.Black;
  boardState.cells[3][4] = BishopColour.Black;

  // Get a reference to the canvas for drawing
  canvas.value!.width = canvasWidth;
  canvas.value!.height = canvasHeight;
  canvasContext.value = canvas.value!.getContext('2d')!;

  // Add event listeners to game board
  canvas.value?.addEventListener('click', onBoardClick);
  // canvas.value.addEventListener('')

  drawBoard();
});

function onBoardClick(event: MouseEvent) {
  const { offsetX, offsetY } = event;
  const canvasRelativeX = (offsetX * canvas.value!.width) / canvas.value!.clientWidth;
  const canvasRelativeY = (offsetY * canvas.value!.height) / canvas.value!.clientHeight;

  // Find corresponding tile
  const tx = Math.floor(canvasRelativeX / tileWidth);
  const ty = Math.floor(canvasRelativeY / tileHeight);

  // User has clicked a tile without a bishop on
  if (boardState.cells[ty][tx] === -1) {
    if (boardState.selectedCell && validMoves.value.length > 0) {
      // Find a matching valid cell for this move
      const i = validMoves.value.findIndex(cell => cell.row === ty && cell.column === tx);

      if (i > -1) {
        boardState.cells[ty][tx] = boardState.selectedCell.state;
        boardState.cells[boardState.selectedCell.row][boardState.selectedCell.column] = -1;
        boardState.selectedCell = null;
        return;
      }
    }

    boardState.selectedCell = null;
    return;
  }

  boardState.selectedCell = {
    row: ty,
    column: tx,
    state: boardState.cells[ty][tx],
  };
}

function drawBoard() {
  // Clear screen
  const ctx = canvasContext.value;
  ctx?.clearRect(0, 0, canvas.value!.width, canvas.value!.height);

  // Draw board tiles
  for (let i = 0; i < boardRows; i += 1) {
    for (let j = 0; j < boardColumns; j += 1) {
      // eslint-disable-next-line no-nested-ternary
      ctx!.fillStyle = i % 2 === 0
        ? j % 2 === 0 ? 'yellow' : 'green'
        : j % 2 === 0 ? 'green' : 'yellow';
      ctx?.fillRect(j * tileWidth, i * tileHeight, tileWidth, tileHeight);
    }
  }

  // Draw bishops
  for (let r = 0; r < boardRows; r += 1) {
    const row = boardState.cells[r];

    for (let c = 0; c < boardColumns; c += 1) {
      if (row[c] === -1) continue;

      ctx!.fillStyle = row[c] === BishopColour.Black
        ? 'black'
        : 'white';
      ctx?.beginPath();
      ctx?.arc(c * tileWidth + 60, r * tileHeight + 60, 40, 0, 2 * Math.PI);
      ctx?.fill();
    }
  }

  // Draw available
  validMoves.value.forEach(cell => {
    canvasContext.value!.fillStyle = 'rgba(63, 127, 191, 0.75)';
    canvasContext.value?.fillRect(cell.column * tileWidth, cell.row * tileHeight, tileWidth, tileHeight);
  });
}

/**
 * Returns a colletion of valid tiles where a bishop can move to
 * @param row The row of the cell to check
 * @param col The column of the cell to check
 */
function getValidMoves(row: number, col: number) {
  const validCells: CellDefinition[] = [];
  const obstructedCells: { row: number, column: number }[] = [];

  const selectedCell = boardState.cells[row][col];
  if (selectedCell < 0 || selectedCell > 1) {
    return obstructedCells;
  }

  // Calculate obstructed tiles
  for (let r = 0; r < boardRows; r++) {
    for (let c = 0; c < boardColumns; c++) {
      const cell = boardState.cells[r][c];

      // Ignore cell if it is empty or the same colour as the selected cell/bishop
      if (cell > -1  && cell !== selectedCell) {
        for (let r2 = 0; r2 < boardRows; r2++) {
          for (let c2 = 0; c2 < boardColumns;  c2++) {
            const rowDiff = Math.abs(r - r2);
            const colDiff = Math.abs(c - c2);

            if (rowDiff === colDiff) {
              obstructedCells.push({
                row: r2,
                column: c2
              });
            }
          }
        }
      }

    }
  }

  // Calculate possible tiles available to move to
  for (let r = 0; r < boardRows; r++) {
    for (let c = 0; c < boardColumns; c++) {
      const cell = boardState.cells[r][c];
      const rowDiff = Math.abs(row - r);
      const colDiff = Math.abs(col - c);

      if (r !== row && rowDiff === colDiff && cell === -1 && !obstructedCells.some(x => x.row === r && x.column === c)) {
        validCells.push({
          row: r,
          column: c,
          state: cell,
        });
      }
    }
  }

  return validCells;
}

const validMoves = computed(() => {
  if (!boardState.selectedCell) {
    return [];
  }

  const { row, column } = boardState.selectedCell;
  return getValidMoves(row, column);
});

watch(validMoves, () => {
  drawBoard();
});
</script>

<template>
  <div class="stack-vertical">
    <h2>Swap the bishops</h2>
    <canvas 
      id="game-canvas"
      ref="canvas"
    />

    <div class="stack-horizontal">
      {{ boardState.selectedCell }}
    </div>
  </div>
</template>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

#game-canvas {
  width: 50%;
}

.stack-vertical {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.stack-horizontal {
  display: flex;
  flex-direction: row;
}
</style>
