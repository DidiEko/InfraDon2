<script setup lang="ts">
import { ref, onMounted } from "vue"
import PouchDB from "pouchdb"
import PouchDBFind from "pouchdb-find"

// Activation du plugin Find
PouchDB.plugin(PouchDBFind)

// =========================
// 🔹 TYPES
// =========================
interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: string[]
  likes?: number
  comments?: { text: string; date: string }[]
  created_at?: string
  updated_at?: string
}

// =========================
// 🔹 STATE
// =========================
const localDB = ref<any>(null)
const remoteDB = ref<any>(null)
const posts = ref<Post[]>([])

const searchTerm = ref("")
const newPost = ref<Post>({
  post_name: "",
  post_content: "",
  attributes: [],
})
const isEditing = ref(false)
const selectedPost = ref<Post | null>(null)

const online = ref(true)
const syncing = ref(false)
const newComment = ref("")
const factoryAmount = ref(30)

let syncHandler: any = null

const remoteURL = "http://admin:170451@localhost:5984/infradon2-eko"

// =========================
// 🔹 INIT DB
// =========================
const initDB = async () => {
  localDB.value = new PouchDB("infradon2-eko")
  remoteDB.value = new PouchDB(remoteURL)

  await localDB.value.createIndex({ index: { fields: ["post_name"] } })
  console.log("📌 Index créé")

  startLocalChangeListener()

  if (online.value) startLiveSync()
}

// =========================
// 🔹 LIVE SYNC (+ events)
// =========================
const startLiveSync = () => {
  syncing.value = true

  syncHandler = localDB.value
    .sync(remoteDB.value, { live: true, retry: true })
    .on("change", loadPosts)
    .on("paused", () => console.log("⏸ Pause sync"))
    .on("active", () => console.log("▶ Sync Active"))
    .on("error", (err: any) => console.error("❌ Sync ERROR:", err))
}

const stopLiveSync = () => {
  syncing.value = false
  syncHandler?.cancel()
}

// =========================
// 🔹 ONLINE/OFFLINE
// =========================
const toggleOnline = async () => {
  online.value = !online.value

  if (online.value) {
    startLiveSync()
    await manualSync()
  } else {
    stopLiveSync()
  }
}

// =========================
// 🔹 FETCH
// =========================
const loadPosts = async () => {
  const result = await localDB.value.allDocs({ include_docs: true })
  posts.value = result.rows
    .map((r: any) => r.doc)
    .sort((a: any, b: any) => new Date(b.created_at) - new Date(a.created_at))
}

// =========================
// 🔹 LISTEN LOCAL CHANGES
// =========================
const startLocalChangeListener = () => {
  localDB.value
    .changes({ since: "now", live: true, include_docs: true })
    .on("change", loadPosts)
}

// =========================
// 🔹 SEARCH (index + regex)
// =========================
const searchPosts = async () => {
  if (searchTerm.value.trim() === "") return loadPosts()

  const selector =
    searchTerm.value.length < 3
      ? { post_name: searchTerm.value }
      : { post_name: { $regex: searchTerm.value } }

  const result = await localDB.value.find({ selector })
  posts.value = result.docs
}

// =========================
// 🔹 FACTORY
// =========================
const generateFake = async () => {
  const batch = []
  const t = Date.now()

  for (let i = 0; i < factoryAmount.value; i++) {
    batch.push({
      _id: `${t}-${i}`,
      post_name: `Personne ${i}`,
      post_content: `Contenu aléatoire ${Math.random()}`,
      attributes: ["auto", `grp_${i}`],
      likes: Math.floor(Math.random() * 40),
      comments: [],
      created_at: new Date().toISOString(),
    })
  }

  await localDB.value.bulkDocs(batch)
  loadPosts()
}

// =========================
// 🔹 CRUD
// =========================
const handleSubmit = () => {
  isEditing.value ? updatePost() : addPost()
}

const addPost = async () => {
  const doc: Post = {
    _id: new Date().toISOString(),
    post_name: newPost.value.post_name,
    post_content: newPost.value.post_content,
    attributes: newPost.value.attributes,
    likes: 0,
    comments: [],
    created_at: new Date().toISOString(),
  }

  await localDB.value.put(doc)
  resetForm()
}

const selectPost = (post: Post) => {
  isEditing.value = true
  selectedPost.value = post

  newPost.value = {
    post_name: post.post_name,
    post_content: post.post_content,
    attributes: [...post.attributes],
  }
}

const updatePost = async () => {
  if (!selectedPost.value) return

  const updated: Post = {
    ...selectedPost.value,
    post_name: newPost.value.post_name,
    post_content: newPost.value.post_content,
    attributes: newPost.value.attributes,
    updated_at: new Date().toISOString(),
  }

  await localDB.value.put(updated)
  resetForm()
}

const deletePost = async (post: Post) => {
  await localDB.value.remove(post._id, post._rev)
}

const resetForm = () => {
  selectedPost.value = null
  isEditing.value = false
  newPost.value = { post_name: "", post_content: "", attributes: [] }
  loadPosts()
}

// =========================
// 🔹 LIKE
// =========================
const likePost = async (post: Post) => {
  post.likes = (post.likes || 0) + 1
  await localDB.value.put(post)
}

// =========================
// 🔹 COMMENTAIRES
// =========================
const addComment = async (post: Post) => {
  if (!newComment.value.trim()) return

  post.comments = post.comments || []
  post.comments.push({
    text: newComment.value,
    date: new Date().toISOString(),
  })

  newComment.value = ""
  await localDB.value.put(post)
}

// =========================
// 🔹 SYNC MANUEL
// =========================
const replicateFromRemote = async () => {
  await localDB.value.replicate.from(remoteDB.value)
  loadPosts()
}

const replicateToRemote = async () => {
  await localDB.value.replicate.to(remoteDB.value)
}

const manualSync = async () => {
  await replicateFromRemote()
  await replicateToRemote()
}

// =========================
// 🔹 MOUNT
// =========================
onMounted(async () => {
  await initDB()
  loadPosts()
})
</script>
