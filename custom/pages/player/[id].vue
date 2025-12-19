<template>
  <section class="bg-white px-3 py-3">
    <h2 class="font-bold text-lg">{{player.name}}</h2>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from '@/utils/pocketbase'
const route = useRoute();
const player = ref({});
const pb = usePocketBase()

const load = async () => {
player.value = await pb.collection('players').getOne(route.params.id);
}

onMounted(()=>{
  load();
})

</script>