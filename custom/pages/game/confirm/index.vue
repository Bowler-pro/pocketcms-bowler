<template>
  Confirm 123
  {{ route.query.game }}
  <button v-if="!game.locked" @click="update()" class="btn btn-primary">
    confirm
  </button>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";

const route = useRoute();
const router = useRouter();
const pb = usePocketBase()

const game = ref({});

const load = async () => {
  game.value = await pb.collection('players_game').getOne(route.query.game)
}

const update = async () => {
  await pb.collection('players_game').update(
      game.value.id, {
        locked: true
      }
  )
  history.back();
}

onMounted(() => {
  load();
});
</script>