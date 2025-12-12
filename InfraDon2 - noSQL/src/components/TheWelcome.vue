<script setup>
import { onMounted, ref, computed, watch } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);

// Configuration des bases de données
const DB_CONFIG = {
  local: 'local_app_db',
  remote: 'http://admin:admin@localhost:5984/firstdbinfradon2',
  docTypes: {
    country: 'country',
    message: 'message',
    comment: 'comment'
  }
};

// État réactif
const state = ref({
  localDB: null,
  remoteDB: null,
  syncHandler: null,
  isOnline: true,
  isLoading: false,
  
  countries: [],
  messages: [],
  comments: [],
  allCommentsForMessage: {},
  topLikedMessages: [],  // Stockage des top 10 messages
  
  selectedCountryId: null,
  expandedMessageId: null,
  
  newCountry: {
    name: '',
    capital: '',
    population: 0,
    area: 0,
    currency: '',
    languages: '',
    region: '',
    subregion: '',
    flag: 'https://flagcdn.com/160x120/fr.png'
  },
  newMessage: {
    title: '',
    content: ''
  },
  
  // Formulaires de commentaires séparés par message
  commentForms: {},
  
  // Édition
  editingCountryId: null,
  editingMessageId: null,
  editingCommentId: null,
  editingCommentData: { content: '', author: '' },
  
  searchRegion: '',
  searchMessageTerm: '',
  sortBy: 'date',
  
  userLikedMessages: new Set()
});

// Initialiser le formulaire de commentaire pour un message
const initCommentForm = (messageId) => {
  if (!state.value.commentForms[messageId]) {
    state.value.commentForms[messageId] = {
      content: '',
      author: ''
    };
  }
};

const methods = {
  async initDatabases() {
    try {
      state.value.isLoading = true;
      state.value.localDB = new PouchDB(DB_CONFIG.local);
      state.value.remoteDB = new PouchDB(DB_CONFIG.remote);
      console.log('✅ Bases de données initialisées');
      
      await methods.createIndexes();
      await methods.replicateFromRemote();
      await methods.fetchAllData();
      
      if (state.value.isOnline) {
        methods.startContinuousSync();
      }
    } catch (error) {
      console.error('❌ Erreur d\'initialisation:', error);
    } finally {
      state.value.isLoading = false;
    }
  },

  async createIndexes() {
    try {
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'region'] }
      });
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'countryId', 'likes'] }
      });
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'countryId', 'createdAt'] }
      });
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'countryId', 'title'] }
      });
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'messageId', 'createdAt'] }
      });
      // Index pour le top 10 des messages les plus likés (tous pays)
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'likes'] }
      });
      console.log('✅ Index créés');
    } catch (error) {
      console.error('❌ Erreur index:', error);
    }
  },

  async replicateFromRemote() {
    try {
      console.log('🔄 Réplication...');
      await state.value.localDB.replicate.from(state.value.remoteDB);
      console.log('✅ Réplication terminée');
    } catch (error) {
      console.error('❌ Erreur réplication:', error);
    }
  },

  startContinuousSync() {
    if (!state.value.isOnline) return;
    console.log('🔄 Sync continue...');
    
    state.value.syncHandler = state.value.localDB.sync(state.value.remoteDB, {
      live: true,
      retry: true
    })
    .on('change', () => methods.fetchAllData())
    .on('error', (err) => console.error('❌ Erreur sync:', err));
  },

  stopContinuousSync() {
    if (state.value.syncHandler) {
      state.value.syncHandler.cancel();
      state.value.syncHandler = null;
    }
  },

  toggleOnlineOffline() {
    state.value.isOnline = !state.value.isOnline;
    if (state.value.isOnline) {
      methods.startContinuousSync();
    } else {
      methods.stopContinuousSync();
    }
  },

  async fetchAllData() {
    await methods.fetchCountries();
    await methods.fetchTopLikedMessages(); // Récupérer le top 10
    if (state.value.selectedCountryId) {
      await methods.fetchMessages();
      await methods.fetchComments();
    }
  },

  async fetchTopLikedMessages() {
    if (!state.value.localDB) return;
    try {
      const result = await state.value.localDB.find({
        selector: {
          type: DB_CONFIG.docTypes.message
        },
        sort: [{ type: 'asc' }, { likes: 'desc' }],
        limit: 10
      });
      state.value.topLikedMessages = result.docs || [];
    } catch (error) {
      console.error('❌ Erreur fetchTopLikedMessages:', error);
      state.value.topLikedMessages = [];
    }
  },

  async fetchCountries() {
    if (!state.value.localDB) return;
    try {
      const result = await state.value.localDB.allDocs({
        include_docs: true,
        startkey: `${DB_CONFIG.docTypes.country}_`,
        endkey: `${DB_CONFIG.docTypes.country}_\uffff`
      });
      state.value.countries = result.rows.map(row => row.doc).filter(doc => doc);
    } catch (error) {
      console.error('❌ Erreur fetchCountries:', error);
      state.value.countries = [];
    }
  },

  async fetchMessages() {
    if (!state.value.localDB || !state.value.selectedCountryId) {
      state.value.messages = [];
      return;
    }
    try {
      let selector = {
        type: DB_CONFIG.docTypes.message,
        countryId: state.value.selectedCountryId
      };
      
      let sort = [{ type: 'asc' }, { countryId: 'asc' }];
      if (state.value.sortBy === 'likes') {
        sort.push({ likes: 'desc' });
      } else {
        sort.push({ createdAt: 'desc' });
      }
      
      const result = await state.value.localDB.find({ selector, sort });
      state.value.messages = result.docs || [];
      
      // Initialiser les formulaires de commentaires pour chaque message
      state.value.messages.forEach(msg => initCommentForm(msg._id));
      
    } catch (error) {
      console.error('❌ Erreur fetchMessages:', error);
      state.value.messages = [];
    }
  },

  async fetchComments() {
    if (!state.value.localDB || !state.value.messages.length) {
      state.value.comments = [];
      return;
    }
    try {
      const comments = [];
      for (const message of state.value.messages) {
        const result = await state.value.localDB.find({
          selector: {
            type: DB_CONFIG.docTypes.comment,
            messageId: message._id
          },
          sort: [{ type: 'asc' }, { messageId: 'asc' }, { createdAt: 'asc' }],
          limit: 1
        });
        if (result.docs && result.docs.length > 0) {
          comments.push(result.docs[0]);
        }
      }
      state.value.comments = comments;
    } catch (error) {
      console.error('❌ Erreur fetchComments:', error);
      state.value.comments = [];
    }
  },

  async fetchAllCommentsForMessage(messageId) {
    if (!state.value.localDB) return;
    try {
      const result = await state.value.localDB.find({
        selector: {
          type: DB_CONFIG.docTypes.comment,
          messageId: messageId
        },
        sort: [{ type: 'asc' }, { messageId: 'asc' }, { createdAt: 'asc' }]
      });
      state.value.allCommentsForMessage[messageId] = result.docs || [];
    } catch (error) {
      console.error('❌ Erreur fetchAllComments:', error);
      state.value.allCommentsForMessage[messageId] = [];
    }
  },

  // PAYS
  async createCountry() {
    if (!state.value.localDB) return;
    try {
      const doc = {
        _id: `${DB_CONFIG.docTypes.country}_${Date.now()}`,
        type: DB_CONFIG.docTypes.country,
        ...state.value.newCountry,
        population: Number(state.value.newCountry.population),
        area: Number(state.value.newCountry.area),
        languages: state.value.newCountry.languages.split(',').map(l => l.trim()),
        createdAt: new Date().toISOString()
      };
      await state.value.localDB.put(doc);
      methods.resetCountryForm();
      await methods.fetchCountries();
      console.log('✅ Pays créé');
    } catch (error) {
      console.error('❌ Erreur création pays:', error);
    }
  },

  async updateCountry() {
    if (!state.value.localDB || !state.value.editingCountryId) return;
    try {
      const doc = await state.value.localDB.get(state.value.editingCountryId);
      Object.assign(doc, {
        ...state.value.newCountry,
        population: Number(state.value.newCountry.population),
        area: Number(state.value.newCountry.area),
        languages: state.value.newCountry.languages.split(',').map(l => l.trim()),
        updatedAt: new Date().toISOString()
      });
      await state.value.localDB.put(doc);
      methods.resetCountryForm();
      state.value.editingCountryId = null;
      await methods.fetchCountries();
      console.log('✅ Pays mis à jour');
    } catch (error) {
      console.error('❌ Erreur update pays:', error);
    }
  },

  async deleteCountry(id) {
    if (!state.value.localDB) return;
    try {
      const doc = await state.value.localDB.get(id);
      await state.value.localDB.remove(doc);
      if (state.value.selectedCountryId === id) {
        state.value.selectedCountryId = null;
        state.value.messages = [];
        state.value.comments = [];
      }
      await methods.fetchCountries();
      console.log('✅ Pays supprimé');
    } catch (error) {
      console.error('❌ Erreur delete pays:', error);
    }
  },

  startEditingCountry(country) {
    state.value.editingCountryId = country._id;
    state.value.newCountry = {
      name: country.name,
      capital: country.capital,
      population: country.population,
      area: country.area,
      currency: country.currency,
      languages: Array.isArray(country.languages) ? country.languages.join(', ') : country.languages,
      region: country.region,
      subregion: country.subregion,
      flag: country.flag
    };
  },

  cancelEditingCountry() {
    methods.resetCountryForm();
    state.value.editingCountryId = null;
  },

  resetCountryForm() {
    state.value.newCountry = {
      name: '',
      capital: '',
      population: 0,
      area: 0,
      currency: '',
      languages: '',
      region: '',
      subregion: '',
      flag: 'https://flagcdn.com/160x120/fr.png'
    };
  },

  // MESSAGES
  selectCountry(countryId) {
    state.value.selectedCountryId = countryId;
    state.value.searchMessageTerm = '';
    state.value.sortBy = 'date';
    methods.fetchMessages();
    methods.fetchComments();
  },

  async createMessage() {
    if (!state.value.localDB || !state.value.selectedCountryId) return;
    try {
      const doc = {
        _id: `${DB_CONFIG.docTypes.message}_${Date.now()}`,
        type: DB_CONFIG.docTypes.message,
        countryId: state.value.selectedCountryId,
        title: state.value.newMessage.title,
        content: state.value.newMessage.content,
        likes: 0,
        createdAt: new Date().toISOString()
      };
      await state.value.localDB.put(doc);
      methods.resetMessageForm();
      await methods.fetchMessages();
      await methods.fetchTopLikedMessages(); // Rafraîchir le top 10
      console.log('✅ Message créé');
    } catch (error) {
      console.error('❌ Erreur création message:', error);
    }
  },

  async updateMessage() {
    if (!state.value.localDB || !state.value.editingMessageId) return;
    try {
      const doc = await state.value.localDB.get(state.value.editingMessageId);
      doc.title = state.value.newMessage.title;
      doc.content = state.value.newMessage.content;
      doc.updatedAt = new Date().toISOString();
      await state.value.localDB.put(doc);
      methods.resetMessageForm();
      state.value.editingMessageId = null;
      await methods.fetchMessages();
      console.log('✅ Message mis à jour');
    } catch (error) {
      console.error('❌ Erreur update message:', error);
    }
  },

  async deleteMessage(id) {
    if (!state.value.localDB) return;
    try {
      const doc = await state.value.localDB.get(id);
      await state.value.localDB.remove(doc);
      
      const commentsResult = await state.value.localDB.find({
        selector: { 
          type: DB_CONFIG.docTypes.comment,
          messageId: id 
        }
      });
      for (const comment of commentsResult.docs) {
        await state.value.localDB.remove(comment);
      }
      
      await methods.fetchMessages();
      await methods.fetchComments();
      await methods.fetchTopLikedMessages(); // Rafraîchir le top 10
      console.log('✅ Message supprimé');
    } catch (error) {
      console.error('❌ Erreur delete message:', error);
    }
  },

  async toggleLike(message) {
    if (!state.value.localDB) return;
    try {
      const doc = await state.value.localDB.get(message._id);
      if (state.value.userLikedMessages.has(message._id)) {
        doc.likes = Math.max(0, (doc.likes || 0) - 1);
        state.value.userLikedMessages.delete(message._id);
      } else {
        doc.likes = (doc.likes || 0) + 1;
        state.value.userLikedMessages.add(message._id);
      }
      await state.value.localDB.put(doc);
      await methods.fetchMessages();
      await methods.fetchTopLikedMessages(); // Rafraîchir le top 10
      console.log('✅ Like mis à jour');
    } catch (error) {
      console.error('❌ Erreur like:', error);
    }
  },

  startEditingMessage(message) {
    state.value.editingMessageId = message._id;
    state.value.newMessage = {
      title: message.title,
      content: message.content
    };
  },

  cancelEditingMessage() {
    methods.resetMessageForm();
    state.value.editingMessageId = null;
  },

  resetMessageForm() {
    state.value.newMessage = { title: '', content: '' };
  },

  // COMMENTAIRES
  async createComment(messageId) {
    if (!state.value.localDB) return;
    
    initCommentForm(messageId);
    const form = state.value.commentForms[messageId];
    
    if (!form.content?.trim() || !form.author?.trim()) {
      alert('Veuillez remplir le commentaire et votre nom');
      return;
    }
    
    try {
      const doc = {
        _id: `${DB_CONFIG.docTypes.comment}_${Date.now()}_${Math.random()}`,
        type: DB_CONFIG.docTypes.comment,
        messageId: messageId,
        content: form.content.trim(),
        author: form.author.trim(),
        createdAt: new Date().toISOString()
      };
      
      await state.value.localDB.put(doc);
      
      // Réinitialiser le formulaire
      state.value.commentForms[messageId] = { content: '', author: '' };
      
      await methods.fetchComments();
      
      if (state.value.expandedMessageId === messageId) {
        await methods.fetchAllCommentsForMessage(messageId);
      }
      
      console.log('✅ Commentaire créé');
    } catch (error) {
      console.error('❌ Erreur création commentaire:', error);
      alert('Erreur lors de la création du commentaire');
    }
  },

  async updateComment() {
    if (!state.value.localDB || !state.value.editingCommentId) return;
    
    const editData = state.value.editingCommentData;
    
    if (!editData.content?.trim() || !editData.author?.trim()) {
      alert('Veuillez remplir le commentaire et votre nom');
      return;
    }
    
    try {
      const doc = await state.value.localDB.get(state.value.editingCommentId);
      doc.content = editData.content.trim();
      doc.author = editData.author.trim();
      doc.updatedAt = new Date().toISOString();
      
      await state.value.localDB.put(doc);
      
      state.value.editingCommentId = null;
      state.value.editingCommentData = { content: '', author: '' };
      
      await methods.fetchComments();
      
      if (state.value.expandedMessageId === doc.messageId) {
        await methods.fetchAllCommentsForMessage(doc.messageId);
      }
      
      console.log('✅ Commentaire mis à jour');
    } catch (error) {
      console.error('❌ Erreur update commentaire:', error);
    }
  },

  async deleteComment(comment) {
    if (!state.value.localDB) return;
    
    if (!confirm('Supprimer ce commentaire ?')) return;
    
    try {
      const doc = await state.value.localDB.get(comment._id);
      await state.value.localDB.remove(doc);
      
      await methods.fetchComments();
      
      if (state.value.expandedMessageId === comment.messageId) {
        await methods.fetchAllCommentsForMessage(comment.messageId);
      }
      
      console.log('✅ Commentaire supprimé');
    } catch (error) {
      console.error('❌ Erreur delete commentaire:', error);
    }
  },

  startEditingComment(comment) {
    state.value.editingCommentId = comment._id;
    state.value.editingCommentData = {
      content: comment.content,
      author: comment.author
    };
  },

  cancelEditingComment() {
    state.value.editingCommentId = null;
    state.value.editingCommentData = { content: '', author: '' };
  },

  async toggleCommentsVisibility(messageId) {
    if (state.value.expandedMessageId === messageId) {
      state.value.expandedMessageId = null;
    } else {
      state.value.expandedMessageId = messageId;
      await methods.fetchAllCommentsForMessage(messageId);
    }
  },

  async searchByRegion() {
    if (!state.value.localDB || !state.value.searchRegion.trim()) {
      await methods.fetchCountries();
      return;
    }
    try {
      const result = await state.value.localDB.find({
        selector: { 
          type: DB_CONFIG.docTypes.country,
          region: { $eq: state.value.searchRegion } 
        }
      });
      state.value.countries = result.docs;
      console.log('✅ Recherche effectuée');
    } catch (error) {
      console.error('❌ Erreur recherche:', error);
    }
  },

  async generateTestData() {
    try {
      console.log('🔧 Génération...');
      
      const testCountries = [
        { name: 'France', capital: 'Paris', population: 67000000, area: 551695, currency: 'EUR', languages: ['Français'], region: 'Europe', subregion: 'Western Europe', flag: 'https://flagcdn.com/160x120/fr.png' },
        { name: 'Germany', capital: 'Berlin', population: 83000000, area: 357022, currency: 'EUR', languages: ['Deutsch'], region: 'Europe', subregion: 'Western Europe', flag: 'https://flagcdn.com/160x120/de.png' },
        { name: 'Italy', capital: 'Rome', population: 60000000, area: 301340, currency: 'EUR', languages: ['Italiano'], region: 'Europe', subregion: 'Southern Europe', flag: 'https://flagcdn.com/160x120/it.png' }
      ];
      
      for (const countryData of testCountries) {
        const countryDoc = {
          _id: `${DB_CONFIG.docTypes.country}_${Date.now()}_${Math.random()}`,
          type: DB_CONFIG.docTypes.country,
          ...countryData,
          createdAt: new Date().toISOString()
        };
        await state.value.localDB.put(countryDoc);
        
        for (let i = 0; i < 10; i++) {
          const messageDoc = {
            _id: `${DB_CONFIG.docTypes.message}_${Date.now()}_${Math.random()}`,
            type: DB_CONFIG.docTypes.message,
            countryId: countryDoc._id,
            title: `Message ${i + 1} about ${countryData.name}`,
            content: `Test message ${i + 1} about ${countryData.name}.`,
            likes: Math.floor(Math.random() * 50),
            createdAt: new Date(Date.now() - Math.random() * 10000000000).toISOString()
          };
          await state.value.localDB.put(messageDoc);
          
          for (let j = 0; j < 3; j++) {
            const commentDoc = {
              _id: `${DB_CONFIG.docTypes.comment}_${Date.now()}_${Math.random()}`,
              type: DB_CONFIG.docTypes.comment,
              messageId: messageDoc._id,
              content: `Comment ${j + 1} on this message.`,
              author: `User ${j + 1}`,
              createdAt: new Date(Date.now() - Math.random() * 10000000000).toISOString()
            };
            await state.value.localDB.put(commentDoc);
          }
        }
      }
      
      await methods.fetchAllData();
      console.log('✅ Données générées');
    } catch (error) {
      console.error('❌ Erreur génération:', error);
    }
  }
};

const computedData = computed(() => {
  const firstCommentByMessage = {};
  state.value.comments.forEach(comment => {
    if (!firstCommentByMessage[comment.messageId]) {
      firstCommentByMessage[comment.messageId] = comment;
    }
  });

  // Utiliser directement les données récupérées depuis la base de données
  return { 
    topLikedMessages: state.value.topLikedMessages,
    firstCommentByMessage 
  };
});

watch(() => state.value.searchMessageTerm, () => {
  if (state.value.selectedCountryId) methods.fetchMessages();
});

watch(() => state.value.sortBy, () => {
  if (state.value.selectedCountryId) methods.fetchMessages();
});

onMounted(() => {
  methods.initDatabases();
});
</script>

<template>
  <div class="app-container">
    <div class="app-header">
      <h1>Infrastructure de données 2 - noSQL</h1>
      <div class="header-actions">
        <button @click="methods.toggleOnlineOffline" class="btn-toggle">
          {{ state.isOnline ? '🌐 ONLINE' : '📵 OFFLINE' }}
        </button>
        <span class="status">{{ state.isOnline ? 'Connecté' : 'Déconnecté' }}</span>
        <button @click="methods.generateTestData" class="btn secondary small">
          🔧 Générer données test
        </button>
      </div>
    </div>

    <div v-if="state.isLoading" class="loading">Chargement...</div>

    <div v-else class="main-content">
      <!-- PAYS -->
      <section class="section">
        <h2>Gestion des Pays</h2>

        <div class="search-bar">
          <input
            v-model="state.searchRegion"
            placeholder="Rechercher par région"
            @keyup.enter="methods.searchByRegion"
            class="input"
          />
          <button @click="methods.searchByRegion" class="btn primary">Rechercher</button>
          <button @click="methods.fetchCountries" class="btn secondary">Réinitialiser</button>
        </div>

        <div class="form-container">
          <h3>{{ state.editingCountryId ? 'Modifier un pays' : 'Ajouter un pays' }}</h3>
          <form @submit.prevent="state.editingCountryId ? methods.updateCountry() : methods.createCountry()">
            <div class="form-grid">
              <div class="form-group">
                <label>Nom</label>
                <input v-model="state.newCountry.name" required class="input" />
              </div>
              <div class="form-group">
                <label>Capitale</label>
                <input v-model="state.newCountry.capital" required class="input" />
              </div>
              <div class="form-group">
                <label>Population</label>
                <input v-model.number="state.newCountry.population" type="number" required class="input" />
              </div>
              <div class="form-group">
                <label>Superficie</label>
                <input v-model.number="state.newCountry.area" type="number" required class="input" />
              </div>
              <div class="form-group">
                <label>Monnaie</label>
                <input v-model="state.newCountry.currency" required class="input" />
              </div>
              <div class="form-group">
                <label>Langues</label>
                <input v-model="state.newCountry.languages" required class="input" />
              </div>
              <div class="form-group">
                <label>Région</label>
                <input v-model="state.newCountry.region" required class="input" />
              </div>
              <div class="form-group">
                <label>Sous-région</label>
                <input v-model="state.newCountry.subregion" required class="input" />
              </div>
            </div>
            <div class="form-actions">
              <button type="submit" class="btn primary">
                {{ state.editingCountryId ? 'Modifier' : 'Ajouter' }}
              </button>
              <button v-if="state.editingCountryId" @click="methods.cancelEditingCountry" type="button" class="btn secondary">
                Annuler
              </button>
            </div>
          </form>
        </div>

        <div v-if="state.countries.length === 0" class="no-data">Aucun pays</div>
        <div v-else class="cards-grid">
          <div 
            v-for="country in state.countries" 
            :key="country._id"
            class="card"
            :class="{ selected: state.selectedCountryId === country._id }"
            @click="methods.selectCountry(country._id)"
          >
            <div class="card-header">
              <h3>{{ country.name }}</h3>
              <img :src="country.flag" :alt="country.name" class="flag" />
            </div>
            <div class="card-body">
              <p><strong>Capitale:</strong> {{ country.capital }}</p>
              <p><strong>Population:</strong> {{ country.population?.toLocaleString() }}</p>
              <p><strong>Région:</strong> {{ country.region }}</p>
            </div>
            <div class="card-actions">
              <button @click.stop="methods.startEditingCountry(country)" class="btn secondary small">Modifier</button>
              <button @click.stop="methods.deleteCountry(country._id)" class="btn danger small">Supprimer</button>
            </div>
          </div>
        </div>
      </section>

      <!-- MESSAGES -->
      <section v-if="state.selectedCountryId" class="section">
        <h2>Messages pour {{ state.countries.find(c => c._id === state.selectedCountryId)?.name }}</h2>

        <div class="toolbar">
          <input
            v-model="state.searchMessageTerm"
            placeholder="Rechercher un message..."
            class="input flex-1"
          />
          <select v-model="state.sortBy" class="select">
            <option value="date">Par date</option>
            <option value="likes">Par likes</option>
          </select>
        </div>

        <div class="form-container">
          <h3>{{ state.editingMessageId ? 'Modifier un message' : 'Nouveau message' }}</h3>
          <form @submit.prevent="state.editingMessageId ? methods.updateMessage() : methods.createMessage()">
            <div class="form-group">
              <label>Titre</label>
              <input v-model="state.newMessage.title" required class="input" />
            </div>
            <div class="form-group">
              <label>Contenu</label>
              <textarea v-model="state.newMessage.content" required class="textarea"></textarea>
            </div>
            <div class="form-actions">
              <button type="submit" class="btn primary">
                {{ state.editingMessageId ? 'Modifier' : 'Publier' }}
              </button>
              <button v-if="state.editingMessageId" @click="methods.cancelEditingMessage" type="button" class="btn secondary">
                Annuler
              </button>
            </div>
          </form>
        </div>

        <div v-if="state.messages.length === 0" class="no-data">Aucun message</div>
        <div v-else class="messages-list">
          <div v-for="message in state.messages" :key="message._id" class="message-card">
            <div class="message-header">
              <h4>{{ message.title }}</h4>
              <div class="message-actions">
                <button @click="methods.toggleLike(message)" class="btn-like">
                  {{ state.userLikedMessages.has(message._id) ? '❤️' : '🤍' }}
                  {{ message.likes || 0 }}
                </button>
                <button @click="methods.startEditingMessage(message)" class="btn secondary tiny">Modifier</button>
                <button @click="methods.deleteMessage(message._id)" class="btn danger tiny">Supprimer</button>
              </div>
            </div>
            <p class="message-content">{{ message.content }}</p>

            <!-- COMMENTAIRES -->
            <div class="comments-section">
              <h5>Commentaires</h5>
              
              <div v-if="computedData.firstCommentByMessage[message._id]" class="comment">
                <p><strong>{{ computedData.firstCommentByMessage[message._id].author }}:</strong> 
                   {{ computedData.firstCommentByMessage[message._id].content }}
                </p>
                <div class="comment-actions">
                  <span class="comment-date">{{ new Date(computedData.firstCommentByMessage[message._id].createdAt).toLocaleDateString() }}</span>
                  <button @click="methods.startEditingComment(computedData.firstCommentByMessage[message._id])" class="btn secondary tiny">Modifier</button>
                  <button @click="methods.deleteComment(computedData.firstCommentByMessage[message._id])" class="btn danger tiny">Supprimer</button>
                </div>
              </div>
              <div v-else class="no-comment">
                Aucun commentaire
              </div>

              <button @click="methods.toggleCommentsVisibility(message._id)" class="btn secondary small" style="margin: 10px 0;">
                {{ state.expandedMessageId === message._id ? 'Masquer' : 'Voir tous' }}
              </button>

              <!-- TOUS LES COMMENTAIRES -->
              <div v-if="state.expandedMessageId === message._id" class="all-comments">
                <h6>Tous les commentaires :</h6>
                <div v-for="comment in state.allCommentsForMessage[message._id]" :key="comment._id" class="comment">
                  <div v-if="state.editingCommentId === comment._id">
                    <div class="form-group">
                      <label>Commentaire</label>
                      <textarea v-model="state.editingCommentData.content" class="textarea-small"></textarea>
                    </div>
                    <div class="form-group">
                      <label>Auteur</label>
                      <input v-model="state.editingCommentData.author" class="input-small" />
                    </div>
                    <div class="comment-actions">
                      <button @click="methods.updateComment()" class="btn primary tiny">Enregistrer</button>
                      <button @click="methods.cancelEditingComment()" class="btn secondary tiny">Annuler</button>
                    </div>
                  </div>
                  <div v-else>
                    <p><strong>{{ comment.author }}:</strong> {{ comment.content }}</p>
                    <div class="comment-actions">
                      <span class="comment-date">{{ new Date(comment.createdAt).toLocaleDateString() }}</span>
                      <button @click="methods.startEditingComment(comment)" class="btn secondary tiny">Modifier</button>
                      <button @click="methods.deleteComment(comment)" class="btn danger tiny">Supprimer</button>
                    </div>
                  </div>
                </div>
              </div>

              <!-- FORMULAIRE AJOUT COMMENTAIRE -->
              <div class="comment-form">
                <h6>💬 Ajouter un commentaire</h6>
                <form @submit.prevent="methods.createComment(message._id)">
                  <div class="form-group">
                    <label>Votre commentaire</label>
                    <textarea 
                      v-model="state.commentForms[message._id].content"
                      placeholder="Écrivez votre commentaire..." 
                      required 
                      class="textarea-small"
                    ></textarea>
                  </div>
                  <div class="form-group">
                    <label>Votre nom</label>
                    <input 
                      v-model="state.commentForms[message._id].author"
                      placeholder="Votre nom" 
                      required 
                      class="input-small" 
                    />
                  </div>
                  <button type="submit" class="btn primary tiny">✅ Publier le commentaire</button>
                </form>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- TOP 10 -->
      <section class="section">
        <h2>🏆 Top 10 Messages les plus likés</h2>
        <div v-if="computedData.topLikedMessages.length === 0" class="no-data">Aucun message</div>
        <div v-else class="top-messages">
          <div v-for="message in computedData.topLikedMessages" :key="message._id" class="top-message">
            <h4>{{ message.title }}</h4>
            <p>{{ message.content?.substring(0, 100) }}...</p>
            <div class="top-message-meta">
              <span>❤️ {{ message.likes || 0 }}</span>
              <span>{{ state.countries.find(c => c._id === message.countryId)?.name || 'Inconnu' }}</span>
            </div>
          </div>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
* { box-sizing: border-box; }

.app-container {
  max-width: 1400px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', sans-serif;
  background: #f5f7fa;
}

.app-header {
  background: white;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 30px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.app-header h1 {
  margin: 0;
  color: #2c3e50;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 15px;
}

.btn-toggle {
  padding: 10px 20px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
}

.status {
  font-weight: 600;
  color: #7f8c8d;
}

.loading {
  text-align: center;
  padding: 40px;
  font-size: 18px;
  color: #7f8c8d;
}

.section {
  background: white;
  padding: 25px;
  border-radius: 8px;
  margin-bottom: 30px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.section h2 {
  margin: 0 0 20px 0;
  color: #2c3e50;
  border-bottom: 2px solid #3498db;
  padding-bottom: 10px;
}

.search-bar, .toolbar {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.flex-1 { flex: 1; }

.input, .select, .textarea {
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
  font-family: inherit;
}

.textarea {
  min-height: 100px;
  resize: vertical;
}

.textarea-small {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
  min-height: 60px;
  width: 100%;
  resize: vertical;
  font-family: inherit;
}

.input-small {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
  width: 100%;
}

.form-container {
  background: #f8f9fa;
  padding: 20px;
  border-radius: 6px;
  margin-bottom: 20px;
}

.form-container h3 {
  margin: 0 0 15px 0;
  color: #34495e;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 15px;
  margin-bottom: 15px;
}

.form-group {
  display: flex;
  flex-direction: column;
  margin-bottom: 10px;
}

.form-group label {
  margin-bottom: 5px;
  font-weight: 600;
  color: #555;
  font-size: 13px;
}

.form-actions {
  display: flex;
  gap: 10px;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
}

.btn.primary { background: #3498db; color: white; }
.btn.secondary { background: #95a5a6; color: white; }
.btn.danger { background: #e74c3c; color: white; }
.btn.small { padding: 6px 12px; font-size: 12px; }
.btn.tiny { padding: 4px 8px; font-size: 11px; }

.btn-like {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 14px;
  color: #e74c3c;
  display: flex;
  align-items: center;
  gap: 5px;
}

.cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.card {
  border: 2px solid #ecf0f1;
  border-radius: 8px;
  padding: 15px;
  cursor: pointer;
  transition: all 0.3s;
}

.card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

.card.selected {
  border-color: #3498db;
  background: #ebf5fb;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.card-header h3 {
  margin: 0;
  color: #2c3e50;
}

.flag {
  width: 50px;
  height: auto;
  border: 1px solid #ddd;
}

.card-body p {
  margin: 5px 0;
  font-size: 14px;
  color: #555;
}

.card-actions {
  display: flex;
  gap: 8px;
  margin-top: 10px;
}

.messages-list {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.message-card {
  border: 1px solid #ecf0f1;
  border-radius: 8px;
  padding: 15px;
  background: #fafbfc;
}

.message-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 10px;
}

.message-header h4 {
  margin: 0;
  color: #2c3e50;
  flex: 1;
}

.message-actions {
  display: flex;
  gap: 8px;
  align-items: center;
}

.message-content {
  margin: 10px 0;
  line-height: 1.6;
  color: #555;
}

.comments-section {
  margin-top: 15px;
  padding-top: 15px;
  border-top: 1px dashed #ddd;
}

.comments-section h5 {
  margin: 0 0 10px 0;
  color: #7f8c8d;
  font-size: 14px;
}

.comment {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  padding: 10px;
  margin-bottom: 10px;
}

.comment p {
  margin: 0 0 8px 0;
  font-size: 13px;
}

.comment-actions {
  display: flex;
  gap: 6px;
  align-items: center;
}

.comment-date {
  font-size: 11px;
  color: #95a5a6;
  margin-right: auto;
}

.all-comments {
  margin: 15px 0;
}

.all-comments h6 {
  margin: 0 0 10px 0;
  color: #555;
  font-size: 13px;
}

.comment-form {
  margin-top: 15px;
  padding: 15px;
  background: #f0f8ff;
  border-radius: 6px;
  border: 2px dashed #3498db;
}

.comment-form h6 {
  margin: 0 0 10px 0;
  color: #2c3e50;
  font-size: 14px;
}

.comment-form form {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.no-comment {
  padding: 10px;
  color: #95a5a6;
  font-style: italic;
  font-size: 13px;
}

.top-messages {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 15px;
}

.top-message {
  border-left: 3px solid #3498db;
  padding: 15px;
  background: #f8f9fa;
  border-radius: 6px;
}

.top-message h4 {
  margin: 0 0 8px 0;
  color: #2c3e50;
  font-size: 16px;
}

.top-message p {
  margin: 0 0 10px 0;
  font-size: 13px;
  color: #555;
}

.top-message-meta {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: #7f8c8d;
}

.no-data {
  text-align: center;
  padding: 30px;
  color: #95a5a6;
  font-style: italic;
}

@media (max-width: 768px) {
  .form-grid {
    grid-template-columns: 1fr;
  }
  .cards-grid {
    grid-template-columns: 1fr;
  }
  .app-header {
    flex-direction: column;
    gap: 15px;
  }
  .message-header {
    flex-direction: column;
    gap: 10px;
  }
}
</style>