<template>
  <section class="bg-white px-3 py-3">
    <div class="overflow-x-auto">
      <table class="table">
        <!-- head -->
        <thead>
        <tr>
          <th>Name</th>
          <th>Aktionen</th>
        </tr>
        </thead>
        <tbody>
        <!-- row 1 -->
        <tr v-for="(liga,index) in games" :class="{
          'bg-gray-400': index % 2 == 0,
          'bg-gray-300': index % 2 != 0
        }">
          <td>
            {{ liga.name }} - {{new Date(liga.date).toLocaleDateString('de')}} - {{ getTeamNames(liga) }}
          </td>
          <td>
            <a :href="'/de/game/'+liga.id" class="btn btn-sm btn-primary">
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
import {useRoute} from "vue-router";
import {addBreadcrumb} from "@/utils/breadcrumb";

const pb = usePocketBase()
const route = useRoute()

const games = ref([]);

const load = async () => {
  games.value = (await pb.collection('league_game').getList(1, 10, {
    filter: 'league="' + route.params.id + '"',
    expand: 'teams',
    sort: '-created'
  })).items;

  addBreadcrumb({
    code: 'dashboard',
    icon: 'question-mark',
    link: 'dashboard'
  })
}

const getTeamNames = (liga) => {
  if (!liga.expand?.teams || liga.expand.teams.length === 0) return '';
  const teamNames = liga.expand.teams.map(team => team.name);
  return teamNames.join(' vs. ');
}

onMounted(() => {
  load();
})
</script>