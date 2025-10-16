<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// 📌 Structure du document
declare interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: string[]
}

// Référence à la base
const storage = ref<any>(null)
const postsData = ref<Post[]>([])

// 📋 Formulaire d'ajout
const newPost = ref<Post>({
  post_name: '',
  post_content: '',
  attributes: []
})

// ✅ Connexion à la base
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:170451@localhost:5984/infradon2-eko')
  if (db) {
    console.log('Connecté à la collection : ' + db.name)
    storage.value = db
  } else {
    console.warn('Échec lors de la connexion à la base de données')
  }
}

// 📥 Récupération des données
const fetchData = async () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    postsData.value = result.rows.map((row: any) => row.doc)
    console.log('📥 Documents récupérés :', postsData.value)
  } catch (error) {
    console.error('❌ Erreur lors de la récupération des données :', error)
  }
}

// ➕ Ajout d'un nouveau document dans la base
const addPost = async () => {
  if (!storage.value) return

  // Création d'un ID unique (timestamp simple ici)
  const doc = {
    _id: new Date().toISOString(),
    post_name: newPost.value.post_name,
    post_content: newPost.value.post_content,
    attributes: newPost.value.attributes
  }

  try {
    await storage.value.put(doc)
    console.log('✅ Document ajouté :', doc)

    // Vide le formulaire
    newPost.value.post_name = ''
    newPost.value.post_content = ''
    newPost.value.attributes = []

    // Rafraîchit la liste
    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de l’ajout :', error)
  }
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
</script>

<template>
  <div class="container">
    <h1>📡 CouchDB + Vue 3</h1>

    <!-- 📝 Formulaire d'ajout -->
    <div class="form">
      <h2>Ajouter une personne</h2>
      <input
        v-model="newPost.post_name"
        placeholder="Nom"
        type="text"
      />
      <input
        v-model="newPost.post_content"
        placeholder="Contenu / Description"
        type="text"
      />
      <input
        v-model="newPost.attributes"
        placeholder="Attributs séparés par des virgules"
        type="text"
        @input="newPost.attributes = ($event.target as HTMLInputElement).value.split(',')"
      />
      <button @click="addPost">Ajouter</button>
    </div>

    <hr />

    <!-- 📃 Liste des documents -->
    <div v-if="postsData.length === 0">
      <p>Aucune donnée trouvée.</p>
    </div>

    <article
      v-for="post in postsData"
      :key="post._id"
      class="item"
    >
      <h2>{{ post.post_name }}</h2>
      <p>{{ post.post_content }}</p>
      <p>Attributs : {{ post.attributes.join(', ') }}</p>
    </article>
  </div>
</template>

<style scoped>
.container {
  padding: 1.5rem;
  color: white;
  max-width: 600px;
  margin: auto;
}
.form {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}
input {
  padding: 0.5rem;
  border-radius: 4px;
  border: none;
}
button {
  background: #42b883;
  color: white;
  padding: 0.5rem;
  border: none;
  cursor: pointer;
  border-radius: 4px;
}
button:hover {
  background: #2a9d6e;
}
.item {
  background: #1e1e1e;
  padding: 1rem;
  margin-top: 0.5rem;
  border-radius: 6px;
}
</style>
