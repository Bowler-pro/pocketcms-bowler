<template>
  <section class="bg-white px-3 py-3">
    <section class="grid grid-cols-6 gap-3">
      <div class="col-span-6 md:col-span-3">
        <form @submit.prevent="add()">
          <label class="floating-label mt-3">
            <span>Round</span>
            <select v-model="form.round" class="select w-full">
              <option value="1">1</option>
              <option value="2">2</option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Team</span>
            <select v-model="form.team" class="select w-full">
              <option v-for="team in teams" :value="team.id">{{team.name}}</option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Game</span>
            <select v-model="form.game" class="select w-full">
              <option v-for="game in games" :value="game.id">{{game.name}} - {{ getTeamNames(game) }}</option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Baker Score</span>
            <input type="number" required v-model="form.score" class="validator input w-full" />
          </label>
          <section class="actions flex justify-end mt-3">
            <button type="submit" class="btn btn-primary">
              Add Baker
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
const teams = ref([]);
const games = ref([]);
const allTeams = ref([]);
const form = ref({
  game: '',
  round: 1,
  score: '',
  team: '',
});

onMounted(async ()=>{
  if(!pb.authStore.isValid){
    navigateTo('/')
  }

  form.value.game = route.query.game;
  form.value.team = route.query.team;

  // Fetch teams
  teams.value = await pb.collection('teams').getFullList(100);
  allTeams.value = teams.value;

  // Fetch games with expanded teams
  games.value = await pb.collection('league_game').getFullList(100, {
    expand: 'teams'
  });
});

const getTeamNames = (game) => {
  if (!game.expand?.teams || game.expand.teams.length === 0) {
    if (!game.teams || game.teams.length === 0) return 'N/A';
    const teamNames = game.teams.map(teamId => {
      const team = allTeams.value.find(t => t.id === teamId);
      return team ? team.name : teamId;
    });
    return teamNames.join(' vs. ');
  }
  const teamNames = game.expand.teams.map(team => team.name);
  return teamNames.join(' vs. ');
};

const add = async () => {
  await pb.collection('teams_baker').create(form.value);
  router.push('/game/'+form.value.game)
}
</script>
