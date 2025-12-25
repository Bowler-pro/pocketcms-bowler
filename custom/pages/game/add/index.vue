<template>
  <section class="bg-white px-3 py-3">
    <section class="grid grid-cols-6 gap-3">
      <div class="col-span-6 md:col-span-3">
        <form @submit.prevent="add()">
          <label class="floating-label mt-3">
            <span>Your Round</span>
            <select v-model="form.round" class="select w-full">
              <option value="1">1</option>
              <option value="2">2</option>
              <option value="3">3</option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Your Player</span>
            <select v-model="form.player" class="select w-full">
              <option v-for="player in players" :value="player.id">{{player.name}}</option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Your Game</span>
            <select v-model="form.game" class="select w-full">
              <option v-for="game in games" :value="game.id">{{game.name}} - {{ getTeamNames(game) }}</option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Your Score</span>
            <input type="number" required v-model="form.score" class="validator input w-full" />
          </label>
          <section class="actions flex justify-end mt-3">
            <button  type="submit" class="btn btn-primary">
              add
            </button>
          </section>
        </form>
      </div>
    </section>
  </section>
</template>
<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";

const route = useRoute()
const router = useRouter()
const pb = usePocketBase()
const players = ref([]);
const games = ref([]);
const teams = ref([]);
const form = ref({
  game: '',
  round: 1,
  score: '',
  player: '',
});

onMounted(async ()=>{
  if(!pb.authStore.isValid){
    navigateTo('/')
  }

  form.value.player = pb.authStore.record.id;
  form.value.game = route.query.game;
  form.value.team = route.query.team;

  players.value = await pb.collection('players').getFullList(100, {
    filter: 'team="'+route.query.team+'"'
  });
  games.value = await pb.collection('league_game').getFullList(100, {
    expand: 'teams'
  });

  // Fetch all teams for name lookup
  const allTeams = await pb.collection('teams').getFullList(100);
  teams.value = allTeams;
});

const getTeamNames = (game) => {
  if (!game.expand?.teams || game.expand.teams.length === 0) {
    // Fallback if expand didn't work - try to get team names by ID
    if (!game.teams || game.teams.length === 0) return 'N/A';
    const teamNames = game.teams.map(teamId => {
      const team = teams.value.find(t => t.id === teamId);
      return team ? team.name : teamId;
    });
    return teamNames.join(' vs. ');
  }
  const teamNames = game.expand.teams.map(team => team.name);
  return teamNames.join(' vs. ');
};

const add = async () => {
  await pb.collection('players_game').create(form.value);
  router.push('/game/'+form.value.game)
}
</script>