<template>
  <div class="signup-container">
    <div class="signup-box">
      <h2>Sign Up</h2>
      <input type="email" v-model="email" placeholder="Email" />
      <input type="password" v-model="password" placeholder="Password" />
      <button @click="signUp" class="btn signup-btn">Sign Up</button>
      <router-link to="/login">Already have an account? Log in</router-link>

      <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    </div>
  </div>
</template>

<script>
import { auth } from '../firebase';
import { createUserWithEmailAndPassword } from 'firebase/auth';

export default {
  data() {
    return {
      email: '',
      password: '',
      errorMessage: '',
    };
  },
  methods: {
    async signUp() {
      try {
        this.errorMessage = '';

        if (this.emptyFields()) {
          return;
        }

        await createUserWithEmailAndPassword(auth, this.email, this.password);
        this.$router.push('/dashboard'); // Redirect to dashboard after successful sign-up
      } catch (error) {
        this.errorMessage = error.message;
      }
    },
    emptyFields() {
      if (!this.email || !this.password) {
        this.errorMessage = "You must enter an email and password.";
        return true;
      }
      return false;
    }
  }
};
</script>

<style scoped>
.signup-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.signup-box {
  background-color: white;
  padding: 2rem;
  border-radius: 12px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
  text-align: center;
  width: 300px;
}

.signup-box:hover {
  box-shadow: 0 0 24px rgba(0, 0, 0, 0.5);
}

.signup-box h2 {
  margin-bottom: 1.5rem;
  color: #333;
  font-family: 'Roboto', sans-serif;
  text-transform: uppercase;
}

input {
  width: 100%;
  padding: 0.75rem;
  margin: 0.5rem 0;
  border: 1px solid #ccc;
  border-radius: 8px;
  font-size: 1rem;
  outline: none;
  transition: border-color 0.3s ease;
}

input:focus {
  border-color: #4b79a1;
}

.btn {
  width: 100%;
  padding: 0.75rem;
  margin: 0.5rem 0;
  border: none;
  border-radius: 8px;
  color: white;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.signup-btn {
  background-color: #56ab2f;
}

.signup-btn:hover {
  background-color: #3b791c;
}

.error {
  color: #e74c3c;
  font-size: 0.9rem;
}
</style>
