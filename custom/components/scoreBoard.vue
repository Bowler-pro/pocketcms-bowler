<template>
  <div class="scoreboard">
    <h1>Scoreboard</h1>

    <div v-if="teamScores.length">
      <!-- Display team points and scores for every round -->
      <div v-for="(teamPoints, roundIndex) in teamScores" :key="roundIndex">
        <h2>Round {{ roundIndex + 1 }}</h2>
        <ul>
          <li v-for="(points, team) in teamPoints" :key="team">
            Team {{ team }}: {{ points }} points (Score: {{ scoresByRound[roundIndex][team] || 0 }})
          </li>
        </ul>
      </div>

      <!-- Display baker points only for rounds 1 and 2 -->
      <div v-for="(bakerPoints, roundIndex) in bakerScores" v-if="roundIndex < 2" :key="'baker-' + roundIndex">
        <h2>Baker Scores - Round {{ roundIndex + 1 }}</h2>
        <ul>
          <li v-for="(points, baker) in bakerPoints" :key="'baker-' + baker">
            Baker {{ baker }}: {{ points }} points (Score: {{ scoresByRoundBakers[roundIndex][baker] || 0 }})
          </li>
        </ul>
      </div>

      <!-- Display total team scores and points -->
      <h2>Total Team Scores & Points</h2>
      <ul>
        <li v-for="(team, index) in sortedTeams" :key="team">
          <strong>Team {{ team }}:</strong>
          <br />
          Total Score: {{ totalTeamScores[team] || 0 }} (Games: {{ totalGameScores[team] || 0 }} + Bakers: {{ totalBakerScores[team] || 0 }})
          <br />
          Total Points: {{ totalTeamPoints[team] || 0 }} (Games: {{ gamePoints[team] || 0 }} + Bakers: {{ bakerPoints[team] || 0 }} + Bonus: {{ bonusPoints[team] || 0 }})
        </li>
      </ul>
    </div>
    <div v-else>
      <p>No games found or scores to display for this game.</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
import { usePocketBase } from "@/utils/pocketbase";

const props = defineProps({
  game: { type: String, required: true }, // Current game ID
  teams: { type: Array as () => string[], required: true }, // List of teams
});

const pb = usePocketBase();

const games = ref([]); // Stores the game data
const bakers = ref([]); // Stores baker data
const scoresByRound = ref<Record<string, number>[]>([]); // Scores per team per round
const teamScores = ref<Record<string, number>[]>([]); // Points for win/draw/loss per team per round
const scoresByRoundBakers = ref<Record<string, number>[]>([]); // Scores per baker per round
const bakerScores = ref<Record<string, number>[]>([]); // Points for bakers per round
const totalTeamPoints = ref<Record<string, number>>({}); // Final total points for teams (including bonus)
const totalBakerPoints = ref<Record<string, number>>({}); // Final total points for bakers
const totalTeamScores = ref<Record<string, number>>({}); // Total scores (games + bakers)
const totalGameScores = ref<Record<string, number>>({}); // Total game scores only
const totalBakerScores = ref<Record<string, number>>({}); // Total baker scores only
const gamePoints = ref<Record<string, number>>({}); // Total points from games
const bakerPoints = ref<Record<string, number>>({}); // Total points from bakers
const bonusPoints = ref<Record<string, number>>({}); // Bonus points
const sortedTeams = ref<string[]>([]); // Teams sorted by total points

onMounted(async () => {
  try {
    // Fetch games data
    games.value = (
        await pb.collection("players_game").getList(1, 50, {
          filter: `game="${props.game}"`,
          expand: "player",
        })
    ).items;

    // Fetch baker data
    bakers.value = (
        await pb.collection("teams_baker").getList(1, 50, {
          filter: `game="${props.game}"`,
        })
    ).items;

    // Process and calculate scores and points
    calculateScores();
  } catch (error) {
    console.error("Error fetching game data:", error);
  }
});

// Function to calculate scores for teams and bakers
function calculateScores() {
  // Initialize scores for each round
  const roundScores: Record<string, number>[] = Array(3)
      .fill(null)
      .map(() => ({}));
  const roundPoints: Record<string, number>[] = Array(3)
      .fill(null)
      .map(() => ({}));
  const roundScoresBakers: Record<string, number>[] = Array(3)
      .fill(null)
      .map(() => ({}));
  const roundPointsBakers: Record<string, number>[] = Array(3)
      .fill(null)
      .map(() => ({}));

  // Count players per team per round
  const playerCounts: Record<string, number>[] = Array(3)
      .fill(null)
      .map(() => ({}));

  // Populate team scores and count players
  games.value.forEach((game: any) => {
    const team = game.expand?.player?.team;
    const round = Number(game.round);
    const score = Number(game.score);

    if (team && score >= 0 && round > 0 && round <= 3) {
      roundScores[round - 1][team] =
          (roundScores[round - 1][team] || 0) + score;
      playerCounts[round - 1][team] =
          (playerCounts[round - 1][team] || 0) + 1;
    }
  });

  // Add 130 points for each missing player (each team should have 4 players)
  // Ensure all teams from props are accounted for in every round
  props.teams.forEach((team) => {
    roundScores.forEach((scores, roundIndex) => {
      if (!scores[team]) {
        scores[team] = 0;
      }
      const actualPlayers = playerCounts[roundIndex][team] || 0;
      const missingPlayers = Math.max(0, 4 - actualPlayers);
      scores[team] += missingPlayers * 130;
    });
  });

  // Populate baker scores (only for rounds 1 and 2)
  bakers.value.forEach((baker: any) => {
    const team = baker.team; // Assuming baker is tied to a team
    const round = Number(baker.round);
    const score = Number(baker.score);

    if (team && score >= 0 && round > 0 && round < 3) {
      // Only for rounds 1 and 2
      roundScoresBakers[round - 1][team] =
          (roundScoresBakers[round - 1][team] || 0) + score;
    }
  });

  // Calculate points for teams
  roundScores.forEach((scores, roundIndex) => {
    const teams = Object.keys(scores);
    const maxScore = Math.max(...Object.values(scores));

    teams.forEach((team) => {
      const teamScore = scores[team];
      const teamsWithMaxScore = teams.filter((t) => scores[t] === maxScore);

      if (teamScore === maxScore && teamsWithMaxScore.length > 1) {
        roundPoints[roundIndex][team] = 1; // Tie
      } else if (teamScore === maxScore) {
        roundPoints[roundIndex][team] = 2; // Win
      } else {
        roundPoints[roundIndex][team] = 0; // Loss
      }
    });
  });

  // Calculate points for bakers (only for rounds 1 and 2)
  roundScoresBakers.forEach((scores, roundIndex) => {
    if (roundIndex >= 2) return; // Skip round 3 for bakers
    const bakers = Object.keys(scores);

    // If no baker entries, give 1 point to each team
    if (bakers.length === 0) {
      props.teams.forEach((team) => {
        roundPointsBakers[roundIndex][team] = 1;
      });
      return;
    }

    const maxScore = Math.max(...Object.values(scores));

    bakers.forEach((baker) => {
      const bakerScore = scores[baker];
      const bakersWithMaxScore = bakers.filter((b) => scores[b] === maxScore);

      if (bakerScore === maxScore && bakersWithMaxScore.length > 1) {
        roundPointsBakers[roundIndex][baker] = 1; // Tie - each team gets 1 point
      } else if (bakerScore === maxScore) {
        roundPointsBakers[roundIndex][baker] = 2; // Win
      } else {
        roundPointsBakers[roundIndex][baker] = 0; // Loss
      }
    });
  });

  scoresByRound.value = roundScores;
  teamScores.value = roundPoints;
  scoresByRoundBakers.value = roundScoresBakers;
  bakerScores.value = roundPointsBakers;

  // Calculate total points for teams and bakers (including the highest total score bonus)
  calculateTotalPointsWithBonus();
}

// Function to calculate total points for both teams and bakers and add bonus for highest total score
function calculateTotalPointsWithBonus() {
  const teamTotals: Record<string, number> = {};
  const bakerTotals: Record<string, number> = {};
  const gamePts: Record<string, number> = {};
  const bakerPts: Record<string, number> = {};
  const gameScores: Record<string, number> = {};
  const bakerScoresTotal: Record<string, number> = {};
  const bonus: Record<string, number> = {};

  // Sum up team points from all rounds
  teamScores.value.forEach((roundPoints) => {
    Object.keys(roundPoints).forEach((team) => {
      teamTotals[team] = (teamTotals[team] || 0) + roundPoints[team];
      gamePts[team] = (gamePts[team] || 0) + roundPoints[team];
    });
  });

  // Sum up game scores from all rounds
  scoresByRound.value.forEach((roundScores) => {
    Object.keys(roundScores).forEach((team) => {
      gameScores[team] = (gameScores[team] || 0) + roundScores[team];
    });
  });

  // Sum up baker points from rounds 1 and 2
  bakerScores.value.forEach((roundPoints, roundIndex) => {
    if (roundIndex >= 2) return; // Skip round 3 for bakers
    Object.keys(roundPoints).forEach((baker) => {
      bakerTotals[baker] = (bakerTotals[baker] || 0) + roundPoints[baker];
      bakerPts[baker] = (bakerPts[baker] || 0) + roundPoints[baker];
      teamTotals[baker] = (teamTotals[baker] || 0) + roundPoints[baker];
    });
  });

  // Sum up baker scores from rounds 1 and 2
  scoresByRoundBakers.value.forEach((roundScores, roundIndex) => {
    if (roundIndex >= 2) return; // Skip round 3 for bakers
    Object.keys(roundScores).forEach((team) => {
      bakerScoresTotal[team] = (bakerScoresTotal[team] || 0) + roundScores[team];
    });
  });

  // Initialize bonus points to 0 for all teams
  Object.keys(teamTotals).forEach((team) => {
    bonus[team] = 0;
  });

  // Add bonus points for teams based on highest total scores
  const teamTotalScores = Object.values(teamTotals);
  const highestTotalScore = Math.max(...teamTotalScores);
  const teamsWithHighestScore = Object.keys(teamTotals).filter(
      (team) => teamTotals[team] === highestTotalScore
  );

  Object.keys(teamTotals).forEach((team) => {
    if (teamsWithHighestScore.length > 1) {
      // Tie scenario: All teams with the highest score get +1 point
      teamTotals[team] += 1;
      bonus[team] = 1;
    } else if (teamsWithHighestScore.includes(team)) {
      // Single winner: Team gets +2 bonus points
      teamTotals[team] += 2;
      bonus[team] = 2;
    }
  });

  totalTeamPoints.value = teamTotals;
  totalBakerPoints.value = bakerTotals;
  gamePoints.value = gamePts;
  bakerPoints.value = bakerPts;
  bonusPoints.value = bonus;
  totalGameScores.value = gameScores;
  totalBakerScores.value = bakerScoresTotal;

  // Calculate total scores (games + bakers)
  const totalScores: Record<string, number> = {};
  Object.keys(teamTotals).forEach((team) => {
    totalScores[team] = (gameScores[team] || 0) + (bakerScoresTotal[team] || 0);
  });
  totalTeamScores.value = totalScores;

  // Sort teams by total points (descending)
  sortedTeams.value = Object.keys(teamTotals).sort((a, b) => teamTotals[b] - teamTotals[a]);
}
</script>

<style scoped>
.scoreboard {
  padding: 16px;
}
h1 {
  text-align: center;
}
h2 {
  margin-top: 16px;
}
ul {
  list-style: none;
  padding: 0;
}
li {
  margin: 8px 0;
}
</style>