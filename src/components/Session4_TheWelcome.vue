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

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://Nero:Penafiel29Snaky25@localhost:5984/infradon2')
  if (db) {
    console.log('Connecté à la collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

// Récupération des données
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

const deleteDocument = (): any => {
  if (postsData.value.length === 0) {
    console.warn('Aucun document à supprimer')
    return
  }

  const docToDelete = postsData.value[0]

  storage.value
    .remove(docToDelete)
    .then(() => {
      console.log('Document supprimé')
      fetchData()
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la suppression du document :', error)
    })
}
</script>

<template>
  <h1>Fetch Data</h1>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
  <button @click="addDocument">Ajouter un nouveau document</button>
  <button @click="deleteDocument">Supprimer un document</button>
</template>
