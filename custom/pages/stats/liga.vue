<template>
  <div class="bg-white mx-auto p-4">
    <h1 class="text-3xl font-bold text-center mb-8">Liga Tabelle</h1>

    <div v-if="noGames" class="alert alert-warning shadow-lg max-w-2xl mx-auto">
      <div>
        <svg xmlns="http://www.w3.org/2000/svg" class="stroke-current flex-shrink-0 h-6 w-6" fill="none" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" /></svg>
        <div>
          <h3 class="font-bold">Keine Spiele gefunden</h3>
          <div class="text-xs">Für diese Liga wurden noch keine Spiele angelegt.</div>
        </div>
      </div>
      <div class="flex-none">
        <a href="/de/dashboard" class="btn btn-sm btn-primary">Zurück zum Dashboard</a>
      </div>
    </div>

    <div v-else-if="ligaTable.length" class="card bg-base-100 shadow-xl">
      <div class="card-body">
        <div class="overflow-x-auto">
          <table class="table table-zebra w-full">
            <thead class="bg-green-300">
              <tr>
                <th>Platz</th>
                <th>Team</th>
                <th>Spiele</th>
                <th>Punkte</th>
                <th>Pins</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(team, index) in ligaTable" :key="team.id" class="hover" :class="{'bg-gray-300': index % 2, 'bg-gray-400': !(index % 2)}">
                <td class="font-bold">{{ index + 1 }}</td>
                <td><strong><a :href="`/de/team/${team.id}`" class="link link-hover link-primary">{{ team.name }}</a></strong></td>
                <td>{{ team.gamesPlayed }}</td>
                <td class="font-bold">{{ team.totalPoints }}</td>
                <td>{{ team.totalPins }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <div v-else-if="loading" class="flex justify-center items-center min-h-screen">
      <span class="loading loading-spinner loading-lg"></span>
      <p class="ml-4 text-lg">Loading liga table...</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
import { usePocketBase } from "@/utils/pocketbase";
import { useRoute, useRouter } from "vue-router";

const pb = usePocketBase();
const route = useRoute();
const router = useRouter();
const ligaTable = ref<any[]>([]);
const loading = ref(true);
const noGames = ref(false);

onMounted(async () => {
  try {
    const ligaId = route.query.id;

    // Redirect to dashboard if no liga id
    if (!ligaId) {
      router.push('/dashboard');
      return;
    }

    // Fetch all league games for this specific liga
    const games = (await pb.collection("league_game").getList(1, 1000, {
      filter: `league="${ligaId}"`
    })).items;

    // Check if there are no games
    if (games.length === 0) {
      noGames.value = true;
      loading.value = false;
      return;
    }

    // Collect all unique team IDs from the games
    const teamIdsInLiga = new Set<string>();
    games.forEach((game: any) => {
      const gameTeams = game.teams || [];
      gameTeams.forEach((teamId: string) => teamIdsInLiga.add(teamId));
    });

    // Fetch only teams that are in this liga (only if there are team IDs)
    let teams = [];
    if (teamIdsInLiga.size > 0) {
      teams = (await pb.collection("teams").getList(1, 1000, {
        filter: Array.from(teamIdsInLiga).map(id => `id="${id}"`).join(' || ')
      })).items;
    }

    // Initialize team stats
    const teamStats: Record<string, any> = {};
    teams.forEach((team: any) => {
      teamStats[team.id] = {
        id: team.id,
        name: team.name,
        gamesPlayed: 0,
        totalPoints: 0,
        totalPins: 0,
      };
    });

    // Calculate stats for each game
    for (const game of games) {
      const gameTeams = game.teams || [];
      if (gameTeams.length !== 2) continue;

      const gameScores = await calculateGameScores(game.id, gameTeams);

      // Add to team stats
      gameTeams.forEach((teamId: string) => {
        if (teamStats[teamId]) {
          teamStats[teamId].gamesPlayed++;
          teamStats[teamId].totalPoints += gameScores.totalPoints[teamId] || 0;
          teamStats[teamId].totalPins += gameScores.totalScores[teamId] || 0;
        }
      });
    }

    // Sort by total points (descending), then by total pins
    // Filter out teams with 0 games played
    ligaTable.value = Object.values(teamStats)
      .filter((team: any) => team.gamesPlayed > 0)
      .sort((a, b) => {
        if (b.totalPoints !== a.totalPoints) {
          return b.totalPoints - a.totalPoints;
        }
        return b.totalPins - a.totalPins;
      });

    loading.value = false;
  } catch (error) {
    console.error("Error calculating liga table:", error);
    loading.value = false;
  }
});

async function calculateGameScores(gameId: string, teams: string[]) {
  // Fetch player games
  const playerGames = (
    await pb.collection("players_game").getList(1, 50, {
      filter: `game="${gameId}"`,
      expand: "player",
    })
  ).items;

  // Fetch baker games
  const bakerGames = (
    await pb.collection("teams_baker").getList(1, 50, {
      filter: `game="${gameId}"`,
    })
  ).items;

  // Calculate round scores
  const roundScores: Record<string, number>[] = Array(3).fill(null).map(() => ({}));
  const roundPoints: Record<string, number>[] = Array(3).fill(null).map(() => ({}));
  const roundScoresBakers: Record<string, number>[] = Array(3).fill(null).map(() => ({}));
  const roundPointsBakers: Record<string, number>[] = Array(3).fill(null).map(() => ({}));
  const playerCounts: Record<string, number>[] = Array(3).fill(null).map(() => ({}));

  // Process player games
  playerGames.forEach((game: any) => {
    const team = game.expand?.player?.team;
    const round = Number(game.round);
    const score = Number(game.score);

    if (team && score >= 0 && round > 0 && round <= 3) {
      roundScores[round - 1][team] = (roundScores[round - 1][team] || 0) + score;
      playerCounts[round - 1][team] = (playerCounts[round - 1][team] || 0) + 1;
    }
  });

  // Add 130 points for missing players
  teams.forEach((team) => {
    roundScores.forEach((scores, roundIndex) => {
      if (!scores[team]) scores[team] = 0;
      const actualPlayers = playerCounts[roundIndex][team] || 0;
      const missingPlayers = Math.max(0, 4 - actualPlayers);
      scores[team] += missingPlayers * 130;
    });
  });

  // Process baker games (rounds 1 and 2 only)
  bakerGames.forEach((baker: any) => {
    const team = baker.team;
    const round = Number(baker.round);
    const score = Number(baker.score);

    if (team && score >= 0 && round > 0 && round < 3) {
      roundScoresBakers[round - 1][team] = (roundScoresBakers[round - 1][team] || 0) + score;
    }
  });

  // Calculate points for each round
  roundScores.forEach((scores, roundIndex) => {
    const teamList = Object.keys(scores);
    const maxScore = Math.max(...Object.values(scores));

    teamList.forEach((team) => {
      const teamScore = scores[team];
      const teamsWithMaxScore = teamList.filter((t) => scores[t] === maxScore);

      if (teamScore === maxScore && teamsWithMaxScore.length > 1) {
        roundPoints[roundIndex][team] = 1; // Tie
      } else if (teamScore === maxScore) {
        roundPoints[roundIndex][team] = 2; // Win
      } else {
        roundPoints[roundIndex][team] = 0; // Loss
      }
    });
  });

  // Calculate baker points (rounds 1 and 2)
  roundScoresBakers.forEach((scores, roundIndex) => {
    if (roundIndex >= 2) return;
    const bakerList = Object.keys(scores);

    // If no baker entries, give 1 point to each team
    if (bakerList.length === 0) {
      teams.forEach((team) => {
        roundPointsBakers[roundIndex][team] = 1;
      });
      return;
    }

    const maxScore = Math.max(...Object.values(scores));

    bakerList.forEach((baker) => {
      const bakerScore = scores[baker];
      const bakersWithMaxScore = bakerList.filter((b) => scores[b] === maxScore);

      if (bakerScore === maxScore && bakersWithMaxScore.length > 1) {
        roundPointsBakers[roundIndex][baker] = 1; // Tie
      } else if (bakerScore === maxScore) {
        roundPointsBakers[roundIndex][baker] = 2; // Win
      } else {
        roundPointsBakers[roundIndex][baker] = 0; // Loss
      }
    });
  });

  // Calculate totals
  const gamePts: Record<string, number> = {};
  const bakerPts: Record<string, number> = {};
  const totalPts: Record<string, number> = {};
  const gameScores: Record<string, number> = {};
  const bakerScoresTotal: Record<string, number> = {};
  const bonus: Record<string, number> = {};

  // Sum game points and scores
  roundPoints.forEach((points) => {
    Object.keys(points).forEach((team) => {
      gamePts[team] = (gamePts[team] || 0) + points[team];
      totalPts[team] = (totalPts[team] || 0) + points[team];
    });
  });

  roundScores.forEach((scores) => {
    Object.keys(scores).forEach((team) => {
      gameScores[team] = (gameScores[team] || 0) + scores[team];
    });
  });

  // Sum baker points and scores
  roundPointsBakers.forEach((points, roundIndex) => {
    if (roundIndex >= 2) return;
    Object.keys(points).forEach((team) => {
      bakerPts[team] = (bakerPts[team] || 0) + points[team];
      totalPts[team] = (totalPts[team] || 0) + points[team];
    });
  });

  roundScoresBakers.forEach((scores, roundIndex) => {
    if (roundIndex >= 2) return;
    Object.keys(scores).forEach((team) => {
      bakerScoresTotal[team] = (bakerScoresTotal[team] || 0) + scores[team];
    });
  });

  // Initialize bonus
  teams.forEach((team) => {
    bonus[team] = 0;
  });

  // Calculate bonus points
  const highestTotalScore = Math.max(...Object.values(totalPts));
  const teamsWithHighestScore = Object.keys(totalPts).filter(
    (team) => totalPts[team] === highestTotalScore
  );

  Object.keys(totalPts).forEach((team) => {
    if (teamsWithHighestScore.length > 1) {
      totalPts[team] += 1;
      bonus[team] = 1;
    } else if (teamsWithHighestScore.includes(team)) {
      totalPts[team] += 2;
      bonus[team] = 2;
    }
  });

  // Calculate total scores
  const totalScores: Record<string, number> = {};
  teams.forEach((team) => {
    totalScores[team] = (gameScores[team] || 0) + (bakerScoresTotal[team] || 0);
  });

  return {
    totalPoints: totalPts,
    gamePoints: gamePts,
    bakerPoints: bakerPts,
    bonusPoints: bonus,
    totalScores: totalScores,
    bakerScoresOnly: bakerScoresTotal,
  };
}
</script>
