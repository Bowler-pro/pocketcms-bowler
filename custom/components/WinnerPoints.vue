<template>
  <button class="btn text-center btn-block btn-primary">
    Gewinner:
    {{ winners }}
  </button>
  {{ result }}
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";

const props = defineProps({
  league_game: {type: String, required: true},
  teams: {type: Object, required: true},
});

const pb = usePocketBase()

const games = ref([]);
const bakers = ref([]);
const winners = ref({});
const result = ref([]);

function calculatePointsForGame(teamScoresByRound) {
  const roundResults = []; // To store formatted results for each round
  const totalPoints = {}; // To store cumulative points across all rounds

  // Iterate through each round
  Object.keys(teamScoresByRound).forEach((round) => {
    const scores = teamScoresByRound[round];
    const teams = Object.keys(scores); // Teams in this round

    // Determine the highest score for this round
    const maxScore = Math.max(...Object.values(scores));
    const winners = teams.filter((team) => scores[team] === maxScore); // Teams with the max score

    const roundPoints = {};
    teams.forEach((team) => {
      if (winners.includes(team)) {
        // Winners each get 2 points
        roundPoints[team] = 2;
      } else {
        // Losers get 0 points
        roundPoints[team] = 0;
      }

      // Add points to the cumulative total
      if (!totalPoints[team]) {
        totalPoints[team] = 0; // Initialize total points if not already present
      }
      totalPoints[team] += roundPoints[team];
    });

    // Format the result for this round
    const formattedRound = teams
        .map((team) => `${team} (${roundPoints[team]})`)
        .join(" : ");
    const winnerText =
        winners.length === 1 ? `won by "${winners[0]}"` : `It's a draw`;

    roundResults.push({
      round,
      result: `${formattedRound}, ${winnerText}`
    });
  });

  // Determine overall winner(s) based on total points
  const maxTotalPoints = Math.max(...Object.values(totalPoints));
  const overallWinners = Object.keys(totalPoints).filter(
      (team) => totalPoints[team] === maxTotalPoints
  );

  return { roundResults, totalPoints, overallWinners };
}




function calculateScoresByRound(games) {
  const scoresByRound = {};

  // Loop through each game
  games.value.forEach((game) => {
    const round = game.round; // Get the round (e.g., "1", "2", "3")
    const team = game.expand.player.team; // Access the player's team
    const score = game.score || 0; // Get the score, default to 0 if not set

    // Initialize the round if not already added
    if (!scoresByRound[round]) {
      scoresByRound[round] = {};
    }

    // Initialize the team's score for this round if not already added
    if (!scoresByRound[round][team]) {
      scoresByRound[round][team] = 0;
    }

    // Sum up the team's score for this round
    scoresByRound[round][team] += score;
  });

  return scoresByRound;
}

function findWinnersByRound(teamScoresByRound) {
  const winnersByRound = {};

  Object.keys(teamScoresByRound).forEach((round) => {
    const scores = teamScoresByRound[round];
    let maxScore = -Infinity; // Start with the smallest number
    let winner = null;

    // Loop through each team's scores to find the highest score
    Object.keys(scores).forEach((team) => {
      if (scores[team] > maxScore) {
        maxScore = scores[team];
        winner = team; // Update the winner
      }
    });

    // Save the winner and their score for this round
    winnersByRound[round] = { winner: winner, score: maxScore };
  });

  return winnersByRound;
}

// Function to find the winner of the game by points
// Function to find the game winner or "remi" in case of a tie (same points for all teams)
function findGameWinner(teamScoresByRound) {
  const totalPoints = {}; // To aggregate points for all teams

  // Iterate through each round to calculate total points
  Object.keys(teamScoresByRound).forEach((round) => {
    const scores = teamScoresByRound[round];
    const teams = Object.keys(scores);
    const maxScore = Math.max(...Object.values(scores)); // Find the highest score in the round
    const winners = teams.filter((team) => scores[team] === maxScore); // Teams that are winners this round

    // Assign points (2 for winners, 0 for others)
    teams.forEach((team) => {
      if (!totalPoints[team]) totalPoints[team] = 0; // Initialize team total
      if (winners.includes(team)) {
        totalPoints[team] += 2; // Winners get 2 points
      }
    });
  });

  // Check if all teams have the same points
  const uniquePoints = new Set(Object.values(totalPoints));
  if (uniquePoints.size === 1) {
    return "remi"; // All teams have the same points
  }

  // Otherwise, find the team with the maximum points
  let maxPoints = -Infinity;
  let overallWinner = null;

  Object.keys(totalPoints).forEach((team) => {
    if (totalPoints[team] > maxPoints) {
      maxPoints = totalPoints[team];
      overallWinner = team;
    }
  });

  return overallWinner; // Return the single team ID or null if no rounds are played
}

async function calculateBakerWinnerWithTeam(bakers) {
  // Aggregate points for each team (team IDs grouped)
  const teamPoints = {}; // { teamId: totalPoints, ... }
  const bakerPoints = {}; // { bakerId: { teamId, points }, ... }

  bakers.value.forEach((baker) => {
    const bakerId = baker.id; // Unique baker ID
    const teamId = baker.team; // Use the baker's team field
    const points = baker.points || 0; // Use `points` column or appropriate field, default 0

    // Track points for each baker
    bakerPoints[bakerId] = {
      team: teamId,
      points: points
    };

    // Aggregate points for each team
    if (!teamPoints[teamId]) {
      teamPoints[teamId] = 0; // Initialize team total points
    }
    teamPoints[teamId] += points;
  });

  // Find the baker(s) with the highest score
  const maxPoints = Math.max(...Object.values(bakerPoints).map((baker) => baker.points));
  const topBakers = Object.keys(bakerPoints).filter(
      (bakerId) => bakerPoints[bakerId].points === maxPoints
  );

  // Award 2 bonus points to the top-performing baker(s)
  topBakers.forEach((bakerId) => {
    const teamId = bakerPoints[bakerId].team; // Get the team ID
    const bonusPoints = 2;

    // Add bonus points to the baker and their team
    bakerPoints[bakerId].points += bonusPoints;
    teamPoints[teamId] += bonusPoints;
  });

  // Find the team with the highest total points
  const maxTeamPoints = Math.max(...Object.values(teamPoints));
  const potentialWinningTeams = Object.keys(teamPoints).filter(
      (teamId) => teamPoints[teamId] === maxTeamPoints
  );

  // Determine overall winner (if tied, results in "remi")
  const overallWinner = potentialWinningTeams.length > 1 ? "remi" : potentialWinningTeams[0];

  return {
    overallWinner,
    bakerPoints,
    teamPoints
  };
}


onMounted(async () => {
  games.value = (await pb.collection('players_game').getList(1, 100, {
    filter: 'game="' + props.league_game + '"',
    expand: 'player'
  })).items;

  bakers.value = (await pb.collection('teams_baker').getList(1, 100, {
    filter: 'game="' + props.league_game + '"'
  })).items;

  const result = await calculateBakerWinnerWithTeam(bakers);

  console.log("Baker Points:", result.teamPoints);


  // Calculate team scores by round
  const teamScoresByRound = calculateScoresByRound(games);

  winners.value = findWinnersByRound(teamScoresByRound);
// Execute the function with the sample data
  const { roundResults, totalPoints, overallWinners } = calculatePointsForGame(
      teamScoresByRound
  );
// Total Points
  console.log("\nTotal Points:");
  Object.keys(totalPoints).forEach((team) =>
      console.log(`${team}: ${totalPoints[team]}`)
  );

  const gameWinner = findGameWinner(teamScoresByRound);
  console.log('Gewinner: '+gameWinner);
})
</script>