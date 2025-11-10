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

// === Données réactives pour le formulaire ===
const newPost = ref<NewPost>({
  name: '',
  capital: '',
  population: 0,
  area: 0,
  currency: '',
  languages: '',
  region: '',
  subregion: '',
  flag: ''
});

// === Variables réactives pour gérer l'édition ===
const editingPost = ref<Post | null>(null);

// === Initialisation des bases locale et distante ===
const initDatabases = () => {
  localDB.value = new PouchDB('local_biblio_db');
  remoteDB.value = new PouchDB('http://admin:admin@localhost:5984/firstdbinfradon2');
  console.log('✅ Bases locale et distante initialisées');
};

// === Synchronisation des bases ===
const syncDatabases = () => {
  if (!localDB.value || !remoteDB.value) return;
  localDB.value.sync(remoteDB.value, {
    live: true, // Synchronisation en temps réel
    retry: true, // Réessayer en cas d'échec
  })
  .on('change', () => {
    console.log('↔️ Synchronisation en cours...');
    fetchData(); // Rafraîchir les données locales
  })
  .on('error', (err) => {
    console.error('❌ Erreur de synchronisation :', err);
  });
};

// === Récupération des données locales ===
const fetchData = async () => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const result = await localDB.value.allDocs({ include_docs: true });
    postsData.value = result.rows.map(row => row.doc);
    console.log('📋 Données locales mises à jour :', postsData.value);
  } catch (error) {
    console.error('❌ Erreur lors de la récupération :', error);
  }
};

// === Ajouter un document localement ===
const createPost = async (newPost: NewPost) => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const languagesArray = newPost.languages.split(',').map(lang => lang.trim());
    const doc = {
      ...newPost,
      languages: languagesArray,
      _id: `country_${Date.now()}`,
    };
    await localDB.value.post(doc);
    console.log('📝 Document ajouté localement (sera synchronisé)');
    await fetchData(); // Rafraîchir la liste
  } catch (error) {
    console.error('❌ Erreur lors de l\'ajout :', error);
  }
};

// === Modifier un document ===
const updatePost = async (post: Post) => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const doc = await localDB.value.get(post._id);
    const updatedDoc = { ...doc, ...post };
    await localDB.value.put(updatedDoc);
    console.log('🔄 Document mis à jour localement (sera synchronisé)');
    await fetchData();
  } catch (error) {
    console.error('❌ Erreur lors de la mise à jour :', error);
  }
};

// === Supprimer un document ===
const deletePost = async (post: Post) => {
  if (!localDB.value) {
    console.warn('Base locale non initialisée');
    return;
  }
  try {
    const doc = await localDB.value.get(post._id);
    await localDB.value.remove(doc);
    console.log('🗑️ Document supprimé localement (sera synchronisé)');
    await fetchData();
  } catch (error) {
    console.error('❌ Erreur lors de la suppression :', error);
  }
};

// === Gestion du formulaire d'ajout ===
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
    flag: ''
  };
};

// === Gestion de l'édition ===
const editPost = (post: Post) => {
  const languagesString = Array.isArray(post.languages) ? post.languages.join(', ') : post.languages;
  editingPost.value = { ...post, languages: languagesString };
};

const cancelEdit = () => {
  editingPost.value = null;
};

const updateSubmit = () => {
  if (!editingPost.value) return;
  const languagesArray = editingPost.value.languages.toString().split(',').map(lang => lang.trim());
  updatePost({ ...editingPost.value, languages: languagesArray });
  cancelEdit();
};

// === Montage du composant ===
onMounted(() => {
  console.log('🚀 Composant initialisé');
  initDatabases();
  syncDatabases();
  fetchData();
});
</script>

<template>
  <!-- Affichage des pays (inchangé) -->
  <h1>Gestion des pays</h1>
  <article v-for="post in postsData" :key="post._id">
    <h2>{{ post.name }}</h2>
    <img :src="post.flag" alt="flag" width="100" />
    <p>Capital: {{ post.capital }}</p>
    <p>Population: {{ post.population.toLocaleString() }}</p>
    <p>Region: {{ post.region }} ({{ post.subregion }})</p>
    <p>Currency: {{ post.currency }}</p>
    <p>Languages: {{ Array.isArray(post.languages) ? post.languages.join(', ') : post.languages }}</p>
    <div>
      <button @click="editPost(post)">Modifier</button>
      <button @click="deletePost(post)">Supprimer</button>
    </div>
  </article>

  <!-- Formulaire d'édition (inchangé) -->
  <section v-if="editingPost">
    <h2>Modifier un pays</h2>
    <form @submit.prevent="updateSubmit">
      <!-- Champs du formulaire (inchangés) -->
      <div>
        <label for="editName">Nom :</label>
        <input type="text" id="editName" v-model="editingPost.name" required />
      </div>
      <!-- ... (autres champs) ... -->
      <button type="submit">Enregistrer</button>
      <button type="button" @click="cancelEdit">Annuler</button>
    </form>
  </section>

  <!-- Formulaire d'ajout (inchangé) -->
  <section>
    <h2>Ajouter un pays</h2>
    <form @submit.prevent="handleSubmit">
      <!-- Champs du formulaire (inchangés) -->
      <div>
        <label for="name">Nom :</label>
        <input type="text" id="name" v-model="newPost.name" required />
      </div>
      <!-- ... (autres champs) ... -->
      <button type="submit">Ajouter</button>
    </form>
  </section>
</template>

<style scoped>
/* Styles inchangés */
ul { list-style: none; padding: 0; }
form div { margin-bottom: 10px; }
label { display: inline-block; width: 150px; }
</style>
