<template>
  <button class="btn text-center btn-block btn-primary">
    {{props.data}}
  </button>
</template>

<script setup lang="ts">
const props = defineProps({
  data: { type: Object ,required:true},
});
</script>