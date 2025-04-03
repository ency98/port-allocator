<!-- src/views/Register.vue -->
<template>
  <div class="register">
    <h2>Register</h2>
    <div v-if="error" class="error">{{ error }}</div>
    <form @submit.prevent="register">
      <div class="form-group">
        <label for="email">Email</label>
        <input type="email" id="email" v-model="email" required />
      </div>
      <div class="form-group">
        <label for="password">Password</label>
        <input type="password" id="password" v-model="password" required />
      </div>
      <div class="form-group">
        <label for="confirmPassword">Confirm Password</label>
        <input type="password" id="confirmPassword" v-model="confirmPassword" required />
      </div>
      <button type="submit" :disabled="loading">{{ loading ? 'Registering...' : 'Register' }}</button>
    </form>
    <p>Already have an account? <router-link to="/login">Login</router-link></p>
  </div>
</template>

<script>
import { ref } from 'vue';
import { getAuth, createUserWithEmailAndPassword } from 'firebase/auth';
import { useRouter } from 'vue-router';

export default {
  name: 'Register',
  setup() {
    const email = ref('');
    const password = ref('');
    const confirmPassword = ref('');
    const error = ref('');
    const loading = ref(false);
    const router = useRouter();

    const register = async () => {
      loading.value = true;
      error.value = '';

      if (password.value !== confirmPassword.value) {
        error.value = 'Passwords do not match';
        loading.value = false;
        return;
      }

      try {
        const auth = getAuth();
        await createUserWithEmailAndPassword(auth, email.value, password.value);
        router.push('/');
      } catch (err) {
        console.error('Registration error:', err);
        error.value = err.message || 'Failed to register';
      } finally {
        loading.value = false;
      }
    };

    return {
      email,
      password,
      confirmPassword,
      error,
      loading,
      register
    };
  }
}
</script>

<style scoped>
.register {
  max-width: 400px;
  margin: 0 auto;
  padding: 20px;
  border: 1px solid #eee;
  border-radius: 8px;
}

.form-group {
  margin-bottom: 15px;
  text-align: left;
}

label {
  display: block;
  margin-bottom: 5px;
}

input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.error {
  color: red;
  margin-bottom: 15px;
}
</style>