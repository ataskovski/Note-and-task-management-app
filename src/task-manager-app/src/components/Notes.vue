<template>
  <div>

    <div class="title_and_button">
      <span id="title">{{ taskTitle }}</span>
      <button class="add-note-btn" @click="toggleNoteForm">+ Add Note</button>
    </div>

    <div v-if="showNoteForm" class="note-form">
      <input v-model="newNoteContent" placeholder="Note Content" />
      <button @click="addNote">Submit</button>
      <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    </div>

    <ul class="notes-list">
      <li v-for="note in notes" :key="note.id" class="note-item">
        <div v-if="editingNoteId !== note.id" class="note-display">
          <span class="note-text" :class="{ done: note.done }">{{ note.content }}</span>
          <div class="note-actions">
            <button @click="markAsDone(note)" title="Mark as done" class="icon-btn">
              <i class="fas fa-check"></i>
            </button>
            <button @click="startEditingNote(note.id)" title="Edit note" class="icon-btn">
              <i class="fas fa-pencil-alt"></i>
            </button>
            <button @click="deleteNote(note.id)" title="Delete note" class="icon-btn">
              <i class="fas fa-trash"></i>
            </button>
          </div>
        </div>

        <div v-else class="edit-note-form">
          <input v-model="editingNoteContent" placeholder="Edit Note Content" />
          <div class="edit-note-buttons">
            <button @click="updateNote(note.id)" class="edit-button">Update</button>
            <button @click="cancelEdit" class="edit-button">Cancel</button>
          </div>
        </div>
      </li>
    </ul>
  </div>
</template>

<script>
import { collection, addDoc, getDocs, deleteDoc, updateDoc, doc } from "firebase/firestore";
import { db } from '../firebase';
import { auth } from '../firebase';

export default {
  props: {
    taskId: {
      type: String,
      required: true
    },
    taskTitle: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      showNoteForm: false,
      newNoteContent: '',
      notes: [],
      errorMessage: '',
      user: null,
      editingNoteId: null,
      editingNoteContent: ''
    };
  },
  watch: {
    taskId: 'fetchNotes'
  },
  created() {
    auth.onAuthStateChanged(user => {
      this.user = user;
      if (this.user) {
        this.fetchNotes();
      } else {
        console.error("No user is authenticated.");
        this.errorMessage = "You must be logged in to view notes.";
      }
    });
  },
  methods: {
    async fetchNotes() {
      if (this.user) {
        try {
          const notesCollection = collection(db, `users/${this.user.uid}/tasks/${this.taskId}/notes`);
          const notesSnapshot = await getDocs(notesCollection);
          this.notes = notesSnapshot.docs
          .map(doc => ({ id: doc.id, ...doc.data() }))
          .sort((a, b) => a.createdAt.toDate() - b.createdAt.toDate());
        } catch (error) {
          console.error("Error fetching notes:", error);
          this.errorMessage = "Failed to fetch notes. Please try again.";
        }
      }
    },

    async addNote() {
      if (this.newNoteContent.trim() === '') {
        this.errorMessage = "Note content cannot be empty.";
        return;
      }

      if (this.user) {
        try {
          const notesCollection = collection(db, `users/${this.user.uid}/tasks/${this.taskId}/notes`);
          await addDoc(notesCollection, {
            content: this.newNoteContent,
            done: false,
            createdAt: new Date()
          });
          this.newNoteContent = '';
          this.errorMessage = '';
          this.fetchNotes();
          this.toggleNoteForm();
        } catch (error) {
          console.error("Error adding note:", error);
          this.errorMessage = "Failed to add note. Please try again.";
        }
      }
    },

    async deleteNote(noteId) {
      if (this.user && noteId) {
        try {
          const noteRef = doc(db, `users/${this.user.uid}/tasks/${this.taskId}/notes/${noteId}`);
          await deleteDoc(noteRef);
          this.fetchNotes();
        } catch (error) {
          console.error("Error deleting note:", error);
        }
      } else {
        console.error("Unable to delete note. Either the user is not authenticated or noteId is missing.");
      }
    },

    startEditingNote(noteId) {
      const note = this.notes.find(n => n.id === noteId);
      if (note) {
        this.editingNoteId = noteId;
        this.editingNoteContent = note.content;
      }
    },

    async updateNote(noteId) {
      if (this.user && noteId) {
        try {
          const noteRef = doc(db, `users/${this.user.uid}/tasks/${this.taskId}/notes/${noteId}`);
          await updateDoc(noteRef, { content: this.editingNoteContent });
          this.editingNoteId = null;
          this.editingNoteContent = '';
          this.fetchNotes();
        } catch (error) {
          console.error("Error updating note:", error);
        }
      } else {
        console.error("Unable to update note. Either the user is not authenticated or note is missing.");
      }
    },

    async markAsDone(note) {
      if (this.user && note.id) {
        try {
          const noteRef = doc(db, `users/${this.user.uid}/tasks/${this.taskId}/notes/${note.id}`);
          await updateDoc(noteRef, { done: !note.done });
          this.fetchNotes();
        } catch (error) {
          console.error("Error marking note as done:", error);
        }
      } else {
        console.error("Unable to mark note as done. Either the user is not authenticated or noteId is missing.");
      }
    },

    cancelEdit() {
      this.editingNoteId = null;
      this.editingNoteContent = '';
    },

    toggleNoteForm() {
      this.showNoteForm = !this.showNoteForm;
    }
  }
};
</script>

<style scoped>
/* Title and Button Styles*/
.title_and_button {
  margin: 1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

#title {
  font-family: 'Lucida Sans', 'Lucida Sans Regular', 'Lucida Grande', 'Lucida Sans Unicode', Geneva, Verdana, sans-serif;
  font-size: 1.45rem;
  font-weight: bold;
  color: #333;
  display: inline-block;
  max-width: 90%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.add-note-btn {
  background-color: #56ab2f;
  color: white;
  font-size: 0.9rem;
  font-weight: bold;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.2s ease;
}

.add-note-btn:hover {
  background-color: #3b791c;
  transform: scale(1.05);
}

/* Add Note Form Styles */
.note-form {
  margin-top: 1rem;
  padding: 1rem;
  border-radius: 8px;
  background: #fff;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.note-form input {
  width: calc(100% - 1rem);
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  border-radius: 5px;
  border: 1px solid #ccc;
  box-sizing: border-box;
}

.note-form button {
  background-color: #3498db;
  color: white;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.2s ease;
}

.note-form button:hover {
  background-color: #2980b9;
  transform: scale(1.05);
}

.error {
  color: #e74c3c;
  font-size: 0.9rem;
  margin-top: 0.5rem;
}

/* Notes List Styles */
.notes-list {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

.note-item {
  display: flex;
  flex-direction: column;
  padding: 0.75rem;
  margin-bottom: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 5px;
  background: #fff;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.note-display {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.note-text {
  display: inline-block;
  max-width: 90%;
  overflow: hidden;
  flex: 1;
  font-size: 1rem;
  color: #333;
  word-wrap: break-word;
}

.note-actions {
  display: flex;
  gap: 0.5rem;
}

.note-actions button {
  font-size: 1rem;
  background: none;
  border: none;
  cursor: pointer;
  transition: color 0.2s ease;
}

.note-actions button:hover i {
  color: #3498db;
}

.done {
  text-decoration: line-through;
  color: #7f8c8d;
}

/* Edit Note Form Styles */
.edit-note-form {
  display: flex;
  flex-direction: column;
  margin-top: 0.5rem;
}

.edit-note-form input {
  width: calc(100% - 1rem);
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  border-radius: 5px;
  border: 1px solid #ccc;
  box-sizing: border-box;
}

.edit-note-buttons {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
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