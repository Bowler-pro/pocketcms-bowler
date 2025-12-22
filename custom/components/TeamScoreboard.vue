<template>
  <section class="card col-span-6 md:col-span-3">
    <section class="flex justify-between">
      <h1 class="font-bold text-primary">{{ team.name }}</h1>
      <section class="actions space-x-3">
        <button v-if="isPlayer" @click="add()" class="btn btn-primary btn-sm">
          <font-awesome-icon :icon="['fas', 'play']"/>
        </button>
        <button v-if="!isLeader && isPlayer" @click="modalBaker=true" class="btn btn-primary btn-sm">
          <font-awesome-icon :icon="['fas', 'cake-candles']"/>
        </button>
      </section>
    </section>
    <h2 class="font-bold">Spiele:</h2>
    <div class="overflow-x-auto" v-if="games.length">
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
        <tr v-for="(spiel,index) in games" class="my-3">
          <td>
            <span>
              Spiel #{{ index + 1 }}
              - <a :href="'/de/player/'+spiel.expand.player.id" class="text-primary">
              <b>
                {{ spiel.expand.player.name }}
              </b>
            </a>
            </span>
          </td>
          <td>
            {{ spiel.round }}
          </td>
          <td>
            {{ spiel.score }}
          </td>
          <td class="space-x-3">
            <a :href="'/de/game/edit?game='+spiel.id"
               v-if="spiel.expand?.player?.id == pb.authStore.record.id && spiel.locked == false"
               class="btn btn-sm btn-primary ml-3">bearbeiten</a>
            <a :href="'/de/game/edit?game='+spiel.id" v-if="spiel.expand?.player?.id != pb.authStore.record.id && isTeamLeader && spiel.locked == false"
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
    <div v-else>
      <section class="alert alert-error">
        Keine Spiele
      </section>
    </div>
    <h2 class="font-bold">Baker:</h2>
    <section class="flex headline justify-between">
      <section class="actions">
        <a :href="'/de/baker/edit?game='+spiel.id" v-if="isTeamLeader && spiel?.locked == false"
           class="btn btn-info btn-sm">bearbeiten</a>
      </section>
    </section>
    <div v-if="baker.length == 0">
      <section class="alert alert-error">
        Keine Baker
      </section>
    </div>
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
        <tr v-for="(spiel,index) in baker" class="my-3">
          <td>
            Baker #{{index+1}} - <a :href="'/de/team/'+spiel.expand.team.id" class="text-primary">
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
    </div>
    <div class="">
      {{score}}
    </div>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import {FontAwesomeIcon} from "@fortawesome/vue-fontawesome";

const emit = defineEmits(['score','result']);
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
const modalBaker = ref(false);

const load = async () => {
  pb.autoCancellation(false);
  games.value = (await pb.collection('players_game').getList(1, 50, {
    filter: 'game="' + props.game + '" && player.team ~"' + props.team + '"',
    expand: 'player',
    sort: '-round,created'
  })).items;
  baker.value = (await pb.collection('teams_baker').getList(1, 50, {
    filter: 'game="' + props.game + '" && team ="' + props.team + '"',
    expand: 'team'
  })).items;
  team.value = await pb.collection('teams').getOne(props.team);
}

const score = computed(() => {
  let points = 0;
  baker.value.forEach((game) => {
    points += game.score;
  })

  games.value.forEach((game) => {
    points += game.score
  })

  emit('score', points)
  return points;
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