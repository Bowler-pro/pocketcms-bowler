<template>
  {{player.name}}
</template>

<script setup lang="ts">
import {usePocketBase} from '@/utils/pocketbase'
const pb = usePocketBase();
const props = defineProps({
  identifier: String,
});
const player = ref({});

const load = async () => {
  pb.autoCancellation(false);
  player.value = await pb.collection('players').getOne(props.identifier);
}

onMounted(()=>{
  load();
});
</script>