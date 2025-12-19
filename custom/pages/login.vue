<template>
  <div class="flex h-screen">
    <!-- Login Card -->
    <div class="m-auto p-6 max-w-md bg-base-200 rounded-lg shadow-xl">
      <h1 class="text-center text-2xl font-bold text-primary mb-6">Login to PocketBase</h1>

      <form @submit.prevent="handleLogin">
        <!-- Email Input -->
        <div class="form-control mb-4">
          <label class="label" for="email">
            <span class="label-text">Email</span>
          </label>
          <input
              id="email"
              type="email"
              v-model="email"
              class="input input-bordered input-primary"
              placeholder="Enter your email"
              required
          />
        </div>

        <!-- Password Input -->
        <div class="form-control mb-6">
          <label class="label" for="password">
            <span class="label-text">Password</span>
          </label>
          <input
              id="password"
              type="password"
              v-model="password"
              class="input input-bordered input-primary"
              placeholder="Enter your password"
              required
          />
        </div>

        <!-- Submit Button -->
        <div class="form-control">
          <button
              type="submit"
              class="btn btn-primary w-full"
              :class="{ 'loading': isLoading }"
              :disabled="isLoading"
          >
            {{ isLoading ? "Logging in..." : "Login" }}
          </button>
        </div>

        <!-- Error Message -->
        <div v-if="errorMessage" class="text-error mt-4 text-sm text-center">
          {{ errorMessage }}
        </div>
      </form>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import {usePocketBase} from "@/utils/pocketbase";

// Define state using Composition API
const email = ref("");
const password = ref("");
const isLoading = ref(false);
const errorMessage = ref("");
// Create PocketBase client
const pb = usePocketBase()

onMounted(()=>{
  if(pb.authStore.isValid){
    navigateTo('/dashboard');
  }
})

// Login handler function
async function handleLogin() {
  isLoading.value = true;
  errorMessage.value = "";

  try {
    // Authenticate with email and password
    const user = await pb.collection("players").authWithPassword(email.value, password.value);
    console.log("User authenticated successfully:", user);

    navigateTo('/dashboard');
  } catch (error) {
    console.error("Login failed:", error);
    errorMessage.value = "Invalid email or password. Please try again.";
  } finally {
    isLoading.value = false;
  }
}
</script>

<style scoped>
/* Optionally place custom styles */
</style>