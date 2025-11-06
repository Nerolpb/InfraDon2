<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  title: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

const storage = ref()
const postsData = ref<Post[]>([])
const localDB = ref()
const remoteDB = ref()

const initDatabase = () => {
  console.log('=> Initialisation des bases PouchDB')

  localDB.value = new PouchDB('infradon_local')
  storage.value = localDB.value
  remoteDB.value = new PouchDB('http://Nero:Penafiel29Snaky25@localhost:5984/infradon2')

  if (localDB.value && remoteDB.value) {
    console.log('Bases locale et distante prêtes.')
  }
}

const replicateDataLocal = () => {
  localDB.value.replicate(remoteDB.value, { live: true, retry: true }).on('change', (info: any) => {
    console.log('=> Réplication locale - changement détecté :', info)
    fetchData()
  })
}

const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}

const addDocument = (): any => {
  storage.value
    .post({
      title: 'New',
    })
    .then(() => {
      console.log('Document ajouté')
      fetchData()
    })
    .catch((error: any) => {
      console.error("=> Erreur lors de l'ajout du document :", error)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
console.log(postsData.value)

const deleteDocument = (id: string): any => {
  storage.value
    .get(id)
    .then((document) => {
      return storage.value.remove(document)
    })
    .then(() => {
      console.log('Document supprimé')
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
      const updatedDocument = { ...document, ...updatedData }
      return storage.value.put(updatedDocument)
    })
    .then(() => {
      console.log('Document mis à jour')
      fetchData()
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la mise à jour du document :', error)
    })
}
</script>

<template>
  <h1>Fetch Data</h1>
  <button @click="addDocument">Ajouter un nouveau document</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.post_content }}</p>
    <button @click="deleteDocument(post._id)">Supprimer un document</button>
    <button @click="updateDocument(post._id, { title: 'New (Modified)' })">
      Mettre à jour un document
    </button>
  </article>
</template>
