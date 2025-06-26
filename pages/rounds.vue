<script setup>
import { useRouter } from 'vue-router'
import { createClient } from '@supabase/supabase-js'
import { onMounted, ref, reactive, watch } from 'vue'
import draggable from 'vuedraggable'

const config = useRuntimeConfig()
const supabase = createClient(config.public.supabaseUrl, config.public.supabaseAnonKey)

const groups = ref([])
const selectedGroupId = ref(null)
const selectedGroup = ref(null)
const rounds = reactive([])
const pacePresets = ref([])

const router = useRouter()

onMounted(async () => {
    await fetchGroups()
})

async function fetchGroups() {
    const { data: groupData } = await supabase.from('groups').select('*')
    const { data: presetData } = await supabase.from('pace_presets').select('*')
    groups.value = groupData
    pacePresets.value = presetData
    if (groupData.length > 0) {
        selectedGroupId.value = groupData[0].id
    }
}

watch(selectedGroupId, async () => {
    selectedGroup.value = groups.value.find(g => g.id === selectedGroupId.value)
    rounds.splice(0, rounds.length)
    await fetchRoundsForGroup()
})

async function fetchRoundsForGroup() {
    if (!selectedGroupId.value) return

    const { data: savedRounds } = await supabase
        .from('rounds')
        .select('*')
        .eq('group_id', selectedGroupId.value)
        .order('position', { ascending: true })

    rounds.splice(0, rounds.length, ...savedRounds.map(r => ({
        preset: detectPreset(r.pace_s),
        customPace: secondsToParts(r.pace_s)
    })))
}

function detectPreset(pace_s) {
    const match = pacePresets.value.find(p =>
        p.group_id === selectedGroupId.value && p.pace_s === pace_s
    )
    return match?.pace_type ?? 'custom'
}

function secondsToParts(s) {
    const sec = s % 60
    const min = Math.floor(s / 60)
    return { min, sec }
}

function paceToSeconds(pace) {
    const m = Number(pace.min)
    const s = Number(pace.sec)
    return isNaN(m) || isNaN(s) ? null : m * 60 + s
}

function addRound() {
    rounds.push({
        preset: 'custom',
        customPace: { min: 0, sec: 0 }
    })
}

function deleteRound(index) {
    rounds.splice(index, 1)
}

function getPresetPace(presetType) {
    const preset = pacePresets.value.find(
        p => p.group_id === selectedGroupId.value && p.pace_type === presetType
    )
    return preset?.pace_s ?? null
}

async function saveRounds() {
    if (!selectedGroupId.value) return
    await supabase.from('rounds').delete().eq('group_id', selectedGroupId.value)

    for (const [i, round] of rounds.entries()) {
        let pace_s = null
        if (round.preset !== 'custom') {
            pace_s = getPresetPace(round.preset)
        } else {
            pace_s = paceToSeconds(round.customPace)
        }

        if (pace_s === null) continue

        await supabase.from('rounds').insert([{
            group_id: selectedGroupId.value,
            position: i,
            pace_s
        }])
    }

    alert('✅ Runden gespeichert')
}

function goToTracking() {
    router.push(`/tracking?group=${selectedGroupId.value}`)
}
</script>

<template>
    <div class="max-w-xl mx-auto p-4 space-y-4">
        <h2 class="text-xl font-bold mb-4">Rundenplanung</h2>

        <label class="block mb-1 text-sm font-medium">Gruppe wählen:</label>
        <select v-model="selectedGroupId" class="w-full border p-2 rounded text-sm mb-4">
            <option v-for="g in groups" :key="g.id" :value="g.id">{{ g.name }}</option>
        </select>

        <draggable v-model="rounds" item-key="index" class="space-y-2" handle=".handle">
            <template #item="{ element, index }">
                <div class="p-3 border rounded bg-white shadow-sm flex flex-col gap-2">
                    <div class="flex justify-between items-center">
                        <strong>Runde {{ index + 1 }}</strong>
                        <span class="cursor-move handle text-gray-400">↕</span>
                    </div>

                    <div class="flex gap-2 items-center text-sm">
                        <label>Tempo:</label>
                        <select v-model="element.preset" class="border rounded p-1 text-sm">
                            <option value="langsam">Langsam</option>
                            <option value="mittel">Mittel</option>
                            <option value="schnell">Schnell</option>
                            <option value="custom">Eigene Pace</option>
                        </select>

                        <template v-if="element.preset === 'custom'">
                            <select v-model.number="element.customPace.min" class="border rounded p-1">
                                <option v-for="m in 10" :key="m" :value="m">{{ m }} min</option>
                            </select>
                            <select v-model.number="element.customPace.sec" class="border rounded p-1">
                                <option v-for="s in [0, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 55]" :key="s" :value="s">{{ s }} sek
                                </option>
                            </select>
                        </template>
                    </div>

                    <button @click="deleteRound(index)" class="text-red-500 text-sm mt-1 self-end">🗑️ Löschen</button>
                </div>
            </template>
        </draggable>

        <button @click="addRound" class="w-full bg-green-600 text-white py-2 rounded hover:bg-green-500">
            ➕ Runde hinzufügen
        </button>

        <button @click="saveRounds" class="w-full bg-blue-600 text-white py-2 rounded hover:bg-blue-500">
            💾 Runden speichern
        </button>

        <button v-if="rounds.length > 0" @click="goToTracking"
            class="w-full bg-purple-600 text-white py-2 rounded hover:bg-purple-500">
            ▶ Weiter zum Tracking
        </button>
    </div>
</template>
