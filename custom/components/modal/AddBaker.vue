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

watch(modalAddBaker, ()=>{
  form.value.game = modalAddBakerGame.value;
  form.value.team = modalAddBakerTeam.value;
})

onMounted(() => {
  modalAddBaker.value = false
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
          <input type="text" v-model="form.team" required class="input w-full validator"/>
        </label>
        <label class="floating-label">
          <span>Your Game</span>
          <input type="text" v-model="form.game" required class="input w-full validator"/>
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