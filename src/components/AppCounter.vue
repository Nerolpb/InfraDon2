<script setup lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
import { onMounted } from 'vue'

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

const storage = ref()

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://Nero:Penafiel29Snaky25@localhost:5984/infra_53_1527')
  if (db) {
    console.log('Connected to collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}

//DATA => Model
//mise à jour automatique du DOM lorsque la valeur de count change
const count = ref(0)

//METHODS => Controller
const increment = () => {
  count.value++
}

const decrement = () => {
  count.value--
}
</script>

<!-- TEMPLATE => View -->
<template>
  <div>
    <p>compteur : {{ count }}</p>
    <button @click="increment">increment</button>
    <button @click="decrement">decrement</button>
  </div>
</template>

<style scoped>
button {
  margin-right: 10px;
  background-color: black;
  color: white;
  border: none;
  padding: 10px;
}
</style>
