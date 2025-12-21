<template>
  <section class="bg-white px-3 py-3">
    <section class="grid grid-cols-6 gap-3">
      <div class="col-span-6 md:col-span-3">
        <form @submit.prevent="update()">
          <label class="floating-label mt-3">
            <span>Your Player</span>
            <input type="text" disabled v-model="player.name" class="validator input w-full"/>
          </label>
          <label class="floating-label mt-3">
            <span>Your League</span>
            <input type="text" disabled v-model="league.name" class="validator input w-full"/>
          </label>
          <label class="floating-label mt-3">
            <span>Your Game</span>
            <input type="text" disabled v-model="game.name" class="validator input w-full"/>
          </label>
          <label class="floating-label mt-3 hidden">
            <span>Your Game</span>
            <input type="text" required v-model="form.game" class="validator input w-full"/>
          </label>
          <label class="floating-label mt-3">
            <span>Your Score</span>
            <input type="number" required v-model="form.score" class="validator input w-full"/>
          </label>
          <section class="actions flex justify-between mt-3">
            <button @click="remove(form.id)" class="btn btn-error">löschen</button>
            <button type="submit" class="btn btn-primary">
              update
            </button>
          </section>
        </form>
      </div>
    </section>
  </section>
</template>
<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";

const route = useRoute()
const router = useRouter()
const pb = usePocketBase()
const form = ref({
  game: '',
  score: '',
  player: '',
});

const player = ref({});
const game = ref({});
const league = ref({});

const load = async () => {
  form.value = await pb.collection('players_game').getOne(route.query.game);
  player.value = await pb.collection('players').getOne(form.value.player);
  game.value = await pb.collection('league_game').getOne(form.value.game);
  league.value = await pb.collection('leagues').getOne(game.value.league);
}

onMounted(async () => {
  if (!pb.authStore.isValid) {
    navigateTo('/')
  }
  await load();
  addBreadcrumb({
    icon: null,
    link: 'game/' + form.value.game,
    code: 'back-to-game'
  })
});

const remove = async (id) => {
  if (confirm('Eintrag löschen ?')) {
    await pb.collection('players_game').delete(id)
    router.push('/game/' + form.value.game);
  }
}

const update = async () => {
  await pb.collection('players_game').update(form.value.id, form.value);
  router.push('/game/' + form.value.game)
}
</script>