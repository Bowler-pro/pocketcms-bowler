<template>
  Players
  <section class="grid grid-cols-6 gap-3">
    <div v-for="player in players" class="col-span-6 md:col-span-2">
      <PlayerCard :data="player" />
    </div>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import PlayerCard from "@/components/PlayerCard.vue";

const pb = usePocketBase()
const route = useRoute()
const players = ref({});

onMounted(async () => {
  players.value = (await pb.collection('players').getList(1,10, {
    filter: 'team="'+route.query.team+'"'
  })).items;
});
</script>