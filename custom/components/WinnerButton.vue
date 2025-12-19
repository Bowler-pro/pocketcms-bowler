<template>
  <button class="btn text-center btn-block btn-primary">
    {{props.data}}
    <TeamName :identifier="props.data.team" />
  </button>
</template>

<script setup lang="ts">
import TeamName from "./TeamName.vue";

const props = defineProps({
  data: { type: Object ,required:true},
});
</script>