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

// 📂 Références globales
const storage = ref<any>(null)
const postsData = ref<Post[]>([])

// 📝 Formulaire d'ajout/modification
const newPost = ref<Post>({
  post_name: '',
  post_content: '',
  attributes: []
})

// 🎯 Mode édition
const isEditing = ref(false)
const selectedPost = ref<Post | null>(null)

// 🌍 URL serveur CouchDB
const remoteCouch = 'http://admin:170451@localhost:5984/infradon2-eko'

// ✅ Initialisation de la base et synchronisation live
const initDatabase = () => {
  console.log('=> Initialisation de la base locale PouchDB')

  const db = new PouchDB('infradon2-eko')
  storage.value = db

  console.log('📦 Base locale prête : infradon2-eko')

  // 🔄 Synchronisation automatique bidirectionnelle
  db.sync(remoteCouch, { live: true, retry: true })
    .on('change', info => {
      console.log('🟢 Changement détecté (sync auto) :', info)
      fetchData()
    })
    .on('paused', err => console.log('⏸️ Synchro en pause', err || ''))
    .on('active', () => console.log('▶️ Synchro reprise'))
    .on('error', err => console.error('❌ Erreur de synchronisation :', err))

  console.log('🌍 Synchronisation CouchDB ↔️ PouchDB activée')
}

// 📥 Récupération des données locales
const fetchData = async () => {
  if (!storage.value) return console.warn('Base non initialisée')

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    postsData.value = result.rows.map((row: any) => row.doc)
    console.log('📥 Documents récupérés :', postsData.value)
  } catch (error) {
    console.error('❌ Erreur lors de la récupération :', error)
  }
}

// 🔁 Réplication DISTANT → LOCAL (bouton)
const replicateFromDistant = async () => {
  if (!storage.value) return

  console.log('⬇️ Réplication DISTANT → LOCAL...')
  try {
    const result = await storage.value.replicate.from(remoteCouch)
    console.log('✅ Réplication depuis serveur :', result)
    fetchData()
  } catch (err) {
    console.error('❌ Erreur replicateFromDistant :', err)
  }
}

// 🔁 Réplication LOCAL → DISTANT (bouton)
const replicateToDistant = async () => {
  if (!storage.value) return

  console.log('⬆️ Réplication LOCAL → DISTANT...')
  try {
    const result = await storage.value.replicate.to(remoteCouch)
    console.log('✅ Réplication vers serveur :', result)
  } catch (err) {
    console.error('❌ Erreur replicateToDistant :', err)
  }
}

// 👂 Watch des changements DISTANTS (serveur CouchDB)
const watchDistantChanges = () => {
  const remote = new PouchDB(remoteCouch)

  remote
    .changes({
      since: 'now',
      live: true,
      include_docs: true
    })
    .on('change', info => {
      console.log('🌍 Changement distant détecté :', info)
      fetchData()
    })
    .on('error', err => console.error('❌ Erreur watch distant :', err))
}

// ➕ Ajout d’un document
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
    fetchData()
  } catch (error) {
    console.error('❌ Erreur ajout :', error)
  }
}

// 🎯 Sélection pour modification
const selectPost = (post: Post) => {
  isEditing.value = true
  selectedPost.value = post

  newPost.value = {
    post_name: post.post_name,
    post_content: post.post_content,
    attributes: [...post.attributes]
  }

  console.log('🎯 Document en édition :', post)
}

// ✏️ Modification d’un document
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
    fetchData()
  } catch (error) {
    console.error('❌ Erreur modification :', error)
  }
}

// 🗑️ Suppression d’un document
const deletePost = async (post: Post) => {
  if (!storage.value || !confirm('Supprimer ce document ?')) return

  try {
    await storage.value.remov
