<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue"
import PouchDB from "pouchdb"
import PouchDBFind from "pouchdb-find"

PouchDB.plugin(PouchDBFind)

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

// --- STATE ---
const storage = ref<any>(null)
const posts = ref<Post[]>([])
const search = ref("")
const online = ref(true)
const syncing = ref(false)
const sortLikes = ref(false)

const newPost = ref<Post>({
  post_name:"", post_content:"", attributes:[], likes:0, comments:[]
})
const isEditing = ref(false)
const selected = ref<Post|null>(null)
const remoteCouch = "http://admin:170451@localhost:5984/infradon2-eko"
let syncHandler: any = null

// --- Factory ---
const factoryCount = ref(1)

// --- Comments ---
const showComments = ref<{[key:string]:boolean}>({})
const newComment = ref<{[key:string]:string}>({})

// --- INIT ---
const initDB = async () => {
  storage.value = new PouchDB("infradon2-eko")
  await storage.value.createIndex({ index:{fields:["post_name"]} })
  await storage.value.createIndex({ index:{fields:["likes"]} })
  listenLocal()
  if (online.value) startSync()
}

function listenLocal() {
  storage.value.changes({
    since:"now", live:true, include_docs:true
  }).on("change", fetchPosts)
}

function startSync() {
  if (syncHandler) syncHandler.cancel()
  syncing.value = true
  syncHandler = storage.value.sync(remoteCouch, { live:true, retry:true })
    .on("change", fetchPosts)
    .on("paused", (info:any) => {
      syncing.value = false
    })
    .on("active", () => {
      syncing.value = true
    })
    .on("denied", (err:any) => {
      syncing.value = false
      console.error('Denied:', err)
    })
    .on("error", (err:any) => {
      online.value = false
      syncing.value = false
      console.error('Sync error:', err)
    })
}

function stopSync() {
  if (syncHandler) {
    syncHandler.cancel()
    syncHandler = null
  }
  syncing.value = false
}

async function toggleOnline() {
  online.value = !online.value
  if (online.value) {
    startSync()
  } else {
    stopSync()
  }
}

async function fetchPosts() {
  if (!storage.value) return
  const result = sortLikes.value
    ? await storage.value.find({ selector:{likes:{$gte:0}}, sort:[{likes:"desc"}] })
    : await storage.value.allDocs({ include_docs:true })
  posts.value = sortLikes.value
    ? result.docs.filter((doc:any) => !doc._id?.startsWith('_design'))
    : result.rows.map((r:any)=>r.doc).filter((d:any)=>!d._id?.startsWith('_design'))
      .sort((a:any,b:any)=>new Date(b.created_at).getTime()-new Date(a.created_at).getTime())
}

async function searchByName() {
  if (!search.value.trim()) return fetchPosts()
  const result = await storage.value.find({
    selector:{ post_name:{ $regex:`(?i)${search.value}` } }
  })
  posts.value = result.docs
}

// --- Factory personnalisée ---
async function generateFake(n = factoryCount.value) {
  const docs = []
  for (let i=0; i<n; i++) {
    docs.push({
      _id: `fake_${Date.now()}_${i}`,
      post_name: `Message ${i+1}`,
      post_content: `Contenu du message ${i+1}`,
      attributes: ["auto"],
      likes: 0,
      comments: [],
      created_at: new Date(Date.now() - Math.random()*1e10).toISOString()
    })
  }
  await storage.value.bulkDocs(docs)
  fetchPosts()
}

// --- CRUD Posts ---
async function handleSubmit() {
  isEditing.value ? updatePost() : addPost()
}
async function addPost() {
  if (!newPost.value.post_name.trim() || !newPost.value.post_content.trim()) return
  await storage.value.put({
    ...newPost.value,
    _id: new Date().toISOString(),
    created_at: new Date().toISOString()
  })
  resetForm()
  fetchPosts()
}
function selectPost(post: Post) {
  isEditing.value = true
  selected.value = post
  newPost.value = { ...post, attributes: [...post.attributes], likes: post.likes||0, comments: post.comments||[] }
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
  await storage.value.remove(post._id, post._rev)
  fetchPosts()
}
function resetForm() {
  isEditing.value = false
  selected.value = null
  newPost.value = { post_name:"", post_content:"", attributes:[], likes:0, comments:[] }
}

// --- Likes ---
async function likePost(post: Post) {
  await storage.value.put({ ...post, likes: (post.likes||0)+1 })
  fetchPosts()
}

// --- Comments ---
function toggleComments(id: string) {
  showComments.value[id] = !showComments.value[id]
}
async function addComment(post: Post) {
  const txt = newComment.value[post._id!]?.trim()
  if (!txt) return
  const updated = {
    ...post,
    comments: [...(post.comments||[]), { text: txt, date: new Date().toISOString() }]
  }
  await storage.value.put(updated)
  newComment.value[post._id!] = ""
  fetchPosts()
}

// --- Sync Manual ---
async function replicateFromDistant() {
  if (!online.value) return
  await storage.value.replicate.from(remoteCouch)
  fetchPosts()
}
async function replicateToDistant() {
  if (!online.value) return
  await storage.value.replicate.to(remoteCouch)
}
async function manualSync() {
  if (!online.value) return
  await replicateFromDistant()
  await replicateToDistant()
  fetchPosts()
}

onMounted(async () => {
  await initDB()
  await fetchPosts()
})
onUnmounted(() => {
  stopSync()
})
</script>

<template>
  <div class="app">
    <h1>📡 CouchDB + Vue 3</h1>
    <section>
      <span :class="online ? 'online' : 'offline'">
        ● {{ online ? "ONLINE" : "OFFLINE" }}
      </span>
      <button @click="toggleOnline">{{ online ? 'Mode Offline' : 'Mode Online' }}</button>
    </section>
    <section>
      <input v-model="search" @input="searchByName" placeholder="Rechercher par nom..." />
      <input type="number" v-model="factoryCount" min="1" style="width:80px" />
      <button @click="generateFake()">🧪 Générer</button>
      <button @click="sortLikes = !sortLikes; fetchPosts()" :class="{active:sortLikes}">
        {{ sortLikes ? 'Tri: Likes' : 'Tri: Date' }}
      </button>
    </section>
    <section>
      <button @click="replicateFromDistant" :disabled="!online">⬇️ CouchDB → Local</button>
      <button @click="replicateToDistant" :disabled="!online">⬆️ Local → CouchDB</button>
      <button @click="manualSync" :disabled="!online">🔁 Sync Bidirectionnelle</button>
    </section>
    <section>
      <h2>{{ isEditing ? "Modifier" : "Ajouter" }}</h2>
      <input v-model="newPost.post_name" placeholder="Nom *" />
      <textarea v-model="newPost.post_content" rows="3" placeholder="Contenu *"></textarea>
      <input :value="newPost.attributes.join(', ')" placeholder="Attributs (virgule)" 
        @input="newPost.attributes = ($event.target as HTMLInputElement).value.split(',').map(s=>s.trim()).filter(s=>s)" />
      <button @click="handleSubmit">{{ isEditing ? "Enregistrer":"Créer" }}</button>
      <button v-if="isEditing" @click="resetForm">Annuler</button>
    </section>
    <section>
      <h2>Messages <span>{{ posts.length }}</span></h2>
      <p v-if="posts.length===0">Aucun message.</p>
      <article v-for="post in posts" :key="post._id">
        <h3>{{ post.post_name }}</h3>
        <p>{{ post.post_content }}</p>
        <div v-if="post.attributes?.length">Attributs: <span v-for="attr in post.attributes" :key="attr">{{ attr }}</span></div>
        <div>👍 <button @click="likePost(post)">{{ post.likes||0 }}</button></div>
        <div>Créé: {{ new Date(post.created_at).toLocaleString('fr-FR') }}</div>
        <button @click="toggleComments(post._id!)">💬 {{ (post.comments||[]).length }} commentaire(s)</button>
        <div v-if="showComments[post._id!]">
          <h4>Commentaires</h4>
          <div v-for="(c,i) in post.comments" :key="i">{{ c.text }} <small>{{ new Date(c.date).toLocaleString('fr-FR') }}</small></div>
          <input v-model="newComment[post._id!]" placeholder="Commenter..." />
          <button
            @click="addComment(post)"
            :disabled="!newComment[post._id!] || !newComment[post._id!].trim()"
          >Envoyer</button>
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
input, textarea { margin-bottom: .4em; padding:.5em; width:100%; }
button { margin:.2em .2em 0 0; padding:.4em 1em; border-radius:6px; }
.active { background:#42b883; color:#fff; }
</style>
