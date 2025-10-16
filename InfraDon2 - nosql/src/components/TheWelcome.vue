<script lang="ts">
import { defineComponent, ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

interface Character {
  name: string
  age: number
  affiliation: string
  lightsaber: boolean
}

export default defineComponent({
  name: 'ListePersonnages',

  setup() {
    const characters = ref<Character[]>([])
    const storage = ref<any>(null)

    // Connexion à la base CouchDB
    const initDatabase = async () => {
      const db = new PouchDB('http://admin:admin@localhost:5984/database')
      storage.value = db
    }

    // Récupération des personnages depuis le JSON stocké en base
    const fetchData = async () => {
      if (!storage.value) return

      try {
        const result = await storage.value.allDocs({ include_docs: true })
        console.log('=> Données récupérées :', result)

        // Extraction des personnages dans tous les documents
        const allChars: Character[] = result.rows.flatMap(
          (row: any) => row.doc.characters || []
        )
        characters.value = allChars
      } catch (err) {
        console.error('Erreur lors de la récupération :', err)
      }
    }

    onMounted(async () => {
      await initDatabase()
      await fetchData()
    })

    return {
      characters
    }
  }
})
</script>

<template>
  <h1>Liste des personnages</h1>

  <article
    v-for="(char, index) in characters"
    :key="index"
    class="p-4 border-b"
  >
    <h2>{{ char.name }}</h2>
    <p>Âge : {{ char.age }}</p>
    <p>Affiliation : {{ char.affiliation }}</p>
    <p>Sabre laser : {{ char.lightsaber ? 'Oui' : 'Non' }}</p>
  </article>
</template>
