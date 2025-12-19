<template>
  <section class="card col-span-6 md:col-span-3">
    <section class="flex justify-between">
      <h1 class="font-bold text-primary">{{ team.name }}</h1>
      <section class="actions">
        <button v-if="isPlayer" @click="add()" class="btn btn-primary btn-sm">
          add score
        </button>
      </section>
    </section>
    <h2 class="font-bold">Spiele:</h2>
    <div class="overflow-x-auto">
      <table class="table">
        <!-- head -->
        <thead>
        <tr>
          <th>Name - Spieler - Score</th>
          <th>Name</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="spiel in games" class="my-3">
          <td>
            <span>
                  {{ spiel.name }} - {{ spiel.expand.player.name }} - {{ spiel.score }}
            </span>
          </td>
          <td>
            <a :href="'/de/game/edit?game='+spiel.id" v-if="spiel.expand.player.id == pb.authStore.record.id"
               class="btn btn-sm btn-primary ml-3">bearbeiten</a>
          </td>
        </tr>
        </tbody>
      </table>
    </div>
    <h2 class="font-bold">Baker:</h2>
    <div v-if="baker.length == 0">Test</div>
    <div v-else class="overflow-x-auto">
      <table class="table">
        <!-- head -->
        <thead>
        <tr>
          <th>Name - Spieler - Score</th>
          <th>Name</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="spiel in baker" class="my-3">
          <td>
            <span>
                  {{ spiel.expand.team.name }} - {{ spiel.score }}
            </span>
          </td>
          <td>
            <a :href="'/de/game/edit?game='+spiel.id" v-if="spiel.expand.player.id == pb.authStore.record.id"
               class="btn btn-sm btn-primary ml-3">bearbeiten</a>
          </td>
        </tr>
        </tbody>
      </table>
    </div>

    {{ score }}
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";

const emit = defineEmits(['score']);
const pb = usePocketBase()
const props = defineProps({
  team: {
    type: String, required: true
  },
  game: {
    type: String, required: true
  },
});
const team = ref({});
const games = ref([]);
const baker = ref([]);

const load = async () => {
  pb.autoCancellation(false);
  games.value = (await pb.collection('players_game').getList(1, 50, {
    filter: 'game="' + props.game + '" && player.team ~"' + props.team + '"',
    expand: 'player',
    sort: 'name'
  })).items;
  baker.value = (await pb.collection('teams_baker').getList(1, 50, {
    filter: 'game="' + props.game + '" && team ="' + props.team + '"',
    expand: 'team'
  })).items;
  team.value = await pb.collection('teams').getOne(props.team);
}

const score = computed(() => {
  let tmp = 0;
  baker.value.forEach((game) => {
    tmp += game.score;
  })

  games.value.forEach((game) => {
    tmp += game.score;
  })
  emit('score', tmp)
  return tmp;
});

const isPlayer = computed(() => {
  return pb.authStore.record.team == props.team;
});

const add = () => {
  navigateTo('/game/add/?team=' + props.team + '&game=' + props.game)
}

onMounted(async () => {
  await load();

  // Subscribe to changes in any players_game record
  pb.collection('players_game').subscribe('*', function (e) {
    load();
  }, {});
  pb.collection('teams_baker').subscribe('*', function (e) {
    load();
  }, {});

});
</script>