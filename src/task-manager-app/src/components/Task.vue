<template>
  <button class="add-task-btn" @click="toggleTaskForm">+ Add Task</button>
  <div>
    <div v-if="showTaskForm" class="task-form">
      <input v-model="newTaskTitle" placeholder="Task Title" />
      <button @click="addTask">Submit</button>
      <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    </div>

    <ul class="task-list">
      <li v-for="task in tasks" :key="task.id" class="task-item" @click="selectTask(task)">
        <div v-if="editingTaskId !== task.id" class="task-display">
          <span :class="{ done: task.done }" class="task-title">{{ task.title }}</span>
          <div class="task-actions">
            <button @click.stop="markAsDone(task)" title="Mark as done" class="icon-btn">
              <i class="fas fa-check"></i>
            </button>
            <button @click.stop="startEditingTask(task.id)" title="Edit task" class="icon-btn">
              <i class="fas fa-pencil-alt"></i>
            </button>
            <button @click.stop="deleteTask(task.id)" title="Delete task" class="icon-btn">
              <i class="fas fa-trash"></i>
            </button>
          </div>
        </div>

        <div v-else class="edit-task-form">
          <input v-model="editTitle" placeholder="Edit Task Title" />
          <div class="edit-task-buttons">
            <button @click="updateTask(task.id)" class="edit-button">Update</button>
            <button @click="cancelEdit" class="edit-button">Cancel</button>
          </div>
        </div>
      </li>
    </ul>
  </div>
</template>

<script>
import { collection, addDoc, getDocs, deleteDoc, writeBatch, updateDoc, doc } from "firebase/firestore";
import { db } from '../firebase';
import { auth } from '../firebase';

export default {
  emits: ['task-selected'],
  data() {
    return {
      showTaskForm: false,
      newTaskTitle: '',
      tasks: [],
      errorMessage: '',
      user: null,
      editingTaskId: null, 
      editTitle: '',
    };
  },
  created() {
    this.checkAuthState();
  },
  methods: {
    checkAuthState() {
      auth.onAuthStateChanged(async (user) => {
        if (user) {
          this.user = user;
          await this.fetchTasks();
        } else {
          console.error("No user is signed in.");
        }
      });
    },
    toggleTaskForm() {
      this.showTaskForm = !this.showTaskForm;
    },
    async addTask() {
      if (this.newTaskTitle.trim() === '') {
        this.errorMessage = "Task title cannot be empty.";
        return;
      }

      if (this.user) {
        try {
          const userTasksCollection = collection(db, `users/${this.user.uid}/tasks`);
          await addDoc(userTasksCollection, {
            title: this.newTaskTitle,
            createdAt: new Date(),
            done: false
          });
          this.newTaskTitle = '';
          this.showTaskForm = false;
          this.fetchTasks();
          this.errorMessage = '';
        } catch (error) {
          console.error("Error adding task:", error);
          this.errorMessage = "Failed to add task. Please try again.";
        }
      } else {
        console.error("User is not authenticated.");
        this.errorMessage = "You must be logged in to add a task.";
      }
    },
    async fetchTasks() {
      if (this.user) {
        try {
          const userTasksCollection = collection(db, `users/${this.user.uid}/tasks`);
          const taskSnapshot = await getDocs(userTasksCollection);
          this.tasks = taskSnapshot.docs
            .map(doc => ({ id: doc.id, ...doc.data() }))
            .sort((a, b) => a.createdAt.toDate() - b.createdAt.toDate());
        } catch (error) {
          console.error("Error fetching tasks:", error);
        }
      } else {
        console.error("User is not authenticated.");
      }
    },
    async deleteTask(taskId) {
      if (this.user && taskId) {
        try {
          const batch = writeBatch(db);

          const taskRef = doc(db, `users/${this.user.uid}/tasks/${taskId}`);
          batch.delete(taskRef);

          const notesCollection = collection(db, `users/${this.user.uid}/tasks/${taskId}/notes`);
          const notesSnapshot = await getDocs(notesCollection);
          
          notesSnapshot.docs.forEach(noteDoc => {
            batch.delete(doc(db, `users/${this.user.uid}/tasks/${taskId}/notes/${noteDoc.id}`));
          });

          await batch.commit();
          this.fetchTasks();
          this.$emit('task-deleted');
        } catch (error) {
          console.error("Error deleting task and notes:", error);
        }
      } else {
        console.error("Unable to delete task. Either the user is not authenticated or taskId is missing.");
      }
    },
    async markAsDone(task) {
      if (this.user && task.id) {
        try {
          const taskRef = doc(db, `users/${this.user.uid}/tasks/${task.id}`);
          await updateDoc(taskRef, { done: !task.done });
          this.fetchTasks();
        } catch (error) {
          console.error("Error marking task as done:", error);
        }
      } else {
        console.error("Unable to mark task as done. Either the user is not authenticated or taskId is missing.");
      }
    },
    startEditingTask(taskId) {
      const task = this.tasks.find(t => t.id === taskId);
      if (task) {
        this.editingTaskId = taskId;
        this.editTitle = task.title;
      }
    },
    async updateTask(taskId) {
      if (this.user && taskId) {
        try {
          const taskRef = doc(db, `users/${this.user.uid}/tasks/${taskId}`);
          await updateDoc(taskRef, { title: this.editTitle });
          this.editingTaskId = null;
          this.fetchTasks();
        } catch (error) {
          console.error("Error updating task:", error);
        }
      } else {
        console.error("Unable to update task. Either the user is not authenticated or task is missing.");
      }
    },
    cancelEdit() {
      this.editingTaskId = null;
    },
    selectTask(task) {
      this.$emit('task-selected', task);
    }
  }
};
</script>

<style scoped>
/* Button Styles */
.icon-btn {
  background: none;
  border: none;
  cursor: pointer;
  margin: 0 5px;
}

.icon-btn i {
  font-size: 1rem;
  transition: color 0.2s ease;
}

.icon-btn:hover i {
  color: #3498db;
}

/* Add Task Button Styles */
.add-task-btn {
  background-color: #56ab2f;
  color: white;
  font-size: 15px;
  font-weight: bold;
  padding: 0.75rem 1.5rem;
  border: none;
  cursor: pointer;
  box-sizing: border-box;
  width: 100%;
  transition: background-color 0.3s ease;
}

.add-task-btn:hover {
  background-color: #3b791c;
}

/* Task Form Styles */
.task-form {
  margin: 0.25rem;
}

.task-form input {
  width: calc(100% - 2rem);
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  border-radius: 8px;
  border: 1px solid #ccc;
}

.task-form button {
  background-color: #3498db;
  color: white;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.task-form button:hover {
  background-color: #2980b9;
}

/* Task List Styles */
.task-list {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

.task-item {
  display: flex;
  flex-direction: column;
  padding: 0.5rem;
  border-bottom: 1px solid #7f8c8d;
  cursor: pointer;
  transition: background-color 0.3s ease;
  box-sizing: border-box;
}

.task-title {
  display: inline-block;
  max-width: 85%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.task-item:hover {
  background: #9a9a9a;
}

.task-actions {
  padding: 0.25rem;
  display: flex;
  justify-content: flex-end;
}

.done {
  text-decoration: line-through;
  color: #7f8c8d;
}

/* Edit Task Form Styles */
.edit-task-form {
  display: flex;
  flex-direction: column; 
  width: 100%; 
  box-sizing: border-box;
}

.edit-task-form input {
  width: 100%;
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  border-radius: 8px;
  border: 1px solid #ccc;
  box-sizing: border-box;
}

.edit-task-buttons {
  display: flex;
  justify-content: space-evenly;
}

.edit-button {
  background-color: #3498db;
  color: white;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.2s ease;
}

.edit-button:hover {
  background-color: #2980b9;
  transform: scale(1.05);
}

</style>
