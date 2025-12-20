<template>
  <div class="card bg-gray-400  shadow-sm">
    <div class="card-body">
      <h2 class="card-title">{{props.data.name}}</h2>
      <p>A card component has a figure, a body part, and inside body there are title and actions parts</p>
      <div class="card-actions justify-end">
        <a :href="'/de/team/'+props.data.id" class="btn btn-primary">View</a>
        <a :href="'/de/players/'+props.data.id" class="btn btn-primary">Players ({{props.data.players.length}})</a>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
const props = defineProps({
  data: { type: Object },
});
</script>