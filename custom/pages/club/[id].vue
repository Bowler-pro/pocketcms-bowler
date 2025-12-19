<template>
  <section class="bg-white px-3 py-3">
    <h2 class="font-bold text-lg">{{club.name}}</h2>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from '@/utils/pocketbase'
const route = useRoute();
const club = ref({});
const pb = usePocketBase()

const load = async () => {
  club.value = await pb.collection('clubs').getOne(route.params.id);
}

onMounted(()=>{
  load();
})
</script>