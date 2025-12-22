<template>
  <section class="bg-white px-3 py-3">
    <section class="grid grid-cols-6 gap-3">
      <div class="col-span-6 md:col-span-3">
        <form @submit.prevent="add()">
          <label class="floating-label mt-3">
            <span>Your Name</span>
            <input type="text" required v-model="form.name" class="validator input w-full" />
          </label>
          <label class="floating-label mt-3">
            <span>Your Name</span>
            <select v-model="form.round" class="select w-full">
              <option value="1">1</option>
              <option value="2">2</option>
              <option value="3">3</option>
            </select>
          </label>
          <label class="floating-label mt-3">
            <span>Your Game</span>
            <input type="text" required v-model="form.game" class="validator input w-full" />
          </label>
          <label class="floating-label mt-3">
            <span>Your Score</span>
            <input type="number" required v-model="form.score" class="validator input w-full" />
          </label>
          <section class="actions flex justify-end mt-3">
            <button  type="submit" class="btn btn-primary">
              add
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
  round: '',
  name: '',
  score: '',
  player: '',
});
onMounted(()=>{
  if(!pb.authStore.isValid){
    navigateTo('/')
  }

  form.value.player = pb.authStore.record.id;
  form.value.game = route.query.game;
  form.value.team = route.query.team;
});

const add = async () => {
  await pb.collection('players_game').create(form.value);
  router.push('/game/'+form.value.game)
}
</script>