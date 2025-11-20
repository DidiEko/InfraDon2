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
const sortByLikes = ref(false)

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
const editingComment = ref<{ [key: string]: number | null }>({})
const editCommentText = ref<{ [key: string]: string }>({})

// URL CouchDB
const remoteCouch = "http://admin:170451@localhost:5984/infradon2-eko"

let syncHandler: any = null

// ------------------ INIT DB ------------------
const initDatabase = async () => {
  const db = new PouchDB("infradon2-eko")
  storage.value = db

  // INDEXATION sur post_name ET likes pour le tri
  await db.createIndex({
    index: { fields: ["post_name"] }
  })
  
  await db.createIndex({
    index: { fields: ["likes"] }
  })
  
  console.log("🔍 Index 'post_name' et 'likes' créés")

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
  let result
  
  if (sortByLikes.value) {
    // TRI PAR LIKES (décroissant) - Effectué côté base de données
    result = await storage.value.find({
      selector: { likes: { $gte: 0 } },
      sort: [{ likes: "desc" }]
    })
    postsData.value = result.docs.filter((doc: any) => !doc._id.startsWith('_design'))
  } else {
    // Récupération normale
    result = await storage.value.allDocs({ include_docs: true })
    postsData.value = result.rows
      .map((r: any) => r.doc)
      .filter((doc: any) => !doc._id.startsWith('_design'))
      .sort((a: any, b: any) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
  }
}

// ------------------ SEARCH (INDEXED) - CÔTÉ BASE DE DONNÉES ------------------
const searchByName = async () => {
  if (searchTerm.value.trim() === "") {
    return fetchData()
  }

  // Recherche effectuée côté base de données avec l'index
  const result = await storage.value.find({
    selector: {
      post_name: { $regex: `(?i)${searchTerm.value}` }
    }
  })

  postsData.value = result.docs
  console.log(`🔍 Recherche effectuée côté DB: ${result.docs.length} résultats`)
}

// ------------------ TRI PAR LIKES - CÔTÉ BASE DE DONNÉES ------------------
const toggleSortByLikes = async () => {
  sortByLikes.value = !sortByLikes.value
  await fetchData()
  console.log(`📊 Tri par likes ${sortByLikes.value ? 'activé' : 'désactivé'} - effectué côté DB`)
}

// ------------------ FACTORY ------------------
const generateFake = async (n = 50) => {
  const docs = []
  const base = Date.now()

  for (let i = 0; i < n; i++) {
    docs.push({
      _id: `fake_${base}_${i}`,
      post_name: `Message ${i}`,
      post_content: `Contenu du message automatique numéro ${i}. ${Math.random().toFixed(3)}`,
      attributes: ["auto", `groupe_${i % 5}`],
      likes: Math.floor(Math.random() * 100),
      comments: [],
      created_at: new Date(Date.now() - Math.random() * 10000000000).toISOString()
    })
  }

  await storage.value.bulkDocs(docs)
  fetchData()
  console.log(`🧪 ${n} messages générés`)
}

// ------------------ CRUD POSTS ------------------
const addPost = async () => {
  if (!newPost.value.post_name || !newPost.value.post_content) {
    alert("Le nom et le contenu sont obligatoires")
    return
  }

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
  console.log("✅ Message ajouté")
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
  console.log("✅ Message mis à jour")
}

const deletePost = async (post: Post) => {
  if (!confirm("Supprimer ce message ?")) return
  
  await storage.value.remove(post._id, post._rev)
  fetchData()
  console.log("🗑️ Message supprimé")
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
  console.log(`👍 Like ajouté au message "${post.post_name}"`)
}

// ------------------ COMMENTS CRUD ------------------
const toggleComments = (postId: string) => {
  showComments.value[postId] = !showComments.value[postId]
}

const addComment = async (post: Post) => {
  const commentText = newComment.value[post._id!]
  if (!commentText || !commentText.trim()) {
    alert("Le commentaire ne peut pas être vide")
    return
  }

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
  console.log(`💬 Commentaire ajouté au message "${post.post_name}"`)
}

const startEditComment = (postId: string, index: number, text: string) => {
  editingComment.value[postId] = index
  editCommentText.value[postId] = text
}

const cancelEditComment = (postId: string) => {
  editingComment.value[postId] = null
  editCommentText.value[postId] = ""
}

const updateComment = async (post: Post, index: number) => {
  const newText = editCommentText.value[post._id!]
  if (!newText || !newText.trim()) {
    alert("Le commentaire ne peut pas être vide")
    return
  }

  const updatedComments = [...(post.comments || [])]
  updatedComments[index] = {
    ...updatedComments[index],
    text: newText
  }

  const updatedPost = {
    ...post,
    comments: updatedComments,
    updated_at: new Date().toISOString()
  }

  await storage.value.put(updatedPost)
  editingComment.value[post._id!] = null
  editCommentText.value[post._id!] = ""
  fetchData()
  console.log(`✏️ Commentaire modifié`)
}

const deleteComment = async (post: Post, index: number) => {
  if (!confirm("Supprimer ce commentaire ?")) return

  const updatedComments = [...(post.comments || [])]
  updatedComments.splice(index, 1)

  const updatedPost = {
    ...post,
    comments: updatedComments,
    updated_at: new Date().toISOString()
  }

  await storage.value.put(updatedPost)
  fetchData()
  console.log(`🗑️ Commentaire supprimé`)
}

// ------------------ REPLICATION MANUELLE ------------------
const replicateFromDistant = async () => {
  console.log("⬇️ Début réplication distante → locale")
  await storage.value.replicate.from(remoteCouch)
  fetchData()
  console.log("✅ Synchronisation distante → locale terminée")
}

const replicateToDistant = async () => {
  console.log("⬆️ Début réplication locale → distante")
  await storage.value.replicate.to(remoteCouch)
  console.log("✅ Synchronisation locale → distante terminée")
}

const manualSync = async () => {
  console.log("🔁 Synchronisation bidirectionnelle en cours...")
  await replicateFromDistant()
  await replicateToDistant()
  console.log("✅ Synchronisation bidirectionnelle terminée")
}

// ------------------ MOUNT ------------------
onMounted(async () => {
  console.log("🚀 Initialisation de l'application")
  await initDatabase()
  fetchData()
})
</script>

<template>
  <div class="app">

    <h1>📡 CouchDB + Vue 3 — Application de Messagerie</h1>
    <p class="subtitle">Infrastructure de Données 2 - TP Réplication & Indexation</p>

    <!-- ONLINE/OFFLINE -->
    <section class="status-bar">
      <span :class="online ? 'online' : 'offline'">
        ● {{ online ? "ONLINE (sync temps réel actif)" : "OFFLINE (mode local uniquement)" }}
      </span>

      <button @click="toggleOnline" class="secondary small">
        {{ online ? '📴 Mode Offline' : '📡 Mode Online' }}
      </button>
    </section>

    <!-- 🔍 Recherche indexée -->
    <section class="search-bar">
      <input
        v-model="searchTerm"
        @input="searchByName"
        placeholder="🔍 Rechercher un message par nom (indexé)..."
      />
      <button @click="generateFake(50)" class="factory-btn">🧪 +50 messages</button>
      <button 
        @click="toggleSortByLikes" 
        :class="['sort-btn', { active: sortByLikes }]"
      >
        {{ sortByLikes ? '📊 Tri: Likes ↓' : '📅 Tri: Date' }}
      </button>
    </section>

    <!-- 🔁 Sync -->
    <div class="sync-buttons">
      <button class="sync-btn" @click="replicateFromDistant">⬇️ CouchDB → Local</button>
      <button class="sync-btn" @click="replicateToDistant">⬆️ Local → CouchDB</button>
      <button class="sync-btn primary-sync" @click="manualSync">🔁 Sync Bidirectionnelle</button>
    </div>

    <!-- FORM -->
    <section class="card">
      <h2>{{ isEditing ? "✏️ Modifier un message" : "➕ Ajouter un message" }}</h2>

      <input v-model="newPost.post_name" placeholder="Nom du message *" />
      <textarea 
        v-model="newPost.post_content" 
        placeholder="Contenu du message *"
        rows="4"
      ></textarea>
      <input
        :value="newPost.attributes.join(', ')"
        placeholder="Attributs (séparés par virgule)"
        @input="newPost.attributes = ($event.target as HTMLInputElement).value.split(',').map(s => s.trim()).filter(s => s)"
      />

      <div class="btn-row">
        <button @click="handleSubmit" class="primary">
          {{ isEditing ? "💾 Enregistrer" : "➕ Créer" }}
        </button>

        <button v-if="isEditing" @click="resetForm" class="secondary">
          ❌ Annuler
        </button>
      </div>
    </section>

    <!-- LISTE -->
    <section class="card">
      <div class="header-with-count">
        <h2>📃 Messages</h2>
        <span class="count-badge">{{ postsData.length }} message(s)</span>
      </div>

      <p v-if="postsData.length === 0" class="empty-state">
        Aucun message. Ajoutez-en un ou utilisez la factory ! 🧪
      </p>

      <article v-for="post in postsData" :key="post._id" class="doc">
        <div class="doc-content">
          <div class="doc-header">
            <h3>{{ post.post_name }}</h3>
            <div class="likes-display">
              <button @click="likePost(post)" class="like-btn-inline">
                👍 <strong>{{ post.likes || 0 }}</strong>
              </button>
            </div>
          </div>
          
          <p class="content">{{ post.post_content }}</p>
          
          <div class="attributes" v-if="post.attributes && post.attributes.length">
            <span v-for="attr in post.attributes" :key="attr" class="badge">{{ attr }}</span>
          </div>

          <div class="meta">
            <small>🆔 {{ post._id }}</small>
            <small>📅 Créé: {{ new Date(post.created_at!).toLocaleString('fr-FR') }}</small>
            <small v-if="post.updated_at">🔄 Modifié: {{ new Date(post.updated_at).toLocaleString('fr-FR') }}</small>
          </div>

          <!-- INTERACTIONS -->
          <div class="interactions">
            <button @click="toggleComments(post._id!)" class="comment-toggle">
              💬 {{ (post.comments || []).length }} commentaire(s)
            </button>
          </div>

          <!-- SECTION COMMENTAIRES -->
          <div v-if="showComments[post._id!]" class="comments-section">
            <h4>💬 Commentaires</h4>
            
            <div v-if="post.comments && post.comments.length" class="comments-list">
              <div v-for="(comment, idx) in post.comments" :key="idx" class="comment">
                <div v-if="editingComment[post._id!] === idx" class="edit-comment">
                  <textarea 
                    v-model="editCommentText[post._id!]"
                    rows="2"
                  ></textarea>
                  <div class="btn-row">
                    <button @click="updateComment(post, idx)" class="primary small">💾 Sauver</button>
                    <button @click="cancelEditComment(post._id!)" class="secondary small">❌ Annuler</button>
                  </div>
                </div>
                <div v-else>
                  <p>{{ comment.text }}</p>
                  <div class="comment-footer">
                    <small>{{ new Date(comment.date).toLocaleString('fr-FR') }}</small>
                    <div class="comment-actions">
                      <button @click="startEditComment(post._id!, idx, comment.text)" class="icon-btn">✏️</button>
                      <button @click="deleteComment(post, idx)" class="icon-btn danger">🗑️</button>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            
            <div v-else class="no-comments">
              Aucun commentaire pour le moment.
            </div>
            
            <div class="add-comment">
              <input 
                v-model="newComment[post._id!]"
                placeholder="Ajouter un commentaire..."
                @keyup.enter="addComment(post)"
              />
              <button @click="addComment(post)" class="primary small">📤 Envoyer</button>
            </div>
          </div>
        </div>

        <div class="btn-row doc-actions">
          <button @click="selectPost(post)" class="primary small">✏️ Modifier</button>
          <button @click="deletePost(post)" class="danger small">🗑️ Supprimer</button>
        </div>
      </article>
    </section>

  </div>
</template>

<style scoped>
/* === GLOBAL === */
.app {
  padding: 2rem;
  max-width: 900px;
  margin: auto;
  color: #fff;
  font-family: system-ui, -apple-system, sans-serif;
  background: #0a0a0a;
  min-height: 100vh;
}

h1 { 
  margin-bottom: 0.5rem;
  font-size: 1.8rem;
}

.subtitle {
  color: #888;
  margin-bottom: 2rem;
  font-size: 0.9rem;
}

h2 { margin-bottom: 1rem; }

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

.online { 
  color: #42b883; 
  font-weight: bold;
  font-size: 1rem;
}

.offline { 
  color: #ff4d4d; 
  font-weight: bold;
  font-size: 1rem;
}

/* === SEARCH BAR === */
.search-bar {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.search-bar input {
  flex: 1;
  padding: 0.7rem;
  background: #111;
  border: 1px solid #444;
  border-radius: 6px;
  color: white;
  font-size: 0.95rem;
}

.factory-btn {
  background: #6366f1;
  color: white;
  padding: 0.7rem 1rem;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  white-space: nowrap;
}

.factory-btn:hover {
  background: #4f46e5;
}

.sort-btn {
  background: #374151;
  color: white;
  padding: 0.7rem 1rem;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  white-space: nowrap;
  transition: all 0.2s;
}

.sort-btn.active {
  background: #f59e0b;
}

.sort-btn:hover {
  background: #4b5563;
}

.sort-btn.active:hover {
  background: #d97706;
}

/* === BUTTONS SYNC === */
.sync-buttons {
  display: flex;
  overflow: hidden;
  border-radius: 8px;
  margin-bottom: 1rem;
  gap: 1px;
  background: #111;
}

.sync-btn {
  flex: 1;
  background: #2a2a2a;
  color: white;
  padding: 0.8rem 1.2rem;
  font-weight: 600;
  border: none;
  cursor: pointer;
  transition: background 0.2s;
}

.sync-btn:hover { 
  background: #3a3a3a; 
}

.primary-sync {
  background: #42b883;
}

.primary-sync:hover {
  background: #2a9d6e;
}

/* === CARDS === */
.card {
  background: #1e1e1e;
  padding: 1.5rem;
  border-radius: 8px;
  margin-bottom: 1.5rem;
  border: 1px solid #333;
}

.header-with-count {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.count-badge {
  background: #42b883;
  color: white;
  padding: 0.4rem 0.8rem;
  border-radius: 20px;
  font-size: 0.85rem;
  font-weight: 600;
}

.empty-state {
  text-align: center;
  color: #888;
  padding: 2rem;
  font-style: italic;
}

/* === INPUTS === */
input, textarea {
  width: 100%;
  padding: 0.7rem;
  margin-bottom: 0.8rem;
  border-radius: 6px;
  background: #111;
  color: white;
  border: 1px solid #444;
  font-family: inherit;
  font-size: 0.95rem;
}

textarea {
  resize: vertical;
  min-height: 80px;
}

input:focus, textarea:focus {
  outline: none;
  border-color: #42b883;
}

/* === BUTTONS === */
button {
  cursor: pointer;
  border: none;
  border-radius: 6px;
  padding: 0.6rem 1rem;
  font-weight: 500;
  transition: all 0.2s;
  font-size: 0.9rem;
}

.primary { background: #42b883; color: white; }
.primary:hover { background: #2a9d6e; }

.secondary { background: #666; color: white; }
.secondary:hover { background: #555; }

.danger { background: #ef4444; color: white; }
.danger:hover { background: #dc2626; }

.small { font-size: 0.8rem; padding: 0.4rem 0.7rem; }

.btn-row { 
  display: flex; 
  gap: 0.5rem; 
  flex-wrap: wrap; 
}

/* === DOC LIST === */
.doc {
  background: #2b2b2b;
  padding: 1.5rem;
  border-radius: 8px;
  margin-bottom: 1.5rem;
  border: 1px solid #333;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.doc-content {
  flex: 1;
}

.doc-header {
  display: flex;
  justify-content: space-between;
  align-items: start;
  margin-bottom: 0.5rem;
}

.doc h3 {
  margin: 0;
  color: #42b883;
  font-size: 1.3rem;
}

.content {
  margin: 0 0 1rem 0;
  line-height: 1.6;
  color: #ddd;
}

.likes-display {
  display: flex;
  align-items: center;
}

.like-btn-inline {
  background: #374151;
  color: white;
  padding: 0.5rem 1rem;
  font-size: 1rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  border-radius: 20px;
}

.like-btn-inline:hover {
  background: #42b883;
  transform: scale(1.05);
}

/* === ATTRIBUTES === */
.attributes {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-bottom: 0.5rem;
}

.badge {
  background: #4f46e5;
  color: white;
  padding: 0.3rem 0.6rem;
  border-radius: 4px;
  font-size: 0.75rem;
  font-weight: 600;
}

/* === META === */
.meta {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
  margin-bottom: 1rem;
  opacity: 0.7;
  font-size: 0.85rem;
}

/* === INTERACTIONS === */
.interactions {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.comment-toggle {
  background: #333;
  color: white;
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
}

.comment-toggle:hover { 
  background: #444; 
}

.doc-actions {
  border-top: 1px solid #444;
  padding-top: 1rem;
}

/* === COMMENTS === */
.comments-section {
  margin-top: 1rem;
  padding: 1.5rem;
  background: #1a1a1a;
  border-radius: 8px;
  border-left: 4px solid #42b883;
}

.comments-section h4 {
  margin: 0 0 1rem 0;
  color: #42b883;
}

.comments-list {
  margin-bottom: 1rem;
  max-height: 400px;
  overflow-y: auto;
}

.comment {
  background: #2b2b2b;
  padding: 1rem;
  border-radius: 6px;
  margin-bottom: 0.75rem;
  border: 1px solid #333;
}

.comment p {
  margin: 0 0 0.5rem 0;
  line-height: 1.5;
}

.comment-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.comment-actions {
  display: flex;
  gap: 0.5rem;
}

.icon-btn {
  background: transparent;
  padding: 0.3rem 0.5rem;
  font-size: 1rem;
}

.icon-btn:hover {
  background: #444;
}

.icon-btn.danger:hover {
  background: #ef4444;
}

.edit-comment textarea {
  margin-bottom: 0.5rem;
}

.no-comments {
  color: #888;
  font-style: italic;
  padding: 1rem;
  text-align: center;
}

.add-comment {
  display: flex;
  gap: 0.5rem;
  align-items: end;
}

.add-comment input {
  margin: 0;
  flex: 1;
}
</style>