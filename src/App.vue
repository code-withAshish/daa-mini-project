<template>
  <div class="game-wrapper">
    <h1>🏴‍☠️ Treasure Hunt</h1>
    <div class="grid-container">
      <div v-for="cell in grid" :key="`${cell.row}-${cell.col}`"
        :class="['grid-cell', cell.type, isCurrentSprite(cell) ? 'animated' : '']"></div>
    </div>
    <div class="controls">
      <button @click="regenerateObstacles">🔁 Regenerate Obstacles</button>
      <button @click="startGame" v-if="!gameStarted">▶️ Start Game</button>
      <button @click="stopGame" v-if="gameStarted">⏹ Stop Game</button>
      <p v-if="elapsedTime !== null">⏱ Time: {{ elapsedTime }}s</p>
      <button @click="handleFindPath">✨ Auto Find Treasure (A*)</button>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';

const gridSize = 10;
const grid = ref([]);
const currentSpriteRow = ref(0);
const currentSpriteCol = ref(0);
const gameStarted = ref(false);
const startTime = ref(null);
const elapsedTime = ref(null);
let treasureRow = 9;
let treasureCol = 9;

function startGame() {
  gameStarted.value = true;
  startTime.value = Date.now();
  elapsedTime.value = null;
  initializeGrid();
}

function stopGame() {
  gameStarted.value = false;
  const endTime = Date.now();
  elapsedTime.value = Math.floor((endTime - startTime.value) / 1000);
}

function regenerateObstacles() {
  if (!grid.value.length) return;

  const baseGrid = grid.value.map(cell => {
    if (cell.type === 'obstacle' || cell.type === 'path') {
      return { ...cell, type: 'cell' };
    }
    return cell;
  });

  let obstaclesPlaced = 0;
  const maxObstacles = 20;

  const path = aStar(currentSpriteRow.value, currentSpriteCol.value, treasureRow, treasureCol);
  if (!path) {
    alert("⚠️ Can't place obstacles without a valid path.");
    return;
  }

  const pathSet = new Set(path.map(p => `${p.row},${p.col}`));

  while (obstaclesPlaced < maxObstacles) {
    const row = Math.floor(Math.random() * gridSize);
    const col = Math.floor(Math.random() * gridSize);
    const index = row * gridSize + col;
    const key = `${row},${col}`;

    if (
      baseGrid[index].type === 'cell' &&
      !pathSet.has(key) &&
      !(row === currentSpriteRow.value && col === currentSpriteCol.value) &&
      !(row === treasureRow && col === treasureCol)
    ) {
      baseGrid[index].type = 'obstacle';
      obstaclesPlaced++;
    }
  }

  grid.value = baseGrid;
}


function isCurrentSprite(cell) {
  return cell.row === currentSpriteRow.value && cell.col === currentSpriteCol.value;
}

function initializeGrid() {
  const createEmptyGrid = () => {
    const arr = [];
    for (let row = 0; row < gridSize; row++) {
      for (let col = 0; col < gridSize; col++) {
        arr.push({ row, col, type: 'cell' });
      }
    }
    return arr;
  };

  let path = null;
  let tempGrid;

  do {
    tempGrid = createEmptyGrid();
    treasureRow = Math.floor(Math.random() * gridSize);
    treasureCol = Math.floor(Math.random() * gridSize);
  } while (treasureRow === 0 && treasureCol === 0);

  currentSpriteRow.value = 0;
  currentSpriteCol.value = 0;
  tempGrid[0].type = 'sprite';
  tempGrid[treasureRow * gridSize + treasureCol].type = 'treasure';

  grid.value = tempGrid;
  path = aStar(0, 0, treasureRow, treasureCol);

  while (!path);

  const maxObstacles = 20;
  let obstaclesPlaced = 0;

  const pathSet = new Set(path.map(p => `${p.row},${p.col}`));

  while (obstaclesPlaced < maxObstacles) {
    const row = Math.floor(Math.random() * gridSize);
    const col = Math.floor(Math.random() * gridSize);
    const index = row * gridSize + col;
    const key = `${row},${col}`;

    if (
      tempGrid[index].type === 'cell' &&
      !pathSet.has(key)
    ) {
      tempGrid[index].type = 'obstacle';
      obstaclesPlaced++;
    }
  }

  grid.value = tempGrid;
}

function handleKeyDown(event) {
  let newRow = currentSpriteRow.value;
  let newCol = currentSpriteCol.value;

  if (event.key === 'ArrowUp') newRow--;
  else if (event.key === 'ArrowDown') newRow++;
  else if (event.key === 'ArrowLeft') newCol--;
  else if (event.key === 'ArrowRight') newCol++;

  if (
    newRow >= 0 && newRow < gridSize &&
    newCol >= 0 && newCol < gridSize
  ) {
    const newCell = grid.value[newRow * gridSize + newCol];
    if (newCell.type !== 'obstacle') {
      const updatedGrid = grid.value.map((cell) => {
        if (cell.row === currentSpriteRow.value && cell.col === currentSpriteCol.value) {
          return { ...cell, type: 'cell' };
        } else if (cell.row === newRow && cell.col === newCol) {
          return { ...cell, type: 'sprite' };
        }
        return cell;
      });

      grid.value = updatedGrid;
      currentSpriteRow.value = newRow;
      currentSpriteCol.value = newCol;

      if (newRow === treasureRow && newCol === treasureCol) {
        alert('🎉 You found the treasure!');
      }
    }
  }
}

function heuristic(row1, col1, row2, col2) {
  return Math.abs(row1 - row2) + Math.abs(col1 - col2);
}

function getNeighbors(row, col) {
  const neighbors = [];
  const moves = [[0, 1], [0, -1], [1, 0], [-1, 0]];
  for (const [dr, dc] of moves) {
    const newRow = row + dr;
    const newCol = col + dc;
    if (
      newRow >= 0 &&
      newRow < gridSize &&
      newCol >= 0 &&
      newCol < gridSize
    ) {
      const cell = grid.value[newRow * gridSize + newCol];
      if (cell.type !== 'obstacle') {
        neighbors.push({ row: newRow, col: newCol });
      }
    }
  }
  return neighbors;
}

function reconstructPath(node) {
  const path = [];
  let current = node;
  while (current) {
    path.unshift({ row: current.row, col: current.col });
    current = current.parent;
  }
  return path;
}

function aStar(startRow, startCol, endRow, endCol) {
  const openSet = [{
    row: startRow,
    col: startCol,
    g: 0,
    h: heuristic(startRow, startCol, endRow, endCol),
    f: 0,
    parent: null
  }];
  const closedSet = [];

  while (openSet.length > 0) {
    openSet.sort((a, b) => a.f - b.f);
    const current = openSet.shift();
    if (current.row === endRow && current.col === endCol) {
      return reconstructPath(current);
    }

    closedSet.push(current);

    const neighbors = getNeighbors(current.row, current.col);
    for (const neighbor of neighbors) {
      if (closedSet.find((n) => n.row === neighbor.row && n.col === neighbor.col)) continue;

      const g = current.g + 1;
      const h = heuristic(neighbor.row, neighbor.col, endRow, endCol);
      const f = g + h;

      const existing = openSet.find((n) => n.row === neighbor.row && n.col === neighbor.col);
      if (existing && g >= existing.g) continue;

      if (existing) {
        existing.g = g;
        existing.h = h;
        existing.f = f;
        existing.parent = current;
      } else {
        openSet.push({ row: neighbor.row, col: neighbor.col, g, h, f, parent: current });
      }
    }
  }
  return null;
}

function animatePath(path) {
  let index = 1;
  const interval = setInterval(() => {
    if (index < path.length) {
      const updatedGrid = grid.value.map((cell) => {
        if (cell.row === currentSpriteRow.value && cell.col === currentSpriteCol.value) {
          return { ...cell, type: 'path' };
        } else if (cell.row === path[index].row && cell.col === path[index].col) {
          return { ...cell, type: 'sprite' };
        }
        return cell;
      });

      grid.value = updatedGrid;
      currentSpriteRow.value = path[index].row;
      currentSpriteCol.value = path[index].col;
      index++;
    } else {
      clearInterval(interval);
      if (
        currentSpriteRow.value === treasureRow &&
        currentSpriteCol.value === treasureCol
      ) {
        alert('🎉 You found the treasure!');
        stopGame();
      }
    }
  }, 200);
}

function handleFindPath() {
  const path = aStar(currentSpriteRow.value, currentSpriteCol.value, treasureRow, treasureCol);
  if (path) {
    animatePath(path);
  } else {
    alert('❌ No path found!');
  }
}

onMounted(() => {
  initializeGrid();
  document.addEventListener('keydown', handleKeyDown);
});

onBeforeUnmount(() => {
  document.removeEventListener('keydown', handleKeyDown);
});
</script>

<style scoped>
.game-wrapper {
  font-family: 'Comic Sans MS', cursive, sans-serif;
  text-align: center;
  margin-top: 30px;
}

.grid-container {
  display: grid;
  grid-template-columns: repeat(10, 50px);
  grid-template-rows: repeat(10, 50px);
  gap: 4px;
  justify-content: center;
  margin-bottom: 20px;
}

.grid-cell {
  width: 50px;
  height: 50px;
  border: 2px solid #ccc;
  border-radius: 8px;
  box-sizing: border-box;
  transition: all 0.2s ease-in-out;
}

.cell {
  background-color: #f0f0f0;
}

.obstacle {
  background-color: #444;
}

.sprite {
  background-color: #4caf50;
  animation: pop-in 0.2s ease;
}

.treasure {
  background-color: gold;
  background-image: url('https://img.icons8.com/emoji/48/treasure-chest.png');
  background-repeat: no-repeat;
  background-position: center;
  background-size: 32px;
}

.path {
  background-color: lightblue;
}

.animated {
  transition: transform 0.2s ease;
}

.controls {
  margin-top: 10px;
}

button {
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
  border-radius: 12px;
  background-color: #007bff;
  color: white;
  border: none;
}

button:hover {
  background-color: #0056b3;
}

@keyframes pop-in {
  0% {
    transform: scale(0.5);
    opacity: 0.2;
  }

  100% {
    transform: scale(1);
    opacity: 1;
  }
}
</style>
