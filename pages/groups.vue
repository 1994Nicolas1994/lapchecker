<script setup>
import { useRoute, useRouter } from 'vue-router'
import { createClient } from '@supabase/supabase-js'
import { reactive, onMounted } from 'vue'

const config = useRuntimeConfig()
const supabase = createClient(config.public.supabaseUrl, config.public.supabaseAnonKey)

const route = useRoute()
const router = useRouter()
const trainingId = route.query.training

const groupName = ref('')
const lapLength = ref('')
const groups = reactive([])

onMounted(() => {
    if (!trainingId) {
        router.push('/')
    } else {
        fetchGroups()
    }
})

function toParts(sec) {
    sec = Number(sec) || 0
    return {
        min: Math.floor(sec / 60),
        sec: sec % 60
    }
}

async function fetchGroups() {
    const { data: groupData } = await supabase
        .from('groups')
        .select('*')
        .eq('training_id', trainingId)

    const { data: paceData } = await supabase
        .from('pace_presets')
        .select('*')

    groups.splice(0, groups.length, ...groupData.map((g) => {
        const getPace = (type) =>
            toParts(paceData.find(p => p.group_id === g.id && p.pace_type === type)?.pace_s ?? 0)

        return reactive({
            ...g,
            pace: {
                langsam: getPace('langsam'),
                mittel: getPace('mittel'),
                schnell: getPace('schnell')
            }
        })
    }))
}

async function addGroup() {
    const { data, error } = await supabase
        .from('groups')
        .insert([{ training_id: trainingId, name: groupName.value, lap_length_m: parseInt(lapLength.value) }])
        .select()
        .single()

    if (!error) {
        groups.push(reactive({
            ...data,
            pace: {
                langsam: { min: 0, sec: 0 },
                mittel: { min: 0, sec: 0 },
                schnell: { min: 0, sec: 0 }
            }
        }))
        groupName.value = ''
        lapLength.value = ''
    }
}

async function deleteGroup(id) {
    await supabase.from('groups').delete().eq('id', id)
    await supabase.from('pace_presets').delete().eq('group_id', id)
    const index = groups.findIndex((g) => g.id === id)
    if (index !== -1) groups.splice(index, 1)
}

function toSeconds({ min, sec }) {
    const m = Number(min)
    const s = Number(sec)
    if (isNaN(m) || isNaN(s) || m < 0 || s < 0 || s >= 60) return null
    return m * 60 + s
}

async function saveGroupAndPaces(group) {
    const { error: updateError } = await supabase
        .from('groups')
        .update({ name: group.name, lap_length_m: group.lap_length_m })
        .eq('id', group.id)

    if (updateError) return console.error('Fehler beim Speichern der Gruppe:', updateError)

    for (const type of ['langsam', 'mittel', 'schnell']) {
        const seconds = toSeconds(group.pace[type])
        if (seconds === null) {
            console.warn(`⚠️ Ungültige Pace für ${type}`)
            continue
        }

        const { data: existing } = await supabase
            .from('pace_presets')
            .select('id')
            .eq('group_id', group.id)
            .eq('pace_type', type)
            .maybeSingle()

        if (existing) {
            await supabase
                .from('pace_presets')
                .update({ pace_s: seconds })
                .eq('id', existing.id)
        } else {
            await supabase
                .from('pace_presets')
                .insert([{ group_id: group.id, pace_type: type, pace_s: seconds }])
        }
    }

    alert('✅ Gespeichert')
    await fetchGroups()
}

function goToRounds() {
    router.push(`/rounds?training=${trainingId}`)
}
</script>

<template>
    <div class="max-w-md mx-auto p-4 space-y-6">
        <h2 class="text-xl font-bold">Gruppen verwalten</h2>

        <form @submit.prevent="addGroup" class="space-y-2">
            <input v-model="groupName" placeholder="Gruppenname" class="w-full border p-2 rounded text-sm" />
            <input v-model="lapLength" type="number" placeholder="Rundenlänge (m)"
                class="w-full border p-2 rounded text-sm" />
            <button type="submit" class="w-full bg-green-600 text-white py-2 rounded hover:bg-green-500">
                ➕ Gruppe hinzufügen
            </button>
        </form>

        <div v-if="groups.length" class="space-y-4">
            <div v-for="group in groups" :key="group.id" class="border p-4 rounded-lg bg-gray-50">
                <input v-model="group.name" placeholder="Name" class="border p-1 rounded text-sm mb-2 w-full" />
                <input v-model.number="group.lap_length_m" type="number" placeholder="Rundenlänge (m)"
                    class="border p-1 rounded text-sm mb-2 w-full" />

                <div class="grid gap-2 text-sm">
                    <div v-for="type in ['langsam', 'mittel', 'schnell']" :key="type" class="flex gap-2 items-center">
                        <label class="w-20 capitalize">{{ type }}</label>
                        <select v-model.number="group.pace[type].min" class="border rounded p-1">
                            <option v-for="m in 10" :value="m">{{ m }} min</option>
                        </select>
                        <select v-model.number="group.pace[type].sec" class="border rounded p-1">
                            <option v-for="s in Array.from({ length: 12 }, (_, i) => i * 5)" :value="s">
                                {{ s.toString().padStart(2, '0') }} sek
                            </option>
                        </select>
                    </div>
                </div>

                <button @click="saveGroupAndPaces(group)"
                    class="mt-3 bg-blue-600 text-white px-3 py-2 rounded text-sm hover:bg-blue-700 w-full">
                    💾 Speichern
                </button>

                <button @click="deleteGroup(group.id)"
                    class="mt-2 bg-red-500 text-white px-3 py-1 rounded text-sm hover:bg-red-600 w-full">
                    Gruppe löschen
                </button>
            </div>
        </div>

        <button v-if="groups.length" @click="goToRounds"
            class="w-full mt-6 bg-purple-600 text-white py-2 rounded-lg font-semibold hover:bg-purple-500 transition">
            ➡ Weiter zur Rundenplanung
        </button>
    </div>
</template>
