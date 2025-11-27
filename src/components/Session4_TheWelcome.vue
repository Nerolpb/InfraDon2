<script setup lang="ts">
import { onMounted, ref } from 'vue'
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

const storage = ref()
const postsData = ref<Post[]>([])
const localDB = ref()
const remoteDB = ref()
const syncStatus = ref('')
const isOnline = ref(true)
const syncHandler = ref<PouchDB.Replication.Sync<Post> | null>(null)
const searchQuery = ref('')
const filteredPosts = ref<Post[]>([])

const initDatabase = async () => {
  console.log('=> Initialisation des bases PouchDB')

  localDB.value = new PouchDB('infradon_local')
  storage.value = localDB.value
  remoteDB.value = new PouchDB('http://Nero:Penafiel29Snaky25@localhost:5984/infradon2')

  if (localDB.value && remoteDB.value) {
    console.log('Bases locale et distante prêtes.')
  }

  // Création de l'index sur la catégorie
  await createIndex()
}

const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données locales récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données locales :', error)
    })
}

// ÉTAPE 3: Ajouter un document (en local)
const addDocument = (): any => {
  const newPost: Partial<Post> = {
    title: 'Nouveau post (local)',
    post_content: 'Contenu ajouté localement...',
    attributes: {
      creation_date: new Date().toISOString(),
    },
  }

  storage.value
    .post(newPost)
    .then(() => {
      console.log('Document ajouté en local')
      fetchData()
    })
    .catch((error: any) => {
      console.error("=> Erreur lors de l'ajout du document :", error)
    })
}

const syncCollections = () => {
  syncStatus.value = 'Synchronisation en cours...'
  console.log('Lancement de la synchronisation bi-directionnelle...')

  localDB.value
    .sync(remoteDB.value, { retry: true })
    .on('change', (info: any) => {
      console.log('Synchro : changement détecté', info)
      syncStatus.value = `Changement détecté (${info.direction})...`
    })
    .on('complete', (info: any) => {
      syncStatus.value = 'Synchronisation terminée !'
      console.log('Synchro bi-directionnelle terminée', info)

      fetchData()
    })
    .on('error', (err: any) => {
      syncStatus.value = 'Erreur de synchronisation.'
      console.error('Erreur de synchro', err)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()

  fetchData()

  console.log('Lancement de la réplication initiale (pull)...')
  syncStatus.value = 'Récupération des données distantes...'

  remoteDB.value.replicate
    .to(localDB.value)
    .on('complete', () => {
      console.log('Réplication initiale (pull) terminée.')
      syncStatus.value = 'Données locales à jour.'
      // Maintenant que les données locales sont à jour, on rafraîchit la UI
      fetchData()
    })
    .on('error', (err: any) => {
      console.error('Erreur de réplication (pull)', err)
      syncStatus.value = 'Erreur de récupération des données.'
    })
})

const deleteDocument = (id: string): any => {
  storage.value
    .get(id)
    .then((document: any) => {
      return storage.value.remove(document)
    })
    .then(() => {
      console.log('Document supprimé localement')
      fetchData()
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la suppression du document :', error)
    })
}

const updateDocument = (id: string, updatedData: any): any => {
  storage.value
    .get(id)
    .then((document: any) => {
      // L'objet `updatedData` doit inclure ce que vous voulez changer
      // Par exemple: { title: 'New (Modified)' }
      const updatedDocument = { ...document, ...updatedData }
      return storage.value.put(updatedDocument)
    })
    .then(() => {
      console.log('Document mis à jour localement')
      fetchData()
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la mise à jour du document :', error)
    })
}

// ==========================================
// MODULE 1: RÉPLICATION & SIMULATION OFFLINE
// ==========================================

const toggleOnlineOffline = () => {
  isOnline.value = !isOnline.value

  if (isOnline.value) {
    // Passer en mode Online : démarrer la synchronisation live
    console.log('🟢 Mode ONLINE : Synchronisation active')
    syncStatus.value = 'En ligne - Synchronisation active'
    startLiveSync()
  } else {
    // Passer en mode Offline : arrêter la synchronisation
    console.log('🔴 Mode OFFLINE : Synchronisation arrêtée')
    syncStatus.value = 'Hors ligne - Modifications en local uniquement'
    stopLiveSync()
  }
}

const startLiveSync = () => {
  if (syncHandler.value) {
    console.log('Synchronisation déjà active')
    return
  }

  syncHandler.value = localDB.value
    .sync(remoteDB.value, {
      live: true,
      retry: true,
    })
    .on('change', (info: any) => {
      console.log('🔄 Synchro : changement détecté', info)
      syncStatus.value = `Synchronisation (${info.direction})...`
      fetchData()
      if (searchQuery.value) {
        performSearch()
      }
    })
    .on('paused', () => {
      console.log('⏸️  Synchro en pause (à jour)')
      syncStatus.value = 'En ligne - À jour'
    })
    .on('active', () => {
      console.log('▶️  Synchro active')
      syncStatus.value = 'En ligne - Synchronisation en cours'
    })
    .on('error', (err: any) => {
      console.error('❌ Erreur de synchro', err)
      syncStatus.value = 'Erreur de synchronisation'
    })
}

const stopLiveSync = () => {
  if (syncHandler.value) {
    syncHandler.value.cancel()
    syncHandler.value = null
    console.log('Synchronisation arrêtée')
  }
}
</script>

<template>
  <div class="container">
    <h1>PouchDB - Gestion avancée</h1>

    <!-- MODULE 1: Toggle Online/Offline -->
    <section class="sync-section">
      <h2>📡 Mode de connexion</h2>
      <button @click="toggleOnlineOffline" :class="{ online: isOnline, offline: !isOnline }">
        {{ isOnline ? '🟢 EN LIGNE' : '🔴 HORS LIGNE' }}
      </button>
      <p class="status">
        <em>{{ syncStatus }}</em>
      </p>
      <button @click="syncCollections" v-if="!isOnline">
        Synchroniser manuellement (Push/Pull)
      </button>
    </section>

    <hr />

    <!-- MODULE 2: Data Factory -->
    <section class="data-factory-section">
      <h2>📦 Génération de données</h2>
      <button @click="generateFakeData(50)">Générer 50 documents</button>
      <button @click="generateFakeData(100)">Générer 100 documents</button>
    </section>

    <hr />

    <!-- MODULE 2: Recherche par catégorie -->
    <section class="search-section">
      <h2>🔍 Recherche par catégorie</h2>
      <input
        v-model="searchQuery"
        @input="performSearch"
        type="text"
        placeholder="Rechercher une catégorie (Tech, Science, Sport, Culture, Politique)"
      />
      <p v-if="searchQuery">
        Résultats trouvés : <strong>{{ filteredPosts.length }}</strong>
      </p>
    </section>

    <hr />

    <!-- Ajouter un document -->
    <section class="actions-section">
      <h2>➕ Actions</h2>
      <button @click="addDocument">Ajouter un nouveau document (local)</button>
    </section>

    <hr />

    <!-- Affichage des documents -->
    <section class="posts-section">
      <h2>📄 Documents ({{ searchQuery ? 'Filtrés' : 'Tous' }})</h2>
      <div class="posts-grid">
        <article
          v-for="post in searchQuery ? filteredPosts : postsData"
          v-bind:key="post._id"
          class="post-card"
        >
          <h3>{{ post.title }}</h3>
          <p class="category" v-if="post.attributes?.category">🏷️ {{ post.attributes.category }}</p>
          <p class="content">{{ post.post_content }}</p>
          <p class="date" v-if="post.attributes?.creation_date">
            📅 {{ new Date(post.attributes.creation_date).toLocaleString() }}
          </p>
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
