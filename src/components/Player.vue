<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, shallowRef, watch } from "vue";

const props = defineProps({
  videoId: { type: String, required: true },
  subtitleUrl: { type: String, required: true },
  isPrettyGray: { type: Boolean, required: true },
});

const videoWrapper = ref<HTMLDivElement | null>(null);
const playerContainer = ref<HTMLDivElement | null>(null);
const subtitleCanvas = ref<HTMLCanvasElement | null>(null);

const player = shallowRef<any>(null); // YT.Player
const octopus = shallowRef<any>(null);
const iframeElement = ref<HTMLElement | null>(null);

const animationFrameId = ref<number | null>(null);
const isFullscreen = ref(false);
const controlsVisible = ref(true);
let hideControlsTimer: ReturnType<typeof setTimeout> | null = null;

onMounted(() => {
  loadYoutubeAPI();
  document.addEventListener("fullscreenchange", onFullscreenChange);
});

onBeforeUnmount(() => {
  document.removeEventListener("fullscreenchange", onFullscreenChange);
  if (animationFrameId.value) cancelAnimationFrame(animationFrameId.value);
  if (player.value && player.value.destroy) player.value.destroy();
  if (octopus.value) octopus.value.dispose();
  if (hideControlsTimer) clearTimeout(hideControlsTimer);
});

// Controla o botão de FullScreen
function showControls() {
  controlsVisible.value = true;

  const canvas = subtitleCanvas.value;
  if (canvas) canvas.style.pointerEvents = "none";
  if (iframeElement.value) iframeElement.value.style.pointerEvents = "auto";

  if (hideControlsTimer) clearTimeout(hideControlsTimer);
  hideControlsTimer = setTimeout(hideControls, 3000);
}

function hideControls() {
  controlsVisible.value = false;

  const canvas = subtitleCanvas.value;
  // Canvas intercepts mouse again so we can detect movement
  if (canvas) canvas.style.pointerEvents = "auto";
  if (iframeElement.value) iframeElement.value.style.pointerEvents = "none";
}

function onCanvasMouseMove() {
  showControls();
}

// Carrega YouTube
function loadYoutubeAPI() {
  const win = window as any;
  if (win.YT && win.YT.Player) {
    initPlayer();
  } else {
    if (
      !document.querySelector(
        'script[src="https://www.youtube.com/iframe_api"]',
      )
    ) {
      const tag = document.createElement("script");
      tag.src = "https://www.youtube.com/iframe_api";
      const firstScriptTag = document.getElementsByTagName("script")[0];
      if (firstScriptTag) {
        firstScriptTag.parentNode?.insertBefore(tag, firstScriptTag);
      }
    }

    win.onYouTubeIframeAPIReady = () => {
      initPlayer();
    };
  }
}

function initPlayer() {
  if (player.value) return;

  const win = window as any;

  player.value = new win.YT.Player(playerContainer.value, {
    height: "100%",
    width: "100%",
    videoId: props.videoId,
    playerVars: {
      playsinline: 1,
      rel: 0,
      modestbranding: 1,
      controls: 1, // Controles normais do YT
      fs: 0, // Remove a função nativa de fullscreen do YT
    },
    events: {
      onReady: onPlayerReady,
    },
  });
}

// Prepara o Octopus
async function onPlayerReady() {
  iframeElement.value = playerContainer.value?.querySelector("iframe") ?? null;
  await initOctopus();
}

async function loadSubtitleTrack(subId: string) {
  try {
    const subLocation = props.isPrettyGray
      ? `/gray/${subId}.ass`
      : `/subs/${subId}.ass`;
    const response = await fetch(subLocation);
    if (!response.ok) throw new Error("Network response was not ok");
    const subText = await response.text();

    if (octopus.value) {
      octopus.value.setCurrentTime(0); // reseta o tempo só por precaução
      octopus.value.setTrack(subText);
    } else {
      return subText;
    }
  } catch (error) {
    console.error("Failed to load subtitles:", error);
  }
}

async function initOctopus() {
  const win = window as any;
  if (!win.SubtitlesOctopus) return;

  const subText = await loadSubtitleTrack(props.subtitleUrl);

  if (!subText || octopus.value) return;

  octopus.value = new win.SubtitlesOctopus({
    canvas: subtitleCanvas.value,
    subContent: subText,
    fonts: [
      "/fonts/BerkshireSwash-Regular.ttf",
      "/fonts/DelaGothicOne-Regular.ttf",
      "/fonts/Grandstander-Black.ttf",
      "/fonts/Grandstander-Bold.ttf",
      "/fonts/Grandstander-Regular.ttf",
      "/fonts/Japan wave.ttf",
      "/fonts/MochiyPopPOne-Regular.ttf",
      "/fonts/MPLUSRounded1c-Bold.ttf",
      "/fonts/NotoSansJP-Bold_0.otf",
      "/fonts/NotoSerifJP-Bold.otf",
      "/fonts/OpenSans-Bold_0.ttf",
      "/fonts/Pangolin-Regular.ttf",
      "/fonts/PermanentMarker-Regular.ttf",
      "/fonts/RobotoMono-VariableFont_wght.ttf",
      "/fonts/SedgwickAve-Regular.ttf",
      "/fonts/SedgwickAveDisplay-Regular.ttf",
      "/fonts/YoungDreamer.otf",
    ],
    workerUrl: "/subtitles-octopus-worker.js",
  });

  startSyncLoop();
}

function startSyncLoop() {
  const sync = () => {
    if (
      player.value &&
      octopus.value &&
      typeof player.value.getCurrentTime === "function"
    ) {
      octopus.value.setCurrentTime(player.value.getCurrentTime());
    }
    animationFrameId.value = requestAnimationFrame(sync);
  };
  sync();
}

// Tratativas do fullscreen
async function toggleFullscreen() {
  if (!videoWrapper.value) return;

  if (!document.fullscreenElement) {
    try {
      await videoWrapper.value.requestFullscreen();
    } catch (err) {
      console.error(`Erro ao tentar entrar em fullscreen: ${err}`);
    }
  } else {
    if (document.exitFullscreen) {
      document.exitFullscreen();
    }
  }
}

function onFullscreenChange() {
  isFullscreen.value = !!document.fullscreenElement;
  if (isFullscreen.value) {
    showControls();
    requestAnimationFrame(repositionCanvas);
  } else {
    showControls();
    if (hideControlsTimer) clearTimeout(hideControlsTimer);
    resetCanvasPosition();
  }
}

/**
 * MUITO PUTO COM ISTO
 * O SubtitlesOctopus renderiza no canvas na resolução 1920x1080 (16:9)
 * quando a tela em si não é 16:9 e tenta colocar em fullscreen,
 * o navegador coloca as barras pretas na vertical ou horizontal do vídeo
 * DENTRO DO ELEMENTO FULLSCREEN.
 * Precisa espelhar esse offset no canvas das legendas para alinhar
 * e não distorcer a porra toda
 */
function repositionCanvas() {
  const canvas = subtitleCanvas.value;
  if (!canvas) return;

  if (!document.fullscreenElement) {
    resetCanvasPosition();
    return;
  }

  const screenW = window.screen.width;
  const screenH = window.screen.height;
  const videoAspect = 16 / 9;
  const screenAspect = screenW / screenH;

  let videoW: number, videoH: number, offsetX: number, offsetY: number;

  if (screenAspect > videoAspect) {
    // Pillarboxed: barras na direita e esquerda
    videoH = screenH;
    videoW = screenH * videoAspect;
    offsetX = (screenW - videoW) / 2;
    offsetY = 0;
  } else {
    // Letterboxed: barras em cima e embaixo
    videoW = screenW;
    videoH = screenW / videoAspect;
    offsetX = 0;
    offsetY = (screenH - videoH) / 2;
  }

  canvas.style.width = `${videoW}px`;
  canvas.style.height = `${videoH}px`;
  canvas.style.left = `${offsetX}px`;
  canvas.style.top = `${offsetY}px`;
}

function resetCanvasPosition() {
  const canvas = subtitleCanvas.value;
  if (!canvas) return;
  canvas.style.width = "";
  canvas.style.height = "";
  canvas.style.left = "";
  canvas.style.top = "";
}

watch(
  () => props.videoId,
  async (newId) => {
    if (player.value && typeof player.value.loadVideoById === "function") {
      player.value.cueVideoById(newId);
      await loadSubtitleTrack(props.subtitleUrl);
    }
  },
);
</script>

<template>
  <div class="responsive-container">
    <span
      >RECOMENDAÇÃO PARA SMARTPHONES: Assista em tela cheia e no modo
      paisagem</span
    >

    <div
      ref="videoWrapper"
      class="video-wrapper"
    >
      <div ref="playerContainer" class="yt-element"></div>

      <canvas
        ref="subtitleCanvas"
        class="subtitle-overlay"
        width="1920"
        height="1080"
        @mousemove="onCanvasMouseMove"
        @touchstart="onCanvasMouseMove"
      ></canvas>

      <div class="custom-controls" :class="{ visible: controlsVisible }">
        <button @click="toggleFullscreen" class="fs-button">
          {{ isFullscreen ? "Sair da Tela Cheia" : "Tela Cheia" }}
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.responsive-container {
  width: 100%;
  max-width: 1280px;
  margin: 0 0;
}

.video-wrapper {
  position: relative;
  width: 100%;
  aspect-ratio: 16 / 9;
  background: #000;
  overflow: hidden;
}

@supports not (aspect-ratio: 16 / 9) {
  .video-wrapper {
    height: 0;
    padding-bottom: 56.25%;
  }
}

.video-wrapper:fullscreen {
  width: 100vw;
  height: 100vh;
  overflow: visible;
}

.yt-element {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.subtitle-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 10;
  transition:
    width 0.1s,
    height 0.1s,
    top 0.1s,
    left 0.1s;
}

.custom-controls {
  position: absolute;
  top: 50%;
  right: 10px;
  z-index: 20;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.4s ease;
}

.custom-controls.visible {
  opacity: 1;
  pointer-events: auto;
}

.fs-button {
  background: rgba(0, 0, 0, 0.6);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.3);
  padding: 8px 12px;
  cursor: pointer;
  border-radius: 4px;
  font-family: sans-serif;
  font-size: 14px;
}

.fs-button:hover {
  background: rgba(0, 0, 0, 0.8);
}
</style>
