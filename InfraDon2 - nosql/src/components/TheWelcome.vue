<script setup lang="ts">
import { ref, onMounted } from "vue"
import PouchDB from "pouchdb"
import PouchDBFind from "pouchdb-find"

// Plugin PouchDB-find
PouchDB.plugin(PouchDBFind)

// ------------------ TYPES ------------------
interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: string[]
  created_at?: string
  updated_at?: string
}

// ------------------ STATE ------------------
const storage = ref<any>(null)
const postsData = ref<Post[]>([])
const searchTerm = ref("")
const online = ref(true)

const syncing = ref(false)

const newPost = ref<Post>({
  post_name: "",
  post_content: "",
  attributes: []
})

const isEditing = ref(false)
const selectedPost = ref<Post | null>(null)

// URL CouchDB
const remoteCouch = "http://admin:170451@localhost:5984/infradon2-eko"

let syncHandler: any = null

// ------------------ INIT DB ------------------
const initDatabase = async () => {
  const db = new PouchDB("infradon2-eko")
  storage.value = db

  // INDEXATION
  await db.createIndex({
    index: { fields: ["post_name"] }
  })
  console.log("🔍 Index 'post_name' OK")

  // LIVE SYNC si online
  if (online.value) startLiveSync()
}

// ------------------ LIVE SYNC ------------------
const startLiveSync = () => {
  console.log("🌍 LIVE SYNC ACTIVÉ")
  syncing.value = true

  syncHandler = storage.value
    .sync(remoteCouch, { live: true, retry: true })
    .on("change", fetchData)
    .on("paused", () => console.log("⏸️ Pause sync"))
    .on("active", () => console.log("▶️ Sync reprise"))
    .on("error", (err: any) => console.error("❌ Sync error", err))
}

const stopLiveSync = () => {
  console.log("🛑 LIVE SYNC STOP")
  syncing.value = false
  syncHandler?.cancel()
}

// ------------------ ONLINE/OFFLINE ------------------
const toggleOnline = async () => {
  online.value = !online.value

  if (online.value) {
    startLiveSync()
    await manualSync()
  } else {
    stopLiveSync()
  }
}

// ------------------ FETCH ------------------
const fetchData = async () => {
  const result = await storage.value.allDocs({ include_docs: true })

  postsData.value = result.rows
    .map((r: any) => r.doc)
    .sort((a: any, b: any) => new Date(b.created_at) - new Date(a.created_at))
}

// ------------------ SEARCH (INDEXED) ------------------
const searchByName = async () => {
  if (searchTerm.value.trim() === "") {
    return fetchData()
  }

  const selector =
    searchTerm.value.length < 3
      ? { post_name: searchTerm.value } // Exact (cours)
      : { post_name: { $regex: `^${searchTerm.value}` } } // Partiel

  const result = await storage.value.find({ selector })

  postsData.value = result.docs
}

// ------------------ FACTORY ------------------
const generateFake = async (n = 50) => {
  const docs = []
  const base = Date.now()

  for (let i = 0; i < n; i++) {
    docs.push({
      _id: `fake_${base}_${i}`,
      post_name: `Personne ${i}`,
      post_content: `Contenu généré automatiquement`,
      attributes: ["fake", `groupe_${i}`],
      created_at: new Date().toISOString()
    })
  }

  await storage.value.bulkDocs(docs)
  fetchData()
  console.log(`🧪 ${n} documents générés`)
}

// ------------------ CRUD ------------------
const addPost = async () => {
  const doc = {
    _id: new Date().toISOString(),
    post_name: newPost.value.post_name,
    post_content: newPost.value.post_content,
    attributes: newPost.value.attributes,
    created_at: new Date().toISOString()
  }

  await storage.value.put(doc)
  resetForm()
  fetchData()
}

const selectPost = (post: Post) => {
  isEditing.value = true
  selectedPost.value = post

  newPost.value = {
    post_name: post.post_name,
    post_content: post.post_content,
    attributes: [...post.attributes]
  }
}

const updatePost = async () => {
  if (!selectedPost.value) return

  const doc = {
    _id: selectedPost.value._id,
    _rev: selectedPost.value._rev,
    post_name: newPost.value.post_name,
    post_content: newPost.value.post_content,
    attributes: newPost.value.attributes,
    created_at: selectedPost.value.created_at,
    updated_at: new Date().toISOString()
  }

  await storage.value.put(doc)
  resetForm()
  fetchData()
}

const deletePost = async (post: Post) => {
  await storage.value.remove(post._id, post._rev)
  fetchData()
}

const resetForm = () => {
  isEditing.value = false
  selectedPost.value = null
  newPost.value = { post_name: "", post_content: "", attributes: [] }
}

const handleSubmit = () => {
  isEditing.value ? updatePost() : addPost()
}

// ------------------ REPLICATION MANUELLE ------------------
const replicateFromDistant = async () => {
  await storage.value.replicate.from(remoteCouch)
  fetchData()
}

const replicateToDistant = async () => {
  await storage.value.replicate.to(remoteCouch)
}

const manualSync = async () => {
  await replicateFromDistant()
  await replicateToDistant()
}

// ------------------ MOUNT ------------------
onMounted(async () => {
  await initDatabase()
  fetchData()
})
</script>

<template>
  <div class="app">

    <h1>📡 CouchDB + Vue 3 — CRUD + Index + Recherche + Sync + Factory</h1>

    <!-- ONLINE/OFFLINE -->
    <section class="status-bar">
      <span :class="online ? 'online' : 'offline'">
        ● {{ online ? "ONLINE (sync actif)" : "OFFLINE (local uniquement)" }}
      </span>

      <button @click="toggleOnline" class="secondary small">
        Passer en mode {{ online ? 'OFFLINE' : 'ONLINE' }}
      </button>
    </section>

    <!-- 🔍 Recherche indexée -->
    <section class="search-bar">
      <input
        v-model="searchTerm"
        @input="searchByName"
        placeholder="Rechercher par nom..."
      />
      <button @click="generateFake(50)">+50 docs</button>
    </section>

    <!-- 🔁 Sync -->
    <div class="sync-buttons">
      <button class="sync-btn" @click="replicateFromDistant">⬇️ Distant → Local</button>
      <button class="sync-btn" @click="replicateToDistant">⬆️ Local → Distant</button>
      <button class="sync-btn" @click="manualSync">🔁 Sync 2 Sens</button>
    </div>

    <!-- FORM -->
    <section class="card">
      <h2>{{ isEditing ? "✏ Modifier" : "➕ Ajouter" }} une personne</h2>

      <input v-model="newPost.post_name" placeholder="Nom" />
      <input v-model="newPost.post_content" placeholder="Contenu" />
      <input
        v-model="newPost.attributes"
        placeholder="Attributs (séparés par virgule)"
        @input="newPost.attributes = $event.target.value.split(',')"
      />

      <div class="btn-row">
        <button @click="handleSubmit" class="primary">
          {{ isEditing ? "Enregistrer" : "Créer" }}
        </button>

        <button v-if="isEditing" @click="resetForm" class="secondary">
          Annuler
        </button>
      </div>
    </section>

    <!-- LISTE -->
    <section class="card">
      <h2>📃 Documents</h2>

      <p v-if="postsData.length === 0">Aucun document.</p>

      <article v-for="post in postsData" :key="post._id" class="doc">
        <div>
          <h3>{{ post.post_name }}</h3>
          <p>{{ post.post_content }}</p>
          <small>ID: {{ post._id }}</small><br />
          <small v-if="post.updated_at">🔄 Maj: {{ post.updated_at }}</small>
        </div>

        <div class="btn-row">
          <button @click="selectPost(post)" class="primary small">Modifier</button>
          <button @click="deletePost(post)" class="danger small">Supprimer</button>
        </div>
      </article>
    </section>

  </div>
</template>

<style scoped>
/* === GLOBAL === */
.app {
  padding: 2rem;
  max-width: 700px;
  margin: auto;
  color: #fff;
  font-family: system-ui;
}

h1, h2 { margin-bottom: 1rem; }

/* === STATUS BAR === */
.status-bar {
  display: flex;
  justify-content: space-between;
  margin-bottom: 1rem;
  align-items: center;
}

.online { color: #42b883; }
.offline { color: #ff4d4d; }

/* === SEARCH BAR === */
.search-bar {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.search-bar input {
  flex: 1;
  padding: 0.6rem;
  background: #111;
  border: 1px solid #444;
  border-radius: 6px;
  color: white;
}

/* === BUTTONS SYNC === */
.sync-buttons {
  display: flex;
  overflow: hidden;
  border-radius: 8px;
  margin-bottom: 1rem;
}

.sync-btn {
  background: #d6d6d6;
  color: #111;
  padding: 0.7rem 1.2rem;
  border-right: 1px solid #aaa;
  font-weight: 600;
}
.sync-btn:last-child { border-right: none; }

/* === CARDS === */
.card {
  background: #1e1e1e;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1.5rem;
  border: 1px solid #333;
}

/* === INPUTS === */
input {
  width: 100%;
  padding: 0.6rem;
  margin-bottom: 0.8rem;
  border-radius: 6px;
  background: #111;
  color: white;
  border: 1px solid #444;
}

/* === BUTTONS === */
button {
  cursor: pointer;
  border: none;
  border-radius: 6px;
  padding: 0.6rem 1rem;
  font-weight: 500;
}

/* === STYLES DES BOUTONS === */
.primary { background: #42b883; }
.primary:hover { background: #2a9d6e; }

.secondary { background: #666; }
.secondary:hover { background: #555; }

.danger { background: #ef4444; }
.danger:hover { background: #dc2626; }

.small { font-size: 0.8rem; padding: 0.4rem 0.7rem; }

.btn-row { display: flex; gap: 0.5rem; }

/* === DOC LIST === */
.doc {
  background: #2b2b2b;
  padding: 1rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  border: 1px solid #333;
  display: flex;
  justify-content: space-between;
}
</style>
