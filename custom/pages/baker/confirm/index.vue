<template>
  <section class="bg-white px-3 py-3">
    <div class="text-center">
      <h2 class="text-2xl font-bold mb-4">Baker Bestätigen</h2>
      <p class="mb-4">Baker ID: {{ route.query.game }}</p>
      <button v-if="!baker.locked" @click="update()" class="btn btn-primary">
        Bestätigen
      </button>
      <p v-else class="text-success">Bereits bestätigt</p>
    </div>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";

const route = useRoute();
const router = useRouter();
const pb = usePocketBase()

const baker = ref({});

const load = async () => {
  // The parameter is called 'game' but it actually contains the baker record ID
  const bakerId = route.query.game;
  if (!bakerId) {
    router.push('/dashboard');
    return;
  }
  baker.value = await pb.collection('teams_baker').getOne(bakerId as string)
}

const update = async () => {
  await pb.collection('teams_baker').update(
      baker.value.id, {
        locked: true
      }
  )
  history.back();
}

onMounted(() => {
  if (!pb.authStore.isValid) {
    navigateTo('/')
  }
  load();
});
</script>
