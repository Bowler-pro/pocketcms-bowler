<template>
  <button class="btn text-center btn-block btn-primary">
    Gewinner:
  </button>
  {{ tmp1 }}
  {{ tmp2 }}
</template>
<script setup lang="ts">
import { ref, onMounted } from "vue";
import { usePocketBase } from "@/utils/pocketbase";

// Props declaration
const props = defineProps({
  league_game: { type: String, required: true },
  teams: { type: Object, required: true }, // Adjust type based on expected structure
});

// PocketBase instance
const pb = usePocketBase();

// Reactive variables
const games = ref([]);
const bakers = ref([]);
const winners = ref({});
const tmp1 = ref({});
const tmp2 = ref({});
const teamScores = ref({
  gameScores: {}, // Raw scores from games
  bakerScores: {}, // Raw scores from bakers
  totalScores: {}, // Combined raw scores: games + bakers
  gamePoints: {}, // Calculated points from games
  bakerPoints: {}, // Calculated points from bakers
  outcomePoints: {}, // Final 2/1/0 points added to total points
  totalPoints: {}, // Grand total points: gamePoints + bakerPoints + outcomePoints
});

// Function to calculate raw scores from games
function calculateRawGameScores(gamesList: typeof games) {
  const gameScoreTotals: Record<string, number> = {};

  gamesList.value.forEach((game: any) => {
    const team = game.expand.player.team; // Team from expanded player object
    const score = parseInt(game.score, 10) || 0; // Parse and default to 0
    gameScoreTotals[team] = (gameScoreTotals[team] || 0) + score;
  });

  return gameScoreTotals;
}

// Function to calculate raw scores from bakers
function calculateRawBakerScores(bakersList: typeof bakers) {
  const bakerScoreTotals: Record<string, number> = {};

  bakersList.value.forEach((baker: any) => {
    const team = baker.team; // Team from baker record
    const score = parseInt(baker.score, 10) || 0; // Parse and default to 0
    bakerScoreTotals[team] = (bakerScoreTotals[team] || 0) + score;
  });

  return bakerScoreTotals;
}

// Function to calculate scores by round from games
function calculateScoresByRound(gamesList: typeof games) {
  const scoresByRound: Record<string, Record<string, number>> = {};

  gamesList.value.forEach((game: any) => {
    const round = game.round; // Round identifier (e.g., "1", "2", ...)
    const team = game.expand.player.team; // Team from expanded player object
    const score = parseInt(game.score, 10) || 0; // Parse and default to 0 for missing scores

    // Initialize the round's score object if it doesn't exist
    if (!scoresByRound[round]) {
      scoresByRound[round] = {};
    }

    // Sum the scores for the team per round
    scoresByRound[round][team] = (scoresByRound[round][team] || 0) + score;
  });

  return scoresByRound;
}

// Function to calculate points from games with proper tie handling (remi = 1 point for every team in case of a tie)
function calculateGamePoints(teamScoresByRound: Record<string, Record<string, number>>) {
  const totalPoints: Record<string, number> = {};

  Object.keys(teamScoresByRound).forEach((round) => {
    const roundScores = teamScoresByRound[round]; // Scores for the current round
    const maxScore = Math.max(...Object.values(roundScores)); // Get maximum score
    const tiedTeams = Object.keys(roundScores).filter((team) => roundScores[team] === maxScore);

    Object.keys(roundScores).forEach((team) => {
      if (tiedTeams.length > 1) {
        // If there is a tie, every team gets 1 point
        totalPoints[team] = (totalPoints[team] || 0) + 1;
      } else if (roundScores[team] === maxScore) {
        // Winner gets 2 points
        totalPoints[team] = (totalPoints[team] || 0) + 2;
      } else {
        // Loser gets 0 points (this is implicit since we don't add any points for other teams)
        totalPoints[team] = totalPoints[team] || 0;
      }
    });
  });

  return totalPoints;
}

// Function to calculate points from bakers with proper tie handling
function calculateBakerPoints(bakersList: typeof bakers) {
  const teamPoints: Record<string, number> = {};
  const bakerScoreTotals: Record<string, number> = calculateRawBakerScores(bakersList);

  const maxScore = Math.max(...Object.values(bakerScoreTotals)); // Get maximum baker score
  const tiedTeams = Object.keys(bakerScoreTotals).filter((team) => bakerScoreTotals[team] === maxScore);

  Object.keys(bakerScoreTotals).forEach((team) => {
    if (tiedTeams.length > 1) {
      // If there is a tie, every team gets 1 point
      teamPoints[team] = (teamPoints[team] || 0) + 1;
    } else if (bakerScoreTotals[team] === maxScore) {
      // Winner gets 2 points
      teamPoints[team] = (teamPoints[team] || 0) + 2;
    } else {
      // Loser gets 0 points (this is implicit)
      teamPoints[team] = teamPoints[team] || 0;
    }
  });

  return teamPoints;
}

// Function to calculate outcome points for total scores
function calculateOutcomePoints(totalPoints: Record<string, number>) {
  const outcomePoints: Record<string, number> = {};
  const maxScore = Math.max(...Object.values(totalPoints)); // Determine highest total points
  const tiedTeams = Object.keys(totalPoints).filter((team) => totalPoints[team] === maxScore);

  Object.keys(totalPoints).forEach((team) => {
    if (tiedTeams.length > 1) {
      // On tie, all tied teams get 1 point
      outcomePoints[team] = 1;
    } else if (totalPoints[team] === maxScore) {
      // Winner gets 2 points
      outcomePoints[team] = 2;
    } else {
      // All other teams get 0
      outcomePoints[team] = 0;
    }
  });

  return outcomePoints;
}

// onMounted lifecycle hook
onMounted(async () => {
  // Fetch games data
  games.value = (await pb.collection("players_game").getList(1, 100, {
    filter: `game="${props.league_game}"`,
    expand: "player", // Expand player relationships for team info
  })).items;

  // Fetch bakers data
  bakers.value = (await pb.collection("teams_baker").getList(1, 100, {
    filter: `game="${props.league_game}"`,
  })).items;

  // 1. Calculate raw game scores
  const gameScores = calculateRawGameScores(games);

  // 2. Calculate raw baker scores
  const bakerScores = calculateRawBakerScores(bakers);

  // 3. Combine gameScores and bakerScores into totalScores
  const totalScores: Record<string, number> = { ...gameScores };
  Object.keys(bakerScores).forEach((team) => {
    totalScores[team] = (totalScores[team] || 0) + bakerScores[team];
  });

  // 4. Calculate game points
  const teamScoresByRound = calculateScoresByRound(games); // Round-wise team scores
  const gamePoints = calculateGamePoints(teamScoresByRound);

  // 5. Calculate baker points
  const bakerPoints = calculateBakerPoints(bakers);

  // 6. Combine points into totalPoints
  const totalPoints: Record<string, number> = { ...gamePoints };
  Object.keys(bakerPoints).forEach((team) => {
    totalPoints[team] = (totalPoints[team] || 0) + bakerPoints[team];
  });

  // 7. Calculate outcome points
  const outcomePoints = calculateOutcomePoints(totalPoints);

  // 8. Add the final outcome points to totalPoints
  const finalPoints: Record<string, number> = { ...totalPoints };
  Object.keys(outcomePoints).forEach((team) => {
    finalPoints[team] += outcomePoints[team];
  });

  // 9. Store all scores and points into the reactive state
  teamScores.value = {
    gameScores,
    bakerScores,
    totalScores,
    gamePoints,
    bakerPoints,
    outcomePoints,
    totalPoints: finalPoints, // Grand total
  };

  // 10. Determine the winner
  const maxFinalPoints = Math.max(...Object.values(finalPoints));
  const potentialWinners = Object.keys(finalPoints).filter(
      (team) => finalPoints[team] === maxFinalPoints
  );

  winners.value = {
    winner: potentialWinners.length > 1 ? "remi" : potentialWinners[0],
    tied: potentialWinners.length > 1 ? potentialWinners : null, // List tied teams explicitly
    scores: teamScores.value,
  };

  // Debugging
  console.log("Winner:", winners.value.scores.totalPoints);
  console.log("Winner:", winners.value.scores.totalScores);
  tmp1.value = winners.value.scores.totalPoints;
  tmp2.value = winners.value.scores.totalScores;
});
</script>