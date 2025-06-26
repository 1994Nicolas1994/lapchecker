<script setup>
import { useRouter } from 'vue-router'
import { createClient } from '@supabase/supabase-js'
import { onMounted, ref, reactive, watch } from 'vue'
import draggable from 'vuedraggable'

/* ──────────────────  Supabase  ────────────────── */
const cfg = useRuntimeConfig()
const supabase = createClient(cfg.public.supabaseUrl, cfg.public.supabaseAnonKey)

/* ──────────────────  State  ────────────────── */
const groups = ref([])
const pacePresets = ref([])
const rounds = reactive([])
const selectedGroupId = ref(null)
const router = useRouter()

/* ──────────────────  Lifecycle  ────────────────── */
onMounted(async () => {
    const { data: g } = await supabase.from('groups').select('*')
    const { data: pp } = await supabase.from('pace_presets').select('*')
    groups.value = g
    pacePresets.value = pp
    if (g.length) selectedGroupId.value = g[0].id
})

watch(selectedGroupId, async () => {
    rounds.splice(0)           // reset
    const { data } = await supabase
        .from('rounds')
        .select('*')
        .eq('group_id', selectedGroupId.value)
        .order('position')
    data.forEach(r => rounds.push(mapDBRound(r)))
})

/* ──────────────────  Helpers  ────────────────── */
const mapDBRound = r => ({
    preset: detectPreset(r.pace_s),
    customPace: secToMinSec(r.pace_s)
})

const detectPreset = s =>
    pacePresets.value.find(p => p.group_id === selectedGroupId.value && p.pace_s === s)?.pace_type ?? 'custom'

const secToMinSec = s => ({ min: Math.floor(s / 60), sec: s % 60 })
const minSecToSec = obj => Number(obj.min) * 60 + Number(obj.sec)

/* ──────────────────  Round CRUD  ────────────────── */
const addRound = () => rounds.push({ preset: 'custom', customPace: { min: 0, sec: 0 } })
const deleteRound = i => rounds.splice(i, 1)

const presetToSec = type =>
    pacePresets.value.find(p => p.group_id === selectedGroupId.value && p.pace_type === type)?.pace_s ?? null

async function saveRounds() {
    if (!selectedGroupId.value) return
    await supabase.from('rounds').delete().eq('group_id', selectedGroupId.value)

    for (const [i, r] of rounds.entries()) {
        const pace_s = r.preset === 'custom' ? minSecToSec(r.customPace) : presetToSec(r.preset)
        if (!pace_s) continue
        await supabase.from('rounds').insert([{ group_id: selectedGroupId.value, position: i, pace_s }])
    }
    alert('✅ Runden gespeichert')
}

const goToTracking = () => router.push(`/tracking?group=${selectedGroupId.value}`)
</script>

<template>
    <div class="max-w-lg mx-auto flex flex-col gap-6 p-4">
        <!-- Header -->
        <h1 class="text-2xl font-bold text-center">Rundenplanung</h1>

        <!-- Gruppenwahl -->
        <select v-model="selectedGroupId"
            class="w-full rounded border px-3 py-2 text-base focus:ring-2 focus:ring-purple-500">
            <option v-for="g in groups" :key="g.id" :value="g.id">{{ g.name }}</option>
        </select>

        <!-- Rundenliste (drag) -->
        <draggable v-model="rounds" handle=".drag" item-key="index" class="flex flex-col gap-3">
            <template #item="{ element: r, index: i }">
                <div class="rounded-lg bg-white shadow p-3">
                    <!-- Head -->
                    <div class="flex items-center justify-between mb-2">
                        <span class="font-semibold">Runde {{ i + 1 }}</span>
                        <span class="drag cursor-grab text-gray-400 text-lg">⋮⋮</span>
                    </div>

                    <!-- Pace Row -->
                    <div class="flex flex-wrap gap-2 items-center text-sm">
                        <select v-model="r.preset" class="border rounded px-1 py-0.5">
                            <option value="langsam">Langsam</option>
                            <option value="mittel">Mittel</option>
                            <option value="schnell">Schnell</option>
                            <option value="custom">Custom</option>
                        </select>

                        <template v-if="r.preset === 'custom'">
                            <input v-model.number="r.customPace.min" type="number" min="0" max="10"
                                class="w-14 border rounded px-1 py-0.5 text-center" />min
                            <input v-model.number="r.customPace.sec" type="number" min="0" max="59"
                                class="w-14 border rounded px-1 py-0.5 text-center" />s
                        </template>
                    </div>

                    <!-- Delete -->
                    <button @click="deleteRound(i)" class="mt-2 text-red-500 text-sm inline-flex items-center gap-1">
                        <span class="i-heroicons-trash-20-solid"></span> Löschen
                    </button>
                </div>
            </template>
        </draggable>

        <!-- Action Buttons -->
        <div class="flex flex-col gap-3">
            <button @click="addRound" class="btn w-full bg-green-600 hover:bg-green-500">➕ Runde hinzufügen</button>
            <button @click="saveRounds" class="btn w-full bg-blue-600 hover:bg-blue-500">💾 Runden speichern</button>
            <button v-if="rounds.length" @click="goToTracking" class="btn w-full bg-purple-600 hover:bg-purple-500">▶
                Tracking
                starten</button>
        </div>
    </div>
</template>
