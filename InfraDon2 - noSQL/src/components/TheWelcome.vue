<script setup>
// Importations
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);

// Interfaces
const countryProps = {
  _id: String,
  _rev: { type: String, default: null },
  name: String,
  capital: String,
  population: Number,
  area: Number,
  currency: String,
  languages: [String, Array],
  region: String,
  subregion: String,
  flag: String
};

// État réactif
const state = ref({
  localDB: null,
  remoteDB: null,
  countries: [],
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
  editingCountryId: null,
  searchRegion: '',
  isOnline: true,
  syncHandler: null
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
    .on('change', methods.fetchData)
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

  async fetchData() {
    if (!state.value.localDB) return;

    try {
      const result = await state.value.localDB.allDocs({ include_docs: true });
      state.value.countries = result.rows
        .map(row => row.doc)
        .filter(doc => doc && !doc._id.startsWith('_'));
    } catch (error) {
      console.error('Erreur lors de la récupération des données:', error);
    }
  },

  async getDocument(id) {
    if (!state.value.localDB) return null;
    try {
      return await state.value.localDB.get(id);
    } catch (error) {
      console.error('Erreur lors de la récupération du document:', error);
      return null;
    }
  },

  async createCountry() {
    if (!state.value.localDB) return;

    try {
      const doc = {
        ...state.value.newCountry,
        languages: state.value.newCountry.languages.split(',').map(lang => lang.trim()),
        _id: `country_${Date.now()}`
      };

      await state.value.localDB.post(doc);
      methods.resetNewCountryForm();
      await methods.fetchData();
    } catch (error) {
      console.error('Erreur lors de la création:', error);
    }
  },

  async updateCountry() {
    if (!state.value.localDB || !state.value.editingCountryId) return;

    try {
      const doc = await methods.getDocument(state.value.editingCountryId);
      if (!doc) return;

      const updatedDoc = {
        ...doc,
        ...state.value.newCountry,
        languages: state.value.newCountry.languages.split(',').map(lang => lang.trim())
      };

      await state.value.localDB.put(updatedDoc);
      methods.resetNewCountryForm();
      state.value.editingCountryId = null;
      await methods.fetchData();
    } catch (error) {
      console.error('Erreur lors de la mise à jour:', error);
    }
  },

  async deleteCountry(id) {
    if (!state.value.localDB) return;

    try {
      const doc = await methods.getDocument(id);
      if (!doc) return;

      await state.value.localDB.remove(doc);
      await methods.fetchData();
    } catch (error) {
      console.error('Erreur lors de la suppression:', error);
    }
  },

  startEditing(country) {
    state.value.newCountry = {
      ...country,
      languages: Array.isArray(country.languages) ? country.languages.join(', ') : country.languages
    };
    state.value.editingCountryId = country._id;
  },

  cancelEditing() {
    methods.resetNewCountryForm();
    state.value.editingCountryId = null;
  },

  resetNewCountryForm() {
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

  handleSyncError(error) {
    console.error('Erreur de synchronisation:', error);
  },

  async createIndex() {
    if (!state.value.localDB) return;
    try {
      await state.value.localDB.createIndex({ index: { fields: ['region'] } });
      console.log('Index créé sur "region"');
    } catch (error) {
      console.error('Erreur lors de la création de l\'index:', error);
    }
  },

  async searchByRegion() {
    if (!state.value.localDB || !state.value.searchRegion) return;

    try {
      const result = await state.value.localDB.find({
        selector: { region: { $eq: state.value.searchRegion } }
      });
      state.value.countries = result.docs;
    } catch (error) {
      console.error('Erreur lors de la recherche:', error);
    }
  }
};

// Cycle de vie
onMounted(() => {
  methods.initDatabases();
  if (state.value.isOnline) methods.syncDatabases();
  methods.createIndex();
  methods.fetchData();
});
</script>

<template>
  <div class="app-container">
    <h1>Gestion des Pays</h1>

    <!-- Toggle Online/Offline -->
    <section class="toggle-section">
      <button @click="methods.toggleOnlineOffline" class="toggle-button">
        {{ state.isOnline ? 'Passer en mode OFFLINE' : 'Passer en mode ONLINE' }}
      </button>
      <span class="status">Statut : {{ state.isOnline ? 'ONLINE' : 'OFFLINE' }}</span>
    </section>

    <!-- Recherche par région -->
    <section class="search-section">
      <h2>Rechercher par région</h2>
      <div class="search-container">
        <input
          v-model="state.searchRegion"
          placeholder="Ex: Europe"
          @keyup.enter="methods.searchByRegion"
          class="search-input"
        />
        <button @click="methods.searchByRegion" class="search-button">Rechercher</button>
        <button @click="methods.fetchData" class="reset-button">Réinitialiser</button>
      </div>
    </section>

    <!-- Liste des pays -->
    <section class="countries-section">
      <div v-for="country in state.countries" :key="country._id" class="country-card">
        <div class="country-header">
          <h2>{{ country.name }}</h2>
          <img :src="country.flag" alt="flag" class="country-flag" />
        </div>

        <div class="country-details">
          <p><strong>Capitale:</strong> {{ country.capital }}</p>
          <p><strong>Population:</strong> {{ country.population.toLocaleString() }}</p>
          <p><strong>Région:</strong> {{ country.region }} ({{ country.subregion }})</p>
          <p><strong>Monnaie:</strong> {{ country.currency }}</p>
          <p><strong>Langues:</strong>
            {{
              Array.isArray(country.languages)
                ? country.languages.join(', ')
                : country.languages
            }}
          </p>
        </div>

        <div class="country-actions">
          <button @click="methods.startEditing(country)" class="edit-button">
            {{ state.editingCountryId === country._id ? 'Annuler' : 'Modifier' }}
          </button>
          <button @click="methods.deleteCountry(country._id)" class="delete-button">Supprimer</button>
        </div>

        <!-- Formulaire de modification intégré -->
        <div v-if="state.editingCountryId === country._id" class="edit-form">
          <form @submit.prevent="methods.updateCountry" class="form-container">
            <div class="form-row">
              <div class="form-group">
                <label>Nom</label>
                <input v-model="state.newCountry.name" required class="form-input" />
              </div>
              <div class="form-group">
                <label>Capitale</label>
                <input v-model="state.newCountry.capital" required class="form-input" />
              </div>
            </div>

            <div class="form-row">
              <div class="form-group">
                <label>Population</label>
                <input type="number" v-model.number="state.newCountry.population" required class="form-input" />
              </div>
              <div class="form-group">
                <label>Superficie</label>
                <input type="number" v-model.number="state.newCountry.area" required class="form-input" />
              </div>
            </div>

            <div class="form-row">
              <div class="form-group">
                <label>Monnaie</label>
                <input v-model="state.newCountry.currency" required class="form-input" />
              </div>
              <div class="form-group">
                <label>Langues</label>
                <input v-model="state.newCountry.languages" placeholder="Langues séparées par des virgules" required class="form-input" />
              </div>
            </div>

            <div class="form-row">
              <div class="form-group">
                <label>Région</label>
                <input v-model="state.newCountry.region" required class="form-input" />
              </div>
              <div class="form-group">
                <label>Sous-région</label>
                <input v-model="state.newCountry.subregion" required class="form-input" />
              </div>
            </div>

            <div class="form-group">
              <label>Drapeau (URL)</label>
              <input v-model="state.newCountry.flag" required class="form-input" />
            </div>

            <div class="form-actions">
              <button type="submit" class="submit-button">Enregistrer</button>
              <button type="button" @click="methods.cancelEditing" class="cancel-button">Annuler</button>
            </div>
          </form>
        </div>
      </div>
    </section>

    <!-- Formulaire d'ajout -->
    <section class="form-section">
      <h2>Ajouter un pays</h2>
      <form @submit.prevent="methods.createCountry" class="form-container">
        <div class="form-row">
          <div class="form-group">
            <label>Nom</label>
            <input v-model="state.newCountry.name" required class="form-input" />
          </div>
          <div class="form-group">
            <label>Capitale</label>
            <input v-model="state.newCountry.capital" required class="form-input" />
          </div>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label>Population</label>
            <input type="number" v-model.number="state.newCountry.population" required class="form-input" />
          </div>
          <div class="form-group">
            <label>Superficie</label>
            <input type="number" v-model.number="state.newCountry.area" required class="form-input" />
          </div>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label>Monnaie</label>
            <input v-model="state.newCountry.currency" required class="form-input" />
          </div>
          <div class="form-group">
            <label>Langues</label>
            <input v-model="state.newCountry.languages" placeholder="Langues séparées par des virgules" required class="form-input" />
          </div>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label>Région</label>
            <input v-model="state.newCountry.region" required class="form-input" />
          </div>
          <div class="form-group">
            <label>Sous-région</label>
            <input v-model="state.newCountry.subregion" required class="form-input" />
          </div>
        </div>

        <div class="form-group">
          <label>Drapeau (URL)</label>
          <input v-model="state.newCountry.flag" required class="form-input" />
        </div>

        <div class="form-actions">
          <button type="submit" class="submit-button">Ajouter</button>
        </div>
      </form>
    </section>
  </div>
</template>

<style scoped>
.app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Arial', sans-serif;
}

h1, h2 {
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

.search-section {
  margin-bottom: 20px;
}

.search-container {
  display: flex;
  gap: 10px;
  margin-bottom: 10px;
}

.search-input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  flex: 1;
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

.countries-section {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
  margin-bottom: 20px;
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

.country-details {
  margin-bottom: 15px;
}

.country-actions {
  display: flex;
  gap: 10px;
  margin-bottom: 15px;
}

.edit-button, .delete-button {
  padding: 5px 10px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.8em;
}

.edit-button {
  background-color: #f39c12;
  color: white;
}

.edit-button:hover {
  background-color: #e67e22;
}

.delete-button {
  background-color: #e74c3c;
  color: white;
}

.delete-button:hover {
  background-color: #c0392b;
}

.edit-form {
  margin-top: 15px;
  padding: 15px;
  background-color: #f9f9f9;
  border-radius: 4px;
  border: 1px solid #ddd;
}

.form-section {
  margin-bottom: 20px;
  padding: 15px;
  background-color: #f9f9f9;
  border-radius: 8px;
  border: 1px solid #ddd;
}

.form-container {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.form-row {
  display: flex;
  gap: 15px;
}

.form-row .form-group {
  flex: 1;
}

.form-group {
  display: flex;
  flex-direction: column;
}

.form-group label {
  margin-bottom: 5px;
  font-weight: bold;
  font-size: 0.9em;
}

.form-input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 0.9em;
}

.form-actions {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}

.submit-button, .cancel-button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9em;
}

.submit-button {
  background-color: #2ecc71;
  color: white;
}

.submit-button:hover {
  background-color: #27ae60;
}

.cancel-button {
  background-color: #e74c3c;
  color: white;
}

.cancel-button:hover {
  background-color: #c0392b;
}

@media (max-width: 768px) {
  .form-row {
    flex-direction: column;
    gap: 10px;
  }

  .form-row .form-group {
    flex: none;
  }
}
</style>
