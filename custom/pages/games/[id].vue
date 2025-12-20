<template>
  <section class="bg-white px-3 py-3">
    <div class="overflow-x-auto">
      <table class="table">
        <!-- head -->
        <thead>
        <tr>
          <th>Name</th>
          <th>Job</th>
        </tr>
        </thead>
        <tbody>
        <!-- row 1 -->
        <tr v-for="(liga,index) in games" :class="{
          'bg-gray-400': index % 2 == 0,
          'bg-gray-300': index % 2 != 0
        }">
          <td>
            {{ liga.name }}
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
    filter: 'league="' + route.params.id + '"'
  })).items;

  addBreadcrumb({
    code: 'dashboard',
    icon: 'question-mark',
    link: 'dashboard'
  })
}

onMounted(() => {
  load();
})
</script>