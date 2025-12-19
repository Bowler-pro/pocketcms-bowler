<template>
  {{team}}
</template>

<script setup lang="ts">
import {usePocketBase} from '@/utils/pocketbase'
const pb = usePocketBase();
const props = defineProps({
  identifier: String,
});
const team = ref({});

const load = async () => {
  pb.autoCancellation(false);
  team.value = await pb.collection('teams').getOne(props.identifier);
}

onMounted(()=>{
  load();
});
</script>