<script setup lang="ts">
import Player from "./components/Player.vue";
import EpisodesList from "./components/EpisodesList.vue";
import { ref } from "vue";

type Episode = {
  id: string;
  name: string;
  videoId: string;
};

const isPrettyGray = ref(false);

const currentEpisode = ref<Episode>({
  id: "01",
  name: "Episódio 1: Mensageiras da tempestade",
  videoId: "Q6ur7-o_YdI",
} as Episode);

function update(episode: Episode, prettyGray: boolean) {
  currentEpisode.value = episode;
  isPrettyGray.value = prettyGray;
}
</script>

<template>
  <div class="page-container">
    <Player
      :videoId="currentEpisode.videoId"
      :subtitleUrl="currentEpisode.id"
      :isPrettyGray="isPrettyGray"
    />
    <EpisodesList @updateEpisode="update" />
  </div>
</template>

<style scoped>
.page-container {
  width: 90vw;
  height: 95vh;
  hyphens: auto;
  overflow-wrap: anywhere;
  text-align: center;
  display: flex;
}
</style>
