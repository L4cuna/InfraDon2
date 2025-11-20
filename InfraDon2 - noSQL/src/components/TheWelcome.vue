<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';

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
  flag: 'https://example.com/flag.png'
});

const editingPost = ref<Post | null>(null);
const searchRegion = ref<string>('');
const isOnline = ref<boolean>(true);

// === 1. Create/Connect Database (https://pouchdb.com/api.html#create_database) ===
const initDatabases = () => {
  // Base locale
  localDB.value = new PouchDB('local_biblio_db');
  // Base distante
  remoteDB.value = new PouchDB('http://admin:admin@localhost:5984/firstdbinfradon2');
  console.log('Bases de données créées/connectées');
};

// === 2. Replicate Databases (https://pouchdb.com/api.html#replication) ===
const syncDatabases = () => {
  if (!isOnline.value || !localDB.value || !remoteDB.value) return;

  // Synchronisation bidirectionnelle et continue
  syncHandler.value = localDB.value.sync(remoteDB.value, {
    live: true,
    retry: true,
  })
  .on('change', (info) => {
    console.log('Changement synchronisé:', info);
    fetchData();
  })
  .on('denied', (err) => {
    console.error('Conflit de synchronisation:', err);
    alert('Conflit détecté. Vérifie les données dans CouchDB et l\'application.');
  })
  .on('error', (err) => {
    console.error('Erreur de synchronisation:', err);
  });
};

// === 3. Toggle Online/Offline ===
const toggleOnlineOffline = () => {
  isOnline.value = !isOnline.value;
  if (isOnline.value) {
    console.log('Mode ONLINE - Synchronisation activée');
    syncDatabases();
  } else {
    console.log('Mode OFFLINE - Synchronisation désactivée');
    if (syncHandler.value) {
      syncHandler.value.cancel();
    }
  }
};

// === 4. Retrieve All Documents (https://pouchdb.com/api.html#batch_fetch) ===
const fetchData = async () => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const result = await localDB.value.allDocs({ include_docs: true });
    postsData.value = result.rows.map(row => row.doc);
    console.log('Données récupérées:', postsData.value.length, 'documents');
  } catch (error) {
    console.error('Erreur lors de la récupération:', error);
  }
};

// === 5. Retrieve One Document (https://pouchdb.com/api.html#fetch_document) ===
const getDocument = async (id: string) => {
  if (!localDB.value) return null;
  try {
    const doc = await localDB.value.get(id);
    return doc;
  } catch (error) {
    console.error('Erreur lors de la récupération du document:', error);
    return null;
  }
};

// === 6. Create/Update Document (https://pouchdb.com/api.html#create_document) ===
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
    console.log('Document créé:', response.id);
    await fetchData();
  } catch (error) {
    console.error('Erreur lors de la création:', error);
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
    console.log('Document mis à jour:', response.id);
    await fetchData();
  } catch (error) {
    console.error('Erreur lors de la mise à jour:', error);
  }
};

// === 7. Delete Document (https://pouchdb.com/api.html#delete_document) ===
const deletePost = async (post: Post) => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const doc = await getDocument(post._id);
    if (!doc) return;
    const response = await localDB.value.remove(doc);
    console.log('Document supprimé:', response.id);
    await fetchData();
  } catch (error) {
    console.error('Erreur lors de la suppression:', error);
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
  flag: 'https://example.com/flag.png',
});

const generateAndInsertData = async (count: number) => {
  for (let i = 0; i < count; i++) {
    await createPost(generateRandomPost());
  }
  console.log(`${count} documents générés !`);
};

// === 9. Indexation et recherche ===
const createIndex = async () => {
  if (!localDB.value) return;
  try {
    await localDB.value.createIndex({
      index: { fields: ['region'] },
    });
    console.log('Index créé sur "region"');
  } catch (error) {
    console.error('Erreur lors de la création de l\'index:', error);
  }
};

const searchByRegion = async () => {
  if (!localDB.value) return;
  try {
    const result = await localDB.value.find({
      selector: { region: { $eq: searchRegion.value } },
    });
    postsData.value = result.docs;
    console.log('Résultats de recherche:', result.docs.length, 'documents');
  } catch (error) {
    console.error('Erreur lors de la recherche:', error);
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
    flag: 'https://example.com/flag.png',
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
  console.log('Composant initialisé');
  initDatabases();
  if (isOnline.value) syncDatabases();
  createIndex();
  fetchData();
  // generateAndInsertData(10); // Décommente pour générer des données de test
});
</script>

<template>
  <h1>Gestion des pays</h1>

  <!-- Bouton toggle online/offline -->
  <section>
    <button @click="toggleOnlineOffline">
      {{ isOnline ? 'Passer en mode OFFLINE' : 'Passer en mode ONLINE' }}
    </button>
    <span>Statut : {{ isOnline ? 'ONLINE' : 'OFFLINE' }}</span>
  </section>

  <!-- Recherche par région -->
  <section>
    <h2>Rechercher par région</h2>
    <input v-model="searchRegion" placeholder="Ex: Europe" @keyup.enter="searchByRegion" />
    <button @click="searchByRegion">Rechercher</button>
    <button @click="fetchData">Réinitialiser</button>
  </section>

  <!-- Liste des pays -->
  <article v-for="post in postsData" :key="post._id">
    <h2>{{ post.name }}</h2>
    <img :src="post.flag" alt="flag" width="100" />
    <p>Capital: {{ post.capital }}</p>
    <p>Population: {{ post.population.toLocaleString() }}</p>
    <p>Région: {{ post.region }} ({{ post.subregion }})</p>
    <p>Monnaie: {{ post.currency }}</p>
    <p>Langues: {{ Array.isArray(post.languages) ? post.languages.join(', ') : post.languages }}</p>
    <div>
      <button @click="editPost(post)">Modifier</button>
      <button @click="deletePost(post)">Supprimer</button>
    </div>
  </article>

  <!-- Formulaire d'édition -->
  <section v-if="editingPost">
    <h2>Modifier {{ editingPost.name }}</h2>
    <form @submit.prevent="updateSubmit">
      <div><label>Nom: <input v-model="editingPost.name" required /></label></div>
      <div><label>Capitale: <input v-model="editingPost.capital" required /></label></div>
      <div><label>Population: <input type="number" v-model.number="editingPost.population" required /></label></div>
      <div><label>Superficie: <input type="number" v-model.number="editingPost.area" required /></label></div>
      <div><label>Monnaie: <input v-model="editingPost.currency" required /></label></div>
      <div><label>Langues (virgules): <input v-model="editingPost.languages" required /></label></div>
      <div><label>Région: <input v-model="editingPost.region" required /></label></div>
      <div><label>Sous-région: <input v-model="editingPost.subregion" required /></label></div>
      <div><label>Drapeau (URL): <input v-model="editingPost.flag" required /></label></div>
      <button type="submit">Enregistrer</button>
      <button type="button" @click="cancelEdit">Annuler</button>
    </form>
  </section>

  <!-- Formulaire d'ajout -->
  <section>
    <h2>Ajouter un pays</h2>
    <form @submit.prevent="handleSubmit">
      <div><label>Nom: <input v-model="newPost.name" required /></label></div>
      <div><label>Capitale: <input v-model="newPost.capital" required /></label></div>
      <div><label>Population: <input type="number" v-model.number="newPost.population" required /></label></div>
      <div><label>Superficie: <input type="number" v-model.number="newPost.area" required /></label></div>
      <div><label>Monnaie: <input v-model="newPost.currency" required /></label></div>
      <div><label>Langues (virgules): <input v-model="newPost.languages" required /></label></div>
      <div><label>Région: <input v-model="newPost.region" required /></label></div>
      <div><label>Sous-région: <input v-model="newPost.subregion" required /></label></div>
      <div><label>Drapeau (URL): <input v-model="newPost.flag" required /></label></div>
      <button type="submit">Ajouter</button>
    </form>
  </section>
</template>

<style scoped>
button { margin-right: 10px; margin-bottom: 10px; }
span { margin-left: 10px; }
form div { margin-bottom: 40px; }
label { display: inline-block; width: 120px; }
</style>
