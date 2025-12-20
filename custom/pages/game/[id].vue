<template>
  <section class="bg-white px-3 py-3">
   <section class="headline mb-3">
     {{game.expand?.league.name}} -
     {{ game.name }} | {{teamNamesVs}}
   </section> <br>
    <section class="grid grid-cols-6 gap-3">
      <TeamScoreboard v-for="team in game.teams"
                      :game="game.id" :team="team"
                      :isLeader="isEnemy(team).length > 0"
                      @score="handleScore($event,team)"
      />
    </section>
    <WinnerButton :data="winner"></WinnerButton>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import {onMounted} from "vue";
import {useRoute} from 'vue-router'
import TeamScoreboard from "@/components/TeamScoreboard.vue";
import WinnerButton from "@/components/WinnerButton.vue";

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

const teamNamesVs = computed(()=>{
  let teamNames = game.value.expand?.teams?.map(team => team.name) || [];
  return teamNames.join(' vs. ');
});

const isEnemy = (teamID) => {
  return teams.value
      .filter(team => team.leader.includes(pb.authStore.record.id))
      .filter(team => team.id !== teamID);
}

const handleScore = (score, team) => {
  if (winner.value.score < score) {
    winner.value = {
      team: teams.value.find(item => item.id == team),
      score: score
    }
  }
}

onMounted(() => {
  if (!route.params.id && !route.query.team && !route.params.game) {
    navigateTo('/dashboard')
  }
  load();
})
</script>