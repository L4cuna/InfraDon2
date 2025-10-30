<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

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

// === Référence à la base et données ===
const storage = ref<PouchDB.Database>();
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

// === Initialisation de la base ===
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:admin@localhost:5984/firstdbinfradon2');
  if (db) {
    console.log('✅ Connecté à la base :', db.name);
    storage.value = db;
  } else {
    console.warn('❌ Échec de la connexion à la base de données');
  }
};

// === Récupération de tous les documents ===
const fetchData = async () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée');
    return;
  }

  try {
    const result = await storage.value.allDocs({ include_docs: true });
    postsData.value = result.rows.map(row => row.doc);
    console.log('Documents récupérés :', postsData.value);
  } catch (error) {
    console.error('Erreur lors de la récupération des données :', error);
  }
};

// === Fonction pour créer un nouveau document ===
const createPost = async (newPost: NewPost) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée');
    return;
  }

  try {
    const languagesArray = newPost.languages.split(',').map(lang => lang.trim());
    const doc = {
      ...newPost,
      languages: languagesArray,
      _id: `country_${Date.now()}`
    };
    const response = await storage.value.post(doc);
    console.log('Document créé avec succès :', response);
    await fetchData();
  } catch (error) {
    console.error('Erreur lors de la création du document :', error);
  }
};

// === Fonction pour modifier un document ===
const updatePost = async (post: Post) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée');
    return;
  }

  try {
    const doc = await storage.value.get(post._id);
    const updatedDoc = { ...doc, ...post };
    const response = await storage.value.put(updatedDoc);
    console.log('Document mis à jour avec succès :', response);
    await fetchData();
  } catch (error) {
    console.error('Erreur lors de la mise à jour du document :', error);
  }
};

// === Fonction pour supprimer un document ===
const deletePost = async (post: Post) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée');
    return;
  }

  try {
    const doc = await storage.value.get(post._id);
    const response = await storage.value.remove(doc);
    console.log('Document supprimé avec succès :', response);
    await fetchData();
  } catch (error) {
    console.error('Erreur lors de la suppression du document :', error);
  }
};

// === Fonction pour gérer la soumission du formulaire ===
const handleSubmit = () => {
  const languagesArray = newPost.value.languages.split(',').map(lang => lang.trim());
  const postToCreate = {
    ...newPost.value,
    languages: languagesArray
  };
  createPost(postToCreate);
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

// === Fonction pour initialiser l'édition ===
const editPost = (post: Post) => {
  const languagesString = Array.isArray(post.languages) ? post.languages.join(', ') : post.languages;
  editingPost.value = { ...post, languages: languagesString };
};

// === Fonction pour annuler l'édition ===
const cancelEdit = () => {
  editingPost.value = null;
};

// === Fonction pour soumettre les modifications ===
const updateSubmit = () => {
  if (!editingPost.value) return;

  const languagesArray = editingPost.value.languages.toString().split(',').map(lang => lang.trim());
  const updatedPost = {
    ...editingPost.value,
    languages: languagesArray
  };

  updatePost(updatedPost);
  cancelEdit();
};

// === Montage du composant ===
onMounted(() => {
  console.log('🚀 Composant initialisé');
  initDatabase();
  fetchData();
});
</script>

<template>
  <h1>Fetch Data</h1>
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

  <!-- Ajoute un formulaire pour modifier un pays -->
  <section v-if="editingPost">
    <h2>Modifier un pays</h2>
    <form @submit.prevent="updateSubmit">
      <div>
        <label for="editName">Nom :</label>
        <input type="text" id="editName" v-model="editingPost.name" required />
      </div>
      <div>
        <label for="editCapital">Capitale :</label>
        <input type="text" id="editCapital" v-model="editingPost.capital" required />
      </div>
      <div>
        <label for="editPopulation">Population :</label>
        <input type="number" id="editPopulation" v-model.number="editingPost.population" required />
      </div>
      <div>
        <label for="editArea">Superficie :</label>
        <input type="number" id="editArea" v-model.number="editingPost.area" required />
      </div>
      <div>
        <label for="editCurrency">Monnaie :</label>
        <input type="text" id="editCurrency" v-model="editingPost.currency" required />
      </div>
      <div>
        <label for="editLanguages">Langues (séparées par des virgules) :</label>
        <input type="text" id="editLanguages" v-model="editingPost.languages" required />
      </div>
      <div>
        <label for="editRegion">Région :</label>
        <input type="text" id="editRegion" v-model="editingPost.region" required />
      </div>
      <div>
        <label for="editSubregion">Sous-région :</label>
        <input type="text" id="editSubregion" v-model="editingPost.subregion" required />
      </div>
      <div>
        <label for="editFlag">URL du drapeau :</label>
        <input type="url" id="editFlag" v-model="editingPost.flag" required />
      </div>
      <button type="submit">Enregistrer les modifications</button>
      <button type="button" @click="cancelEdit">Annuler</button>
    </form>
  </section>

  <!-- Ajoute le formulaire pour créer un pays -->
  <section>
    <h2>Ajouter un pays</h2>
    <form @submit.prevent="handleSubmit">
      <div>
        <label for="name">Nom :</label>
        <input type="text" id="name" v-model="newPost.name" required />
      </div>
      <div>
        <label for="capital">Capitale :</label>
        <input type="text" id="capital" v-model="newPost.capital" required />
      </div>
      <div>
        <label for="population">Population :</label>
        <input type="number" id="population" v-model.number="newPost.population" required />
      </div>
      <div>
        <label for="area">Superficie :</label>
        <input type="number" id="area" v-model.number="newPost.area" required />
      </div>
      <div>
        <label for="currency">Monnaie :</label>
        <input type="text" id="currency" v-model="newPost.currency" required />
      </div>
      <div>
        <label for="languages">Langues (séparées par des virgules) :</label>
        <input type="text" id="languages" v-model="newPost.languages" required />
      </div>
      <div>
        <label for="region">Région :</label>
        <input type="text" id="region" v-model="newPost.region" required />
      </div>
      <div>
        <label for="subregion">Sous-région :</label>
        <input type="text" id="subregion" v-model="newPost.subregion" required />
      </div>
      <div>
        <label for="flag">URL du drapeau :</label>
        <input type="url" id="flag" v-model="newPost.flag" required />
      </div>
      <button type="submit">Ajouter le pays</button>
    </form>
  </section>
</template>

<style scoped>
ul {
  list-style: none;
  padding: 0;
}
form div {
  margin-bottom: 10px;
}
label {
  display: inline-block;
  width: 150px;
}
</style>
