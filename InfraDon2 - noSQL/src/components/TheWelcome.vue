<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';

// Ajout du plugin pour l'indexation et la recherche
PouchDB.plugin(PouchDBFind);

// === Interfaces ===
interface Post {
  _id: string;
  _rev?: string;
  name: string;
  capital: string;
  population: number;
  area: number;
  currency: string;
  languages: string[] | string;
  region: string;
  subregion: string;
  flag: string;
}

interface NewPost {
  name: string;
  capital: string;
  population: number;
  area: number;
  currency: string;
  languages: string;
  region: string;
  subregion: string;
  flag: string;
}

// === Références aux bases et données ===
const localDB = ref<PouchDB.Database>();
const remoteDB = ref<PouchDB.Database>();
const postsData = ref<Post[]>([]);
const syncHandler = ref<any>(null);

// === Données réactives ===
const newPost = ref<NewPost>({
  name: '',
  capital: '',
  population: 0,
  area: 0,
  currency: '',
  languages: '',
  region: '',
  subregion: '',
  flag: 'https://flagcdn.com/160x120/fr.png'
});

const editingPost = ref<Post | null>(null);
const searchRegion = ref<string>('');
const isOnline = ref<boolean>(true);

// === 1. Create/Connect Database ===
const initDatabases = () => {
  localDB.value = new PouchDB('local_biblio_db');
  remoteDB.value = new PouchDB('http://admin:admin@localhost:5984/firstdbinfradon2');
  console.log('✅ Bases de données créées/connectées');
};

// === 2. Replicate Databases ===
const syncDatabases = () => {
  if (!isOnline.value || !localDB.value || !remoteDB.value) return;

  syncHandler.value = localDB.value.sync(remoteDB.value, {
    live: true,
    retry: true,
  })
  .on('change', (info) => {
    console.log('↔️ Changement synchronisé:', info);
    fetchData();
  })
  .on('denied', (err) => {
    console.error('⚠️ Conflit de synchronisation:', err);
    alert('Conflit détecté. Vérifiez les données dans CouchDB et l\'application.');
  })
  .on('error', (err) => {
    console.error('❌ Erreur de synchronisation:', err);
  });
};

// === 3. Toggle Online/Offline ===
const toggleOnlineOffline = () => {
  isOnline.value = !isOnline.value;
  if (isOnline.value) {
    console.log('🌐 Mode ONLINE - Synchronisation activée');
    syncDatabases();
  } else {
    console.log('🔌 Mode OFFLINE - Synchronisation désactivée');
    if (syncHandler.value) {
      syncHandler.value.cancel();
    }
  }
};

// === 4. Retrieve All Documents ===
const fetchData = () => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }

  localDB.value.allDocs({ include_docs: true })
    .then((docs) => {
      postsData.value = docs.rows
        .map((row) => row.doc)
        .filter((doc): doc is Post => !!doc)
        .filter((doc) => !doc._id.startsWith("_"));
    })
    .catch((err) => console.error("Erreur lors de la récupération des données:", err));
};

// === 5. Retrieve One Document ===
const getDocument = async (id: string) => {
  if (!localDB.value) return null;
  try {
    const doc = await localDB.value.get(id);
    return doc;
  } catch (error) {
    console.error('❌ Erreur lors de la récupération du document:', error);
    return null;
  }
};

// === 6. Create/Update Document ===
const createPost = async (post: NewPost) => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const doc = {
      ...post,
      languages: post.languages.split(',').map(lang => lang.trim()),
      _id: `country_${Date.now()}`,
    };
    const response = await localDB.value.post(doc);
    console.log('📝 Document créé:', response.id);
    await fetchData();
  } catch (error) {
    console.error('❌ Erreur lors de la création:', error);
  }
};

const updatePost = async (post: Post) => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const doc = await getDocument(post._id);
    if (!doc) return;
    const updatedDoc = {
      ...doc,
      ...post,
      languages: post.languages.toString().split(',').map(lang => lang.trim()),
    };
    const response = await localDB.value.put(updatedDoc);
    console.log('🔄 Document mis à jour:', response.id);
    await fetchData();
  } catch (error) {
    console.error('❌ Erreur lors de la mise à jour:', error);
  }
};

// === 7. Delete Document ===
const deletePost = async (post: Post) => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const doc = await getDocument(post._id);
    if (!doc) return;
    const response = await localDB.value.remove(doc);
    console.log('🗑️ Document supprimé:', response.id);
    await fetchData();
  } catch (error) {
    console.error('❌ Erreur lors de la suppression:', error);
  }
};

// === 8. Factory pour générer des données ===
const regions = ['Europe', 'Asia', 'Africa', 'Americas', 'Oceania'];
const currencies = ['EUR', 'USD', 'JPY', 'GBP', 'AUD'];
const languagesList = ['French', 'English', 'Spanish', 'German', 'Chinese'];

const generateRandomPost = (): NewPost => ({
  name: `Country_${Math.floor(Math.random() * 1000)}`,
  capital: `Capital_${Math.floor(Math.random() * 1000)}`,
  population: Math.floor(Math.random() * 100000000),
  area: Math.floor(Math.random() * 1000000),
  currency: currencies[Math.floor(Math.random() * currencies.length)],
  languages: Array.from({ length: 2 }, () =>
    languagesList[Math.floor(Math.random() * languagesList.length)]
  ).join(', '),
  region: regions[Math.floor(Math.random() * regions.length)],
  subregion: `${regions[Math.floor(Math.random() * regions.length)]}_Sub`,
  flag: 'https://flagcdn.com/160x120/fr.png',
});

const generateAndInsertData = async (count: number) => {
  for (let i = 0; i < count; i++) {
    await createPost(generateRandomPost());
  }
  console.log(`✅ ${count} documents générés !`);
};

// === 9. Indexation et recherche ===
const createIndex = async () => {
  if (!localDB.value) return;
  try {
    await localDB.value.createIndex({
      index: { fields: ['region'] },
    });
    console.log('📊 Index créé sur "region"');
  } catch (error) {
    console.error('❌ Erreur lors de la création de l\'index:', error);
  }
};

const searchByRegion = async () => {
  if (!localDB.value) return;
  try {
    const result = await localDB.value.find({
      selector: { region: { $eq: searchRegion.value } },
    });
    postsData.value = result.docs;
    console.log('🔍 Résultats de recherche:', result.docs.length, 'documents');
  } catch (error) {
    console.error('❌ Erreur lors de la recherche:', error);
  }
};

// === 10. Gestion des formulaires ===
const handleSubmit = () => {
  createPost(newPost.value);
  newPost.value = {
    name: '',
    capital: '',
    population: 0,
    area: 0,
    currency: '',
    languages: '',
    region: '',
    subregion: '',
    flag: 'https://flagcdn.com/160x120/fr.png',
  };
};

const editPost = (post: Post) => {
  editingPost.value = {
    ...post,
    languages: Array.isArray(post.languages) ? post.languages.join(', ') : post.languages,
  };
};

const cancelEdit = () => { editingPost.value = null; };

const updateSubmit = () => {
  if (!editingPost.value) return;
  updatePost(editingPost.value);
  cancelEdit();
};

// === 11. Initialisation ===
onMounted(() => {
  console.log('🚀 Composant initialisé');
  initDatabases();
  if (isOnline.value) syncDatabases();
  createIndex();
  fetchData();
  // generateAndInsertData(10); // Décommente pour générer des données de test
});
</script>

<template>
  <div class="app-container">
    <h1>Gestion des Pays</h1>

    <!-- Bouton toggle online/offline -->
    <section class="toggle-section">
      <button @click="toggleOnlineOffline" class="toggle-button">
        {{ isOnline ? 'Passer en mode OFFLINE' : 'Passer en mode ONLINE' }}
      </button>
      <span class="status">Statut : {{ isOnline ? '🌐 ONLINE' : '🔌 OFFLINE' }}</span>
    </section>

    <!-- Recherche par région -->
    <section class="search-section">
      <h2>Rechercher par région</h2>
      <div class="search-container">
        <input v-model="searchRegion" placeholder="Ex: Europe" @keyup.enter="searchByRegion" class="search-input" />
        <button @click="searchByRegion" class="search-button">Rechercher</button>
        <button @click="fetchData" class="reset-button">Réinitialiser</button>
      </div>
    </section>

    <!-- Liste des pays -->
    <section class="countries-section">
      <article v-for="post in postsData" :key="post._id" class="country-card">
        <h2>{{ post.name }}</h2>
        <img :src="post.flag" alt="flag" class="country-flag" />
        <p><strong>Capitale:</strong> {{ post.capital }}</p>
        <p><strong>Population:</strong> {{ post.population.toLocaleString() }}</p>
        <p><strong>Région:</strong> {{ post.region }} ({{ post.subregion }})</p>
        <p><strong>Monnaie:</strong> {{ post.currency }}</p>
        <p><strong>Langues:</strong> {{ Array.isArray(post.languages) ? post.languages.join(', ') : post.languages }}</p>
        <div class="actions">
          <button @click="editPost(post)" class="edit-button">Modifier</button>
          <button @click="deletePost(post)" class="delete-button">Supprimer</button>
        </div>
      </article>
    </section>

    <!-- Formulaire d'édition -->
    <section v-if="editingPost" class="form-section">
      <h2>Modifier {{ editingPost.name }}</h2>
      <form @submit.prevent="updateSubmit" class="form-container">
        <div class="form-group"><label>Nom: <input v-model="editingPost.name" required class="form-input" /></label></div>
        <div class="form-group"><label>Capitale: <input v-model="editingPost.capital" required class="form-input" /></label></div>
        <div class="form-group"><label>Population: <input type="number" v-model.number="editingPost.population" required class="form-input" /></label></div>
        <div class="form-group"><label>Superficie: <input type="number" v-model.number="editingPost.area" required class="form-input" /></label></div>
        <div class="form-group"><label>Monnaie: <input v-model="editingPost.currency" required class="form-input" /></label></div>
        <div class="form-group"><label>Langues (virgules): <input v-model="editingPost.languages" required class="form-input" /></label></div>
        <div class="form-group"><label>Région: <input v-model="editingPost.region" required class="form-input" /></label></div>
        <div class="form-group"><label>Sous-région: <input v-model="editingPost.subregion" required class="form-input" /></label></div>
        <div class="form-group"><label>Drapeau (URL): <input v-model="editingPost.flag" required class="form-input" /></label></div>
        <div class="form-actions">
          <button type="submit" class="submit-button">Enregistrer</button>
          <button type="button" @click="cancelEdit" class="cancel-button">Annuler</button>
        </div>
      </form>
    </section>

    <!-- Formulaire d'ajout -->
    <section class="form-section">
      <h2>Ajouter un pays</h2>
      <form @submit.prevent="handleSubmit" class="form-container">
        <div class="form-group"><label>Nom: <input v-model="newPost.name" required class="form-input" /></label></div>
        <div class="form-group"><label>Capitale: <input v-model="newPost.capital" required class="form-input" /></label></div>
        <div class="form-group"><label>Population: <input type="number" v-model.number="newPost.population" required class="form-input" /></label></div>
        <div class="form-group"><label>Superficie: <input type="number" v-model.number="newPost.area" required class="form-input" /></label></div>
        <div class="form-group"><label>Monnaie: <input v-model="newPost.currency" required class="form-input" /></label></div>
        <div class="form-group"><label>Langues (virgules): <input v-model="newPost.languages" required class="form-input" /></label></div>
        <div class="form-group"><label>Région: <input v-model="newPost.region" required class="form-input" /></label></div>
        <div class="form-group"><label>Sous-région: <input v-model="newPost.subregion" required class="form-input" /></label></div>
        <div class="form-group"><label>Drapeau (URL): <input v-model="newPost.flag" required class="form-input" /></label></div>
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

.country-flag {
  width: 100px;
  height: auto;
  margin-bottom: 10px;
}

.actions {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}

.edit-button, .delete-button {
  padding: 5px 10px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
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

.form-section {
  margin-bottom: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  background-color: #f9f9f9;
}

.form-container {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.form-group {
  display: flex;
  flex-direction: column;
}

.form-input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
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
</style>
