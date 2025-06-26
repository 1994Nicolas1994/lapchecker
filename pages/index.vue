<script setup>
import { useRouter } from 'vue-router'
import { createClient } from '@supabase/supabase-js'

const config = useRuntimeConfig()
const supabase = createClient(config.public.supabaseUrl, config.public.supabaseAnonKey)

const title = ref('')
const date = ref('')
const selectedTrainingId = ref('')
const trainings = ref([])
const router = useRouter()

onMounted(async () => {
    const { data } = await supabase.from('trainings').select('*').order('date', { ascending: false })
    trainings.value = data
})

async function createTraining() {
    const { data, error } = await supabase.from('trainings').insert([
        { title: title.value, date: date.value }
    ]).select().single()

    if (!error) {
        trainings.value.unshift(data)
        selectedTrainingId.value = data.id
        title.value = ''
        date.value = ''
    }
}

function goToGroups() {
    if (selectedTrainingId.value) {
        router.push(`/groups?training=${selectedTrainingId.value}`)
    }
}

function formatDate(dateStr) {
    return new Date(dateStr).toLocaleDateString('de-CH', {
        weekday: 'short',
        day: '2-digit',
        month: '2-digit'
    })
}
</script>

<template>
    <div class="max-w-md mx-auto p-4 space-y-6">
        <h2 class="text-xl font-bold">Training wählen oder erstellen</h2>

        <!-- Auswahl -->
        <div v-if="trainings.length">
            <select v-model="selectedTrainingId"
                class="w-full rounded-lg border p-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500">
                <option disabled value="">Training auswählen</option>
                <option v-for="t in trainings" :key="t.id" :value="t.id">
                    {{ t.title }} ({{ formatDate(t.date) }})
                </option>
            </select>
        </div>

        <!-- Erstellen -->
        <form @submit.prevent="createTraining" class="space-y-2">
            <input v-model="title" type="text" placeholder="Titel" class="w-full rounded-lg border p-2 text-sm"
                required />
            <input v-model="date" type="date" class="w-full rounded-lg border p-2 text-sm" required />
            <button class="w-full bg-blue-600 text-white py-2 rounded-lg hover:bg-blue-500 transition">Training
                erstellen</button>
        </form>

        <!-- Weiter Button -->
        <button v-if="selectedTrainingId" @click="goToGroups"
            class="w-full mt-4 bg-green-600 text-white py-2 rounded-lg font-semibold hover:bg-green-500 transition">
            ➡ Weiter zu Gruppen
        </button>
    </div>
</template>
