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
  created_at: string
  updated_at?: string
}

// ---------- STATE ----------
const dbAM = ref<any>(null)
const dbNZ = ref<any>(null)
const posts = ref<Post[]>([])

const search = ref("")
const online = ref(true)
const syncing = ref(false)
const sortMode = ref<"date" | "alpha" | "likes">("likes") // ➕ ajout tri par likes

const newPost = ref({
  post_name: "",
  post_content: "",
  attributes: [] as string[]
})

const isEditing = ref(false)
const selected = ref<Post | null>(null)
const factoryCount = ref(1)

const remoteAM = "http://admin:170451@localhost:5984/infradon2-posts-am"
const remoteNZ = "http://admin:170451@localhost:5984/infradon2-posts-nz"

let syncAM: any = null
let syncNZ: any = null

// ---------- PAGINATION (top 10 les plus likés) ----------
const page = ref(0)
const pageSize = 10
const totalPosts = ref(0)

// ---------- COMMENTAIRES ----------
const commentsByPost = ref<Record<string, Comment[]>>({})
const lastCommentByPost = ref<Record<string, Comment | null>>({})
const showComments = ref<Record<string, boolean>>({})
const newCommentText = ref<Record<string, string>>({})
const editingComment = ref<Comment | null>(null)
const editingCommentText = ref("")

// ---------- ATTACHMENTS ----------
const attachmentUrls = ref<Record<string, string>>({})

// ---------- UTILS ----------
function getDBForName(name: string) {
  const letter = (name?.trim()?.[0] || "").toUpperCase()

  // Sécurité : nom vide ou caractère non alphabétique
  if (!letter || letter < "A" || letter > "Z") {
    return dbNZ.value
  }

  return letter <= "M" ? dbAM.value : dbNZ.value
}

function getDBForPost(post: Post) {
  return getDBForName(post.post_name)
}

function escapeRegExp(text: string) {
  return text.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")
}

// ---------- INIT ----------
const initDB = async () => {
  dbAM.value = new PouchDB("infradon2-posts-am")
  dbNZ.value = new PouchDB("infradon2-posts-nz")

  // Index de base pour les posts
  await dbAM.value.createIndex({ index: { fields: ["type", "post_name", "likes", "created_at"] } })
  await dbNZ.value.createIndex({ index: { fields: ["type", "post_name", "likes", "created_at"] } })

  // Index pour les commentaires
  await dbAM.value.createIndex({ index: { fields: ["type", "postId", "created_at"] } })
  await dbNZ.value.createIndex({ index: { fields: ["type", "postId", "created_at"] } })

  if (online.value) startSync()
}

// ---------- SYNC ----------
function startSync() {
  syncing.value = true
  syncAM = dbAM.value.sync(remoteAM, { live: true, retry: true })
    .on("paused", () => syncing.value = false)
    .on("active", () => syncing.value = true)

  syncNZ = dbNZ.value.sync(remoteNZ, { live: true, retry: true })
    .on("paused", () => syncing.value = false)
    .on("active", () => syncing.value = true)
}

function stopSync() {
  if (syncAM) syncAM.cancel()
  if (syncNZ) syncNZ.cancel()
  syncing.value = false
}

function toggleOnline() {
  online.value ? stopSync() : startSync()
  online.value = !online.value
}

// ---------- FETCH ----------
async function fetchPosts() {
  const [resAM, resNZ] = await Promise.all([
    dbAM.value.find({ selector: { type: "post" } }),
    dbNZ.value.find({ selector: { type: "post" } })
  ])

  const all = [...resAM.docs, ...resNZ.docs] as Post[]
  totalPosts.value = all.length

  // Tri côté TS (dans le cadre du TP on pourrait pousser plus en DB,
  // mais on reste fidèle à ta logique de base)
  const sorted = all.sort((a: any, b: any) => {
    if (sortMode.value === "alpha") {
      return a.post_name.localeCompare(b.post_name)
    }
    if (sortMode.value === "likes") {
      return (b.likes || 0) - (a.likes || 0)
    }
    return new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
  })

  const start = page.value * pageSize
  const end = start + pageSize
  posts.value = sorted.slice(start, end)

  // Charger les derniers commentaires + attachments
  await fetchLastComments()
  await fetchAttachments()
}

// ---------- SEARCH ----------
async function searchByName() {
  if (!search.value.trim()) {
    page.value = 0
    return fetchPosts()
  }

  const safe = escapeRegExp(search.value)
  const regex = new RegExp(safe, "i")

  const [resAM, resNZ] = await Promise.all([
    dbAM.value.find({
      selector: { type: "post", post_name: { $regex: regex.source } }
    }),
    dbNZ.value.find({
      selector: { type: "post", post_name: { $regex: regex.source } }
    })
  ])

  const all = [...resAM.docs, ...resNZ.docs] as Post[]
  totalPosts.value = all.length
  page.value = 0
  posts.value = all.sort((a, b) => a.post_name.localeCompare(b.post_name))

  await fetchLastComments()
  await fetchAttachments()
}

// ---------- PAGINATION ----------
function nextPage() {
  if ((page.value + 1) * pageSize >= totalPosts.value) return
  page.value++
  fetchPosts()
}

function prevPage() {
  if (page.value === 0) return
  page.value--
  fetchPosts()
}

// ---------- CRUD ----------
function handleSubmit() {
  isEditing.value ? updatePost() : addPost()
}

async function addPost() {
  if (!newPost.value.post_name.trim() || !newPost.value.post_content.trim()) return

  const db = getDBForName(newPost.value.post_name)

  await db.put({
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

  const db = getDBForName(selected.value.post_name)

  await db.put({
    ...selected.value,
    ...newPost.value,
    updated_at: new Date().toISOString()
  })

  resetForm()
  fetchPosts()
}

async function deletePost(post: Post) {
  const db = getDBForName(post.post_name)
  await db.remove(post)
  fetchPosts()
}

function resetForm() {
  isEditing.value = false
  selected.value = null
  newPost.value = { post_name: "", post_content: "", attributes: [] }
}

// ---------- LIKES ----------
async function likePost(post: Post) {
  const db = getDBForPost(post)
  const fresh = await db.get(post._id!)
  await db.put({
    ...fresh,
    likes: (fresh.likes || 0) + 1
  })
  fetchPosts()
}

// ---------- COMMENTAIRES ----------
async function loadComments(post: Post) {
  const db = getDBForPost(post)
  const res = await db.find({
    selector: { type: "comment", postId: post._id },
    sort: [{ created_at: "asc" }]
  })
  commentsByPost.value[post._id!] = res.docs
}

async function fetchLastComments() {
  for (const post of posts.value) {
    const db = getDBForPost(post)
    const res = await db.find({
      selector: { type: "comment", postId: post._id },
      sort: [{ created_at: "desc" }],
      limit: 1
    })
    lastCommentByPost.value[post._id!] = res.docs[0] || null
  }
}

function toggleComments(post: Post) {
  const id = post._id!
  showComments.value[id] = !showComments.value[id]
  if (showComments.value[id]) {
    loadComments(post)
  }
}

async function addComment(post: Post) {
  const text = (newCommentText.value[post._id!] || "").trim()
  if (!text) return

  const db = getDBForPost(post)
  await db.put({
    _id: `comment_${Date.now()}`,
    type: "comment",
    postId: post._id!,
    text,
    created_at: new Date().toISOString()
  })

  newCommentText.value[post._id!] = ""
  await loadComments(post)
  await fetchLastComments()
}

function startEditComment(comment: Comment) {
  editingComment.value = comment
  editingCommentText.value = comment.text
}

async function updateComment() {
  if (!editingComment.value) return
  const postId = editingComment.value.postId
  const post = posts.value.find(p => p._id === postId)
  if (!post) return

  const db = getDBForPost(post)
  const fresh = await db.get(editingComment.value._id!)

  await db.put({
    ...fresh,
    text: editingCommentText.value.trim(),
    updated_at: new Date().toISOString()
  })

  editingComment.value = null
  editingCommentText.value = ""
  await loadComments(post)
  await fetchLastComments()
}

async function deleteComment(comment: Comment) {
  const post = posts.value.find(p => p._id === comment.postId)
  if (!post) return

  const db = getDBForPost(post)
  await db.remove(comment)
  await loadComments(post)
  await fetchLastComments()
}

// ---------- ATTACHMENTS ----------
async function loadAttachmentForPost(post: Post) {
  const db = getDBForPost(post)
  try {
    const blob = await db.getAttachment(post._id!, "media")
    const url = URL.createObjectURL(blob)
    attachmentUrls.value[post._id!] = url
  } catch {
    attachmentUrls.value[post._id!] = ""
  }
}

async function fetchAttachments() {
  for (const post of posts.value) {
    await loadAttachmentForPost(post)
  }
}

async function onAttachmentChange(event: Event, post: Post) {
  const input = event.target as HTMLInputElement
  if (!input.files || !input.files[0]) return
  const file = input.files[0]

  const db = getDBForPost(post)
  const doc = await db.get(post._id!)

  await db.putAttachment(doc._id, "media", doc._rev, file, file.type)
  await loadAttachmentForPost(post)
}

async function removeAttachment(post: Post) {
  const db = getDBForPost(post)
  const doc = await db.get(post._id!)

  await db.removeAttachment(doc._id, "media", doc._rev)
  attachmentUrls.value[post._id!] = ""
}

// ---------- FACTORY ----------
async function generateFake() {
  for (let i = 0; i < factoryCount.value; i++) {
    const name = `Message ${i + 1}`
    const db = getDBForName(name)

    await db.put({
      _id: `post_fake_${Date.now()}_${i}`,
      type: "post",
      post_name: name,
      post_content: `Contenu ${i + 1}`,
      attributes: ["auto"],
      likes: 0,
      created_at: new Date().toISOString()
    })
  }
  fetchPosts()
}

// ---------- EXPORTS ----------
async function replicateFromDistant() {
  await dbAM.value.replicate.from(remoteAM)
  await dbNZ.value.replicate.from(remoteNZ)
  fetchPosts()
}

async function replicateToDistant() {
  await dbAM.value.replicate.to(remoteAM)
  await dbNZ.value.replicate.to(remoteNZ)
}

async function manualSync() {
  await replicateFromDistant()
  await replicateToDistant()
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
    <h1>📡 CouchDB + Vue 3 — Split A–M / N–Z</h1>

    <!-- ONLINE / OFFLINE -->
    <section>
      <span :class="online ? 'online' : 'offline'">
        ● {{ online ? "ONLINE" : "OFFLINE" }}
      </span>
      <button @click="toggleOnline">
        {{ online ? "Mode Offline" : "Mode Online" }}
      </button>
      <span v-if="syncing">⏳ sync en cours...</span>
    </section>

    <!-- SEARCH / SORT / FACTORY -->
    <section>
      <input v-model="search" @input="searchByName" placeholder="Rechercher par nom..." />
      <input type="number" v-model="factoryCount" min="1" style="width:80px" />
      <button @click="generateFake">🧪 Générer</button>

      <button @click="sortMode = 'date'; page = 0; fetchPosts()">
        Tri par date
      </button>
      <button @click="sortMode = 'alpha'; page = 0; fetchPosts()">
        Tri alphabétique
      </button>
      <button @click="sortMode = 'likes'; page = 0; fetchPosts()">
        Tri par likes
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

    <!-- PAGINATION TOP 10 -->
    <section v-if="!search && totalPosts > pageSize">
      <p>Documents : {{ totalPosts }} — Page : {{ page + 1 }}</p>
      <button @click="prevPage" :disabled="page === 0">⬅️ Précédent</button>
      <button @click="nextPage" :disabled="(page + 1) * pageSize >= totalPosts">
        Suivant ➡️
      </button>
    </section>

    <!-- LIST -->
    <article v-for="post in posts" :key="post._id">
      <h3>{{ post.post_name }}</h3>
      <p>{{ post.post_content }}</p>
      <small>{{ post.created_at }}</small><br />

      <!-- Likes -->
      <button @click="likePost(post)">👍 {{ post.likes }}</button>

      <!-- Attachment -->
      <div>
        <label>
          Média :
          <input type="file" @change="onAttachmentChange($event, post)" />
        </label>
        <div v-if="attachmentUrls[post._id!]">
          <p>Prévisualisation :</p>
          <img :src="attachmentUrls[post._id!]" alt="media" style="max-width: 200px; max-height: 200px;" />
          <br />
          <button @click="removeAttachment(post)">🗑️ Supprimer le média</button>
        </div>
      </div>

      <!-- Dernier commentaire -->
      <div v-if="lastCommentByPost[post._id!]">
        <strong>Dernier commentaire :</strong>
        <p>{{ lastCommentByPost[post._id!]!.text }}</p>
      </div>

      <!-- Commentaires -->
      <button @click="toggleComments(post)">
        💬 {{ showComments[post._id!] ? "Masquer" : "Voir" }} les commentaires
      </button>

      <div v-if="showComments[post._id!]">
        <div v-for="comment in commentsByPost[post._id!] || []" :key="comment._id">
          <p>{{ comment.text }}</p>
          <small>
            {{ new Date(comment.created_at).toLocaleString("fr-FR") }}
            <span v-if="comment.updated_at">
              (modifié {{ new Date(comment.updated_at).toLocaleString("fr-FR") }})
            </span>
          </small>
          <div>
            <button @click="startEditComment(comment)">✏️ Modifier</button>
            <button @click="deleteComment(comment)">🗑️ Supprimer</button>
          </div>
        </div>

        <!-- Édition -->
        <div v-if="editingComment && editingComment.postId === post._id">
          <textarea v-model="editingCommentText" />
          <button @click="updateComment">💾 Enregistrer</button>
          <button @click="editingComment = null">❌ Annuler</button>
        </div>

        <!-- Nouveau commentaire -->
        <div>
          <textarea
            v-model="newCommentText[post._id!]"
            placeholder="Ajouter un commentaire..."
          />
          <button @click="addComment(post)">➕ Commenter</button>
        </div>
      </div>

      <!-- Actions message -->
      <button @click="selectPost(post)">Modifier</button>
      <button @click="deletePost(post)">Supprimer</button>
    </article>
  </div>
</template>

<style scoped>
.app { padding:2rem; background:#111; color:#fff; max-width:900px; margin:auto }
.online { color:#42b883; font-weight:bold }
.offline { color:#ff4d4d; font-weight:bold }
article { background:#222; padding:1rem; margin-bottom:.5rem; border-radius:6px }
button { margin:.2rem; padding:.4rem .8rem; border:none; border-radius:4px; cursor:pointer }
input, textarea { margin-bottom:.4rem; padding:.4rem; width:100%; border-radius:4px; border:none }
.hint { font-size:.85em; color:#aaa }
img { border-radius:4px; margin-top:0.3rem }
</style>
