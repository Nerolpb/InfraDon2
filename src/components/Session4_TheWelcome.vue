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
  <div class="container">
    <h1>InfraDon2 - nosql</h1>

    <section class="sync-section">
      <button @click="toggleOnlineOffline" :class="{ online: isOnline, offline: !isOnline }">
        {{ isOnline ? '🟢 EN LIGNE' : '🔴 HORS LIGNE' }}
      </button>
      <p class="status">
        Etat: <strong>{{ syncStatus }}</strong>
      </p>
    </section>

    <hr />

    <section class="categories-section">
      <h2>📂 Collection 2 : Catégories</h2>

      <div class="add-cat-box">
        <input
          v-model="newCategoryName"
          placeholder="Nouvelle catégorie (ex: Gaming)"
          @keyup.enter="addCategory"
        />
        <button @click="addCategory">Ajouter</button>
      </div>

      <div class="tags-container">
        <span v-for="cat in categoriesData" :key="cat._id" class="cat-chip">
          {{ cat.name }}
          <button @click="deleteCategory(cat)" class="btn-x">×</button>
        </span>
        <em v-if="categoriesData.length === 0">Aucune catégorie. Créez-en une pour commencer !</em>
      </div>
    </section>

    <hr />

    <section class="data-factory-section">
      <button @click="generateFakeData(10)">+ 10 Docs</button>
      <button @click="generateFakeData(50)">+ 50 Docs</button>
    </section>

    <hr />

    <section class="search-section">
      <input
        v-model="searchQuery"
        @input="performSearch"
        type="text"
        placeholder="Rechercher catégorie..."
      />
    </section>

    <div style="margin-top: 10px; display: flex; gap: 10px">
      <button @click="sortByLikes">🔥 Trier par Popularité (DB)</button>
      <button @click="fetchData">📅 Trier par Date (Défaut)</button>
    </div>

    <hr />

    <section class="actions-section">
      <div class="action-buttons">
        <button @click="addDocument">Ajouter un nouveau document</button>
        <button class="btn-danger" @click="deleteAllDocuments">☠️ Tout supprimer</button>
      </div>
    </section>

    <section class="posts-section">
      <div class="posts-grid">
        <article
          v-for="post in searchQuery ? filteredPosts : postsData"
          :key="post._id"
          class="post-card"
        >
          <div
            style="
              display: flex;
              justify-content: space-between;
              align-items: flex-start;
              gap: 10px;
            "
          >
            <h3 style="margin: 0">{{ post.title }}</h3>

            <select
              class="cat-select"
              :value="post.attributes?.category || ''"
              @change="changePostCategory(post, $event)"
            >
              <option value="" disabled>Choisir catégorie...</option>
              <option v-for="cat in categoriesData" :key="cat._id" :value="cat.name">
                {{ cat.name }}
              </option>
            </select>
          </div>

          <p>{{ post.post_content }}</p>

          <div class="likes-section">
            <button @click="likePost(post._id)" class="btn-like">
              👍 J'aime ({{ post.likes || 0 }})
            </button>
          </div>

          <div class="comments-section">
            <h4>💬 Commentaires ({{ post.comments?.length || 0 }})</h4>

            <ul>
              <li v-for="(comment, index) in post.comments" :key="index" class="comment-item">
                <div>
                  <small>📅 {{ new Date(comment.date).toLocaleTimeString() }}</small> :
                  <span>{{ comment.text }}</span>
                </div>
                <span
                  class="delete-comment-btn"
                  @click="deleteComment(post._id, index)"
                  title="Supprimer"
                  >❌</span
                >
              </li>
            </ul>

            <div class="add-comment-box">
              <input
                v-model="newCommentText[post._id]"
                placeholder="Écrire un commentaire..."
                @keyup.enter="addCommentToPost(post._id)"
              />
              <button @click="addCommentToPost(post._id)">Envoyer</button>
            </div>
          </div>

          <div class="actions">
            <button
              class="btn-update"
              @click="updateDocument(post._id, { title: post.title + ' (Modifié)' })"
            >
              ✏️ Modifier Post
            </button>
            <button class="btn-delete" @click="deleteDocument(post._id)">🗑️ Supprimer Post</button>
          </div>
        </article>
      </div>
    </section>
  </div>
</template>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: sans-serif;
}

.online {
  background-color: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.offline {
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.status {
  margin-top: 10px;
  font-size: 0.9em;
  color: #666;
}

.cat-chip {
  display: inline-flex;
  align-items: center;
  background: #6c757d;
  color: white;
  padding: 5px 10px;
  border-radius: 15px;
  margin-right: 5px;
  margin-bottom: 5px;
  font-size: 0.9em;
}

.btn-x {
  background: none;
  border: none;
  color: white;
  margin-left: 5px;
  cursor: pointer;
  font-weight: bold;
  padding: 0 5px;
}

.btn-x:hover {
  color: #ffcccc;
}

.add-cat-box {
  margin-bottom: 10px;
  display: flex;
  gap: 10px;
}
/* FIN NOUVEAUX STYLES */

.post-card {
  border: 1px solid #ddd;
  padding: 15px;
  margin-bottom: 10px;
  border-radius: 8px;
  background-color: #fff;
  color: #333;
}

.cat-tag {
  background: #e0e0e0;
  color: #2c3e50;
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: bold;
  font-size: 0.85em;
}

.actions {
  margin-top: 10px;
  display: flex;
  gap: 10px;
}

button {
  cursor: pointer;
  padding: 5px 10px;
}

.btn-danger {
  background-color: #dc3545;
  color: white;
  border: 1px solid #c82333;
  margin-left: 10px;
}

.btn-danger:hover {
  background-color: #c82333;
}

hr {
  margin: 20px 0;
}

.likes-section {
  margin: 10px 0;
}
.btn-like {
  background-color: #e3f2fd;
  border: 1px solid #2196f3;
  color: #0d47a1;
  border-radius: 20px;
  padding: 5px 15px;
  font-weight: bold;
}
.btn-like:hover {
  background-color: #bbdefb;
}

.comments-section {
  background-color: #f8f9fa;
  padding: 10px;
  border-radius: 8px;
  margin-top: 10px;
  border: 1px solid #e9ecef;
}
.comments-section h4 {
  margin-top: 0;
  font-size: 1em;
  color: #495057;
}

.comments-section ul {
  list-style: none;
  padding: 0;
  margin: 10px 0;
}
.comment-item {
  border-bottom: 1px solid #dee2e6;
  padding: 5px 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.9em;
}

.delete-comment-btn {
  cursor: pointer;
  opacity: 0.6;
  margin-left: 10px;
}
.delete-comment-btn:hover {
  opacity: 1;
  transform: scale(1.2);
}

.add-comment-box {
  display: flex;
  gap: 5px;
  margin-top: 10px;
}
.add-comment-box input {
  flex-grow: 1;
  padding: 8px;
  border: 1px solid #ced4da;
  border-radius: 4px;
}

.cat-select {
  padding: 4px 8px;
  border-radius: 15px;
  border: 1px solid #ccc;
  background-color: #f8f9fa;
  font-size: 0.85em;
  color: #333;
  cursor: pointer;
  max-width: 150px; /* Pour ne pas écraser le titre */
}

.cat-select:focus {
  outline: none;
  border-color: #007bff;
  background-color: #fff;
}
</style>
