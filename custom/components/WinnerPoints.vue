<template>
  <button class="btn text-center btn-block btn-primary">
    Gewinner:
  </button>
  <div>
    Total Points: {{ tmp1 }}
  </div>
  <div>
    Total Scores: {{ tmp2 }}
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from "vue";
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

// Fetch data on mounted
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
});

// Function to calculate scores by round from games
function calculateScoresByRound(gamesList) {
  const scoresByRound: Record<string, Record<string, number>> = {};

  gamesList.forEach((game: any) => {
    const round = game.round;
    const team = game.expand.player.team;
    const score = parseInt(game.score, 10) || 0;

    if (!scoresByRound[round]) {
      scoresByRound[round] = {};
    }

    scoresByRound[round][team] = (scoresByRound[round][team] || 0) + score;
  });

  return scoresByRound;
}

// Function to calculate scores by round from bakers
function calculateBakerScoresByRound(bakersList) {
  const scoresByRound: Record<string, Record<string, number>> = {};

  bakersList.forEach((baker: any) => {
    const round = baker.round;
    const team = baker.team;
    const score = parseInt(baker.score, 10) || 0;

    if (!scoresByRound[round]) {
      scoresByRound[round] = {};
    }

    scoresByRound[round][team] = (scoresByRound[round][team] || 0) + score;
  });

  return scoresByRound;
}

// Function to calculate points from scores by round
function calculatePointsByRound(scoresByRound) {
  const totalPoints: Record<string, number> = {};

  Object.keys(scoresByRound).forEach((round) => {
    const roundScores = scoresByRound[round];
    const maxScore = Math.max(...Object.values(roundScores));
    const tiedTeams = Object.keys(roundScores).filter(
        (team) => roundScores[team] === maxScore
    );

    Object.keys(roundScores).forEach((team) => {
      if (tiedTeams.length > 1) {
        totalPoints[team] = (totalPoints[team] || 0) + 1;
      } else if (roundScores[team] === maxScore) {
        totalPoints[team] = (totalPoints[team] || 0) + 2;
      } else {
        totalPoints[team] = totalPoints[team] || 0;
      }
    });
  });

  return totalPoints;
}

// Function to calculate outcome points for total scores
function calculateOutcomePoints(totalPoints) {
  const outcomePoints: Record<string, number> = {};
  const maxScore = Math.max(...Object.values(totalPoints));
  const tiedTeams = Object.keys(totalPoints).filter(
      (team) => totalPoints[team] === maxScore
  );

  Object.keys(totalPoints).forEach((team) => {
    if (tiedTeams.length > 1) {
      outcomePoints[team] = 1;
    } else if (totalPoints[team] === maxScore) {
      outcomePoints[team] = 2;
    } else {
      outcomePoints[team] = 0;
    }
  });

  return outcomePoints;
}

// Function to calculate raw total scores (game + baker scores)
function calculateRawTotalScores(gameScoresByRound, bakerScoresByRound) {
  const totalScores: Record<string, number> = {};

  [gameScoresByRound, bakerScoresByRound].forEach((scoresByRound) => {
    Object.values(scoresByRound).forEach((roundScores) => {
      Object.keys(roundScores).forEach((team) => {
        totalScores[team] = (totalScores[team] || 0) + roundScores[team];
      });
    });
  });

  return totalScores;
}

// Computed property to calculate team scores
const teamScores = computed(() => {
  const gameScoresByRound = calculateScoresByRound(games.value);
  const bakerScoresByRound = calculateBakerScoresByRound(bakers.value);
  const totalScores = calculateRawTotalScores(gameScoresByRound, bakerScoresByRound);

  const gamePoints = calculatePointsByRound(gameScoresByRound);
  const bakerPoints = calculatePointsByRound(bakerScoresByRound);

  const totalPoints: Record<string, number> = { ...gamePoints };
  Object.keys(bakerPoints).forEach((team) => {
    totalPoints[team] = (totalPoints[team] || 0) + bakerPoints[team];
  });

  const outcomePoints = calculateOutcomePoints(totalPoints);

  const finalPoints: Record<string, number> = { ...totalPoints };
  Object.keys(outcomePoints).forEach((team) => {
    finalPoints[team] += outcomePoints[team];
  });

  return {
    gameScoresByRound,
    bakerScoresByRound,
    totalScoresByRound: { gameScoresByRound, bakerScoresByRound },
    gamePoints,
    bakerPoints,
    totalScores,
    outcomePoints,
    totalPoints: finalPoints,
  };
});

// Computed property to determine winners
const computedWinners = computed(() => {
  const { totalPoints } = teamScores.value;
  const maxFinalPoints = Math.max(...Object.values(totalPoints));
  const potentialWinners = Object.keys(totalPoints).filter(
      (team) => totalPoints[team] === maxFinalPoints
  );

  return {
    winner: potentialWinners.length > 1 ? "remi" : potentialWinners[0],
    tied: potentialWinners.length > 1 ? potentialWinners : null,
    scores: teamScores.value,
  };
});

// Bind `tmp1` and `tmp2` directly to computed properties
const tmp1 = computed(() => computedWinners.value.scores.totalPoints);
const tmp2 = computed(() => computedWinners.value.scores.totalScores);
</script>