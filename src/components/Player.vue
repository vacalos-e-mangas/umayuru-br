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

const animationFrameId = ref<number | null>(null);
const isFullscreen = ref(false);

onMounted(() => {
  loadYoutubeAPI();
  document.addEventListener("fullscreenchange", onFullscreenChange);
});

onBeforeUnmount(() => {
  document.removeEventListener("fullscreenchange", onFullscreenChange);
  if (animationFrameId.value) cancelAnimationFrame(animationFrameId.value);
  if (player.value && player.value.destroy) player.value.destroy();
  if (octopus.value) octopus.value.dispose();
});

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

async function onPlayerReady() {
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

async function toggleFullscreen() {
  if (!videoWrapper.value) return;

  if (!document.fullscreenElement) {
    try {
      await videoWrapper.value.requestFullscreen();
    } catch (err) {
      console.error(`Error attempting to enable fullscreen: ${err}`);
    }
  } else {
    if (document.exitFullscreen) {
      document.exitFullscreen();
    }
  }
}

function onFullscreenChange() {
  isFullscreen.value = !!document.fullscreenElement;
}

watch(
  () => props.videoId,
  async (newId) => {
    if (player.value && typeof player.value.loadVideoById === "function") {
      // loadVideoById carrega o vídeo e dá autoplay, cueVideoByID não dá autoplay
      player.value.cueVideoById(newId);
      await loadSubtitleTrack(props.subtitleUrl);
    }
  },
);
</script>

<template>
  <div class="responsive-container">
    <span>RECOMENDAÇÃO PARA SMARTPHONES: Assista em tela cheia e no modo paisagem</span>

    <div ref="videoWrapper" class="video-wrapper">
      <div ref="playerContainer" class="yt-element"></div>

      <canvas
        ref="subtitleCanvas"
        class="subtitle-overlay"
        width="1920"
        height="1080"
      ></canvas>

      <div class="custom-controls">
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
  max-width: 1280px; /* Optional: limit max size */
  margin: 0 0; /* Center it */
}

.video-wrapper {
  position: relative;
  width: 100%;
  /* Modern CSS for 16:9 Aspect Ratio */
  aspect-ratio: 16 / 9;
  background: #000;
  overflow: hidden;
}

@supports not (aspect-ratio: 16 / 9) {
  .video-wrapper {
    height: 0;
    padding-bottom: 56.25%; /* 16:9 calculation */
  }
}

.video-wrapper:fullscreen {
  width: 100vw;
  height: 100vh;
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
  pointer-events: none; /* Crucial: Lets clicks pass through to YouTube */
  z-index: 10;
}

/* Custom Controls */
.custom-controls {
  position: absolute;
  top: 50%;
  right: 10px;
  z-index: 20; /* Must be higher than subtitles (10) */
  opacity: 0; /* Hide by default */
  transition: opacity 0.3s;
}

/* Show controls on hover */
.video-wrapper:hover .custom-controls {
  opacity: 1;
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
