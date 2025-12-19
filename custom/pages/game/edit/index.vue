<template>
  <section class="bg-white px-3 py-3">
    <section class="grid grid-cols-6 gap-3">
      <div class="col-span-6 md:col-span-3">
        <form @submit.prevent="update()">
          <label class="floating-label mt-3">
            <span>Your Name</span>
            <input type="text" required v-model="form.name" class="validator input w-full" />
          </label>
          <label class="floating-label mt-3 hidden">
            <span>Your Game</span>
            <input type="text" required v-model="form.game" class="validator input w-full" />
          </label>
          <label class="floating-label mt-3">
            <span>Your Score</span>
            <input type="number" required v-model="form.score" class="validator input w-full" />
          </label>
          <section class="actions flex justify-between mt-3">
            <button @click="remove(form.id)" class="btn btn-error">löschen</button>
            <button  type="submit" class="btn btn-primary">
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
  name: '',
  score: '',
  player: '',
});

const load = async () => {
  form.value = await pb.collection('players_game').getOne(route.query.game);
}

onMounted(async ()=>{
  if(!pb.authStore.isValid){
    navigateTo('/')
  }
  await load();
  addBreadcrumb({
    icon: null,
    link: 'game?id='+form.value.game,
    code: 'back-to-game'
  })
});

const remove = async (id) =>{
  if(confirm('Eintrag löschen ?')){
    await pb.collection('players_game').delete(id)
    router.push('game?id='+form.value.game);
  }
}

const update = async () => {
  await pb.collection('players_game').update(form.value.id, form.value);
  router.push('/game?id='+form.value.game)
}
</script>