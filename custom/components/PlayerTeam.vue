<template>
  <div v-if="player.expand?.team">{{player.expand.team.name}}</div>
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
  player.value = await pb.collection('player').getOne(props.identifier, {
    expand: 'team'
  });
}

onMounted(() => {
  load();
});
</script>