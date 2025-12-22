<template>
  <input type="checkbox" v-model="modalGameConfirm" class="modal-toggle" />
  <div class="modal" role="dialog">
    <div class="modal-box">
      <h3 class="text-lg font-bold">{{confirmPlayerGame}}</h3>
      <p class="py-4">{{game}}</p>
      <label class="modal-backdrop" @click="modalGameConfirm = false">Close</label>
      <div class="modal-action">
        <button class="btn btn-primary" @click="update()">confirm</button>
        <label @click="modalGameConfirm = false" class="btn btn-error">Close!</label>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useLocalStorage } from "@vueuse/core";
import {usePocketBase} from "@/utils/pocketbase";
const modalGameConfirm = useLocalStorage('modalGameConfirm', false, {})
const confirmPlayerGame = useLocalStorage('confirmPlayerGame', '', {})

const game = ref({});
const pb = usePocketBase()

const load = async () => {
  game.value = await pb.collection('players_game').getOne(confirmPlayerGame.value);
}
const update = async () => {
  await pb.collection('players_game').update(
      game.value.id, {
        locked: true
      }
  )
  modalGameConfirm.value = false
  confirmPlayerGame.value = ''
}


watch(confirmPlayerGame, ()=>load())
onMounted(()=>{
  modalGameConfirm.value = false
  confirmPlayerGame.value = ''
})
</script>