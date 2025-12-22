<template>
  <section class="bg-white px-3 py-3">
    <section class="headline mb-3">
      {{ game.expand?.league.name }} -
      {{ game.name }} | {{ teamNamesVs }}
    </section>
    <br>
    <section class="grid grid-cols-6 gap-3 mb-3">
      <TeamScoreboard v-for="team in game.teams"
                      :game="game.id" :team="team"
                      :isLeader="isEnemy(team).length > 0"
                      @score="handleScore($event,team)"
      />
    </section>

    <WinnerPoints  :league_game="route.params.id" :teams="game.teams"/>
    <WinnerButton :data="winner"></WinnerButton>

  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import {onMounted} from "vue";
import {useRoute} from 'vue-router'
import TeamScoreboard from "@/components/TeamScoreboard.vue";
import WinnerButton from "@/components/WinnerButton.vue";
import WinnerPoints from "../../components/WinnerPoints.vue";

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