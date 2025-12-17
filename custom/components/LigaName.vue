<template>
  {{liga.name}}
</template>

<script setup lang="ts">
import {usePocketBase} from '@/utils/pocketbase'
const pb = usePocketBase();
const props = defineProps({
  identifier: String,
});
const liga = ref({});

const load = async () => {
  liga.value = await pb.collection('leagues').getOne(props.identifier);
}

onMounted(()=>{
  load();
});
</script>