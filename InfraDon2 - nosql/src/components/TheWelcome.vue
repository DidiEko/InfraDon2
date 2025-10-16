<script lang="ts">
import { defineComponent, ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

// Définition du type de données
interface Character {
  name: string
  age: number
  affiliation: string
  lightsaber: boolean
}

export default defineComponent({
  name: 'TestDatabase',

  setup() {
    const counter = ref(110)
    const characters = ref<Character[]>([])
    const storage = ref<any>(null)

    // Connexion à la base de données
    const initDatabase = async () => {
      console.log('=> Connexion à la base de données')
      const db = new PouchDB('http://admin:admin@localhost:5984/database')
      storage.value = db

      try {
        const info = await db.info()
        console.log('Informations de la base :', info)
      } catch (err) {
        console.error('Erreur lors de la connexion :', err)
      }
    }

    // Récupération des données
    const fetchData = async () => {
      if (!storage.value) return

      try {
        const result = await storage.value.allDocs({ include_docs: true })
        console.log('=> Données récupérées :', result)

        const allCharacters: Character[] = result.rows.flatMap(
          (row: any) => row.doc.characters || []
        )
        characters.value = allCharacters
      } catch (err) {
        console.error('Erreur lors de la récupération :', err)
      }
    }

    const increment = () => {
      counter.value++
    }

    // Initialisation automatique au montage
    onMounted(async () => {
      await initDatabase()
      await fetchData()
    })

    // Variables et fonctions exposées au template
    return {
      counter,
      characters,
      increment
    }
  }
})
</script>

<template>
  <h1>C'est moi Cindy</h1>

  <p>Counter {{ counter }}</p>
  <button @click="increment">Increment</button>

  <h1>Liste des personnages</h1>
  <article v-for="(char, index) in characters" :key="index" class="p-4 border-b">
    <h2>{{ char.name }}</h2>
    <p>Âge : {{ char.age }}</p>
    <p>Affiliation : {{ char.affiliation }}</p>
    <p>Sabre laser : {{ char.lightsaber ? 'Oui' : 'Non' }}</p>
  </article>
</template>
