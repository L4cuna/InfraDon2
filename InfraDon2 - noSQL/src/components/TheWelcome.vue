<script setup>
import { onMounted, ref, computed } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);

// État réactif
const state = ref({
  localDB: null,
  remoteDB: null,
  countries: [],
  messages: [],
  comments: [],
  newMessage: {
    title: '',
    content: '',
    countryId: '',
    likes: 0
  },
  newComment: {
    messageId: '',
    content: '',
    author: ''
  },
  editingMessageId: null,
  editingCommentId: null,
  searchTerm: '',
  sortBy: 'date',
  showAllComments: false,
  selectedMessageId: null,
  isOnline: true,
  syncHandler: null,
  isLoading: true
});

// Méthodes
const methods = {
  initDatabases() {
    state.value.localDB = new PouchDB('local_countries_db');
    state.value.remoteDB = new PouchDB('http://admin:admin@localhost:5984/firstdbinfradon2');
    console.log('Bases de données créées/connectées');
  },

  syncDatabases() {
    if (!state.value.isOnline || !state.value.localDB || !state.value.remoteDB) return;

    state.value.syncHandler = state.value.localDB.sync(state.value.remoteDB, {
      live: true,
      retry: true,
    })
    .on('change', methods.fetchAllData)
    .on('denied', methods.handleSyncError)
    .on('error', methods.handleSyncError);
  },

  toggleOnlineOffline() {
    state.value.isOnline = !state.value.isOnline;
    if (state.value.isOnline) {
      console.log('Mode ONLINE - Synchronisation activée');
      methods.syncDatabases();
    } else {
      console.log('Mode OFFLINE - Synchronisation désactivée');
      if (state.value.syncHandler) {
        state.value.syncHandler.cancel();
      }
    }
  },

  async fetchAllData() {
    try {
      state.value.isLoading = true;
      await methods.fetchCountries();
      await methods.fetchMessages();
      await methods.fetchComments();
    } finally {
      state.value.isLoading = false;
    }
  },

  async fetchCountries() {
    if (!state.value.localDB) return;
    try {
      // Utilisation de allDocs pour les pays (pas besoin d'index spécifique)
      const result = await state.value.localDB.allDocs({
        include_docs: true,
        startkey: 'country_',
        endkey: 'country_\uffff'
      });
      state.value.countries = result.rows.map(row => row.doc).filter(doc => doc);
    } catch (error) {
      console.error('Erreur lors de la récupération des pays:', error);
      state.value.countries = [];
    }
  },

  async fetchMessages() {
    if (!state.value.localDB) return;
    try {
      // Utilisation de allDocs pour les messages (tri simple)
      const result = await state.value.localDB.allDocs({
        include_docs: true,
        startkey: 'message_',
        endkey: 'message_\uffff'
      });
      state.value.messages = result.rows.map(row => row.doc).filter(doc => doc);
    } catch (error) {
      console.error('Erreur lors de la récupération des messages:', error);
      state.value.messages = [];
    }
  },

  async fetchComments() {
    if (!state.value.localDB) return;
    try {
      // Utilisation de allDocs pour les commentaires (tri simple)
      const result = await state.value.localDB.allDocs({
        include_docs: true,
        startkey: 'comment_',
        endkey: 'comment_\uffff'
      });
      state.value.comments = result.rows.map(row => row.doc).filter(doc => doc);
    } catch (error) {
      console.error('Erreur lors de la récupération des commentaires:', error);
      state.value.comments = [];
    }
  },

  async createIndex() {
    if (!state.value.localDB) return;
    try {
      // Index pour les recherches par région
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'region'] }
      });

      // Index pour les recherches par likes
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'likes'] }
      });

      // Index pour les recherches par messageId
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'messageId'] }
      });

      console.log('Index créés');
    } catch (error) {
      console.error('Erreur lors de la création des index:', error);
    }
  },

  async createMessage(countryId) {
    if (!state.value.localDB) return;

    try {
      const doc = {
        _id: `message_${Date.now()}`,
        type: 'message',
        title: state.value.newMessage.title,
        content: state.value.newMessage.content,
        countryId: countryId,
        likes: 0,
        createdAt: new Date().toISOString()
      };

      await state.value.localDB.post(doc);
      methods.resetNewMessageForm();
      await methods.fetchMessages();
    } catch (error) {
      console.error('Erreur lors de la création du message:', error);
    }
  },

  async updateMessage() {
    if (!state.value.localDB || !state.value.editingMessageId) return;

    try {
      const doc = await methods.getDocument(state.value.editingMessageId);
      if (!doc) return;

      const updatedDoc = {
        ...doc,
        title: state.value.newMessage.title,
        content: state.value.newMessage.content
      };

      await state.value.localDB.put(updatedDoc);
      methods.resetNewMessageForm();
      state.value.editingMessageId = null;
      await methods.fetchMessages();
    } catch (error) {
      console.error('Erreur lors de la mise à jour du message:', error);
    }
  },

  async deleteMessage(id) {
    if (!state.value.localDB) return;

    try {
      const doc = await methods.getDocument(id);
      if (!doc) return;

      const associatedComments = state.value.comments.filter(c => c.messageId === id);
      for (const comment of associatedComments) {
        await methods.deleteDocument(comment._id);
      }

      await state.value.localDB.remove(doc);
      await methods.fetchMessages();
      await methods.fetchComments();
    } catch (error) {
      console.error('Erreur lors de la suppression du message:', error);
    }
  },

  async createComment(messageId) {
    if (!state.value.localDB) return;

    try {
      const doc = {
        _id: `comment_${Date.now()}`,
        type: 'comment',
        messageId: messageId,
        content: state.value.newComment.content,
        author: state.value.newComment.author,
        createdAt: new Date().toISOString()
      };

      await state.value.localDB.post(doc);
      methods.resetNewCommentForm();
      await methods.fetchComments();
    } catch (error) {
      console.error('Erreur lors de la création du commentaire:', error);
    }
  },

  async updateComment() {
    if (!state.value.localDB || !state.value.editingCommentId) return;

    try {
      const doc = await methods.getDocument(state.value.editingCommentId);
      if (!doc) return;

      const updatedDoc = {
        ...doc,
        content: state.value.newComment.content,
        author: state.value.newComment.author
      };

      await state.value.localDB.put(updatedDoc);
      methods.resetNewCommentForm();
      state.value.editingCommentId = null;
      await methods.fetchComments();
    } catch (error) {
      console.error('Erreur lors de la mise à jour du commentaire:', error);
    }
  },

  async deleteComment(id) {
    if (!state.value.localDB) return;

    try {
      const doc = await methods.getDocument(id);
      if (!doc) return;

      await state.value.localDB.remove(doc);
      await methods.fetchComments();
    } catch (error) {
      console.error('Erreur lors de la suppression du commentaire:', error);
    }
  },

  async likeMessage(id) {
    if (!state.value.localDB) return;

    try {
      const doc = await methods.getDocument(id);
      if (!doc) return;

      const updatedDoc = {
        ...doc,
        likes: doc.likes + 1
      };

      await state.value.localDB.put(updatedDoc);
      await methods.fetchMessages();
    } catch (error) {
      console.error('Erreur lors du like:', error);
    }
  },

  async searchMessages() {
    if (!state.value.localDB || !state.value.searchTerm) return;

    try {
      // Utilisation de find avec index pour la recherche
      const result = await state.value.localDB.find({
        selector: {
          type: 'message',
          $or: [
            { title: { $regex: new RegExp(state.value.searchTerm, 'i') } },
            { content: { $regex: new RegExp(state.value.searchTerm, 'i') } }
          ]
        },
        sort: [{ createdAt: 'desc' }]
      });
      state.value.messages = result.docs;
    } catch (error) {
      console.error('Erreur lors de la recherche:', error);
    }
  },

  async sortMessages() {
    if (!state.value.localDB) return;

    try {
      // Utilisation de find avec index pour le tri
      const result = await state.value.localDB.find({
        selector: {
          type: 'message'
        },
        sort: state.value.sortBy === 'likes' ?
          [{ likes: 'desc' }] :
          [{ createdAt: 'desc' }]
      });
      state.value.messages = result.docs;
    } catch (error) {
      console.error('Erreur lors du tri:', error);
    }
  },

  async getTopLikedMessages() {
    if (!state.value.localDB) return [];

    try {
      // Utilisation de find avec index pour les messages les plus likés
      const result = await state.value.localDB.find({
        selector: {
          type: 'message'
        },
        sort: [{ likes: 'desc' }],
        limit: 10
      });
      return result.docs || [];
    } catch (error) {
      console.error('Erreur lors de la récupération des messages les plus likés:', error);
      return [];
    }
  },

  async getCountriesByRegion(region) {
    if (!state.value.localDB || !region) return [];

    try {
      // Utilisation de find avec index pour filtrer par région
      const result = await state.value.localDB.find({
        selector: {
          type: 'country',
          region: region
        },
        sort: [{ name: 'asc' }]
      });
      return result.docs;
    } catch (error) {
      console.error('Erreur lors de la récupération des pays par région:', error);
      return [];
    }
  },

  async getCommentsByMessageId(messageId) {
    if (!state.value.localDB || !messageId) return [];

    try {
      // Utilisation de find avec index pour filtrer par messageId
      const result = await state.value.localDB.find({
        selector: {
          type: 'comment',
          messageId: messageId
        },
        sort: [{ createdAt: 'asc' }]
      });
      return result.docs;
    } catch (error) {
      console.error('Erreur lors de la récupération des commentaires par message:', error);
      return [];
    }
  },

  async getCountryById(id) {
    if (!state.value.localDB || !id) return null;
    try {
      const doc = await state.value.localDB.get(id);
      return doc.type === 'country' ? doc : null;
    } catch (error) {
      console.error('Erreur lors de la récupération du pays:', error);
      return null;
    }
  },

  async getMessageById(id) {
    if (!state.value.localDB || !id) return null;
    try {
      const doc = await state.value.localDB.get(id);
      return doc.type === 'message' ? doc : null;
    } catch (error) {
      console.error('Erreur lors de la récupération du message:', error);
      return null;
    }
  },

  startEditingMessage(message) {
    if (!message) return;
    state.value.newMessage = {
      title: message.title || '',
      content: message.content || '',
      countryId: message.countryId || '',
      likes: message.likes || 0
    };
    state.value.editingMessageId = message._id || null;
  },

  cancelEditingMessage() {
    methods.resetNewMessageForm();
    state.value.editingMessageId = null;
  },

  startEditingComment(comment) {
    if (!comment) return;
    state.value.newComment = {
      messageId: comment.messageId || '',
      content: comment.content || '',
      author: comment.author || ''
    };
    state.value.editingCommentId = comment._id || null;
  },

  cancelEditingComment() {
    methods.resetNewCommentForm();
    state.value.editingCommentId = null;
  },

  resetNewMessageForm() {
    state.value.newMessage = {
      title: '',
      content: '',
      countryId: '',
      likes: 0
    };
  },

  resetNewCommentForm() {
    state.value.newComment = {
      messageId: '',
      content: '',
      author: ''
    };
  },

  async getDocument(id) {
    if (!state.value.localDB || !id) return null;
    try {
      return await state.value.localDB.get(id);
    } catch (error) {
      console.error('Erreur lors de la récupération du document:', error);
      return null;
    }
  },

  async deleteDocument(id) {
    if (!state.value.localDB || !id) return;
    try {
      const doc = await methods.getDocument(id);
      if (!doc) return;
      await state.value.localDB.remove(doc);
    } catch (error) {
      console.error('Erreur lors de la suppression du document:', error);
    }
  },

  handleSyncError(error) {
    console.error('Erreur de synchronisation:', error);
  }
};

// Données calculées
const computedData = {
  topLikedMessages: computed(async () => {
    return await methods.getTopLikedMessages();
  }),

  messagesWithComments: computed(() => {
    return state.value.messages.map(message => {
      if (!message || !message._id) return null;
      return {
        ...message,
        country: methods.getCountryById(message.countryId)
      };
    }).filter(msg => msg !== null);
  }),

  filteredComments: computed(() => {
    if (!state.value.selectedMessageId) return [];
    return state.value.comments.filter(c => c && c.messageId === state.value.selectedMessageId);
  })
};

// Cycle de vie
onMounted(() => {
  methods.initDatabases();
  if (state.value.isOnline) methods.syncDatabases();
  methods.createIndex();
  methods.fetchAllData();
});
</script>

<template>
  <div class="app-container">
    <h1>Gestion des Pays et Messages</h1>

    <!-- Message de chargement -->
    <div v-if="state.isLoading" class="loading-message">
      Chargement des données...
    </div>

    <div v-else>
      <!-- Toggle Online/Offline -->
      <section class="toggle-section">
        <button @click="methods.toggleOnlineOffline" class="toggle-button">
          {{ state.isOnline ? 'Passer en mode OFFLINE' : 'Passer en mode ONLINE' }}
        </button>
        <span class="status">Statut : {{ state.isOnline ? 'ONLINE' : 'OFFLINE' }}</span>
      </section>

      <!-- Recherche et tri des messages -->
      <section class="message-controls">
        <div class="search-container">
          <input
            v-model="state.searchTerm"
            placeholder="Rechercher un message..."
            @keyup.enter="methods.searchMessages"
            class="search-input"
          />
          <button @click="methods.searchMessages" class="search-button">Rechercher</button>
          <button @click="methods.fetchMessages" class="reset-button">Réinitialiser</button>
        </div>

        <div class="sort-container">
          <label>Trier par :</label>
          <select v-model="state.sortBy" @change="methods.sortMessages" class="sort-select">
            <option value="date">Date (le plus récent)</option>
            <option value="likes">Nombre de likes</option>
          </select>
        </div>
      </section>

      <!-- Liste des pays -->
      <section class="countries-section">
        <h2>Pays</h2>
        <div v-if="state.countries.length === 0" class="no-data">
          Aucun pays trouvé.
        </div>
        <div v-else class="country-grid">
          <div v-for="country in state.countries" :key="country._id" class="country-card">
            <div v-if="country && country._id" class="country-header">
              <h3>{{ country.name }}</h3>
              <img :src="country.flag" alt="flag" class="country-flag" />
            </div>

            <div class="country-details">
              <p><strong>Capitale:</strong> {{ country.capital }}</p>
              <p><strong>Population:</strong> {{ country.population?.toLocaleString() }}</p>
              <p><strong>Région:</strong> {{ country.region }}</p>
            </div>

            <!-- Formulaire pour ajouter un message -->
            <div class="message-form">
              <h4>Ajouter un message</h4>
              <form @submit.prevent="methods.createMessage(country._id)" class="form-container">
                <input
                  v-model="state.newMessage.title"
                  placeholder="Titre"
                  required
                  class="form-input"
                />
                <textarea
                  v-model="state.newMessage.content"
                  placeholder="Contenu"
                  required
                  class="form-input"
                ></textarea>
                <button type="submit" class="submit-button">Ajouter</button>
              </form>
            </div>

            <!-- Messages associés au pays -->
            <div class="messages-list">
              <h4>Messages</h4>
              <div v-if="state.messages.filter(m => m.countryId === country._id).length === 0">
                Aucun message pour ce pays.
              </div>
              <div
                v-for="message in state.messages.filter(m => m.countryId === country._id)"
                :key="message._id"
                class="message-card"
              >
                <div v-if="message && message._id" class="message-header">
                  <h5>{{ message.title }}</h5>
                  <div class="message-actions">
                    <button @click="methods.likeMessage(message._id)" class="like-button">
                      👍 {{ message.likes }}
                    </button>
                    <button
                      @click="state.selectedMessageId = message._id"
                      class="show-comments-button"
                    >
                      {{ state.selectedMessageId === message._id ? 'Masquer' : 'Voir' }} commentaires
                    </button>
                  </div>
                </div>

                <p>{{ message.content }}</p>

                <!-- Tous les commentaires si sélectionnés -->
                <div v-if="state.selectedMessageId === message._id" class="comments-list">
                  <h6>Commentaires</h6>
                  <div v-if="state.comments.filter(c => c.messageId === message._id).length === 0">
                    Aucun commentaire pour ce message.
                  </div>
                  <div v-else>
                    <div v-for="comment in state.comments.filter(c => c.messageId === message._id)" :key="comment._id" class="comment">
                      <div v-if="comment && comment._id">
                        <p>{{ comment.content }}</p>
                        <p><em>Par {{ comment.author }}</em></p>
                        <div class="comment-actions">
                          <button @click="methods.startEditingComment(comment)" class="edit-button">Modifier</button>
                          <button @click="methods.deleteComment(comment._id)" class="delete-button">Supprimer</button>
                        </div>

                        <!-- Formulaire de modification de commentaire -->
                        <div v-if="state.editingCommentId === comment._id" class="edit-comment-form">
                          <form @submit.prevent="methods.updateComment" class="form-container">
                            <textarea v-model="state.newComment.content" required class="form-input"></textarea>
                            <input v-model="state.newComment.author" placeholder="Auteur" required class="form-input" />
                            <button type="submit" class="submit-button">Enregistrer</button>
                            <button type="button" @click="methods.cancelEditingComment" class="cancel-button">Annuler</button>
                          </form>
                        </div>
                      </div>
                    </div>

                    <!-- Formulaire pour ajouter un commentaire -->
                    <form @submit.prevent="methods.createComment(message._id)" class="form-container">
                      <textarea
                        v-model="state.newComment.content"
                        placeholder="Ajouter un commentaire"
                        required
                        class="form-input"
                      ></textarea>
                      <input
                        v-model="state.newComment.author"
                        placeholder="Votre nom"
                        required
                        class="form-input"
                      />
                      <button type="submit" class="submit-button">Commenter</button>
                    </form>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- Messages les plus likés -->
      <section class="top-messages-section">
        <h2>Top 10 Messages les plus likés</h2>
        <div v-if="computedData.topLikedMessages.length === 0" class="no-data">
          Aucun message populaire trouvé.
        </div>
        <div v-else class="message-grid">
          <div v-for="message in computedData.topLikedMessages" :key="message._id" class="message-card popular-message">
            <div v-if="message && message._id">
              <h3>{{ message.title }}</h3>
              <p>{{ truncateText(message.content, 100) }}</p>
              <div class="message-meta">
                <span class="likes-count">❤️ {{ message.likes }}</span>
                <span class="country-name" v-if="message.country">
                  Pays: {{ message.country.name }}
                </span>
              </div>
            </div>
          </div>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  max-width: 1400px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Arial', sans-serif;
}

h1, h2, h3, h4, h5, h6 {
  color: #2c3e50;
}

.toggle-section {
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.toggle-button {
  padding: 8px 16px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.toggle-button:hover {
  background-color: #2980b9;
}

.status {
  font-weight: bold;
}

.message-controls {
  margin-bottom: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 20px;
}

.search-container {
  display: flex;
  gap: 10px;
}

.search-input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.search-button, .reset-button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.search-button {
  background-color: #2ecc71;
  color: white;
}

.search-button:hover {
  background-color: #27ae60;
}

.reset-button {
  background-color: #e74c3c;
  color: white;
}

.reset-button:hover {
  background-color: #c0392b;
}

.sort-container {
  display: flex;
  align-items: center;
  gap: 10px;
}

.sort-select {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.countries-section {
  margin-bottom: 40px;
}

.country-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(400px, 1fr));
  gap: 20px;
}

.country-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  background-color: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.country-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.country-flag {
  width: 60px;
  height: auto;
  border: 1px solid #eee;
}

.message-form {
  margin: 15px 0;
  padding: 10px;
  background-color: #f9f9f9;
  border-radius: 4px;
}

.message-form h4 {
  margin-bottom: 10px;
}

.messages-list {
  margin-top: 15px;
}

.message-card {
  border: 1px solid #eee;
  border-radius: 4px;
  padding: 10px;
  margin-bottom: 10px;
}

.message-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 5px;
}

.message-actions {
  display: flex;
  gap: 10px;
}

.like-button {
  background: none;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 5px;
}

.first-comment {
  margin-top: 10px;
  padding: 10px;
  background-color: #f5f5f5;
  border-radius: 4px;
  font-size: 0.9em;
}

.comments-list {
  margin-top: 15px;
  border-left: 2px solid #ddd;
  padding-left: 15px;
}

.comment {
  margin-bottom: 10px;
  padding: 10px;
  background-color: #f9f9f9;
  border-radius: 4px;
}

.comment-actions {
  display: flex;
  gap: 10px;
  margin-top: 5px;
}

.edit-comment-form {
  margin-top: 10px;
  padding: 10px;
  background-color: white;
  border-radius: 4px;
  border: 1px solid #ddd;
}

.top-messages-section {
  margin-top: 40px;
}

.message-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.popular-message {
  background-color: #f0f8ff;
  border-left: 4px solid #3498db;
}

.message-meta {
  display: flex;
  justify-content: space-between;
  margin-top: 10px;
  font-size: 0.9em;
  color: #666;
}

.likes-count {
  color: #e74c3c;
  font-weight: bold;
}

.loading-message {
  padding: 20px;
  text-align: center;
  font-size: 1.2em;
  color: #666;
}

.no-data {
  padding: 20px;
  text-align: center;
  color: #999;
  font-style: italic;
}

@media (max-width: 768px) {
  .country-grid {
    grid-template-columns: 1fr;
  }

  .message-controls {
    flex-direction: column;
    align-items: stretch;
  }

  .sort-container {
    margin-top: 10px;
  }
}
</style>
