<script setup lang="ts">
//import WelcomeItem from './WelcomeItem.vue'
//import DocumentationIcon from './icons/IconDocumentation.vue'
//import ToolingIcon from './icons/IconTooling.vue'
//import EcosystemIcon from './icons/IconEcosystem.vue'
//import CommunityIcon from './icons/IconCommunity.vue'
//import SupportIcon from './icons/IconSupport.vue'

//const openReadmeInEditor = () => fetch('/__open-in-editor?file=README.md')

import { ref } from 'vue'
import PouchDB from 'pouchdb'
//DATA VARIABLES
const counter = ref(110);

//FUNCTIONS METHODES CONSTRUCTEURS
function mounted() {
    this.initDatabase()
}

const increment = () => {
  counter.value++
}
</script>

<template>  
  <h1>Welcome</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>
