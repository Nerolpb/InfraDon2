<script setup lang="ts">
import { onMounted, ref, onUnmounted } from 'vue'
import PouchDB from 'pouchdb'
import findPlugin from 'pouchdb-find'

PouchDB.plugin(findPlugin)

declare interface Post {
  _id: string
  _rev?: string
  title: string
  post_content: string
  attributes?: {
    creation_date: any
    category?: string
  }
}

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

  await createIndex()
  await fetchData()

  if (isOnline.value) {
    startLiveSync()
  }
}

const createIndex = async () => {
  try {
    await localDB.value.createIndex({
      index: {
        fields: ['attributes.category'],
        ddoc: 'idx-category',
      },
    })
  } catch (error) {
    console.error(error)
  }
}

const fetchData = async () => {
  if (!localDB.value) return

  try {
    const result = await localDB.value.allDocs({ include_docs: true, descending: true })
    const docs = result.rows
      .map((row: any) => row.doc)
      .filter((doc: any) => !doc._id.startsWith('_design/'))

    postsData.value = docs

    if (searchQuery.value) {
      performSearch()
    }
  } catch (error) {
    console.error(error)
  }
}

const addDocument = async () => {
  const categories = ['Tech', 'Science', 'Sport', 'Culture', 'Politique']
  const randomCategory = categories[Math.floor(Math.random() * categories.length)]

  const newPost: Partial<Post> = {
    _id: new Date().toISOString(),
    title: 'Nouveau post (local)',
    post_content: 'Contenu ajouté localement...',
    attributes: {
      creation_date: new Date().toISOString(),
      category: randomCategory,
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
  const categories = ['Tech', 'Science', 'Sport', 'Culture', 'Politique']
  const documents: any[] = []

  for (let i = 0; i < count; i++) {
    documents.push({
      _id: new Date().getTime() + '_' + i,
      title: `Doc factice #${i + 1}`,
      post_content: `Contenu généré...`,
      attributes: {
        creation_date: new Date().toISOString(),
        category: categories[Math.floor(Math.random() * categories.length)],
      },
    })
  }

  try {
    await localDB.value.bulkDocs(documents)
    fetchData()
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
          <h3>{{ post.title }}</h3>
          <small v-if="post.attributes?.category" class="cat-tag">{{
            post.attributes.category
          }}</small>
          <p>{{ post.post_content }}</p>

          <div class="actions">
            <button
              class="btn-update"
              @click="updateDocument(post._id, { title: post.title + ' (Modifié)' })"
            >
              ✏️ Modifier
            </button>
            <button class="btn-delete" @click="deleteDocument(post._id)">🗑️ Supprimer</button>
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

.post-card {
  border: 1px solid #ddd;
  padding: 15px;
  margin-bottom: 10px;
  border-radius: 8px;
  background-color: #fff; /* Assure un fond blanc pour la carte */
  color: #333; /* Assure un texte sombre par défaut */
}

.cat-tag {
  background: #e0e0e0; /* Gris un peu plus soutenu */
  color: #2c3e50; /* PROBLÈME RÉSOLU ICI : On force le texte en gris foncé */
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
  background-color: #dc3545; /* Rouge */
  color: white;
  border: 1px solid #c82333;
  margin-left: 10px; /* Petit espace avec le bouton d'à côté */
}

.btn-danger:hover {
  background-color: #c82333;
}
</style>
