<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue"
import PouchDB from "pouchdb"
import PouchDBFind from "pouchdb-find"

PouchDB.plugin(PouchDBFind)

// ---------- TYPES ----------
interface Post {
  _id?: string
  _rev?: string
  type: "post"
  post_name: string
  post_content: string
  attributes: string[]
  likes: number
  created_at?: string
  updated_at?: string
}

interface Comment {
  _id?: string
  _rev?: string
  type: "comment"
  postId: string
  text: string
  date: string
}

// ---------- STATE ----------
const storage = ref<any>(null)
const posts = ref<Post[]>([])
const commentsByPost = ref<Record<string, Comment[]>>({})

const search = ref("")
const online = ref(true)
const syncing = ref(false)
const sortLikes = ref(false)

const newPost = ref({
  post_name: "",
  post_content: "",
  attributes: [] as string[]
})

const isEditing = ref(false)
const selected = ref<Post | null>(null)

const showComments = ref<Record<string, boolean>>({})
const newComment = ref<Record<string, string>>({})

const factoryCount = ref(1)

// ✅ pour surligner le bouton cliqué
type SyncAction = "from" | "to" | "bi" | null
const activeAction = ref<SyncAction>(null)

const remoteCouch = "http://admin:170451@localhost:5984/infradon2-eko"
let syncHandler: any = null

// ---------- INIT ----------
const initDB = async () => {
  storage.value = new PouchDB("infradon2-eko")

  // Indexation
  await storage.value.createIndex({ index: { fields: ["type"] } })
  await storage.value.createIndex({ index: { fields: ["type", "post_name"] } })
  await storage.value.createIndex({ index: { fields: ["type", "likes"] } })
  await storage.value.createIndex({ index: { fields: ["type", "postId"] } })

  listenLocal()
  if (online.value) startSync()
}

function listenLocal() {
  storage.value.changes({ since: "now", live: true })
    .on("change", fetchPosts)
}

// ---------- SYNC LIVE ----------
function startSync() {
  if (syncHandler) syncHandler.cancel()
  syncing.value = true
  syncHandler = storage.value.sync(remoteCouch, { live: true, retry: true })
    .on("paused", () => syncing.value = false)
    .on("active", () => syncing.value = true)
}

function stopSync() {
  if (syncHandler) syncHandler.cancel()
  syncing.value = false
}

function toggleOnline() {
  online.value ? stopSync() : startSync()
  online.value = !online.value
}

// ---------- UTILS ----------
function markAction(action: SyncAction) {
  activeAction.value = action
  setTimeout(() => {
    if (activeAction.value === action) {
      activeAction.value = null
    }
  }, 3000)
}

// ---------- FETCH ----------
async function fetchPosts() {
  const result = sortLikes.value
    ? await storage.value.find({
        selector: { type: "post", likes: { $gte: 0 } },
        sort: [{ likes: "desc" }]
      })
    : await storage.value.find({ selector: { type: "post" } })

  posts.value = result.docs.sort(
    (a: any, b: any) =>
      new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
  )

  for (const post of posts.value) {
    await loadComments(post._id!)
  }
}

async function searchByName() {
  if (!search.value.trim()) return fetchPosts()
  const result = await storage.value.find({
    selector: { type: "post", post_name: { $regex: `(?i)${search.value}` } }
  })
  posts.value = result.docs
}

// ---------- CRUD POSTS ----------
async function handleSubmit() {
  isEditing.value ? updatePost() : addPost()
}

async function addPost() {
  if (!newPost.value.post_name || !newPost.value.post_content) return

  await storage.value.put({
    _id: `post_${Date.now()}`,
    type: "post",
    post_name: newPost.value.post_name,
    post_content: newPost.value.post_content,
    attributes: newPost.value.attributes,
    likes: 0,
    created_at: new Date().toISOString()
  })

  resetForm()
  fetchPosts()
}

function selectPost(post: Post) {
  isEditing.value = true
  selected.value = post
  newPost.value = {
    post_name: post.post_name,
    post_content: post.post_content,
    attributes: [...post.attributes]
  }
}

async function updatePost() {
  if (!selected.value) return
  await storage.value.put({
    ...selected.value,
    ...newPost.value,
    updated_at: new Date().toISOString()
  })
  resetForm()
  fetchPosts()
}

async function deletePost(post: Post) {
  await storage.value.remove(post)
  fetchPosts()
}

function resetForm() {
  isEditing.value = false
  selected.value = null
  newPost.value = { post_name: "", post_content: "", attributes: [] }
}

// ---------- LIKES ----------
async function likePost(post: Post) {
  await storage.value.put({ ...post, likes: post.likes + 1 })
  fetchPosts()
}

// ---------- COMMENTS ----------
async function loadComments(postId: string) {
  const result = await storage.value.find({
    selector: { type: "comment", postId }
  })
  commentsByPost.value[postId] = result.docs
}

function toggleComments(postId: string) {
  showComments.value[postId] = !showComments.value[postId]
}

async function addComment(post: Post) {
  const txt = newComment.value[post._id!]?.trim()
  if (!txt) return

  await storage.value.put({
    _id: `comment_${Date.now()}`,
    type: "comment",
    postId: post._id!,
    text: txt,
    date: new Date().toISOString()
  })

  newComment.value[post._id!] = ""
  loadComments(post._id!)
}

// ---------- FACTORY ----------
async function generateFake(n = factoryCount.value) {
  const docs = []
  for (let i = 0; i < n; i++) {
    docs.push({
      _id: `post_fake_${Date.now()}_${i}`,
      type: "post",
      post_name: `Message ${i + 1}`,
      post_content: `Contenu ${i + 1}`,
      attributes: ["auto"],
      likes: 0,
      created_at: new Date().toISOString()
    })
  }
  await storage.value.bulkDocs(docs)
  fetchPosts()
}

// ---------- MANUAL SYNC (OFFLINE) ----------
async function replicateFromDistant() {
  markAction("from")
  await storage.value.replicate.from(remoteCouch)
}

async function replicateToDistant() {
  markAction("to")
  await storage.value.replicate.to(remoteCouch)
}

async function manualSync() {
  markAction("bi")
  await storage.value.replicate.from(remoteCouch)
  await storage.value.replicate.to(remoteCouch)
}

onMounted(async () => {
  await initDB()
  await fetchPosts()
})
onUnmounted(stopSync)
</script>

<template>
  <div class="app">
    <h1>📡 CouchDB + Vue 3</h1>

    <section>
      <span :class="online ? 'online' : 'offline'">
        ● {{ online ? "ONLINE" : "OFFLINE" }}
      </span>
      <button @click="toggleOnline">
        {{ online ? "Mode Offline" : "Mode Online" }}
      </button>
    </section>

    <section>
      <input v-model="search" @input="searchByName" placeholder="Rechercher par nom..." />
      <input type="number" v-model="factoryCount" min="1" style="width:80px" />
      <button @click="generateFake()">🧪 Générer</button>
      <button @click="sortLikes=!sortLikes;fetchPosts()" :class="{active:sortLikes}">
        {{ sortLikes ? "Tri: Likes" : "Tri: Date" }}
      </button>
    </section>

    <section>
      <p class="hint">
        • CouchDB → Local : récupérer les données du serveur<br>
        • Local → CouchDB : envoyer les données locales<br>
        • Sync bidirectionnelle : synchronisation complète
      </p>

      <button
        @click="replicateFromDistant"
        :disabled="online"
        :class="{ active: activeAction === 'from' }"
      >
        ⬇️ CouchDB → Local
      </button>

      <button
        @click="replicateToDistant"
        :disabled="online"
        :class="{ active: activeAction === 'to' }"
      >
        ⬆️ Local → CouchDB
      </button>

      <button
        @click="manualSync"
        :disabled="online"
        :class="{ active: activeAction === 'bi' }"
      >
        🔁 Sync Bidirectionnelle
      </button>
    </section>

    <section>
      <h2>Messages <span>{{ posts.length }}</span></h2>
      <p v-if="posts.length === 0">Aucun message.</p>

      <article v-for="post in posts" :key="post._id">
        <h3>{{ post.post_name }}</h3>
        <p>{{ post.post_content }}</p>
        <div v-if="post.attributes?.length">
          Attributs :
          <span v-for="attr in post.attributes" :key="attr">{{ attr }}</span>
        </div>
        <div>
          👍 <button @click="likePost(post)">{{ post.likes }}</button>
        </div>
        <div>
          Créé :
          {{ post.created_at ? new Date(post.created_at).toLocaleString('fr-FR') : 'N/A' }}
        </div>

        <button @click="toggleComments(post._id!)">
          💬 {{ commentsByPost[post._id!]?.length || 0 }} commentaire(s)
        </button>

        <div v-if="showComments[post._id!]">
          <h4>Commentaires</h4>
          <div v-for="c in commentsByPost[post._id!]" :key="c._id">
            {{ c.text }}
            <small> – {{ new Date(c.date).toLocaleString('fr-FR') }}</small>
          </div>
          <input v-model="newComment[post._id!]" placeholder="Commenter..." />
          <button
            @click="addComment(post)"
            :disabled="!newComment[post._id!] || !newComment[post._id!].trim()"
          >
            Envoyer
          </button>
        </div>

        <button @click="selectPost(post)">Modifier</button>
        <button @click="deletePost(post)">Supprimer</button>
      </article>
    </section>
  </div>
</template>

<style scoped>
.app { padding:2rem; color:#fff; background:#111; max-width:700px; margin:auto; }
.online { color:#42b883; font-weight:bold; }
.offline { color:#ff4d4d; font-weight:bold; }
article { background:#222; margin-bottom:1em; padding:1em; border-radius:8px; }
input, textarea { margin-bottom:.4em; padding:.5em; width:100%; }
button { margin:.2em .2em 0 0; padding:.4em 1em; border-radius:6px; }
.active { background:#4da6ff; color:#000; }
.hint { font-size:.85em; color:#aaa; margin-bottom:.5em; }
button:disabled { opacity:.4; cursor:not-allowed; }
</style>
