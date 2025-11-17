<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  _rev?: string
  title: string
  post_content: string
  attributes?: {
    creation_date: any
  }
}

const storage = ref()
const postsData = ref<Post[]>([])
const localDB = ref()
const remoteDB = ref()
const syncStatus = ref('')

const initDatabase = () => {
  console.log('=> Initialisation des bases PouchDB')

  localDB.value = new PouchDB('infradon_local')
  storage.value = localDB.value
  remoteDB.value = new PouchDB('http://Nero:Penafiel29Snaky25@localhost:5984/infradon2')

  if (localDB.value && remoteDB.value) {
    console.log('Bases locale et distante prêtes.')
  }
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
</script>

<template>
  <h1>Fetch Data</h1>

  <button @click="syncCollections">Synchroniser (Push/Pull)</button>
  <p>
    <em>{{ syncStatus }}</em>
  </p>

  <hr />

  <button @click="addDocument">Ajouter un nouveau document (local)</button>

  <hr />

  <article v-for="post in postsData" v-bind:key="post._id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.post_content }}</p>
    <button @click="deleteDocument(post._id)">Supprimer (local)</button>
    <button @click="updateDocument(post._id, { title: post.title + ' (Modifié)' })">
      Mettre à jour (local)
    </button>
  </article>
</template>
