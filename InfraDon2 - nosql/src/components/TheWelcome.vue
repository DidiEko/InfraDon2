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

// 📋 Formulaire d'ajout/modification
const newPost = ref<Post>({
  post_name: '',
  post_content: '',
  attributes: []
})

// 🎯 Mode édition
const isEditing = ref(false)
const selectedPost = ref<Post | null>(null)

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

  const doc = {
    _id: new Date().toISOString(),
    post_name: newPost.value.post_name,
    post_content: newPost.value.post_content,
    attributes: newPost.value.attributes
  }

  try {
    await storage.value.put(doc)
    console.log('✅ Document ajouté :', doc)
    resetForm()
    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de lajout :', error)
  }
}

// 🎯 Sélectionner un document pour modification
const selectPost = (post: Post) => {
  isEditing.value = true
  selectedPost.value = post
  newPost.value = {
    post_name: post.post_name,
    post_content: post.post_content,
    attributes: [...post.attributes]
  }
  console.log('🎯 Document sélectionné pour modification :', post)
}

// ✏️ Modifier un document existant
const updatePost = async () => {
  if (!storage.value || !selectedPost.value) return

  const doc = {
    _id: selectedPost.value._id,
    _rev: selectedPost.value._rev,
    post_name: newPost.value.post_name,
    post_content: newPost.value.post_content,
    attributes: newPost.value.attributes
  }

  try {
    await storage.value.put(doc)
    console.log('✏️ Document modifié :', doc)
    resetForm()
    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de la modification :', error)
  }
}

// 🗑️ Supprimer un document
const deletePost = async (post: Post) => {
  if (!storage.value || !confirm('Êtes-vous sûr de vouloir supprimer ce document ?')) return

  try {
    await storage.value.remove(post._id, post._rev)
    console.log('🗑️ Document supprimé :', post)
    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de la suppression :', error)
  }
}

// 🔄 Réinitialiser le formulaire
const resetForm = () => {
  newPost.value = {
    post_name: '',
    post_content: '',
    attributes: []
  }
  isEditing.value = false
  selectedPost.value = null
}

// 📤 Gestion de la soumission (ajouter ou modifier)
const handleSubmit = () => {
  if (isEditing.value) {
    updatePost()
  } else {
    addPost()
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
    <h1>📡 CouchDB + Vue 3 - CRUD Complet</h1>

    <!-- 📝 Formulaire d'ajout/modification -->
    <div class="form">
      <h2>{{ isEditing ? '✏️ Modifier' : '➕ Ajouter' }} une personne</h2>
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
      
      <div class="button-group">
        <button @click="handleSubmit" class="btn-primary">
          {{ isEditing ? '✏️ Modifier' : '➕ Ajouter' }}
        </button>
        <button v-if="isEditing" @click="resetForm" class="btn-secondary">
          ❌ Annuler
        </button>
      </div>
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
      :class="{ selected: selectedPost?._id === post._id }"
    >
      <h2>{{ post.post_name }}</h2>
      <p>{{ post.post_content }}</p>
      <p>Attributs : {{ post.attributes.join(', ') }}</p>
      
      <div class="actions">
        <button @click="selectPost(post)" class="btn-edit">
          ✏️ Modifier
        </button>
        <button @click="deletePost(post)" class="btn-delete">
          🗑️ Supprimer
        </button>
      </div>
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
.button-group {
  display: flex;
  gap: 0.5rem;
}
button {
  padding: 0.5rem 1rem;
  border: none;
  cursor: pointer;
  border-radius: 4px;
  font-weight: 500;
}
.btn-primary {
  background: #42b883;
  color: white;
  flex: 1;
}
.btn-primary:hover {
  background: #2a9d6e;
}
.btn-secondary {
  background: #666;
  color: white;
}
.btn-secondary:hover {
  background: #555;
}
.item {
  background: #1e1e1e;
  padding: 1rem;
  margin-top: 0.5rem;
  border-radius: 6px;
  border: 2px solid transparent;
  transition: all 0.2s;
}
.item.selected {
  border-color: #42b883;
  background: #252525;
}
.actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
}
.btn-edit {
  background: #3b82f6;
  color: white;
  flex: 1;
}
.btn-edit:hover {
  background: #2563eb;
}
.btn-delete {
  background: #ef4444;
  color: white;
  flex: 1;
}
.btn-delete:hover {
  background: #dc2626;
}
</style>