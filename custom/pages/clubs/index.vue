<template>
  Club
<section class="grid grid-cols-6 gap-3">
  <div v-for="club in clubs" class="col-span-6 md:col-span-2">
    <ClubCard :data="club" />
  </div>
</section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import ClubCard from "@/components/ClubCard.vue";

const pb = usePocketBase()
const clubs = ref([]);

onMounted(async () => {
  clubs.value = await pb.collection('clubs').getFullList(25);
});
</script>