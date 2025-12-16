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

// ---------- STATE ----------
const storage = ref<any>(null)
const posts = ref<Post[]>([])

const search = ref("")
const online = ref(true)
const syncing = ref(false)

const sortMode = ref<"date" | "alpha">("date")

const newPost = ref({
  post_name: "",
  post_content: "",
  attributes: [] as string[]
})

const isEditing = ref(false)
const selected = ref<Post | null>(null)

const factoryCount = ref(1)

const remoteCouch = "http://admin:170451@localhost:5984/infradon2-eko"
let syncHandler: any = null

// ---------- INIT ----------
const initDB = async () => {
  storage.value = new PouchDB("infradon2-eko")

  await storage.value.createIndex({ index: { fields: ["type"] } })
  await storage.value.createIndex({ index: { fields: ["type", "post_name"] } })

  listenLocal()
  if (online.value) startSync()
}

function listenLocal() {
  storage.value.changes({ since: "now", live: true })
    .on("change", fetchPosts)
}

// ---------- ONLINE / OFFLINE ----------
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

// ---------- FETCH + TRI ----------
async function fetchPosts() {
  const result = await storage.value.find({
    selector: { type: "post" }
  })

  posts.value = result.docs.sort((a: any, b: any) => {
    if (sortMode.value === "alpha") {
      return a.post_name.localeCompare(b.post_name)
    }
    return new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
  })
}

// ---------- SEARCH ----------
function escapeRegExp(text: string) {
  return text.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")
}

async function searchByName() {
  if (!search.value.trim()) return fetchPosts()

  const safe = escapeRegExp(search.value)
  const regex = new RegExp(safe, "i")

  const result = await storage.value.find({
    selector: { type: "post", post_name: { $regex: regex.source } }
  })

  posts.value = result.docs
}

// ---------- CRUD ----------
function handleSubmit() {
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

// ---------- FACTORY ----------
async function generateFake() {
  const docs = []
  for (let i = 0; i < factoryCount.value; i++) {
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

// ---------- EXPORTS ----------
async function replicateFromDistant() {
  await storage.value.replicate.from(remoteCouch)
  fetchPosts()
}

async function replicateToDistant() {
  await storage.value.replicate.to(remoteCouch)
}

async function manualSync() {
  await storage.value.replicate.from(remoteCouch)
  await storage.value.replicate.to(remoteCouch)
  fetchPosts()
}

// ---------- LIFECYCLE ----------
onMounted(async () => {
  await initDB()
  await fetchPosts()
})

onUnmounted(stopSync)
</script>

<template>
  <div class="app">
    <h1>📡 CouchDB + Vue 3</h1>

    <!-- ONLINE / OFFLINE -->
    <section>
      <span :class="online ? 'online' : 'offline'">
        ● {{ online ? "ONLINE" : "OFFLINE" }}
      </span>
      <button @click="toggleOnline">
        {{ online ? "Mode Offline" : "Mode Online" }}
      </button>
    </section>

    <!-- SEARCH / FACTORY / SORT -->
    <section>
      <input v-model="search" @input="searchByName" placeholder="Rechercher par nom..." />
      <input type="number" v-model="factoryCount" min="1" style="width:80px" />
      <button @click="generateFake">🧪 Générer</button>

      <button @click="sortMode = sortMode === 'date' ? 'alpha' : 'date'; fetchPosts()">
        Tri : {{ sortMode === "date" ? "Date" : "Alphabétique" }}
      </button>
    </section>

    <!-- EXPORTS -->
    <section>
      <p class="hint">
        • CouchDB → Local : récupérer les données du serveur<br />
        • Local → CouchDB : envoyer les données locales<br />
        • Sync bidirectionnelle : synchronisation complète
      </p>

      <button :disabled="online" @click="replicateFromDistant">
        ⬇️ CouchDB → Local
      </button>
      <button :disabled="online" @click="replicateToDistant">
        ⬆️ Local → CouchDB
      </button>
      <button :disabled="online" @click="manualSync">
        🔁 Sync Bidirectionnelle
      </button>
    </section>

    <!-- FORM -->
    <form @submit.prevent="handleSubmit">
      <input v-model="newPost.post_name" placeholder="Nom" required />
      <input v-model="newPost.post_content" placeholder="Contenu" required />
      <button type="submit">{{ isEditing ? "Modifier" : "Ajouter" }}</button>
      <button v-if="isEditing" type="button" @click="resetForm">Annuler</button>
    </form>

    <!-- LIST -->
    <article v-for="post in posts" :key="post._id">
      <h3>{{ post.post_name }}</h3>
      <p>{{ post.post_content }}</p>
      <small>{{ post.created_at }}</small><br />
      <button @click="selectPost(post)">Modifier</button>
      <button @click="deletePost(post)">Supprimer</button>
    </article>
  </div>
</template>

<style scoped>
.app { padding:2rem; background:#111; color:#fff; max-width:700px; margin:auto }
.online { color:#42b883; font-weight:bold }
.offline { color:#ff4d4d; font-weight:bold }
article { background:#222; padding:1rem; margin-bottom:.5rem; border-radius:6px }
button { margin:.2rem; padding:.4rem .8rem }
input { margin-bottom:.4rem; padding:.4rem; width:100% }
.hint { font-size:.85em; color:#aaa }
</style>
