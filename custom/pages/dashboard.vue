<template>
  <section class="bg-white px-3 py-3">
    <div class="overflow-x-auto">
      <table class="table">
        <!-- head -->
        <thead>
        <tr>
          <th>Name</th>
          <th>Job</th>
        </tr>
        </thead>
        <tbody>
        <!-- row 1 -->
        <tr v-for="(liga,index) in ligen" :class="{
          'bg-gray-400': index % 2 == 0,
          'bg-gray-300': index % 2 != 0
        }">
          <td>
            {{liga.name}}
          </td>
          <td>
            <a :href="'/de/games/'+liga.id" class="btn btn-sm btn-primary">
              anschauen
            </a>
          </td>
        </tr>
        </tbody>
      </table>
    </div>
  </section>
</template>

<script setup lang="ts">
import {usePocketBase} from "@/utils/pocketbase";
import {onMounted} from "vue";

const pb = usePocketBase()

const ligen = ref([]);

const load = async () => {
  ligen.value = (await pb.collection('leagues').getList(1,10,{})).items;
}

onMounted(()=>{
  if(!pb.authStore.isValid){
    navigateTo('/login');
  }

  load();
})
</script>