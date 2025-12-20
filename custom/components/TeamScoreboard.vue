<template>
  <section class="card col-span-6 md:col-span-3">
    <section class="flex justify-between">
      <h1 class="font-bold text-primary">{{ team.name }}</h1>
      <section class="actions">
        <button v-if="isPlayer" @click="add()" class="btn btn-primary btn-sm">
          <font-awesome-icon :icon="['fas', 'plus']"/>
        </button>
      </section>
    </section>
    <h2 class="font-bold">Spiele:</h2>
    <div class="overflow-x-auto">
      <table class="table">
        <!-- head -->
        <thead>
        <tr>
          <th>Name - Spieler</th>
          <th>Score</th>
          <th>Aktionen</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="spiel in games" class="my-3">
          <td>
            <span>
              {{ spiel.name }}
              - <a :href="'/de/player/'+spiel.expand.player.id" class="text-primary">
              <b>
                {{ spiel.expand.player.name }}
              </b>
            </a>
            </span>
          </td>
          <td>
            {{ spiel.score }}
          </td>
          <td class="space-x-3">
            <a :href="'/de/game/edit?game='+spiel.id"
               v-if="spiel.expand?.player?.id == pb.authStore.record.id && spiel.locked == false"
               class="btn btn-sm btn-primary ml-3">bearbeiten</a>
            <a :href="'/de/game/edit?game='+spiel.id" v-if="isTeamLeader && spiel.locked == false"
               class="btn btn-info btn-sm">
              <span>bearbeiten</span>
              <font-awesome-icon :icon="['fas', 'user-tie']"/>
            </a>
            <a :href="'/de/game/confirm?game='+spiel.id" v-if="props.isLeader && spiel.locked == false"
               class="btn btn-success btn-sm">
              <font-awesome-icon :icon="['fas', 'check']"/>
            </a>
          </td>
        </tr>
        </tbody>
      </table>
    </div>
    <h2 class="font-bold">Baker:</h2>
    <div v-if="baker.length == 0">Kein Baker</div>
    <div v-else class="overflow-x-auto">
      <table class="table">
        <!-- head -->
        <thead>
        <tr>
          <th>Team Name</th>
          <th>Score</th>
          <th>Aktionen</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="spiel in baker" class="my-3">
          <td>
            <a :href="'/de/team/'+spiel.expand.team.id" class="text-primary">
              {{ spiel.expand.team.name }}
            </a>
          </td>
          <td>
            {{ spiel.score }}
          </td>
          <td>
            <a :href="'/de/baker/edit?game='+spiel.id"
               v-if="spiel.expand?.player?.id == pb.authStore.record?.id && spiel.locked == false"
               class="btn btn-sm btn-primary ml-3">bearbeiten</a>
            <a :href="'/de/baker/edit?game='+spiel.id" v-if="isTeamLeader && spiel.locked == false"
               class="btn btn-info btn-sm">bearbeiten</a>
            <a :href="'/de/baker/confirm?game='+spiel.id" v-if="props.isLeader && spiel.locked == false"
               class="btn btn-success btn-sm">
              <font-awesome-icon :icon="['fas', 'check']"/>
            </a>
          </td>
        </tr>
        </tbody>
      </table>
<div class="hidden">{{score}}</div>
    </div>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import {FontAwesomeIcon} from "@fortawesome/vue-fontawesome";

const emit = defineEmits(['score']);
const pb = usePocketBase()
const props = defineProps({
  team: {
    type: String, required: true
  },
  isLeader: {
    type: Boolean, required: true
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

const isTeamLeader = computed(() => {
  return team.value.leader ? toRaw(team.value.leader).includes(pb.authStore.record.id) : false;
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