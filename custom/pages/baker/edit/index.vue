<template>
  <section class="bg-white px-3 py-3">
    <section class="grid grid-cols-6 gap-3">
      <div class="col-span-6 md:col-span-3">
        <form @submit.prevent="update()">
          <label class="floating-label mt-3">
            <span>Team</span>
            <input type="text" disabled v-model="team.name" class="validator input w-full"/>
          </label>
          <label class="floating-label mt-3">
            <span>League</span>
            <input type="text" disabled v-model="league.name" class="validator input w-full"/>
          </label>
          <label class="floating-label mt-3">
            <span>Game</span>
            <select v-model="form.game" class="select w-full" disabled>
              <option v-for="availableGame in games" :value="availableGame.id">
                {{availableGame.name}} - {{ getTeamNames(availableGame) }}
              </option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Round</span>
            <select v-model="form.round" class="select w-full" disabled>
              <option value="1">1</option>
              <option value="2">2</option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Baker Score</span>
            <input type="number" required v-model="form.score" class="validator input w-full"/>
          </label>
          <section class="actions flex justify-between mt-3">
            <button @click="remove(form.id)" type="button" class="btn btn-error">löschen</button>
            <button type="submit" class="btn btn-primary">
              update
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
const form = ref({
  game: '',
  score: '',
  team: '',
  round: 1,
});

const team = ref({});
const game = ref({});
const league = ref({});
const games = ref([]);
const allTeams = ref([]);

const load = async () => {
  // The parameter is called 'game' but it actually contains the baker record ID
  const bakerId = route.query.game;

  if (!bakerId) {
    router.push('/dashboard');
    return;
  }

  form.value = await pb.collection('teams_baker').getOne(bakerId as string);
  team.value = await pb.collection('teams').getOne(form.value.team);
  game.value = await pb.collection('league_game').getOne(form.value.game);
  league.value = await pb.collection('leagues').getOne(game.value.league);

  // Fetch all games with expanded teams
  games.value = await pb.collection('league_game').getFullList(100, {
    expand: 'teams'
  });

  // Fetch all teams for name lookup
  allTeams.value = await pb.collection('teams').getFullList(100);
}

const getTeamNames = (gameItem) => {
  if (!gameItem.expand?.teams || gameItem.expand.teams.length === 0) {
    if (!gameItem.teams || gameItem.teams.length === 0) return 'N/A';
    const teamNames = gameItem.teams.map(teamId => {
      const team = allTeams.value.find(t => t.id === teamId);
      return team ? team.name : teamId;
    });
    return teamNames.join(' vs. ');
  }
  const teamNames = gameItem.expand.teams.map(t => t.name);
  return teamNames.join(' vs. ');
};

onMounted(async () => {
  if (!pb.authStore.isValid) {
    navigateTo('/')
    return;
  }
  await load();
  if (form.value.game) {
    addBreadcrumb({
      icon: null,
      link: 'game/' + form.value.game,
      code: 'back-to-game'
    })
  }
});

const remove = async (id) => {
  if (confirm('Eintrag löschen ?')) {
    await pb.collection('teams_baker').delete(id)
    router.push('/game/' + form.value.game);
  }
}

const update = async () => {
  await pb.collection('teams_baker').update(form.value.id, form.value);
  router.push('/game/' + form.value.game)
}
</script>
