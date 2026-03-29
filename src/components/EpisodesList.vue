<template>
  <div id="playlist-side">
    <div class="playlist-header">
      <button class="playlist-tab" @click="whichList = 0">Umayuru</button>
      <button class="playlist-tab" @click="whichList = 1">Pretty Gray</button>
    </div>
    <div id="playlist-container">
      <div
        class="playlist-item"
        v-for="item in list"
        :key="item.id"
        @click="onClick(item)"
      >
        <div class="playlist-image-container">
          <img
            class="playlist-image"
            width="107"
            height="60"
            :src="`https://img.youtube.com/vi/${item.videoId}/mqdefault.jpg`"
          />
        </div>
        <div class="playlist-info-container">
          <p>{{ item.name }}</p>
        </div>
      </div>
    </div>
  </div>
</template>
<script setup lang="ts">
import { computed, ref } from "vue";

type Episode = {
  id: string;
  name: string;
  videoId: string;
};
const emit = defineEmits<{
  updateEpisode: [episode: Episode, prettyGray: boolean];
}>();

const whichList = ref(0);

const listUmaYuru = [
  {
    id: "01",
    name: "Episódio 1: Mensageiras da tempestade",
    videoId: "Q6ur7-o_YdI",
  },
  { id: "02", name: "Episódio 2: A silenciosa escura", videoId: "7y4DpHsz6Wg" },
  {
    id: "03",
    name: "Episódio 3: Coqueteis no bar após o pôr do sol",
    videoId: "oD4nNApSrd0",
  },
  {
    id: "04",
    name: "Episódio 4: Tsuyoshi fica bombada",
    videoId: "2U2OTcgyubY",
  },
  {
    id: "05",
    name: "Episódio 5: Já estamos de saco cheio!",
    videoId: "36NHQCYjBvs",
  },
  {
    id: "06",
    name: "Episódio 6: Resolvido até o café da manhã",
    videoId: "sLV7WHXcOu0",
  },
  {
    id: "07",
    name: "Episódio 7: A favorita da Sirius",
    videoId: "uXauD1uRZy0",
  },
  {
    id: "08",
    name: "Episódio 8: Entrevista da Etsuko!",
    videoId: "u0LuV8HNd8k",
  },
  { id: "09", name: "Episódio 9: Initial Derby!", videoId: "4KzUSqUW9LU" },
  {
    id: "10",
    name: "Episódio 10: Uma Musume e histórias de terror",
    videoId: "proK-e-v3Eg",
  },
  { id: "11", name: "Episódio 11: Chaos e Ordem", videoId: "ag8dpHBhoCY" },
  {
    id: "12",
    name: "Episódio 12: Yakuza: Como um Cavalo",
    videoId: "aVK4qqhbMhA",
  },
  {
    id: "13",
    name: "Episódio 13: Bota as mão para o céu! UmaYuru Rap!",
    videoId: "_XvzHdO7nrk",
  },
  {
    id: "14",
    name: "Episódio 14: Similares não se dão bem",
    videoId: "xh3FU5NoCzQ",
  },
  { id: "15", name: "Episódio 15: O fim das Cercas", videoId: "VfqRYC8_UPo" },
  { id: "16", name: "Episódio 16: As garras do mal", videoId: "Iz1xs-gZF-g" },
  { id: "17", name: "Episódio 17: Ninja novatas!", videoId: "Vwbj7I47qas" },
  {
    id: "18",
    name: "Episódio 18: O campeonato da expert da Rudolf!",
    videoId: "EOhRP6r9SOQ",
  },
  { id: "19", name: "Episódio 19: Pela vontade da G", videoId: "Moka2TV2eLs" },
  {
    id: "20",
    name: "Episódio 20: O covil das apostadoras",
    videoId: "WrymFqwvaoo",
  },
  { id: "21", name: "Episódio 21: Canal Fine Ramen!", videoId: "HmRbWlbXtIY" },
  {
    id: "22",
    name: "Episódio 22: Cozinha gostosa com a Spe e a Ogurin!",
    videoId: "2Jrb-wi87Dw",
  },
  { id: "23", name: "Episódio 23: Quer Derby comigo?", videoId: "lS54NCyHxko" },
  { id: "24", name: "Episódio 24: Aonde a luz brilha", videoId: "iGExuigHsUw" },
] as Episode[];

const listPrettyGray = [
  {
    id: "01",
    name: "Episódio 1: Duelo brutal",
    videoId: "zxJXquB3YiM",
  },
  {
    id: "02",
    name: "Episódio 2: Os arquivos de casos da Seiun: A Maldição de Yatsuhashi",
    videoId: "cb73PRzHo4w",
  },
  {
    id: "03",
    name: "Episódio 3: A onda do destino - o revezamento turbulento",
    videoId: "AGUgeelUUuo",
  },
  {
    id: "04",
    name: "Episódio 4: A arte de assistir",
    videoId: "a1q3XU_6Jgk",
  },
];

const list = computed(() => {
  return whichList.value ? listPrettyGray : listUmaYuru;
});

function onClick(episode: Episode) {
  emit("updateEpisode", episode, whichList.value ? true : false);
}
</script>
<style scoped>
#playlist-side {
  width: 25%;
  display: flex;
  flex-direction: column;
  padding-left: 0.5em;
}

.playlist-header {
  width: 100%;
  height: 50px;
  display: flex;
  margin: 10px 0;
}

.playlist-tab {
  width: 50%;
  height: 50px;
  border: 1px;
  background-color: #505050;
  color: white
}

#playlist-container {
  flex: 1 1 auto;
  overflow-y: auto;
  width: 100%;
  scrollbar-width: thin;
}

#playlist-container::-webkit-scrollbar {
  width: 0.5em;
}

#playlist-container::-webkit-scrollbar-thumb {
  border-radius: 1em;
}
.playlist-item {
  display: flex;
  gap: 0.5em;
  padding: 0.2em;
  border-radius: 0.4rem;
  cursor: pointer;
}

.playlist-item-selected {
  background-color: #e1e1e1;
}

.playlist-image-container {
  height: 60px;
  width: auto;
  flex: 0 0 100px;
  display: flex;
  align-items: center;
}

.playlist-image {
  height: 100%;
  border-radius: 0.5em;
}

.playlist-info-container {
  flex: 1 1 auto;
  text-align: left;
  font-size: 0.8em;
  font-weight: bold;
  display: flex;
  align-items: center;
}
</style>
