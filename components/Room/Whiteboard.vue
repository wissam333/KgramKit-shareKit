<template>
  <div class="wb-wrap">
    <div class="wb-toolbar">
      <div class="tool-group">
        <button
          v-for="tool in tools"
          :key="tool.key"
          class="wb-tool"
          :class="{ active: activeTool === tool.key }"
          :title="tool.label"
          @click="activeTool = tool.key"
        >
          <Icon :name="tool.icon" size="16" />
        </button>
      </div>

      <div class="tb-sep" />

      <div class="tool-group colors">
        <button
          v-for="c in colors"
          :key="c"
          class="color-dot"
          :class="{ active: strokeColor === c }"
          :style="{ background: c }"
          @click="strokeColor = c"
        />
        <label class="color-picker-wrap" :title="$t('customColor')">
          <input
            v-model="strokeColor"
            type="color"
            class="color-picker-input"
          />
          <Icon name="mdi:palette" size="14" />
        </label>
      </div>

      <div class="tb-sep" />

      <div class="tool-group">
        <input
          v-model.number="strokeWidth"
          type="range"
          min="1"
          max="32"
          class="stroke-slider"
        />
        <span
          class="stroke-preview"
          :style="{
            width: Math.min(strokeWidth, 20) + 'px',
            height: Math.min(strokeWidth, 20) + 'px',
            background: strokeColor,
          }"
        />
      </div>

      <div class="tb-sep" />

      <div class="tool-group">
        <button class="wb-tool" :title="$t('undo')" @click="undo">
          <Icon name="mdi:undo" size="16" />
        </button>
        <button class="wb-tool" :title="$t('clear')" @click="clearBoard">
          <Icon name="mdi:trash-can-outline" size="16" />
        </button>
        <button class="wb-tool" :title="$t('download')" @click="downloadBoard">
          <Icon name="mdi:download" size="16" />
        </button>
      </div>

      <div class="drawing-members">
        <div
          v-for="[peerId, state] in remoteStates"
          :key="peerId"
          class="drawing-pip"
          :class="{ active: state.isDrawing }"
          :title="state.name"
        >
          {{ state.name.charAt(0).toUpperCase() }}
        </div>
      </div>
    </div>

    <div ref="containerEl" class="wb-canvas-wrap">
      <canvas
        ref="canvasEl"
        class="wb-canvas"
        :style="{ cursor: cursorStyle }"
        @mousedown="startDraw"
        @mousemove="onMove"
        @mouseup="endDraw"
        @mouseleave="endDraw"
        @touchstart.prevent="startDraw"
        @touchmove.prevent="onMove"
        @touchend.prevent="endDraw"
      />
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch, nextTick } from "vue";
import { useResizeObserver } from "@vueuse/core";

const props = defineProps({
  room: { type: Object, required: true },
});

const { t } = useI18n();
const canvasEl = ref(null);
const containerEl = ref(null);
const activeTool = ref("pen");
const strokeColor = ref("#14b8a6");
const strokeWidth = ref(3);
const isDrawing = ref(false);

// State tracking for all participants
const remoteStates = ref(new Map());

let ctx = null;
let lastX = 0;
let lastY = 0;
let snapshot = null;
const history = [];
const MAX_HISTORY = 20;

const tools = [
  { key: "pen", icon: "mdi:draw", label: "Pen" },
  { key: "eraser", icon: "mdi:eraser", label: "Eraser" },
  { key: "line", icon: "mdi:vector-line", label: "Line" },
  { key: "rect", icon: "mdi:rectangle-outline", label: "Rectangle" },
  { key: "circle", icon: "mdi:circle-outline", label: "Circle" },
];

const colors = [
  "#14b8a6",
  "#6366f1",
  "#ef4444",
  "#f59e0b",
  "#22c55e",
  "#ffffff",
  "#0f172a",
];

const cursorStyle = computed(() =>
  activeTool.value === "eraser" ? "cell" : "crosshair",
);

// ── Internal Helpers ──────────────────────────────────────
const getPos = (e) => {
  const rect = canvasEl.value.getBoundingClientRect();
  const clientX = e.touches ? e.touches[0].clientX : e.clientX;
  const clientY = e.touches ? e.touches[0].clientY : e.clientY;
  return {
    x: (clientX - rect.left) * (canvasEl.value.width / rect.width),
    y: (clientY - rect.top) * (canvasEl.value.height / rect.height),
  };
};

const setupCtx = () => {
  if (!canvasEl.value) return;
  ctx = canvasEl.value.getContext("2d", { willReadFrequently: true });
  ctx.lineCap = "round";
  ctx.lineJoin = "round";
};

const resizeCanvas = () => {
  if (!containerEl.value || !canvasEl.value) return;
  const { width, height } = containerEl.value.getBoundingClientRect();
  if (width === 0 || height === 0) return;

  const tempCanvas = document.createElement("canvas");
  tempCanvas.width = canvasEl.value.width;
  tempCanvas.height = canvasEl.value.height;
  if (tempCanvas.width > 0) {
    tempCanvas.getContext("2d").drawImage(canvasEl.value, 0, 0);
  }

  canvasEl.value.width = width;
  canvasEl.value.height = height;
  setupCtx();
  ctx.drawImage(tempCanvas, 0, 0);
};

// ── Drawing Core ──────────────────────────────────────────
const broadcast = (payload) => {
  props.room.broadcast({ type: "board", ...payload });
};

const drawLine = (x1, y1, x2, y2, color, width, tool) => {
  if (!ctx) return;
  ctx.beginPath();
  ctx.globalCompositeOperation =
    tool === "eraser" ? "destination-out" : "source-over";
  ctx.strokeStyle = color;
  ctx.lineWidth = tool === "eraser" ? width * 5 : width;
  ctx.moveTo(x1, y1);
  ctx.lineTo(x2, y2);
  ctx.stroke();
  ctx.closePath();
};

const startDraw = (e) => {
  isDrawing.value = true;
  const { x, y } = getPos(e);
  lastX = x;
  lastY = y;

  if (activeTool.value === "pen" || activeTool.value === "eraser") {
    saveHistory();
    broadcast({
      event: "start",
      x,
      y,
      color: strokeColor.value,
      width: strokeWidth.value,
      tool: activeTool.value,
    });
  } else {
    snapshot = ctx.getImageData(
      0,
      0,
      canvasEl.value.width,
      canvasEl.value.height,
    );
  }
};

const onMove = (e) => {
  if (!isDrawing.value) return;
  const { x, y } = getPos(e);

  if (activeTool.value === "pen" || activeTool.value === "eraser") {
    drawLine(
      lastX,
      lastY,
      x,
      y,
      strokeColor.value,
      strokeWidth.value,
      activeTool.value,
    );
    broadcast({ event: "move", x, y });
    lastX = x;
    lastY = y;
  } else if (snapshot) {
    ctx.putImageData(snapshot, 0, 0);
    ctx.globalCompositeOperation = "source-over";
    ctx.strokeStyle = strokeColor.value;
    ctx.lineWidth = strokeWidth.value;
    ctx.beginPath();
    if (activeTool.value === "line") {
      ctx.moveTo(lastX, lastY);
      ctx.lineTo(x, y);
    } else if (activeTool.value === "rect") {
      ctx.strokeRect(lastX, lastY, x - lastX, y - lastY);
    } else if (activeTool.value === "circle") {
      const r = Math.sqrt((x - lastX) ** 2 + (y - lastY) ** 2);
      ctx.arc(lastX, lastY, r, 0, Math.PI * 2);
    }
    ctx.stroke();
  }
};

const endDraw = () => {
  if (!isDrawing.value) return;
  isDrawing.value = false;
  if (activeTool.value !== "pen" && activeTool.value !== "eraser") {
    saveHistory();
    broadcast({
      event: "full-sync",
      data: canvasEl.value.toDataURL("image/webp", 0.3),
    });
  }
  broadcast({ event: "end" });
};

// ── Remote Message Handling ───────────────────────────────
const handleRemoteData = (data, peerId) => {
  if (!ctx) return;

  // Get or create state for this specific peer
  if (!remoteStates.value.has(peerId)) {
    const member = props.room.members.value.find((m) => m.peerId === peerId);
    remoteStates.value.set(peerId, {
      name: member?.name || "User",
      isDrawing: false,
    });
  }
  const state = remoteStates.value.get(peerId);

  switch (data.event) {
    case "start":
      state.isDrawing = true;
      state.lastX = data.x;
      state.lastY = data.y;
      state.color = data.color;
      state.width = data.width;
      state.tool = data.tool;
      break;

    case "move":
      if (state.isDrawing) {
        drawLine(
          state.lastX,
          state.lastY,
          data.x,
          data.y,
          state.color,
          state.width,
          state.tool,
        );
        state.lastX = data.x;
        state.lastY = data.y;
      }
      break;

    case "end":
      state.isDrawing = false;
      break;

    case "full-sync":
      const img = new Image();
      img.onload = () => {
        ctx.globalCompositeOperation = "source-over";
        ctx.drawImage(img, 0, 0);
      };
      img.src = data.data;
      break;

    case "clear":
      ctx.clearRect(0, 0, canvasEl.value.width, canvasEl.value.height);
      break;
  }
};

// ── Actions ───────────────────────────────────────────────
const saveHistory = () => {
  if (history.length >= MAX_HISTORY) history.shift();
  history.push(
    ctx.getImageData(0, 0, canvasEl.value.width, canvasEl.value.height),
  );
};

const undo = () => {
  if (!history.length) return;
  ctx.putImageData(history.pop(), 0, 0);
  broadcast({
    event: "full-sync",
    data: canvasEl.value.toDataURL("image/webp", 0.3),
  });
};

const clearBoard = () => {
  ctx.clearRect(0, 0, canvasEl.value.width, canvasEl.value.height);
  broadcast({ event: "clear" });
};

const downloadBoard = () => {
  const link = document.createElement("a");
  link.download = `whiteboard-${Date.now()}.png`;
  link.href = canvasEl.value.toDataURL("image/png");
  link.click();
};

// ── Lifecycle ─────────────────────────────────────────────
let unsubscribe;

useResizeObserver(containerEl, () => {
  resizeCanvas();
});

onMounted(() => {
  setupCtx();
  nextTick(() => resizeCanvas());

  unsubscribe = props.room.onMessage((data, fromPeerId) => {
    if (data.type === "board") {
      handleRemoteData(data, fromPeerId);
    }
  });
});

onUnmounted(() => {
  unsubscribe?.();
});

// Sync full board to late joiners
watch(
  () => props.room.members.value.length,
  (newCount, oldCount) => {
    if (newCount > oldCount && canvasEl.value) {
      // Small delay to ensure the new peer's listener is ready
      setTimeout(() => {
        broadcast({
          event: "full-sync",
          data: canvasEl.value.toDataURL("image/webp", 0.3),
        });
      }, 1500);
    }
  },
);
</script>

<style scoped lang="scss">
.wb-wrap {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: var(--bg-surface);
  overflow: hidden;
}

.wb-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  border-bottom: 1px solid var(--border-color);
  flex-wrap: wrap;
}

.tool-group {
  display: flex;
  align-items: center;
  gap: 4px;
}

.wb-tool {
  width: 32px;
  height: 32px;
  border-radius: 8px;
  border: 1px solid var(--border-color);
  background: var(--bg-elevated);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--text-muted);

  &.active {
    border-color: #14b8a6;
    color: #14b8a6;
    background: rgba(20, 184, 166, 0.1);
  }
}

.tb-sep {
  width: 1px;
  height: 20px;
  background: var(--border-color);
}

.color-dot {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid transparent;
  &.active {
    border-color: var(--text-primary);
    transform: scale(1.1);
  }
}

.color-picker-wrap {
  position: relative;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  border: 1px solid var(--border-color);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  .color-picker-input {
    position: absolute;
    opacity: 0;
    inset: 0;
    cursor: pointer;
  }
}

.stroke-slider {
  width: 60px;
  accent-color: #14b8a6;
}

.drawing-members {
  display: flex;
  margin-left: auto;
  gap: 4px;
}

.drawing-pip {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: var(--border-color);
  font-size: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 2px solid transparent;
  transition: all 0.3s;

  &.active {
    border-color: #14b8a6;
    box-shadow: 0 0 8px rgba(20, 184, 166, 0.4);
    transform: scale(1.1);
  }
}

.wb-canvas-wrap {
  flex: 1;
  background: var(--bg-surface);
  position: relative;
}

.wb-canvas {
  display: block;
  width: 100%;
  height: 100%;
  touch-action: none;
}
</style>
