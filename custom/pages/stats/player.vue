<template>
  <div class=" mx-auto  bg-white">
    <h1 class="text-3xl font-bold text-center mb-6">Player Stats & Ranking</h1>

    <div v-if="playerStats.length" class="overflow-x-auto">
      <table class="table table-zebra w-full shadow-lg">
        <thead class="bg-gray-300">
        <tr>
          <th class="text-left">Rank</th>
          <th class="text-left">Name</th>
          <th class="text-left">Game Count</th>
          <th class="text-left">Average</th>
        </tr>
        </thead>
        <tbody>
        <tr
            v-for="(player, index) in playerStats"
            :key="player.id" :class="{
              'bg-gray-300': index % 2,
              'bg-gray-400': !(index % 2),
            }"
        >
          <td>{{ index + 1 }}</td>
          <td class="font-semibold">{{ player.name }}</td>
          <td>{{ player.gameCount }}</td>
          <td>{{ player.average.toFixed(0) }}</td>
        </tr>
        </tbody>
      </table>
    </div>
    <div v-else class="text-center">
      <p class="text-lg">No player stats available.</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import {ref, onMounted} from "vue";
import {usePocketBase} from "@/utils/pocketbase";

const pb = usePocketBase();

const games = ref([]);
const playerStats = ref([]);

onMounted(async () => {
  games.value = (
      await pb.collection("players_game").getList(1, 1000, {
        expand: "player",
      })
  ).items;

  // Calculate stats per player
  const statsMap = {};

  games.value.forEach((game) => {
    const player = game.expand?.player;
    if (!player) return;

    const playerId = player.id;
    const playerName = player.name;
    const score = Number(game.score) || 0;

    if (!statsMap[playerId]) {
      statsMap[playerId] = {
        id: playerId,
        name: playerName,
        gameCount: 0,
        totalScore: 0,
      };
    }

    statsMap[playerId].gameCount++;
    statsMap[playerId].totalScore += score;
  });

  // Convert to array and calculate averages
  const stats = Object.values(statsMap).map((player) => ({
    ...player,
    average: player.gameCount > 0 ? player.totalScore / player.gameCount : 0,
  }));

  // Sort by average (descending)
  stats.sort((a, b) => b.average - a.average);

  playerStats.value = stats;
});
</script>
