<template>
  <div id="app">
    <header class="app-header">
      <h1>Note & Task Management App</h1>
      <div v-if="user" class="user-info">
        <p>Welcome, {{ user.email }}</p>
        <button @click="signOut" class="btn logout-btn">Sign Out</button>
      </div>
    </header>

    <div class="app-body" v-if="user">
      <aside class="sidebar">
        <Task @task-selected="selectTask" @task-deleted="handleTaskDeletion" />
      </aside>

      <section class="main-content">
        <Notes v-if="selectedTask" :taskId="selectedTask.id" :taskTitle="selectedTask.title" @note-added="fetchTasks" />
      </section>
    </div>

    <main v-else>
      <router-view />
    </main>
  </div>
</template>

<script>
import { auth } from './firebase';
import { signOut as firebaseSignOut } from 'firebase/auth';
import Task from './components/Task.vue';
import Notes from './components/Notes.vue';

export default {
  name: 'App',
  data() {
    return {
      user: null,
      unsubscribe: null,
      selectedTask: null
    };
  },
  components: {
    Task,
    Notes
  },
  created() {
    this.unsubscribe = auth.onAuthStateChanged(user => {
      this.user = user;
      if (!user) {
        this.$router.push('/login');
      }
    });
  },
  methods: {
    async signOut() {
      try {
        await firebaseSignOut(auth);
        this.user = null;
        this.$router.push('/login');
      } catch (error) {
        console.error("Error signing out:", error.message);
      }
    },
    selectTask(task) {
      this.selectedTask = task;
    },
    async fetchTasks() {
      if (this.user) {
        try {
          const tasksCollection = collection(db, `users/${this.user.uid}/tasks`);
          const tasksSnapshot = await getDocs(tasksCollection);
          this.tasks = tasksSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        } catch (error) {
          console.error("Error fetching tasks:", error);
        }
      }
    },
    handleTaskDeletion() {
      this.selectedTask = null;
      this.fetchTasks();
    }
  },
  beforeDestroy() {
    if (this.unsubscribe) {
      this.unsubscribe();
    }
  }
};
</script>

<style>
body {
  margin: 0;
}

#app {
  font-family: 'Roboto', sans-serif;
  color: #2c3e50;
}

.app-header {
  background-color: #4b79a1;
  color: white;
  padding: 1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.app-header h1 {
  margin: 0;
  font-size: 2rem;
}

.user-info {
  display: flex;
  align-items: center;
}

.user-info p {
  margin: 0;
  margin-right: 1rem;
}

.logout-btn {
  background-color: #e74c3c;
  border: none;
  border-radius: 8px;
  color: white;
  padding: 0.5rem 1rem;
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.2s ease;
}

.logout-btn:hover {
  background-color: #c0392b;
  transform: scale(1.05);
}

/* Main body layout with sidebar and main content */
.app-body {
  display: flex;
  height: calc(100vh - 60px);
}

.sidebar {
  width: 20%;
  height: 100%;
  overflow-y: auto;
  background-color: #f5f5f5;
  border-right: 1px solid #7f8c8d;
}

.main-content {
  flex: 1;
  padding: 0 1rem;
  background-color: #eef2f3;
  overflow-y: auto;
  background: rgb(195, 221, 229);
}

main {
  display: flex;
  justify-content: center;
  align-items: center;
  height: calc(100vh - 60px);
  padding: 1rem;
  background: rgb(195, 221, 229);
}
</style>
