<template>
  <section class="bg-white px-3 py-3">
    <div class="overflow-x-auto">
      <table class="table">
        <!-- head -->
        <thead>
        <tr>
          <th>League ID</th>
          <th>Game Name</th>
          <th>Player Name</th>
          <th>Player Score</th>
        </tr>
        </thead>
        <tbody>
        <!-- row 1 -->
        <tr v-for="(liga,index) in games" :class="{
          'bg-gray-400': index % 2 == 0,
          'bg-gray-300': index % 2 != 0
        }">
          <td>
            <LigaName :identifier="liga.league"/>
          </td>
          <td>
            {{ liga.name }}
          </td>
          <td>
            <PlayerName :identifier="liga.player"/>
            -
            <PlayerTeam :identifier="liga.player" />
          </td>
          <td>
            {{ liga.score }}
          </td>
          <td>
            <a :href="'/de/game?id='+liga.id" class="btn btn-sm btn-primary">
              anschauen
            </a>
          </td>
        </tr>
        </tbody>
      </table>
    </div>
  </section>
</template>

<script setup lang="ts">

import {usePocketBase} from "@/utils/pocketbase";
import {onMounted} from "vue";
import {useRoute} from 'vue-router'
import PlayerName from "../components/PlayerName.vue";
import PlayerTeam from "../components/PlayerTeam.vue";

const pb = usePocketBase()
const route = useRoute()

const games = ref([]);

const load = async () => {
  games.value = (await pb.collection('games').getList(1, 10, {
    filter: 'id="' + route.query.id + '"',
  })).items;
}

onMounted(() => {
  if (!route.query.id) {
    navigateTo('/dashboard')
  }
  load();
})
</script>