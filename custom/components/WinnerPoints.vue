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
const blinde = ref(12);
const games = ref([]);
const bakers = ref([]);
const winners = ref({});
const tmp1 = ref({});
const tmp2 = ref({});
const teamScores = ref({
  gameScoresByRound: {}, // Raw scores from games (grouped by round)
  bakerScoresByRound: {}, // Raw scores from bakers (grouped by round)
  totalScoresByRound: {}, // Combined scores: games + bakers (raw scores by round)
  gamePoints: {}, // Calculated points from games
  bakerPoints: {}, // Calculated points from bakers
  outcomePoints: {}, // Final 2/1/0 points added to total points
  totalPoints: {}, // Grand total points: gamePoints + bakerPoints + outcomePoints
  totalScores: {}, // Raw total scores: Sum of gameScoresByRound + bakerScoresByRound
});

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

// Function to calculate scores by round from bakers
function calculateBakerScoresByRound(bakersList: typeof bakers) {
  const scoresByRound: Record<string, Record<string, number>> = {};

  bakersList.value.forEach((baker: any) => {
    const round = baker.round; // Round identifier (e.g., "1", "2", ...)
    const team = baker.team; // Team identifier
    const score = parseInt(baker.score, 10) || 0; // Parse and default to 0 for any missing scores

    // Initialize the round's score object if it doesn't exist
    if (!scoresByRound[round]) {
      scoresByRound[round] = {};
    }

    // Sum the scores for the team per round
    scoresByRound[round][team] = (scoresByRound[round][team] || 0) + score;
  });

  return scoresByRound;
}

// Function to calculate points from scores by round
function calculatePointsByRound(scoresByRound: Record<string, Record<string, number>>) {
  const totalPoints: Record<string, number> = {};

  Object.keys(scoresByRound).forEach((round) => {
    const roundScores = scoresByRound[round]; // Scores for the current round
    const maxScore = Math.max(...Object.values(roundScores)); // Get maximum score
    const tiedTeams = Object.keys(roundScores).filter((team) => roundScores[team] === maxScore);

    Object.keys(roundScores).forEach((team) => {
      if (tiedTeams.length > 1) {
        // Tie: every team gets 1 point
        totalPoints[team] = (totalPoints[team] || 0) + 1;
      } else if (roundScores[team] === maxScore) {
        // Winner: team gets 2 points
        totalPoints[team] = (totalPoints[team] || 0) + 2;
      } else {
        // Loser: no points are added
        totalPoints[team] = totalPoints[team] || 0;
      }
    });
  });

  return totalPoints;
}

// Function to calculate outcome points for total scores
function calculateOutcomePoints(totalPoints: Record<string, number>) {
  const outcomePoints: Record<string, number> = {};
  const maxScore = Math.max(...Object.values(totalPoints)); // Determine highest total points
  const tiedTeams = Object.keys(totalPoints).filter((team) => totalPoints[team] === maxScore);

  Object.keys(totalPoints).forEach((team) => {
    if (tiedTeams.length > 1) {
      // Tie: all tied teams get 1 point
      outcomePoints[team] = 1;
    } else if (totalPoints[team] === maxScore) {
      // Winner: team gets 2 points
      outcomePoints[team] = 2;
    } else {
      // Loser: no extra points
      outcomePoints[team] = 0;
    }
  });

  return outcomePoints;
}

// Function to calculate raw total scores
function calculateRawTotalScores(gameScoresByRound, bakerScoresByRound) {
  const totalScores: Record<string, number> = {};

  // Combine raw game scores and baker scores for each team
  [gameScoresByRound, bakerScoresByRound].forEach((scoresByRound) => {
    Object.values(scoresByRound).forEach((roundScores) => {
      Object.keys(roundScores).forEach((team) => {
        totalScores[team] = (totalScores[team] || 0) + roundScores[team];
      });
    });
  });

  return totalScores;
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

  blinde.value = 12 - games.value.length;

  // 1. Calculate game scores and points by round
  const gameScoresByRound = calculateScoresByRound(games);
  const gamePoints = calculatePointsByRound(gameScoresByRound);

  // 2. Calculate baker scores and points by round
  const bakerScoresByRound = calculateBakerScoresByRound(bakers);
  const bakerPoints = calculatePointsByRound(bakerScoresByRound);

  // 3. Calculate raw total scores (game + baker scores) for each team
  const totalScores = calculateRawTotalScores(gameScoresByRound, bakerScoresByRound);

  // 4. Combine game and baker points into totalPoints
  const totalPoints: Record<string, number> = { ...gamePoints };
  Object.keys(bakerPoints).forEach((team) => {
    totalPoints[team] = (totalPoints[team] || 0) + bakerPoints[team];
  });

  // 5. Calculate outcome points
  const outcomePoints = calculateOutcomePoints(totalPoints);

  // 6. Add outcome points to get the final total points
  const finalPoints: Record<string, number> = { ...totalPoints };
  Object.keys(outcomePoints).forEach((team) => {
    finalPoints[team] += outcomePoints[team];
  });

  // 7. Store all scores and points into the reactive state
  teamScores.value = {
    gameScoresByRound,
    bakerScoresByRound,
    totalScoresByRound: { gameScoresByRound, bakerScoresByRound }, // For debugging purposes
    gamePoints,
    bakerPoints,
    totalScores, // Raw total scores for all teams
    outcomePoints,
    totalPoints: finalPoints, // Grand total points
  };

  // 8. Determine the winner
  const maxFinalPoints = Math.max(...Object.values(finalPoints));
  const potentialWinners = Object.keys(finalPoints).filter(
      (team) => finalPoints[team] === maxFinalPoints
  );

  winners.value = {
    winner: potentialWinners.length > 1 ? "remi" : potentialWinners[0],
    tied: potentialWinners.length > 1 ? potentialWinners : null,
    scores: teamScores.value,
  };

  // Debugging
  console.log("Final Scores:", teamScores.value);

  tmp1.value = winners.value.scores.totalPoints;
  tmp2.value = winners.value.scores.totalScores;
});
</script>