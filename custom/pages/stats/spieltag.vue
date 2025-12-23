<template>
  <div class="bg-white mx-auto">
    <h1 class="text-3xl font-bold text-center mb-8">Spieltag Scoreboard</h1>

    <div v-if="Object.keys(groupedPlaydays).length" class="space-y-12">
      <div v-for="(games, date) in groupedPlaydays" :key="date" class="card bg-base-100 shadow-xl">
        <div class="card-body">
          <h2 class="card-title text-2xl border-b-2 border-success pb-2 mb-4">Spieltag - {{ formatDate(date) }}</h2>

          <div class="overflow-x-auto">
            <table class="table table-zebra w-full">
              <thead class="bg-green-300 text-white">
                <tr>
                  <th>Match</th>
                  <th>Total Points</th>
                  <th>Total Score</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(playday, index) in games" :key="playday.id" class="hover" :class="{'bg-gray-300': index %2,'bg-gray-400': !(index %2)}">
                  <td><strong><a :href="`/de/game/${playday.id}`" class="link link-hover link-primary">{{ getTeamNames(playday.teams) }}</a></strong></td>
                  <td>{{ formatTotalPoints(playday.id) }}</td>
                  <td>{{ formatTotalScore(playday.id) }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
    <div v-else class="flex justify-center items-center min-h-screen">
      <span class="loading loading-spinner loading-lg"></span>
      <p class="ml-4 text-lg">Loading spieltags...</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
import { usePocketBase } from "@/utils/pocketbase";

const pb = usePocketBase();
const playdays = ref([]);
const groupedPlaydays = ref<Record<string, any[]>>({});
const playdayScores = ref<Record<string, any>>({});
const teamNames = ref<Record<string, string>>({});

onMounted(async () => {
  try {
    // Fetch all teams first
    const teams = (await pb.collection("teams").getList(1, 1000)).items;
    teams.forEach((team: any) => {
      teamNames.value[team.id] = team.name;
    });

    // Fetch all playdays
    playdays.value = (
      await pb.collection("league_game").getList(1, 1000, {
        sort: "date",
      })
    ).items;

    // Group playdays by date and merge duplicate team matchups
    const grouped: Record<string, any[]> = {};
    const processedPairs = new Set<string>();

    playdays.value.forEach((playday: any) => {
      const date = playday.date || "no-date";
      if (!grouped[date]) {
        grouped[date] = [];
      }

      // Create a unique key for this team pair (sorted to catch both orderings)
      const teams = playday.teams || [];
      const sortedTeams = [...teams].sort().join("-");
      const pairKey = `${date}-${sortedTeams}`;

      if (!processedPairs.has(pairKey)) {
        grouped[date].push(playday);
        processedPairs.add(pairKey);
      }
    });
    groupedPlaydays.value = grouped;

    // For each playday, calculate aggregated scores across all matchups
    for (const playday of playdays.value) {
      await calculatePlaydayScores(playday);
    }

    // Merge scores for duplicate team pairs
    mergeMatchupScores();
  } catch (error) {
    console.error("Error fetching playdays:", error);
  }
});

function formatDate(dateString: string) {
  if (dateString === "no-date") return "No Date";
  const date = new Date(dateString);
  return date.toLocaleDateString("de-DE", {
    weekday: "long",
    year: "numeric",
    month: "long",
    day: "numeric",
  });
}

function getTeamNames(teams: string[]) {
  if (!teams || teams.length !== 2) return "N/A";
  const team1 = teamNames.value[teams[0]] || teams[0];
  const team2 = teamNames.value[teams[1]] || teams[1];
  return `${team1} vs ${team2}`;
}

function formatTotalPoints(playdayId: string) {
  const scores = playdayScores.value[playdayId];
  if (!scores) return "-";
  const teams = scores.sortedTeams;
  if (teams.length !== 2) return "-";
  return `${scores.totalPoints[teams[0]] || 0}:${scores.totalPoints[teams[1]] || 0}`;
}

function formatTotalScore(playdayId: string) {
  const scores = playdayScores.value[playdayId];
  if (!scores) return "-";
  const teams = scores.sortedTeams;
  if (teams.length !== 2) return "-";
  return `${scores.totalScores[teams[0]] || 0}:${scores.totalScores[teams[1]] || 0}`;
}

function mergeMatchupScores() {
  // Find all games with the same date and team pair
  const matchupMap = new Map<string, string[]>();

  playdays.value.forEach((playday: any) => {
    const teams = playday.teams || [];
    if (teams.length !== 2) return;

    const sortedTeams = [...teams].sort().join("-");
    const date = playday.date || "no-date";
    const key = `${date}-${sortedTeams}`;

    if (!matchupMap.has(key)) {
      matchupMap.set(key, []);
    }
    matchupMap.get(key)!.push(playday.id);
  });

  // Merge scores for each matchup - aggregate raw scores and recalculate points
  matchupMap.forEach((gameIds, key) => {
    if (gameIds.length <= 1) return;

    // Get the first game as the primary one
    const primaryGameId = gameIds[0];
    const primaryGame = playdays.value.find((p: any) => p.id === primaryGameId);
    if (!primaryGame) return;

    // Aggregate only raw scores (pins) from all games
    const teams = primaryGame.teams;
    const aggregated = {
      totalScores: {} as Record<string, number>,
      bakerScoresOnly: {} as Record<string, number>,
    };

    // Initialize
    teams.forEach((team: string) => {
      aggregated.totalScores[team] = 0;
      aggregated.bakerScoresOnly[team] = 0;
    });

    // Sum up ONLY the raw pin scores from all games
    gameIds.forEach((gameId) => {
      const scores = playdayScores.value[gameId];
      if (!scores) return;

      teams.forEach((team: string) => {
        aggregated.totalScores[team] += scores.totalScores[team] || 0;
        aggregated.bakerScoresOnly[team] += scores.bakerScoresOnly[team] || 0;
      });
    });

    // Calculate game scores (total - baker)
    const gameScores: Record<string, number> = {};
    teams.forEach((team: string) => {
      gameScores[team] = aggregated.totalScores[team] - aggregated.bakerScoresOnly[team];
    });

    // Now recalculate points from scratch based on aggregated scores
    // Since we can't separate by rounds after aggregation, we need a different approach
    // We'll compare total game scores and total baker scores

    const gamePts: Record<string, number> = {};
    const bakerPts: Record<string, number> = {};
    const bonusPts: Record<string, number> = {};
    const totalPts: Record<string, number> = {};

    // For aggregated games, we compare total game scores across all rounds
    // Award points based on who has higher total game score
    const team1GameScore = gameScores[teams[0]];
    const team2GameScore = gameScores[teams[1]];

    if (team1GameScore > team2GameScore) {
      gamePts[teams[0]] = 6; // Won all game rounds
      gamePts[teams[1]] = 0;
    } else if (team2GameScore > team1GameScore) {
      gamePts[teams[0]] = 0;
      gamePts[teams[1]] = 6;
    } else {
      gamePts[teams[0]] = 3; // Tied
      gamePts[teams[1]] = 3;
    }

    // Same for baker scores
    const team1BakerScore = aggregated.bakerScoresOnly[teams[0]];
    const team2BakerScore = aggregated.bakerScoresOnly[teams[1]];

    if (team1BakerScore > team2BakerScore) {
      bakerPts[teams[0]] = 4; // Won both baker rounds
      bakerPts[teams[1]] = 0;
    } else if (team2BakerScore > team1BakerScore) {
      bakerPts[teams[0]] = 0;
      bakerPts[teams[1]] = 4;
    } else {
      bakerPts[teams[0]] = 2; // Tied
      bakerPts[teams[1]] = 2;
    }

    // Calculate total points before bonus
    teams.forEach((team: string) => {
      totalPts[team] = gamePts[team] + bakerPts[team];
    });

    // Add bonus points
    const team1TotalPts = totalPts[teams[0]];
    const team2TotalPts = totalPts[teams[1]];

    if (team1TotalPts > team2TotalPts) {
      bonusPts[teams[0]] = 2;
      bonusPts[teams[1]] = 0;
      totalPts[teams[0]] += 2;
    } else if (team2TotalPts > team1TotalPts) {
      bonusPts[teams[0]] = 0;
      bonusPts[teams[1]] = 2;
      totalPts[teams[1]] += 2;
    } else {
      bonusPts[teams[0]] = 1;
      bonusPts[teams[1]] = 1;
      totalPts[teams[0]] += 1;
      totalPts[teams[1]] += 1;
    }

    // Store the recalculated scores under the primary game ID
    playdayScores.value[primaryGameId] = {
      totalPoints: totalPts,
      gamePoints: gamePts,
      bakerPoints: bakerPts,
      bonusPoints: bonusPts,
      totalScores: aggregated.totalScores,
      bakerScoresOnly: aggregated.bakerScoresOnly,
      sortedTeams: teams,
    };
  });
}

async function calculatePlaydayScores(playday: any) {
  try {
    // Get all teams from the playday
    const teams = playday.teams || [];

    if (!teams.length) {
      return;
    }

    // Calculate scores directly for this league_game (playday)
    const gameScores = await calculateGameScores(playday.id, teams);

    // Sort teams by total points (descending)
    const sortedTeams = teams.sort(
      (a: string, b: string) => gameScores.totalPoints[b] - gameScores.totalPoints[a]
    );

    playdayScores.value[playday.id] = {
      totalPoints: gameScores.totalPoints,
      gamePoints: gameScores.gamePoints,
      bakerPoints: gameScores.bakerPoints,
      bonusPoints: gameScores.bonusPoints,
      totalScores: gameScores.totalScores,
      bakerScoresOnly: gameScores.bakerScoresOnly,
      sortedTeams: sortedTeams,
    };
  } catch (error) {
    console.error(`Error calculating scores for playday ${playday.id}:`, error);
  }
}

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