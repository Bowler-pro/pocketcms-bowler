<template>
  Teams
  <section class="grid grid-cols-6 gap-3">
    <div v-for="team in teams" class="col-span-6 md:col-span-2">
      <TeamCard :data="team" />
    </div>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import TeamCard from "../../components/TeamCard.vue";

const pb = usePocketBase()
const route = useRoute()
const clubs = ref({});
const teams = ref([]);

onMounted(async () => {
  clubs.value = await pb.collection('clubs').getOne(route.query.club, {
    expand: 'teams'
  });
  teams.value = clubs.value.expand.teams;
});
</script>