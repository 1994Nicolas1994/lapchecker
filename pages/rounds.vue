<template>
    <client-only>
        <div class="max-w-lg mx-auto flex flex-col gap-6 p-4">
            <!-- Header -->
            <h1 class="text-2xl font-bold text-center">Rundenplanung</h1>

            <!-- Gruppenwahl -->
            <select v-model="selectedGroupId"
                class="w-full rounded border px-3 py-2 text-base focus:ring-2 focus:ring-purple-500">
                <option v-for="g in groups" :key="g.id" :value="g.id" v-text="g.name"></option>
            </select>

            <!-- Draggable Rundenliste -->
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
                        <button @click="deleteRound(i)"
                            class="mt-2 text-red-500 text-sm inline-flex items-center gap-1">
                            🗑️ Löschen
                        </button>
                    </div>
                </template>
            </draggable>

            <!-- Action Buttons -->
            <div class="flex flex-col gap-3">
                <button @click="addRound"
                    class="w-full bg-green-600 hover:bg-green-500 text-white font-semibold py-2 rounded-md transition">➕
                    Runde
                    hinzufügen</button>
                <button @click="saveRounds"
                    class="w-full bg-blue-600 hover:bg-blue-500 text-white font-semibold py-2 rounded-md transition">💾
                    Runden
                    speichern</button>
                <button v-if="rounds.length" @click="goToTracking"
                    class="w-full bg-purple-600 hover:bg-purple-500 text-white font-semibold py-2 rounded-md transition">▶
                    Tracking
                    starten</button>
            </div>
        </div>
    </client-only>
</template>

<script setup>
import { useRouter } from 'vue-router'
import { useRuntimeConfig } from '#imports'
import { createClient } from '@supabase/supabase-js'
import { onMounted, ref, reactive, watch } from 'vue'
import draggable from 'vuedraggable'

/* Supabase init */
const cfg = useRuntimeConfig()
const supabase = createClient(cfg.public.supabaseUrl, cfg.public.supabaseAnonKey)

/* State */
const groups = ref([])
const pacePresets = ref([])
const rounds = reactive([])
const selectedGroupId = ref(null)
const router = useRouter()

/* Lifecycle */
onMounted(async () => {
    const { data: g } = await supabase.from('groups').select('*')
    const { data: pp } = await supabase.from('pace_presets').select('*')
    groups.value = g || []
    pacePresets.value = pp || []
    if (groups.value.length) selectedGroupId.value = groups.value[0].id
})

watch(selectedGroupId, async () => {
    rounds.splice(0)
    if (!selectedGroupId.value) return
    const { data } = await supabase
        .from('rounds')
        .select('*')
        .eq('group_id', selectedGroupId.value)
        .order('position')
        ; (data || []).forEach(r => {
            const preset = pacePresets.value.find(
                p => p.group_id === selectedGroupId.value && p.pace_s === r.pace_s
            )?.pace_type ?? 'custom'
            rounds.push({
                preset,
                customPace: { min: Math.floor(r.pace_s / 60), sec: r.pace_s % 60 }
            })
        })
})

const minSecToSec = o => Number(o.min) * 60 + Number(o.sec)

/* CRUD */
function addRound() {
    rounds.push({ preset: 'custom', customPace: { min: 0, sec: 0 } })
}
function deleteRound(i) {
    rounds.splice(i, 1)
}

async function saveRounds() {
    const gid = selectedGroupId.value
    if (!gid) return
    await supabase.from('rounds').delete().eq('group_id', gid)
    for (const [i, r] of rounds.entries()) {
        const pace_s =
            r.preset === 'custom'
                ? minSecToSec(r.customPace)
                : pacePresets.value.find(
                    p => p.group_id === gid && p.pace_type === r.preset
                )?.pace_s
        if (pace_s == null) continue
        await supabase
            .from('rounds')
            .insert([{ group_id: gid, position: i, pace_s }])
    }
    alert('✅ Runden gespeichert')
}

function goToTracking() {
    router.push(`/tracking?group=${selectedGroupId.value}`)
}
</script>
