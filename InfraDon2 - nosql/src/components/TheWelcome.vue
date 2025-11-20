<script setup lang="ts">
import { ref, onMounted } from "vue"
import PouchDB from "pouchdb"
import PouchDBFind from "pouchdb-find"

// Plugin PouchDB-find
PouchDB.plugin(PouchDBFind)

// ------------------ TYPES ------------------
interface Comment {
  text: string
  date: string
}

interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: string[]
  likes?: number
  comments?: Comment[]
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
  attributes: [],
  likes: 0,
  comments: []
})

const isEditing = ref(false)
const selectedPost = ref<Post | null>(null)

// Nouveaux états pour les commentaires
const newComment = ref<{ [key: string]: string }>({})
const showComments = ref<{ [key: string]: boolean }>({})

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

  // Écouter les changements locaux en temps réel
  listenLocalChanges()

  // LIVE SYNC si online
  if (online.value) startLiveSync()
}

// ------------------ LIVE CHANGES ------------------
const listenLocalChanges = () => {
  storage.value.changes({
    since: "now",
    live: true,
    include_docs: true
  }).on("change", () => {
    console.log("🔄 Changement local détecté")
    fetchData()
  })
}

// ------------------ LIVE SYNC ------------------
const startLiveSync = () => {
  console.log("🌍 LIVE SYNC ACTIVÉ")
  syncing.value = true

  syncHandler = storage.value
    .sync(remoteCouch, { live: true, retry: true })
    .on("change", (info: any) => {
      console.log("🔄 Sync change:", info)
      fetchData()
    })
    .on("paused", () => {
      console.log("⏸️ Pause sync")
      online.value = false
    })
    .on("active", () => {
      console.log("▶️ Sync reprise")
      online.value = true
    })
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
    .filter((doc: any) => !doc._id.startsWith('_design'))
    .sort((a: any, b: any) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
}

// ------------------ SEARCH (INDEXED) ------------------
const searchByName = async () => {
  if (searchTerm.value.trim() === "") {
    return fetchData()
  }

  const result = await storage.value.find({
    selector: {
      post_name: { $regex: searchTerm.value }
    }
  })

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
      post_content: `Contenu généré automatiquement ${Math.random().toFixed(3)}`,
      attributes: ["fake", `groupe_${i % 5}`],
      likes: Math.floor(Math.random() * 50),
      comments: [],
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
    likes: 0,
    comments: [],
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
    attributes: [...post.attributes],
    likes: post.likes || 0,
    comments: post.comments || []
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
    likes: selectedPost.value.likes || 0,
    comments: selectedPost.value.comments || [],
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
  newPost.value = { post_name: "", post_content: "", attributes: [], likes: 0, comments: [] }
}

const handleSubmit = () => {
  isEditing.value ? updatePost() : addPost()
}

// ------------------ LIKES ------------------
const likePost = async (post: Post) => {
  const updatedPost = {
    ...post,
    likes: (post.likes || 0) + 1
  }
  await storage.value.put(updatedPost)
  fetchData()
}

// ------------------ COMMENTS ------------------
const toggleComments = (postId: string) => {
  showComments.value[postId] = !showComments.value[postId]
}

const addComment = async (post: Post) => {
  const commentText = newComment.value[post._id!]
  if (!commentText || !commentText.trim()) return

  const updatedPost = {
    ...post,
    comments: [
      ...(post.comments || []),
      {
        text: commentText,
        date: new Date().toISOString()
      }
    ]
  }

  await storage.value.put(updatedPost)
  newComment.value[post._id!] = ""
  fetchData()
}

// ------------------ REPLICATION MANUELLE ------------------
const replicateFromDistant = async () => {
  await storage.value.replicate.from(remoteCouch)
  fetchData()
  console.log("⬇️ Synchronisation distante → locale OK")
}

const replicateToDistant = async () => {
  await storage.value.replicate.to(remoteCouch)
  console.log("⬆️ Synchronisation locale → distante OK")
}

const manualSync = async () => {
  await replicateFromDistant()
  await replicateToDistant()
  console.log("🔁 Synchronisation bidirectionnelle OK")
}

// ------------------ MOUNT ------------------
onMounted(async () => {
  await initDatabase()
  fetchData()
})
</script>

<template>
  <div class="app">

    <h1>📡 CouchDB + Vue 3 — Version Complète</h1>

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
        :value="newPost.attributes.join(',')"
        placeholder="Attributs (séparés par virgule)"
        @input="newPost.attributes = ($event.target as HTMLInputElement).value.split(',').map(s => s.trim()).filter(s => s)"
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
      <h2>📃 Documents ({{ postsData.length }})</h2>

      <p v-if="postsData.length === 0">Aucun document.</p>

      <article v-for="post in postsData" :key="post._id" class="doc">
        <div class="doc-content">
          <h3>{{ post.post_name }}</h3>
          <p>{{ post.post_content }}</p>
          
          <div class="attributes" v-if="post.attributes && post.attributes.length">
            <span v-for="attr in post.attributes" :key="attr" class="badge">{{ attr }}</span>
          </div>

          <div class="meta">
            <small>ID: {{ post._id }}</small>
            <small v-if="post.updated_at">🔄 Maj: {{ new Date(post.updated_at).toLocaleString() }}</small>
          </div>

          <!-- LIKES & COMMENTS -->
          <div class="interactions">
            <button @click="likePost(post)" class="like-btn">
              👍 {{ post.likes || 0 }}
            </button>
            <button @click="toggleComments(post._id!)" class="comment-toggle">
              💬 {{ (post.comments || []).length }} commentaires
            </button>
          </div>

          <!-- SECTION COMMENTAIRES -->
          <div v-if="showComments[post._id!]" class="comments-section">
            <div v-if="post.comments && post.comments.length" class="comments-list">
              <div v-for="(comment, idx) in post.comments" :key="idx" class="comment">
                <p>{{ comment.text }}</p>
                <small>{{ new Date(comment.date).toLocaleString() }}</small>
              </div>
            </div>
            <div class="add-comment">
              <input 
                v-model="newComment[post._id!]"
                placeholder="Ajouter un commentaire..."
                @keyup.enter="addComment(post)"
              />
              <button @click="addComment(post)" class="primary small">Envoyer</button>
            </div>
          </div>
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
  max-width: 800px;
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
  padding: 1rem;
  background: #1e1e1e;
  border-radius: 8px;
  border: 1px solid #333;
}

.online { color: #42b883; font-weight: bold; }
.offline { color: #ff4d4d; font-weight: bold; }

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
  flex: 1;
  background: #d6d6d6;
  color: #111;
  padding: 0.7rem 1.2rem;
  border-right: 1px solid #aaa;
  font-weight: 600;
  border: none;
  cursor: pointer;
  transition: background 0.2s;
}
.sync-btn:hover { background: #c0c0c0; }
.sync-btn:last-child { border-right: none; }

/* === CARDS === */
.card {
  background: #1e1e1e;
  padding: 1.5rem;
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
  transition: all 0.2s;
}

/* === STYLES DES BOUTONS === */
.primary { background: #42b883; color: white; }
.primary:hover { background: #2a9d6e; }

.secondary { background: #666; color: white; }
.secondary:hover { background: #555; }

.danger { background: #ef4444; color: white; }
.danger:hover { background: #dc2626; }

.small { font-size: 0.8rem; padding: 0.4rem 0.7rem; }

.btn-row { display: flex; gap: 0.5rem; flex-wrap: wrap; }

/* === DOC LIST === */
.doc {
  background: #2b2b2b;
  padding: 1.5rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  border: 1px solid #333;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.doc-content {
  flex: 1;
}

.doc h3 {
  margin: 0 0 0.5rem 0;
  color: #42b883;
}

.doc p {
  margin: 0 0 1rem 0;
  line-height: 1.5;
}

/* === ATTRIBUTES === */
.attributes {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-bottom: 0.5rem;
}

.badge {
  background: #42b883;
  color: white;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  font-size: 0.75rem;
  font-weight: 500;
}

/* === META === */
.meta {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  margin-bottom: 1rem;
  opacity: 0.7;
}

/* === INTERACTIONS === */
.interactions {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.like-btn, .comment-toggle {
  background: #333;
  color: white;
  padding: 0.4rem 0.8rem;
  font-size: 0.85rem;
}

.like-btn:hover { background: #42b883; }
.comment-toggle:hover { background: #444; }

/* === COMMENTS === */
.comments-section {
  margin-top: 1rem;
  padding: 1rem;
  background: #1e1e1e;
  border-radius: 6px;
  border-left: 3px solid #42b883;
}

.comments-list {
  margin-bottom: 1rem;
}

.comment {
  background: #2b2b2b;
  padding: 0.75rem;
  border-radius: 4px;
  margin-bottom: 0.5rem;
}

.comment p {
  margin: 0 0 0.25rem 0;
}

.comment small {
  opacity: 0.6;
}

.add-comment {
  display: flex;
  gap: 0.5rem;
}

.add-comment input {
  margin: 0;
}
</style>