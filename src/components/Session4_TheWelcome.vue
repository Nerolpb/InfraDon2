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
const newPostCategory = ref('')
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
const newPostTitle = ref('')
const newPostContent = ref('')
const selectedFile = ref<File | null>(null)
const previewImageUrl = ref<string | null>(null)
const fileInputRef = ref<HTMLInputElement | null>(null)
const expandedComments = ref<{ [key: string]: boolean }>({})

const toggleComments = (postId: string) => {
  expandedComments.value[postId] = !expandedComments.value[postId]
}

const isCommentVisible = (postId: string, index: number, total: number) => {
  if (expandedComments.value[postId]) return true
  return index === total - 1
}

const handleFileUpload = (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files && target.files[0]) {
    const file = target.files[0]

    if (!file.type.startsWith('image/')) {
      alert('Veuillez sélectionner une image.')
      clearSelectedFile()
      return
    }

    selectedFile.value = file

    previewImageUrl.value = URL.createObjectURL(file)
  }
}

const clearSelectedFile = () => {
  selectedFile.value = null
  if (previewImageUrl.value) {
    URL.revokeObjectURL(previewImageUrl.value)
    previewImageUrl.value = null
  }

  if (fileInputRef.value) {
    fileInputRef.value.value = ''
  }
}

const getPostImageSrc = (post: any) => {
  if (!post._attachments) return null

  const firstKey = Object.keys(post._attachments)[0]
  if (!firstKey) return null

  const attachment = post._attachments[firstKey]

  return `data:${attachment.content_type};base64,${attachment.data}`
}

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
    const result = await localDB.value.allDocs({
      include_docs: true,
      descending: true,
      attachments: true,
    })

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
  if (!newPostTitle.value.trim()) {
    alert('Veuillez donner un titre à votre post.')
    return
  }

  const newPost: any = {
    _id: new Date().toISOString(),
    type: 'post',
    title: newPostTitle.value.trim(),
    post_content: newPostContent.value.trim() || 'Aucune description...',
    likes: 0,
    comments: [],
    attributes: {
      creation_date: new Date().toISOString(),

      category: newPostCategory.value || 'Général',
    },
  }

  if (selectedFile.value) {
    newPost._attachments = {
      [selectedFile.value.name]: {
        content_type: selectedFile.value.type,
        data: selectedFile.value,
      },
    }
  }

  try {
    await localDB.value.put(newPost)

    newPostTitle.value = ''
    newPostContent.value = ''
    newPostCategory.value = ''
    clearSelectedFile()

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

const editingPostId = ref<string | null>(null)

const editForm = ref({
  title: '',
  content: '',
  newFile: null as File | null,
  newFilePreview: null as string | null,
  deleteExistingImage: false,
})

const startEditingPost = (post: Post) => {
  editingPostId.value = post._id

  editForm.value = {
    title: post.title,
    content: post.post_content,
    newFile: null,
    newFilePreview: null,
    deleteExistingImage: false,
  }
}

const cancelEditingPost = () => {
  editingPostId.value = null
  if (editForm.value.newFilePreview) {
    URL.revokeObjectURL(editForm.value.newFilePreview)
  }
}

const handleEditFileUpload = (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files && target.files[0]) {
    const file = target.files[0]
    editForm.value.newFile = file
    editForm.value.newFilePreview = URL.createObjectURL(file)
    editForm.value.deleteExistingImage = false
  }
}

const savePostModifications = async (originalPost: Post) => {
  try {
    const doc: any = await localDB.value.get(originalPost._id)

    doc.title = editForm.value.title
    doc.post_content = editForm.value.content

    if (editForm.value.deleteExistingImage && !editForm.value.newFile) {
      doc._attachments = {} // On vide les pièces jointes
    }

    if (editForm.value.newFile) {
      doc._attachments = {
        [editForm.value.newFile.name]: {
          content_type: editForm.value.newFile.type,
          data: editForm.value.newFile,
        },
      }
    }

    await localDB.value.put(doc)

    cancelEditingPost()
    if (searchQuery.value) performSearch()
    else fetchData()
  } catch (error) {
    console.error('Erreur sauvegarde édition', error)
    alert('Erreur lors de la modification')
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

  const name = newCategoryName.value.trim()
  const newCat: Category = {
    _id: 'cat_' + Date.now(),
    type: 'category',
    name: name,
  }

  try {
    await localDB.value.put(newCat)
    await fetchData()

    newPostCategory.value = name
    newCategoryName.value = '' // Reset input
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
  const documents: any[] = []

  for (let i = 0; i < count; i++) {
    let catName: string

    if (categoriesData.value.length > 0) {
      const randomCat =
        categoriesData.value[Math.floor(Math.random() * categoriesData.value.length)]
      catName = randomCat.name
    } else {
      catName = `Cat-Auto-${Math.floor(Math.random() * 100)}`
    }

    documents.push({
      _id: new Date().getTime() + '_' + i,
      type: 'post',
      title: `Doc factice #${i + 1}`,
      post_content: `Ceci est une description générée automatiquement pour tester l'affichage...`,
      likes: Math.floor(Math.random() * 100),
      comments: [],
      attributes: {
        creation_date: new Date().toISOString(),
        categories: [catName],
      },
    })
  }

  try {
    await localDB.value.bulkDocs(documents)
    console.log(`✅ ${count} documents générés avec succès.`)
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
const editingCommentKey = ref<string | null>(null)

const editCommentInput = ref('')

const startEditingComment = (postId: string, index: number, currentText: string) => {
  editingCommentKey.value = `${postId}-${index}`
  editCommentInput.value = currentText
}

const cancelEditComment = () => {
  editingCommentKey.value = null
  editCommentInput.value = ''
}

const saveEditedComment = async (postId: string, index: number) => {
  if (!editCommentInput.value.trim()) return

  try {
    const doc = await localDB.value.get(postId)

    if (doc.comments && doc.comments[index]) {
      doc.comments[index].text = editCommentInput.value.trim()

      await localDB.value.put(doc)

      cancelEditComment()

      if (searchQuery.value) performSearch()
      else fetchData()
    }
  } catch (error) {
    console.error("Erreur lors de l'édition du commentaire", error)
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

    <section class="panel factory-panel">
      <h2>🏭 Création de Posts</h2>

      <div class="creation-form">
        <input v-model="newPostTitle" class="form-input" placeholder="Titre de votre post..." />

        <div class="category-toolbar">
          <select v-model="newPostCategory" class="form-select">
            <option value="" disabled selected>Choisir une catégorie...</option>
            <option v-for="cat in categoriesData" :key="cat._id" :value="cat.name">
              {{ cat.name }}
            </option>
          </select>

          <span class="divider">ou</span>

          <div class="quick-create-cat">
            <input
              v-model="newCategoryName"
              class="form-input sm"
              placeholder="Nouvelle cat..."
              @keyup.enter="addCategory"
            />
            <button class="btn btn-sm btn-secondary" @click="addCategory" title="Ajouter">+</button>
          </div>
        </div>

        <div class="media-upload-container">
          <input
            type="file"
            ref="fileInputRef"
            accept="image/*"
            style="display: none"
            @change="handleFileUpload"
          />

          <button
            v-if="!previewImageUrl"
            class="btn btn-secondary-outline attachment-btn"
            @click="fileInputRef?.click()"
          >
            📷 Ajouter une image
          </button>

          <div v-else class="preview-container">
            <img :src="previewImageUrl" class="preview-image" alt="Prévisualisation" />
            <button
              class="btn-icon-cancel remove-preview-btn"
              @click="clearSelectedFile"
              title="Supprimer l'image"
            >
              ❌
            </button>
            <span class="filename-preview">{{ selectedFile?.name }}</span>
          </div>
        </div>

        <textarea
          v-model="newPostContent"
          class="form-input textarea-creation"
          placeholder="Quoi de neuf aujourd'hui ?"
          rows="3"
        ></textarea>

        <div class="button-group creation-actions">
          <button class="btn btn-primary btn-block" @click="addDocument">✨ Publier le Post</button>
        </div>

        <div class="factory-actions">
          <p class="factory-label">🛠️ Zone de test (Générateur) :</p>
          <div class="factory-buttons-row">
            <button class="btn btn-secondary btn-sm" @click="generateFakeData(10)">
              🤖 +10 Fakes
            </button>
            <button class="btn btn-secondary btn-sm" @click="generateFakeData(50)">
              🤖 +50 Fakes
            </button>
          </div>
        </div>
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
          :class="{ 'is-editing': editingPostId === post._id }"
        >
          <div v-if="editingPostId === post._id" class="edit-mode-container">
            <div class="edit-header">
              <h3>Mode Édition</h3>
            </div>

            <div class="edit-body">
              <label>Titre :</label>
              <input v-model="editForm.title" class="form-input" />

              <label>Description :</label>
              <textarea
                v-model="editForm.content"
                class="form-input textarea-edit"
                rows="4"
              ></textarea>

              <div class="edit-media-section">
                <p class="label-text">Média :</p>

                <div
                  v-if="post._attachments && !editForm.deleteExistingImage && !editForm.newFile"
                  class="current-media-preview"
                >
                  <p>Image actuelle :</p>
                  <div class="mini-preview-wrapper">
                    <img :src="getPostImageSrc(post)" class="mini-preview" />
                    <button class="btn-icon-cancel" @click="editForm.deleteExistingImage = true">
                      🗑️ Supprimer
                    </button>
                  </div>
                </div>

                <div v-if="editForm.deleteExistingImage && !editForm.newFile" class="deleted-msg">
                  ⚠️ L'image sera supprimée à la sauvegarde.
                  <button class="btn-text" @click="editForm.deleteExistingImage = false">
                    Annuler
                  </button>
                </div>

                <div v-if="editForm.newFilePreview" class="new-media-preview">
                  <p>Nouvelle image :</p>
                  <div class="mini-preview-wrapper">
                    <img :src="editForm.newFilePreview" class="mini-preview new" />
                    <button
                      class="btn-icon-cancel"
                      @click="((editForm.newFile = null), (editForm.newFilePreview = null))"
                    >
                      ❌
                    </button>
                  </div>
                </div>

                <div class="upload-actions">
                  <label class="btn btn-secondary-outline btn-sm">
                    {{
                      (post._attachments && !editForm.deleteExistingImage) || editForm.newFile
                        ? "🔄 Remplacer l'image"
                        : '📷 Ajouter une image'
                    }}
                    <input
                      type="file"
                      @change="handleEditFileUpload"
                      accept="image/*"
                      style="display: none"
                    />
                  </label>
                </div>
              </div>
            </div>

            <div class="edit-footer">
              <button class="btn btn-primary" @click="savePostModifications(post)">
                💾 Sauvegarder
              </button>
              <button class="btn btn-ghost" @click="cancelEditingPost">Annuler</button>
            </div>
          </div>

          <template v-else>
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
              <div v-if="post._attachments" class="post-media">
                <img :src="getPostImageSrc(post)" alt="Image du post" loading="lazy" />
              </div>
              <p>{{ post.post_content }}</p>
            </div>

            <div class="card-interactions">
              <button
                @click="likePost(post._id)"
                class="btn-like"
                :class="{ liked: post.likes > 0 }"
              >
                ❤️ {{ post.likes || 0 }}
              </button>
            </div>

            <div class="comments-section">
              <div class="comments-header">
                <h4>💬 Commentaires ({{ post.comments?.length || 0 }})</h4>

                <button
                  v-if="post.comments && post.comments.length > 1"
                  class="btn-text-xs"
                  @click="toggleComments(post._id)"
                >
                  {{
                    expandedComments[post._id]
                      ? 'Masquer les anciens'
                      : `Voir les ${post.comments.length - 1} précédents...`
                  }}
                </button>
              </div>

              <ul class="comments-list">
                <li
                  v-for="(comment, index) in post.comments"
                  :key="index"
                  class="comment-item"
                  v-show="isCommentVisible(post._id, index, post.comments.length)"
                >
                  <div
                    v-if="editingCommentKey === `${post._id}-${index}`"
                    class="comment-edit-mode"
                  >
                    <input
                      v-model="editCommentInput"
                      class="form-input sm"
                      @keyup.enter="saveEditedComment(post._id, index)"
                    />
                    <div class="edit-actions">
                      <button class="btn-icon-check" @click="saveEditedComment(post._id, index)">
                        ✅
                      </button>
                      <button class="btn-icon-cancel" @click="cancelEditComment">❌</button>
                    </div>
                  </div>

                  <div v-else class="comment-display-mode">
                    <div class="comment-content">
                      <span class="comment-text">{{ comment.text }}</span>
                      <small class="comment-date">{{
                        new Date(comment.date).toLocaleTimeString([], {
                          hour: '2-digit',
                          minute: '2-digit',
                        })
                      }}</small>
                    </div>
                    <div class="comment-actions">
                      <button
                        class="btn-icon-edit"
                        @click="startEditingComment(post._id, index, comment.text)"
                      >
                        ✏️
                      </button>
                      <button class="btn-icon-delete" @click="deleteComment(post._id, index)">
                        ×
                      </button>
                    </div>
                  </div>
                </li>
              </ul>

              <div class="input-group sm">
                <input
                  v-model="newCommentText[post._id]"
                  class="form-input sm"
                  placeholder="Commenter..."
                  @keyup.enter="addCommentToPost(post._id)"
                />
                <button class="btn btn-sm btn-primary" @click="addCommentToPost(post._id)">
                  Envoyer
                </button>
              </div>
            </div>

            <div class="card-footer">
              <button class="btn-text" @click="startEditingPost(post)">✏️ Éditer</button>
              <button class="btn-text danger" @click="deleteDocument(post._id)">
                🗑️ Supprimer
              </button>
            </div>
          </template>
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

.category-toolbar {
  display: flex;
  align-items: center;
  gap: 10px;
  background: rgba(255, 255, 255, 0.03);
  padding: 8px;
  border-radius: 8px;
  border: 1px solid #333;
}

.form-select {
  flex-grow: 1;
  padding: 8px;
  border-radius: 6px;
  background: #2a2a2a;
  color: white;
  border: 1px solid #444;
}

.divider {
  color: #666;
  font-size: 0.9em;
  font-style: italic;
}

.quick-create-cat {
  display: flex;
  gap: 5px;
}

.quick-create-cat input {
  width: 120px;
}

.mini-tags-list {
  display: flex;
  gap: 5px;
  flex-wrap: wrap;
  justify-content: flex-end;
}

.mini-tag {
  font-size: 0.7em;
  background: #222;
  color: #888;
  padding: 2px 6px;
  border-radius: 4px;
  border: 1px solid #333;
  display: flex;
  align-items: center;
  gap: 4px;
}

.del-tag {
  cursor: pointer;
  color: #666;
}
.del-tag:hover {
  color: #ff4d4d;
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

.creation-form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.textarea-creation {
  resize: vertical;
  min-height: 80px;
}

.creation-actions {
  margin-top: 5px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.btn-block {
  width: 100%;
  padding: 12px;
  font-size: 1rem;
}

.fake-data-actions {
  display: flex;
  align-items: center;
  gap: 5px;
  margin-top: 10px;
}

.text-muted {
  color: #666;
  font-size: 0.8em;
}

.factory-actions {
  margin-top: 20px;
  padding-top: 15px;
  border-top: 1px dashed #333;
}

.factory-label {
  font-size: 0.8em;
  color: #666;
  margin-bottom: 8px;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.factory-buttons-row {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.btn-sm {
  padding: 6px 12px;
  font-size: 0.85em;
}

.btn-xs {
  padding: 2px 8px;
  font-size: 0.75em;
  border: 1px solid #333;
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

.comment-display-mode {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.comment-edit-mode {
  display: flex;
  gap: 10px;
  width: 100%;
  align-items: center;
}

.edit-actions,
.comment-actions {
  display: flex;
  gap: 5px;
  align-items: center;
}

.btn-icon-edit,
.btn-icon-check,
.btn-icon-cancel {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1rem;
  padding: 4px;
  border-radius: 4px;
  transition: background 0.2s;
}

.btn-icon-edit:hover {
  background: rgba(255, 255, 255, 0.1);
}
.btn-icon-check:hover {
  background: rgba(66, 184, 131, 0.2);
}
.btn-icon-cancel:hover {
  background: rgba(255, 77, 77, 0.2);
}

.media-upload-container {
  margin-bottom: 15px;
  padding: 10px;
  border: 1px dashed #444;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 60px;
  background: rgba(0, 0, 0, 0.1);
}

.attachment-btn {
  display: flex;
  align-items: center;
  gap: 8px;
}

.comments-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.comments-header h4 {
  margin: 0;
}

.btn-text-xs {
  background: none;
  border: none;
  color: #888;
  font-size: 0.8em;
  cursor: pointer;
  text-decoration: underline;
  padding: 0;
}

.btn-text-xs:hover {
  color: #42b883;
}

.preview-container {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.preview-image {
  max-height: 100px;
  border-radius: 6px;
  border: 2px solid #42b883;
}

.remove-preview-btn {
  position: absolute;
  top: -8px;
  right: -8px;
  background: #1e1e1e;
  border-radius: 50%;
  border: 2px solid #ff4d4d;
  color: #ff4d4d;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  padding: 0;
}

.filename-preview {
  font-size: 0.8em;
  color: #888;
  margin-top: 5px;
  max-width: 200px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.btn-secondary-outline {
  background: transparent;
  border: 1px solid #666;
  color: #ccc;
}
.btn-secondary-outline:hover {
  border-color: #aaa;
  color: #fff;
}

.post-media {
  margin-bottom: 15px;
  border-radius: 8px;
  overflow: hidden;
  background: #000;
}

.post-media img {
  width: 100%;
  height: auto;
  display: block;
  max-height: 400px;
  object-fit: contain;
}

.post-card.is-editing {
  border-color: #42b883;
  background: #252525;
}

.edit-mode-container {
  padding: 20px;
  animation: fadeIn 0.3s ease;
}

.edit-header h3 {
  margin-bottom: 20px;
  color: #42b883;
  border-bottom: 1px solid #333;
  padding-bottom: 10px;
}

.edit-body {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.edit-body label {
  font-size: 0.9em;
  color: #888;
  margin-bottom: -10px;
}

.textarea-edit {
  resize: vertical;
  min-height: 100px;
}

.edit-media-section {
  background: rgba(0, 0, 0, 0.2);
  padding: 15px;
  border-radius: 8px;
  border: 1px dashed #444;
  margin-top: 10px;
}

.label-text {
  font-size: 0.9em;
  color: #888;
  margin: 0 0 10px 0;
}

.mini-preview-wrapper {
  display: flex;
  align-items: center;
  gap: 10px;
  background: #1e1e1e;
  padding: 5px;
  border-radius: 6px;
  width: fit-content;
}

.mini-preview {
  height: 60px;
  width: auto;
  border-radius: 4px;
}
.mini-preview.new {
  border: 2px solid #42b883;
}

.deleted-msg {
  color: #ff4d4d;
  font-size: 0.9em;
  background: rgba(255, 77, 77, 0.1);
  padding: 8px;
  border-radius: 4px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.edit-footer {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #333;
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
</style>
