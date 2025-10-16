<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// === Interfaces adaptées à ton JSON CouchDB ===
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

// --- Référence à la base ---
const storage = ref()

// --- Données : tableau de documents CouchDB ---
const postsData = ref<Post[]>([])

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
</template>

<style scoped>
ul {
  list-style: none;
  padding: 0;
}
</style>
