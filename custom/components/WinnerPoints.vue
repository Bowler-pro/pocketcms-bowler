<template>
  <button class="btn text-center btn-block btn-primary">
    Gewinner:
    {{ props.data.team?.name }}
    {{ props.data.score }}
  </button>
</template>

<script setup lang="ts">
const props = defineProps({
  data: {type: Object, required: true},
});
</script>