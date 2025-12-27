<template>
  <section class="bg-white px-3 py-3">
    <section class="headline mb-3">
      <a :href="'/game/'+game.expand?.league.id" class="text-primary">{{ game.expand?.league.name }}</a> -
      <a :href="'/game/'+game.id" class="text-primary">{{ game.name }}</a> - {{ gameDate }} <br> {{ teamNamesVs }}
    </section>
    <br>
    <section class="grid grid-cols-6 gap-3 mb-3">
      <TeamScoreboard v-for="team in game.teams"
                      :game="game.id" :team="team"
                      :isLeader="isEnemy(team).length > 0"
                      @score="handleScore($event,team)"
      />
    </section>
    <ScoreBoard :game="route.params.id" :teams="game.teams"></ScoreBoard>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import {onMounted} from "vue";
import {useRoute} from 'vue-router'
import TeamScoreboard from "@/components/TeamScoreboard.vue";
import ScoreBoard from "../../components/scoreBoard.vue";

const pb = usePocketBase()
const route = useRoute()

const teams = ref([]);
const game = ref({});
const winner = ref({
  team: null,
  score: 0
})

const load = async () => {
  game.value = await pb.collection('league_game').getOne(route.params.id, {
    expand: 'teams,league'
  });
  teams.value = game.value.expand.teams;
}

const teamNamesVs = computed(() => {
  let teamNames = game.value.expand?.teams?.map(team => team.name) || [];
  return teamNames.join(' vs. ');
});

const gameDate = computed(() => {
  if (!game.value.date) return '';
  const date = new Date(game.value.date);
  return date.toLocaleDateString('de-DE', {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric'
  });
});

const isEnemy = (teamID) => {
  return teams.value
      .filter(team => team.leader.includes(pb.authStore.record.id))
      .filter(team => team.id !== teamID);
}

const handleScore = (points, team) => {
  if (winner.value.score < points) {
    winner.value = {
      team: teams.value.find(item => item.id == team),
      points
    }
  }
}

onMounted(() => {
  if (!route.params.id && !route.query.team && !route.params.game) {
    navigateTo('/dashboard')
  }
  if (!pb.authStore.isValid) {
    router.push('/login')
  }
  load();

})
</script>