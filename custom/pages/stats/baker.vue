<template>
  <div class="bg-white mx-auto">
    <h1 class="text-3xl font-bold text-center mb-6">Baker Team Stats & Ranking</h1>

    <div v-if="bakerStats.length" class="overflow-x-auto">
      <table class="table table-zebra w-full shadow-lg">
        <thead>
          <tr>
            <th class="text-left">Rank</th>
            <th class="text-left">Name</th>
            <th class="text-left">Baker Count</th>
            <th class="text-left">Average</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="(baker, index) in bakerStats"
            :key="baker.team"
            :class="{
              'bg-gray-300': !(index % 2),
              'bg-gray-400': index % 2
            }"
          >
            <td>{{ index + 1 }}</td>
            <td class="font-semibold">{{ baker.name }}</td>
            <td>{{ baker.gameCount }}</td>
            <td>{{ baker.average.toFixed(0) }}</td>
          </tr>
        </tbody>
      </table>
    </div>
    <div v-else class="text-center">
      <p class="text-lg">No baker stats available.</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
import { usePocketBase } from "@/utils/pocketbase";

const pb = usePocketBase();

const bakers = ref([]);
const bakerStats = ref([]);

onMounted(async () => {
  bakers.value = (
    await pb.collection("teams_baker").getList(1, 1000, {
      expand: "team",
    })
  ).items;

  // Calculate stats per baker team
  const statsMap = {};

  bakers.value.forEach((baker) => {
    const team = baker.team;
    const teamName = baker.expand?.team?.name || `Team ${team}`;
    const score = Number(baker.score) || 0;

    if (!statsMap[team]) {
      statsMap[team] = {
        team: team,
        name: teamName,
        gameCount: 0,
        totalScore: 0,
      };
    }

    statsMap[team].gameCount++;
    statsMap[team].totalScore += score;
  });

  // Convert to array and calculate averages
  const stats = Object.values(statsMap).map((baker) => ({
    ...baker,
    average: baker.gameCount > 0 ? baker.totalScore / baker.gameCount : 0,
  }));

  // Sort by average (descending)
  stats.sort((a, b) => b.average - a.average);

  bakerStats.value = stats;
});
</script>
