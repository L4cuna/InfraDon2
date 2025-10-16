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
  languages: string[];
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
const storage = ref()
const postsData = ref<Post[]>([])

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

// === Initialisation de la base ===
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:admin@localhost:5984/firstdbinfradon2')
  if (db) {
    console.log('✅ Connecté à la base :', db.name)
    storage.value = db
  } else {
    console.warn('❌ Échec de la connexion à la base de données')
  }
}

// === Récupération de tous les documents ===
const fetchData = async () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    postsData.value = result.rows.map(row => row.doc)
    console.log('Documents récupérés :', postsData.value)
  } catch (error) {
    console.error('Erreur lors de la récupération des données :', error)
  }
}

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

// === Fonction pour gérer la soumission du formulaire ===
const handleSubmit = () => {
  createPost(newPost.value);

  // Réinitialiser le formulaire
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

// === Montage du composant ===
onMounted(() => {
  console.log('🚀 Composant initialisé')
  initDatabase()
  fetchData()
})
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
    <p>Languages: {{ post.languages.join(', ') }}</p>
  </article>

  <section>
    <h2>Ajouter un pays</h2>
    <form @submit.prevent="handleSubmit">
      <div>
        <label for="name">Name :</label>
        <input type="text" id="name" v-model="newPost.name" required />
      </div>
      <div>
        <label for="capital">Capital :</label>
        <input type="text" id="capital" v-model="newPost.capital" required />
      </div>
      <div>
        <label for="population">Population :</label>
        <input type="number" id="population" v-model.number="newPost.population" required />
      </div>
      <div>
        <label for="area">Area :</label>
        <input type="number" id="area" v-model.number="newPost.area" required />
      </div>
      <div>
        <label for="currency">Currency :</label>
        <input type="text" id="currency" v-model="newPost.currency" required />
      </div>
      <div>
        <label for="languages">Languages :</label>
        <input type="text" id="languages" v-model="newPost.languages" required />
      </div>
      <div>
        <label for="region">Region :</label>
        <input type="text" id="region" v-model="newPost.region" required />
      </div>
      <div>
        <label for="subregion">Sub region :</label>
        <input type="text" id="subregion" v-model="newPost.subregion" required />
      </div>
      <div>
        <label for="flag">URL du drapeau :</label>
        <input type="url" id="flag" v-model="newPost.flag" required />
      </div>
      <button type="submit">Add new country</button>
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
