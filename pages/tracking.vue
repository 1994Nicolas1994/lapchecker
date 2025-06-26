<script setup>
import { ref, onMounted } from 'vue'
import { createClient } from '@supabase/supabase-js'

const config = useRuntimeConfig()
const supabase = createClient(config.public.supabaseUrl, config.public.supabaseAnonKey)

const groups = ref([])
const rounds = ref({})
const timers = ref({})
const selectedGroupId = ref('all')
const bahnlaenge = 400  // Meter – kannst du auch dynamisch machen

onMounted(async () => {
    const { data: groupData } = await supabase.from('groups').select('*')
    groups.value = groupData

    for (const group of groupData) {
        const { data: groupRounds } = await supabase
            .from('rounds')
            .select('*')
            .eq('group_id', group.id)
            .order('position', { ascending: true })

        rounds.value[group.id] = groupRounds
        timers.value[group.id] = {
            isRunning: false,
            startTime: null,
            lapTimes: [],
            currentLapStart: null
        }
    }

    setInterval(updateTimers, 1000)
})

function updateTimers() {
    for (const groupId in timers.value) {
        if (timers.value[groupId].isRunning) {
            timers.value[groupId]._now = Date.now()
        }
    }
}

function toggleTimer(groupId) {
    const t = timers.value[groupId]
    t.isRunning = !t.isRunning
    if (t.isRunning) {
        if (!t.startTime) {
            t.startTime = Date.now()
            t.currentLapStart = Date.now()
        }
        t._now = Date.now()
    }
}

function lap(groupId) {
    const t = timers.value[groupId]
    if (!t.isRunning) return
    const now = Date.now()
    const lapDuration = now - t.currentLapStart
    t.lapTimes.push(lapDuration)
    t.currentLapStart = now
}

function del(groupId) {
    const t = timers.value[groupId]
    if (!t.lapTimes.length) return
    t.lapTimes.pop()
    t.currentLapStart = Date.now()
}

function getCurrentLapTime(groupId) {
    const t = timers.value[groupId]
    if (!t.currentLapStart) return 0
    return (t._now || Date.now()) - t.currentLapStart
}

function formatTime(ms) {
    const sec = Math.floor(ms / 1000)
    const min = Math.floor(sec / 60)
    const rem = sec % 60
    return `${min}:${rem.toString().padStart(2, '0')}`
}

function formatPace(pace_s) {
    if (!pace_s || isNaN(pace_s)) return '--'

    const min = Math.floor(pace_s / 60)
    const sec = Math.round(pace_s % 60)
    const rundenzeit = Math.round(pace_s * bahnlaenge / 1000)

    return `${min}:${sec.toString().padStart(2, '0')}min/km (${rundenzeit}s)`
}

function paceDiff(actualMs, pace_s) {
    if (actualMs == null || pace_s == null || isNaN(actualMs) || isNaN(pace_s)) return ''

    const target = pace_s * bahnlaenge / 1000
    const actual = actualMs / 1000
    const diff = Math.round(actual - target)
    const sign = diff > 0 ? '+' : ''
    return `${sign}${diff}s`
}


function getGroupRounds(groupId) {
    return rounds.value[groupId] || []
}

</script>
<template>
    <div class="p-4">
        <label class="block mb-2 font-semibold">Gruppe wählen:</label>
        <select v-model="selectedGroupId" class="border p-2 rounded mb-4">
            <option value="all">Alle Gruppen</option>
            <option v-for="g in groups" :key="g.id" :value="g.id">{{ g.name }}</option>
        </select>

        <div class="flex gap-4 overflow-x-auto">
            <div v-for="g in groups" :key="g.id" v-show="selectedGroupId === 'all' || selectedGroupId === g.id"
                class="min-w-[300px] border rounded p-4 shadow">
                <h2 class="text-lg font-bold mb-2">{{ g.name }}</h2>

                <div class="space-y-2 mb-4">
                    <button @click="toggleTimer(g.id)" class="w-full bg-blue-600 text-white py-1 rounded">
                        {{ timers[g.id]?.isRunning ? '⏸ Pause' : '▶ Start' }}
                    </button>
                    <button @click="lap(g.id)" class="w-full bg-green-600 text-white py-1 rounded">🏁 Lap</button>
                    <button @click="del(g.id)" class="w-full bg-red-600 text-white py-1 rounded">⏪ Del</button>
                </div>

                <div class="text-sm">
                    <p><strong>Gesamt:</strong> {{ formatTime((timers[g.id]._now || Date.now()) -
                        timers[g.id].startTime) }}</p>
                    <p><strong>Aktuelle Runde:</strong> {{ formatTime(getCurrentLapTime(g.id)) }}</p>
                </div>

                <div class="mt-4">
                    <h3 class="font-semibold mb-1">Runden</h3>
                    <ul class="text-sm space-y-1">
                        <li v-for="(round, index) in getGroupRounds(g.id)" :key="index"
                            :class="index === timers[g.id].lapTimes.length ? 'font-bold' : ''">
                            Runde {{ index + 1 }} –
                            Zeit:
                            <span v-if="index < timers[g.id].lapTimes.length">
                                {{ formatTime(timers[g.id].lapTimes[index]) }}
                            </span>
                            <span v-else-if="index === timers[g.id].lapTimes.length">
                                {{ formatTime(getCurrentLapTime(g.id)) }} (läuft)
                            </span>
                            <span v-else>
                                --
                            </span>
                            –
                            Ziel: {{ formatPace(round.pace_s) }},
                            Diff:
                            <span v-if="index < timers[g.id].lapTimes.length">
                                {{ paceDiff(timers[g.id].lapTimes[index], round.pace_s) }}
                            </span>
                            <span v-else-if="index === timers[g.id].lapTimes.length">
                                {{ paceDiff(getCurrentLapTime(g.id), round.pace_s) }}
                            </span>
                        </li>
                    </ul>
                </div>
            </div>
        </div>
    </div>
</template>
