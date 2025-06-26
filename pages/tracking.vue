<template>
    <client-only>
        <div class="p-4 space-y-4 bg-gray-100 min-h-screen">
            <!-- Gruppen-Filter -->
            <div class="max-w-md mx-auto">
                <label class="block mb-2 font-semibold">Gruppe wählen:</label>
                <select v-model="selectedGroupId" class="w-full border p-2 rounded">
                    <option value="all">Alle Gruppen</option>
                    <option v-for="g in groups" :key="g.id" :value="g.id" v-text="g.name" />
                </select>
            </div>

            <!-- Gruppe Cards -->
            <div class="flex gap-4 overflow-x-auto">
                <div v-for="g in groups" :key="g.id" v-show="selectedGroupId === 'all' || selectedGroupId === g.id"
                    class="min-w-[300px] bg-white shadow rounded-lg p-4 flex flex-col">
                    <h2 class="text-lg font-bold mb-3">{{ g.name }}</h2>

                    <!-- Controls -->
                    <div class="flex flex-col gap-2 mb-4">
                        <button @click="toggleTimer(g.id)"
                            class="w-full py-2 bg-blue-600 hover:bg-blue-700 text-white rounded">
                            {{ timers[g.id].isRunning ? '⏸ Pause' : '▶ Start' }}
                        </button>
                        <button @click="lap(g.id)"
                            class="w-full py-2 bg-green-600 hover:bg-green-700 text-white rounded">🏁 Lap</button>
                        <button @click="undoLap(g.id)"
                            class="w-full py-2 bg-yellow-500 hover:bg-yellow-600 text-white rounded">↩️ Undo
                            Lap</button>
                        <button @click="resetTimer(g.id)"
                            class="w-full py-2 bg-red-600 hover:bg-red-700 text-white rounded">🔄 Reset</button>
                    </div>

                    <!-- Live-Timer -->
                    <p class="text-sm">
                        <strong>Gesamt:</strong> {{ formatTime(totalElapsed(g.id)) }}
                    </p>
                    <p class="text-sm mb-4">
                        <strong>Aktuelle Runde:</strong> {{ formatTime(getCurrentLapTime(g.id)) }}
                    </p>

                    <!-- Rundenliste -->
                    <h3 class="font-semibold mb-2">Runden</h3>
                    <ul class="flex-1 overflow-y-auto text-sm space-y-1">
                        <li v-for="(r, idx) in staticRounds[g.id] || []" :key="idx"
                            :class="idx === timers[g.id].lapTimes.length ? 'font-bold bg-blue-50' : ''"
                            class="p-2 rounded">
                            Runde {{ idx + 1 }} –
                            <span v-if="idx < timers[g.id].lapTimes.length">
                                {{ formatTime(timers[g.id].lapTimes[idx]) }}
                            </span>
                            <span v-else-if="idx === timers[g.id].lapTimes.length">
                                {{ formatTime(getCurrentLapTime(g.id)) }} <em>(läuft)</em>
                            </span>
                            <span v-else>--</span>
                            – Ziel: {{ formatPace(r.pace_s) }},
                            <span v-if="idx < timers[g.id].lapTimes.length">
                                Diff: {{ paceDiff(timers[g.id].lapTimes[idx], r.pace_s) }}
                            </span>
                        </li>
                    </ul>

                    <!-- Total Δ -->
                    <p class="mt-3 text-sm font-semibold text-right">
                        Total Δ: {{ totalDelta(g.id) }} s
                    </p>
                </div>
            </div>
        </div>
    </client-only>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { createClient } from '@supabase/supabase-js'
import { useRuntimeConfig } from '#imports'

/* Supabase */
const cfg = useRuntimeConfig()
const supabase = createClient(cfg.public.supabaseUrl, cfg.public.supabaseAnonKey)

/* State */
const groups = ref([])
const staticRounds = ref({})
const timers = ref({})
const selectedGroupId = ref('all')
const bahnlaenge = 400

/* Anzeige-Helpers */
const formatTime = ms => {
    if (ms == null) return '--:--'
    const s = Math.floor(ms / 1000), m = Math.floor(s / 60)
    return `${m}:${String(s % 60).padStart(2, '0')}`
}
const formatPace = p => {
    if (p == null) return '--'
    const mm = Math.floor(p / 60), ss = String(p % 60).padStart(2, '0')
    return `${mm}:${ss}min/km (${Math.round(p * bahnlaenge / 1000)}s)`
}
const paceDiff = (ms, p) => {
    if (ms == null || p == null) return ''
    const d = Math.round(ms / 1000 - p * bahnlaenge / 1000)
    return `${d > 0 ? '+' : ''}${d}s`
}

/* Persistenz-Helpers */
async function upsertState(id, t) {
    await supabase.from('session_state').upsert({
        group_id: id,
        is_running: t.isRunning,
        current_lap_start: t.currentLapStart,
        pause_start: t.pauseStart ?? null
    })
}
async function insertLap(id, idx, dur, pace) {
    await supabase.from('session_laps').insert({
        group_id: id, lap_index: idx, duration_ms: dur, pace_s: pace
    })
}
async function deleteLap(id, idx) {
    await supabase.from('session_laps')
        .delete().eq('group_id', id).eq('lap_index', idx)
}
async function resetAll(id) {
    await supabase.from('session_laps').delete().eq('group_id', id)
    await supabase.from('session_state').delete().eq('group_id', id)
}

/* Lifecycle */
onMounted(async () => {
    // Gruppen & Runden
    const { data: g } = await supabase.from('groups').select('*')
    groups.value = g || []
    for (const grp of groups.value) {
        const { data: r } = await supabase
            .from('rounds').select('*')
            .eq('group_id', grp.id).order('position')
        staticRounds.value[grp.id] = r || []
        timers.value[grp.id] = {
            isRunning: false,
            currentLapStart: null,
            pauseStart: null,
            lapTimes: [],
            _now: 0
        }
    }

    // Session-State
    const { data: ss } = await supabase.from('session_state').select('*')
    ss?.forEach(s => {
        const t = timers.value[s.group_id]
        if (!t) return
        t.isRunning = s.is_running
        t.currentLapStart = s.current_lap_start
        t.pauseStart = s.pause_start
        t._now = Date.now()
    })

    // gespeicherte Laps
    const { data: sl } = await supabase.from('session_laps').select('*').order('lap_index')
    sl?.forEach(l => {
        const t = timers.value[l.group_id]
        if (t) t.lapTimes[l.lap_index] = l.duration_ms
    })

    // Live-Tick
    setInterval(() => {
        for (const id in timers.value) {
            const t = timers.value[id]
            if (t.isRunning) t._now = Date.now()
        }
    }, 1000)
})

/* Controls */
async function toggleTimer(id) {
    const t = timers.value[id]
    const now = Date.now()

    if (t.isRunning) {
        // Pause starten
        t.isRunning = false
        t.pauseStart = now
    } else {
        // Pause beenden
        if (t.pauseStart) {
            const paused = now - t.pauseStart
            t.currentLapStart += paused
            t.pauseStart = null
        }
        t.isRunning = true
        t._now = now
        if (!t.currentLapStart) t.currentLapStart = now
    }
    await upsertState(id, t)
}

async function lap(id) {
    const t = timers.value[id]
    if (!t.isRunning) return
    const now = Date.now()
    const dur = now - t.currentLapStart
    t.lapTimes.push(dur)
    t.currentLapStart = now
    await insertLap(id, t.lapTimes.length - 1, dur, staticRounds.value[id][t.lapTimes.length - 1]?.pace_s)
    await upsertState(id, t)
}

async function undoLap(id) {
    const t = timers.value[id]
    if (!t.lapTimes.length) return
    const last = t.lapTimes.pop()
    t.currentLapStart -= last
    await deleteLap(id, t.lapTimes.length)
    await upsertState(id, t)
}

async function resetTimer(id) {
    if (!confirm('Training zurücksetzen?')) return
    timers.value[id] = {
        isRunning: false,
        currentLapStart: null,
        pauseStart: null,
        lapTimes: [],
        _now: 0
    }
    await resetAll(id)
}

/* Berechnung */
function getCurrentLapTime(id) {
    const t = timers.value[id]
    if (t.isRunning) return (t._now || Date.now()) - t.currentLapStart
    if (t.pauseStart) return t.pauseStart - t.currentLapStart
    return 0
}
function totalElapsed(id) {
    const t = timers.value[id]
    const laps = t.lapTimes.reduce((a, d) => a + d, 0)
    return laps + getCurrentLapTime(id)
}
function totalDelta(id) {
    const t = timers.value[id]
    const sr = staticRounds.value[id] || []
    return t.lapTimes.reduce((sum, d, i) => {
        const p = sr[i]?.pace_s
        return sum + (p != null ? Math.abs(Math.round(d / 1000 - p * bahnlaenge / 1000)) : 0)
    }, 0)
}
</script>
