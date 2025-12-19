<template>
  <section class="bg-white px-3 py-3">
    <h2 class="font-bold text-lg">{{team.name}}</h2>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from '@/utils/pocketbase'
const route = useRoute();
const team = ref({});
const pb = usePocketBase()

const load = async () => {
  team.value = await pb.collection('teams').getOne(route.params.id);
}

onMounted(()=>{
  load();
})
</script>