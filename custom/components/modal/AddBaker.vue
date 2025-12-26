<script setup lang="ts">
import {useLocalStorage} from "@vueuse/core";
import {usePocketBase} from "@/utils/pocketbase";

const modalAddBaker = useLocalStorage('modalAddBaker', false, {})
const modalAddBakerGame = useLocalStorage('modalAddBakerGame', '', {})
const modalAddBakerTeam = useLocalStorage('modalAddBakerTeam', '', {})

const add = async () => {
  await pb.collection('teams_baker').create(form.value)
  modalAddBaker.value = false;
  modalAddBakerGame.value = '';
  modalAddBakerTeam.value = '';
}

const pb = usePocketBase()

const form = ref({
  team: '',
  game: '',
  score: 0,
  round: 1
});

const games = ref([]);
const teams = ref([]);

watch(modalAddBaker, ()=>{
  form.value.game = modalAddBakerGame.value;
  form.value.team = modalAddBakerTeam.value;
})

onMounted(async() => {
  modalAddBaker.value = false

  games.value = await pb.collection('league_game').getFullList(100)
  teams.value = await pb.collection('teams').getFullList(100)
})
</script>

<template>
  <input type="checkbox" v-model="modalAddBaker" class="modal-toggle"/>
  <div class="modal" role="dialog">
    <div class="modal-box">
      <h2 class="font-bold text-xl mb-5">Add Baker</h2>
      <form class="space-y-5">
        <label class="floating-label">
          <span>Your Team</span>
          <select class="select select-disabled w-full select-sm">
            <option v-for="game in teams" :value="game.id">{{game.name}}</option>
          </select>
        </label>
        <label class="floating-label">
          <span>Your Game</span>
          <select class="select select-disabled w-full select-sm">
            <option v-for="game in games" :value="game.id">{{game.name}}</option>
          </select>
        </label>
        <label class="floating-label">
          <span>Your Score</span>
          <input type="number" required v-model="form.score" class="input w-full validator"/>
        </label>
        <label class="floating-label">
          <span>Your Round</span>
          <input type="number" min="1" max="3" required v-model="form.round" class="input w-full validator"/>
        </label>

      </form>
      <label class="modal-backdrop" @click="modalAddBaker = false">Close</label>
      <div class="modal-action">
        <button class="btn btn-primary" @click="add()">hinzufügen</button>
        <label @click="modalAddBaker = false" class="btn btn-error">Close!</label>
      </div>
    </div>
  </div>
</template>