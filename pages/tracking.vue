<template>
    <client-only>
        <div class="bg-gray-100 min-h-screen p-6">
            <!-- Header -->
            <header class="mb-8 text-center">
                <h1 class="text-3xl font-bold text-gray-800">LapChecker Tracking</h1>
                <p class="text-gray-600 mt-2">Dein Training im Blick behalten</p>
            </header>

            <!-- Gruppen-Filter -->
            <div class="max-w-md mx-auto mb-8">
                <label class="block text-sm font-medium text-gray-700 mb-2">Gruppe wählen</label>
                <select v-model="selectedGroupId"
                    class="w-full border border-gray-300 rounded-md px-3 py-2 bg-white focus:outline-none focus:ring-2 focus:ring-blue-400">
                    <option value="all">Alle Gruppen</option>
                    <option v-for="g in groups" :key="g.id" :value="g.id">
                        {{ g.name }}
                    </option>
                </select>
            </div>

            <!-- Card-Grid -->
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                <div v-for="g in groups" :key="g.id" v-show="selectedGroupId === 'all' || selectedGroupId === g.id"
                    class="bg-white shadow-lg rounded-xl p-6 flex flex-col">
                    <!-- Titel -->
                    <h2 class="text-xl font-semibold text-gray-800 mb-4">{{ g.name }}</h2>

                    <!-- Controls -->
                    <div class="flex flex-col space-y-3 mb-6">
                        <button @click="toggleTimer(g.id)"
                            class="w-full py-2 bg-blue-500 hover:bg-blue-600 active:bg-blue-700 text-white font-semibold rounded-md transition">
                            {{ timers[g.id].isRunning ? '⏸ Pause' : '▶ Start' }}
                        </button>
                        <button @click="lap(g.id)"
                            class="w-full py-2 bg-green-500 hover:bg-green-600 active:bg-green-700 text-white font-semibold rounded-md transition">🏁
                            Lap</button>
                        <button @click="undoLap(g.id)"
                            class="w-full py-2 bg-yellow-500 hover:bg-yellow-600 active:bg-yellow-700 text-white font-semibold rounded-md transition">↩️
                            Undo</button>
                        <button @click="resetTimer(g.id)"
                            class="w-full py-2 bg-red-500 hover:bg-red-600 active:bg-red-700 text-white font-semibold rounded-md transition">🔄
                            Reset</button>
                    </div>

                    <!-- Timer-Anzeigen -->
                    <div class="mb-6 space-y-1">
                        <p class="text-gray-700"><span class="font-medium">Gesamt:</span> {{
                            formatTime((timers[g.id]._now || Date.now()) - (timers[g.id].startTime||0)) }}</p>
                        <p class="text-gray-700"><span class="font-medium">Aktuelle Runde:</span> {{
                            formatTime(getCurrentLapTime(g.id)) }}</p>
                    </div>

                    <!-- Runden-Liste -->
                    <div class="flex-1">
                        <h3 class="text-gray-800 font-semibold mb-2">Runden</h3>
                        <ul class="divide-y divide-gray-200">
                            <li v-for="(r, idx) in staticRounds[g.id] || []" :key="idx"
                                :class="idx === timers[g.id].lapTimes.length ? 'bg-blue-50' : ''"
                                class="py-2 flex justify-between items-center">
                                <div>
                                    <span class="font-medium">Runde {{ idx + 1 }}</span>
                                    <span class="ml-2 text-gray-600">
                                        {{ idx < timers[g.id].lapTimes.length ? formatTime(timers[g.id].lapTimes[idx]) :
                                            idx === timers[g.id].lapTimes.length ? formatTime(getCurrentLapTime(g.id))
                                                + ' (läuft)' : '--' }} </span>
                                </div>
                                <div class="text-right">
                                    <div class="text-gray-700">{{ formatPace(r.pace_s) }}</div>
                                    <div class="text-sm text-gray-500">{{ idx < timers[g.id].lapTimes.length ?
                                        paceDiff(timers[g.id].lapTimes[idx], r.pace_s) : '' }}</div>
                                    </div>
                            </li>
                        </ul>
                    </div>

                    <!-- Total Δ -->
                    <div class="mt-6 text-right">
                        <span class="text-gray-700 font-medium">Total Δ:</span>
                        <span class="text-gray-900 font-semibold">{{ totalDelta(g.id) }} s</span>
                    </div>
                </div>
            </div>
        </div>
    </client-only>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { createClient } from '@supabase/supabase-js'

const config = useRuntimeConfig()
const supabase = createClient(config.public.supabaseUrl, config.public.supabaseAnonKey)

const groups = ref([])
const staticRounds = ref({})
const timers = ref({})
const selectedGroupId = ref('all')
const bahnlaenge = 400

const isReady = computed(() =>
    groups.value.length > 0 &&
    Object.keys(timers.value).length === groups.value.length
)

function formatTime(ms) {
    if (ms == null) return '--:--'
    const s = Math.floor(ms / 1000), m = Math.floor(s / 60)
    return `${m}:${String(s % 60).padStart(2, '0')}`
}
function formatPace(p) {
    if (p == null) return '--'
    const mm = Math.floor(p / 60), ss = String(Math.round(p % 60)).padStart(2, '0')
    const lapSec = Math.round(p * bahnlaenge / 1000)
    return `${mm}:${ss}min/km (${lapSec}s)`
}
function paceDiff(ms, p) {
    if (ms == null || p == null) return ''
    const d = Math.round(ms / 1000 - p * bahnlaenge / 1000)
    return `${d > 0 ? '+' : ''}${d}s`
}
function totalDelta(gid) {
    const t = timers.value[gid]
    const sr = staticRounds.value[gid] || []
    return (t.lapTimes || []).reduce((sum, dur, i) => {
        const pace = sr[i]?.pace_s
        if (pace == null) return sum
        return sum + Math.abs(Math.round(dur / 1000 - pace * bahnlaenge / 1000))
    }, 0)
}
function getCurrentLapTime(gid) {
    const t = timers.value[gid]
    return t.currentLapStart
        ? (t._now || Date.now()) - t.currentLapStart
        : 0
}

async function insertLap(gid, idx, dur, pace) {
    const { error } = await supabase
        .from('session_laps')
        .insert({ group_id: gid, lap_index: idx, duration_ms: dur, pace_s: pace })
    if (error) console.error('insertLap error', error)
}
async function deleteLap(gid, idx) {
    const { error } = await supabase
        .from('session_laps')
        .delete()
        .eq('group_id', gid)
        .eq('lap_index', idx)
    if (error) console.error('deleteLap error', error)
}
async function upsertState(gid, t) {
    const { error } = await supabase
        .from('session_state')
        .upsert({
            group_id: gid,
            is_running: t.isRunning,
            start_time: t.startTime,
            current_lap_start: t.currentLapStart
        })
    if (error) console.error('upsertState error', error)
}
async function resetAllSession(gid) {
    await supabase.from('session_laps').delete().eq('group_id', gid)
    await supabase.from('session_state').delete().eq('group_id', gid)
}

onMounted(async () => {
    const { data: g } = await supabase.from('groups').select('*')
    groups.value = g || []
    for (const grp of groups.value) {
        const { data: r } = await supabase
            .from('rounds')
            .select('*')
            .eq('group_id', grp.id)
            .order('position', { ascending: true })
        staticRounds.value[grp.id] = r || []
        timers.value[grp.id] = {
            isRunning: false,
            startTime: null,
            currentLapStart: null,
            lapTimes: [],
            _now: 0
        }
    }

    const { data: ss } = await supabase.from('session_state').select('*')
    ss?.forEach(s => {
        const t = timers.value[s.group_id]
        if (!t) return
        t.isRunning = s.is_running
        t.startTime = s.start_time
        t.currentLapStart = s.current_lap_start
        t._now = Date.now()
    })

    const { data: sl } = await supabase
        .from('session_laps')
        .select('*')
        .order('lap_index', { ascending: true })
    sl?.forEach(l => {
        const t = timers.value[l.group_id]
        if (!t) return
        t.lapTimes[l.lap_index] = l.duration_ms
    })

    setInterval(() => {
        for (const gid in timers.value) {
            const t = timers.value[gid]
            if (t.isRunning) t._now = Date.now()
        }
    }, 1000)
})

async function toggleTimer(gid) {
    const t = timers.value[gid]
    t.isRunning = !t.isRunning
    if (t.isRunning && t.startTime == null) {
        t.startTime = Date.now()
        t.currentLapStart = t.startTime
    }
    t._now = Date.now()
    await upsertState(gid, t)
}

async function lap(gid) {
    const t = timers.value[gid]
    if (!t.isRunning) return
    const now = Date.now()
    const dur = now - t.currentLapStart
    const idx = t.lapTimes.length
    const pace = staticRounds.value[gid][idx]?.pace_s
    t.lapTimes.push(dur)
    t.currentLapStart = now
    await insertLap(gid, idx, dur, pace)
    await upsertState(gid, t)
}

async function undoLap(gid) {
    const t = timers.value[gid]
    if (!t.lapTimes.length) return
    const last = t.lapTimes.pop()
    t.currentLapStart -= last
    await deleteLap(gid, t.lapTimes.length)
    await upsertState(gid, t)
}

async function resetTimer(gid) {
    if (!confirm('Training wirklich zurücksetzen?')) return
    timers.value[gid] = {
        isRunning: false,
        startTime: null,
        currentLapStart: null,
        lapTimes: [],
        _now: 0
    }
    await resetAllSession(gid)
}
</script>
