<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb'
 
declare interface Post {
  _id: string;
  _rev: string;
  post_name: string;
  post_content: string;
  attributes: string[];
  }
 
 
// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])
 
// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:Ilovecash1925@localhost:5984/infrandon2_db1')
  if (db) {
    console.log("Connecté à la collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}
 
// Récupération des données
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
 
onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
  fetchData()
});
 
</script>
 
<template>
  <h1>Fetch Data</h1>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
    <p>{{ post.attributes.join(',') }}</p>
  </article>
</template>
 
 