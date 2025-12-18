<script setup lang="ts">
import { onMounted, ref, onUnmounted } from 'vue'
import PouchDB from 'pouchdb'
import findPlugin from 'pouchdb-find'

PouchDB.plugin(findPlugin)

declare interface Comment {
  text: string
  date: string
}

declare interface Category {
  _id: string
  _rev?: string
  type: 'category'
  name: string
}

declare interface Post {
  _id: string
  _rev?: string
  title: string
  post_content: string
  likes: number
  comments: Comment[]
  attributes?: {
    creation_date: any
    category?: string
  }
}
const categoriesData = ref<Category[]>([])
const postsData = ref<Post[]>([])
const localDB = ref()
const remoteDB = ref()
const syncStatus = ref('Déconnecté')
const isOnline = ref(true)
const syncHandler = ref<PouchDB.Replication.Sync<Post> | null>(null)
const searchQuery = ref('')
const filteredPosts = ref<Post[]>([])

const REMOTE_DB_URL = 'http://Nero:Penafiel29Snaky25@localhost:5984/infradon2'
const LOCAL_DB_NAME = 'infradon_local'

const initApp = async () => {
  localDB.value = new PouchDB(LOCAL_DB_NAME)
  remoteDB.value = new PouchDB(REMOTE_DB_URL)

  await createIndexes()
  await fetchData()

  if (isOnline.value) {
    startLiveSync()
  }
}

const createIndexes = async () => {
  try {
    await localDB.value.createIndex({
      index: {
        fields: ['attributes.category'],
        ddoc: 'idx-category',
      },
    })

    await localDB.value.createIndex({
      index: {
        fields: ['likes'],
        ddoc: 'idx-likes',
      },
    })
    console.log('✅ Index créés : Catégorie et Likes')
  } catch (error) {
    console.error('❌ Erreur lors de la création des index :', error)
  }
}

const fetchData = async () => {
  if (!localDB.value) return

  try {
    const result = await localDB.value.allDocs({ include_docs: true, descending: true })

    const posts: Post[] = []
    const categories: Category[] = []

    result.rows.forEach((row: any) => {
      const doc = row.doc

      if (doc._id.startsWith('_design/')) return

      if (doc.type === 'category') {
        categories.push(doc as Category)
      } else {
        posts.push(doc as Post)
      }
    })

    postsData.value = posts
    categoriesData.value = categories
  } catch (error) {
    console.error(error)
  }
}

const addDocument = async () => {
  const newPost: Partial<Post> = {
    _id: new Date().toISOString(),
    type: 'post',
    title: 'Nouveau post',
    post_content: 'Contenu du message...',
    likes: 0,
    comments: [],
    attributes: {
      creation_date: new Date().toISOString(),
      category: '',
    },
  }

  try {
    await localDB.value.put(newPost)
    await fetchData()
  } catch (error) {
    console.error(error)
  }
}

const updateDocument = async (id: string, updatedData: any) => {
  try {
    const doc = await localDB.value.get(id)
    const updatedDocument = { ...doc, ...updatedData }
    await localDB.value.put(updatedDocument)
    await fetchData()
  } catch (error: any) {
    if (error.status === 409) {
      alert('Conflit détecté lors de la mise à jour.')
    }
    console.error(error)
  }
}

const changePostCategory = async (post: Post, event: Event) => {
  const selectElement = event.target as HTMLSelectElement
  const newCategory = selectElement.value

  try {
    const doc = await localDB.value.get(post._id)

    if (!doc.attributes) doc.attributes = {}
    doc.attributes.category = newCategory

    await localDB.value.put(doc)

    if (searchQuery.value) performSearch()
    else fetchData()

    console.log(`Catégorie changée pour : ${newCategory}`)
  } catch (error) {
    console.error('Erreur changement catégorie', error)
  }
}

const deleteDocument = async (id: string) => {
  try {
    const doc = await localDB.value.get(id)
    await localDB.value.remove(doc)
    await fetchData()
  } catch (error) {
    console.error(error)
  }
}

const startLiveSync = () => {
  if (syncHandler.value) return

  syncStatus.value = 'Connexion...'

  syncHandler.value = localDB.value
    .sync(remoteDB.value, {
      live: true,
      retry: true,
    })
    .on('change', (info: any) => {
      syncStatus.value = `🔄 Synchro (${info.direction})`
      fetchData()
    })
    .on('paused', (err: any) => {
      if (!err) {
        syncStatus.value = '🟢 En ligne (À jour)'
      } else {
        syncStatus.value = '⚠️ En attente de connexion...'
      }
    })
    .on('active', () => {
      syncStatus.value = '📡 Transmission en cours...'
    })
    .on('error', (err: any) => {
      console.error(err)
      syncStatus.value = '🔴 Erreur de synchronisation'
    })
}

const stopLiveSync = () => {
  if (syncHandler.value) {
    syncHandler.value.cancel()
    syncHandler.value = null
    syncStatus.value = '⏸️ Synchronisation arrêtée'
  }
}

const toggleOnlineOffline = () => {
  isOnline.value = !isOnline.value
  if (isOnline.value) {
    startLiveSync()
  } else {
    stopLiveSync()
  }
}

const newCategoryName = ref('')

const addCategory = async () => {
  if (!newCategoryName.value.trim()) return

  const newCat: Category = {
    _id: 'cat_' + Date.now(), // ID préfixé pour être propre
    type: 'category', // IMPORTANT : Marqueur de la 2ème collection
    name: newCategoryName.value.trim(),
  }

  try {
    await localDB.value.put(newCat)
    newCategoryName.value = '' // Reset input
    await fetchData()
  } catch (error) {
    console.error(error)
  }
}

const deleteCategory = async (cat: Category) => {
  try {
    await localDB.value.remove(cat)
    await fetchData()
  } catch (error) {
    console.error(error)
  }
}

const performSearch = async () => {
  if (!searchQuery.value.trim()) {
    filteredPosts.value = []
    return
  }

  try {
    const result = await localDB.value.find({
      selector: {
        'attributes.category': { $eq: searchQuery.value },
      },
    })
    filteredPosts.value = result.docs as Post[]
  } catch (error) {
    console.error(error)
  }
}

const generateFakeData = async (count: number) => {
  if (categoriesData.value.length === 0) {
    alert(
      "⚠️ Impossible de générer : aucune catégorie trouvée.\nVeuillez créer au moins une catégorie dans la 'Collection 2' d'abord !",
    )
    return
  }

  const documents: any[] = []

  for (let i = 0; i < count; i++) {
    const randomCat = categoriesData.value[Math.floor(Math.random() * categoriesData.value.length)]

    documents.push({
      _id: new Date().getTime() + '_' + i,
      type: 'post',
      title: `Doc factice #${i + 1}`,
      post_content: `Contenu généré automatiquement...`,
      likes: Math.floor(Math.random() * 100),
      comments: [],
      attributes: {
        creation_date: new Date().toISOString(),
        category: randomCat.name,
      },
    })
  }

  try {
    await localDB.value.bulkDocs(documents)
    console.log(`✅ ${count} documents générés.`)
    await fetchData()
  } catch (error) {
    console.error(error)
  }
}

onMounted(() => {
  initApp()
})

onUnmounted(() => {
  stopLiveSync()
})

const deleteAllDocuments = async () => {
  if (
    !confirm(
      '⚠️ Êtes-vous sûr de vouloir TOUT supprimer ? Cette action est irréversible (et se répercutera sur le serveur si connecté).',
    )
  ) {
    return
  }

  try {
    const result = await localDB.value.allDocs({ include_docs: false })

    const docsToDelete = result.rows
      .filter((row: any) => !row.id.startsWith('_design/'))
      .map((row: any) => ({
        _id: row.id,
        _rev: row.value.rev,
        _deleted: true,
      }))

    if (docsToDelete.length === 0) {
      alert('La base est déjà vide.')
      return
    }

    await localDB.value.bulkDocs(docsToDelete)

    console.log(`✅ ${docsToDelete.length} documents supprimés.`)

    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de la suppression totale :', error)
  }
}

const sortByLikes = async () => {
  console.log('Tri par likes')

  try {
    const result = await localDB.value.find({
      selector: {
        likes: { $gte: 0 },
      },
      sort: [{ likes: 'desc' }],
    })

    postsData.value = result.docs as Post[]
  } catch (error) {
    console.error('Erreur lors du tri DB :', error)
    alert("Erreur de tri. Avez-vous bien recréé l'index (étape précédente) ?")
  }
}

const newCommentText = ref<{ [key: string]: string }>({})

const likePost = async (id: string) => {
  try {
    const doc = await localDB.value.get(id)
    doc.likes = (doc.likes || 0) + 1

    await localDB.value.put(doc)

    if (searchQuery.value) performSearch()
    else fetchData()
  } catch (error) {
    console.error(error)
  }
}

const addCommentToPost = async (id: string) => {
  const text = newCommentText.value[id]
  if (!text || !text.trim()) return

  try {
    const doc = await localDB.value.get(id)

    if (!doc.comments) doc.comments = []

    doc.comments.push({
      text: text,
      date: new Date().toISOString(),
    })

    await localDB.value.put(doc)

    newCommentText.value[id] = ''
    fetchData()
  } catch (error) {
    console.error(error)
  }
}

const deleteComment = async (postId: string, commentIndex: number) => {
  try {
    const doc = await localDB.value.get(postId)

    if (doc.comments) {
      doc.comments.splice(commentIndex, 1)
      await localDB.value.put(doc)
      fetchData()
    }
  } catch (error) {
    console.error(error)
  }
}
</script>

<template>
  <div class="app-container">
    <header class="main-header">
      <h1>InfraDon2 <span class="subtitle">NoSQL Edition</span></h1>

      <div class="sync-status">
        <button
          @click="toggleOnlineOffline"
          :class="['status-badge', isOnline ? 'online' : 'offline']"
        >
          {{ isOnline ? '🟢 Connecté' : '🔴 Déconnecté' }}
        </button>
        <span class="status-text">{{ syncStatus }}</span>
      </div>
    </header>

    <section class="global-actions">
      <button class="btn btn-danger-outline" @click="deleteAllDocuments">☠️ Tout supprimer</button>
    </section>

    <section class="panel categories-panel">
      <h2>📂 Gestion des Catégories</h2>

      <div class="input-group">
        <input
          v-model="newCategoryName"
          class="form-input"
          placeholder="Nouvelle catégorie (ex: Gaming)"
          @keyup.enter="addCategory"
        />
        <button class="btn btn-primary" @click="addCategory">Ajouter</button>
      </div>

      <div class="tags-container">
        <transition-group name="list">
          <span v-for="cat in categoriesData" :key="cat._id" class="cat-chip">
            {{ cat.name }}
            <button @click="deleteCategory(cat)" class="btn-x">×</button>
          </span>
        </transition-group>
        <p v-if="categoriesData.length === 0" class="empty-state">
          Aucune catégorie. Créez-en une pour commencer !
        </p>
      </div>
    </section>

    <section class="panel factory-panel">
      <h2>🏭 Création de Posts</h2>
      <div class="button-group">
        <button class="btn btn-secondary" @click="addDocument">📄 Nouveau Doc Vide</button>
        <button class="btn btn-secondary" @click="generateFakeData(10)">🤖 +10 Fake Docs</button>
        <button class="btn btn-secondary" @click="generateFakeData(50)">🤖 +50 Fake Docs</button>
      </div>
    </section>

    <section class="control-bar">
      <div class="search-wrapper">
        <span class="search-icon">🔍</span>
        <input
          v-model="searchQuery"
          @input="performSearch"
          type="text"
          class="form-input search-input"
          placeholder="Filtrer par catégorie..."
        />
      </div>
      <div class="sort-wrapper">
        <button class="btn btn-ghost" @click="sortByLikes">🔥 Trier par Popularité</button>
        <button class="btn btn-ghost" @click="fetchData">📅 Trier par Date</button>
      </div>
    </section>

    <section class="posts-feed">
      <transition-group name="list">
        <article
          v-for="post in searchQuery ? filteredPosts : postsData"
          :key="post._id"
          class="post-card"
        >
          <div class="card-header">
            <h3>{{ post.title }}</h3>
            <select
              class="cat-select"
              :value="post.attributes?.category || ''"
              @change="changePostCategory(post, $event)"
            >
              <option value="" disabled>Catégorie...</option>
              <option v-for="cat in categoriesData" :key="cat._id" :value="cat.name">
                {{ cat.name }}
              </option>
            </select>
          </div>

          <div class="card-body">
            <p>{{ post.post_content }}</p>
          </div>

          <div class="card-interactions">
            <button @click="likePost(post._id)" class="btn-like" :class="{ liked: post.likes > 0 }">
              ❤️ {{ post.likes || 0 }}
            </button>
          </div>

          <div class="comments-section">
            <h4>💬 Commentaires ({{ post.comments?.length || 0 }})</h4>

            <ul class="comments-list">
              <li v-for="(comment, index) in post.comments" :key="index" class="comment-item">
                <div class="comment-content">
                  <span class="comment-text">{{ comment.text }}</span>
                  <small class="comment-date">{{
                    new Date(comment.date).toLocaleTimeString([], {
                      hour: '2-digit',
                      minute: '2-digit',
                    })
                  }}</small>
                </div>
                <button class="btn-icon-delete" @click="deleteComment(post._id, index)">×</button>
              </li>
            </ul>

            <div class="input-group sm">
              <input
                v-model="newCommentText[post._id]"
                class="form-input sm"
                placeholder="Écrire un commentaire..."
                @keyup.enter="addCommentToPost(post._id)"
              />
              <button class="btn btn-sm btn-primary" @click="addCommentToPost(post._id)">
                Envoyer
              </button>
            </div>
          </div>

          <div class="card-footer">
            <button
              class="btn-text"
              @click="updateDocument(post._id, { title: post.title + ' (Modifié)' })"
            >
              ✏️ Éditer
            </button>
            <button class="btn-text danger" @click="deleteDocument(post._id)">🗑️ Supprimer</button>
          </div>
        </article>
      </transition-group>
    </section>
  </div>
</template>

<style scoped>
:root {
  --bg-color: #121212;
  --card-bg: #1e1e1e;
  --primary: #42b883; /* Vue Green */
  --primary-hover: #33a06f;
  --danger: #ff4d4d;
  --text-main: #e0e0e0;
  --text-muted: #a0a0a0;
  --border: #333;
}

.app-container {
  margin: 0 auto;
  padding: 40px 20px;
  font-family:
    'Inter',
    system-ui,
    -apple-system,
    sans-serif;
  color: #e0e0e0;
  background-color: #121212;
  min-height: 100vh;
}

h1,
h2,
h3 {
  margin: 0;
  font-weight: 600;
  color: #fff;
}
h1 {
  font-size: 2rem;
  margin-bottom: 5px;
}
h2 {
  font-size: 1.2rem;
  margin-bottom: 15px;
  border-left: 4px solid #42b883;
  padding-left: 10px;
}
.subtitle {
  font-size: 1rem;
  color: #666;
  font-weight: 400;
}

.main-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
  padding-bottom: 20px;
  border-bottom: 1px solid #333;
}

.sync-status {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 5px;
}

.status-badge {
  padding: 6px 12px;
  border-radius: 20px;
  font-size: 0.85em;
  font-weight: bold;
  border: none;
  cursor: pointer;
  transition: transform 0.2s;
}
.status-badge:active {
  transform: scale(0.95);
}
.online {
  background: #155724;
  color: #d4edda;
  border: 1px solid #1e7e34;
}
.offline {
  background: #721c24;
  color: #f8d7da;
  border: 1px solid #dc3545;
}
.status-text {
  font-size: 0.8em;
  color: #666;
}

.panel {
  background: #1e1e1e;
  padding: 20px;
  border-radius: 12px;
  margin-bottom: 20px;
  border: 1px solid #333;
}

.global-actions {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 20px;
}

.input-group {
  display: flex;
  gap: 10px;
}
.input-group.sm {
  margin-top: 10px;
}

.form-input {
  flex-grow: 1;
  padding: 10px 15px;
  border-radius: 8px;
  border: 1px solid #444;
  background: #2a2a2a;
  color: white;
  font-size: 0.95rem;
  transition: border-color 0.2s;
}
.form-input:focus {
  outline: none;
  border-color: #42b883;
}
.form-input.sm {
  padding: 8px 12px;
  font-size: 0.9rem;
}

.btn {
  padding: 10px 20px;
  border-radius: 8px;
  border: none;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}
.btn-sm {
  padding: 8px 15px;
  font-size: 0.9em;
}

.btn-primary {
  background: #42b883;
  color: #121212;
}
.btn-primary:hover {
  background: #33a06f;
}

.btn-secondary {
  background: #333;
  color: #fff;
}
.btn-secondary:hover {
  background: #444;
}

.btn-danger-outline {
  background: transparent;
  border: 1px solid #ff4d4d;
  color: #ff4d4d;
}
.btn-danger-outline:hover {
  background: #ff4d4d;
  color: white;
}

.btn-ghost {
  background: transparent;
  color: #aaa;
}
.btn-ghost:hover {
  color: #fff;
  background: rgba(255, 255, 255, 0.05);
}

.button-group {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.tags-container {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 15px;
}
.cat-chip {
  background: #2c3e50;
  color: #fff;
  padding: 6px 12px;
  border-radius: 20px;
  font-size: 0.85em;
  display: flex;
  align-items: center;
  gap: 8px;
  border: 1px solid #3e5871;
}
.btn-x {
  background: none;
  border: none;
  color: #aaa;
  cursor: pointer;
  font-size: 1.1em;
  line-height: 0.5;
}
.btn-x:hover {
  color: #ff4d4d;
}
.empty-state {
  color: #666;
  font-style: italic;
  font-size: 0.9em;
}

.control-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 10px;
}
.search-wrapper {
  position: relative;
  flex-grow: 1;
  max-width: 400px;
}
.search-icon {
  position: absolute;
  left: 12px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 0.9em;
  opacity: 0.5;
}
.search-input {
  padding-left: 35px;
}

.posts-feed {
  display: grid;
  gap: 20px;
}

.post-card {
  background: #1e1e1e;
  border-radius: 12px;
  border: 1px solid #333;
  overflow: hidden;
  transition:
    transform 0.2s,
    box-shadow 0.2s;
}
.post-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.3);
  border-color: #444;
}

.card-header {
  padding: 15px 20px;
  background: rgba(255, 255, 255, 0.03);
  border-bottom: 1px solid #333;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.cat-select {
  background: #121212;
  color: #bbb;
  border: 1px solid #444;
  border-radius: 6px;
  padding: 4px 8px;
  font-size: 0.8em;
  outline: none;
}

.card-body {
  padding: 20px;
  color: #ddd;
  line-height: 1.5;
}

.card-interactions {
  padding: 0 20px 15px;
}

.btn-like {
  background: #2a2a2a;
  border: 1px solid #333;
  color: #ccc;
  padding: 6px 15px;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 6px;
}
.btn-like:hover {
  background: #333;
}
.btn-like.liked {
  color: #ff4d4d;
  border-color: #ff4d4d22;
  background: #ff4d4d11;
}

.comments-section {
  background: #181818;
  padding: 15px 20px;
  border-top: 1px solid #333;
}
.comments-section h4 {
  font-size: 0.9em;
  color: #888;
  margin-bottom: 10px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.comments-list {
  list-style: none;
  padding: 0;
  margin: 0 0 15px 0;
}
.comment-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
  border-bottom: 1px solid #222;
}
.comment-item:last-child {
  border-bottom: none;
}
.comment-content {
  display: flex;
  flex-direction: column;
}
.comment-text {
  font-size: 0.95em;
}
.comment-date {
  font-size: 0.75em;
  color: #666;
  margin-top: 2px;
}

.btn-icon-delete {
  background: none;
  border: none;
  color: #444;
  cursor: pointer;
  font-size: 1.2em;
}
.btn-icon-delete:hover {
  color: #ff4d4d;
}

.card-footer {
  padding: 10px 20px;
  background: rgba(0, 0, 0, 0.2);
  display: flex;
  justify-content: flex-end;
  gap: 15px;
}
.btn-text {
  background: none;
  border: none;
  color: #666;
  font-size: 0.85em;
  cursor: pointer;
}
.btn-text:hover {
  color: #fff;
}
.btn-text.danger:hover {
  color: #ff4d4d;
}

.list-enter-active,
.list-leave-active {
  transition: all 0.3s ease;
}
.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateY(10px);
}
</style>
