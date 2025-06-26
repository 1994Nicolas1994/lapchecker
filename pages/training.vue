<template>Training</template>
<script setup>
console.log('test')
const supabase = useSupabaseClient()

const { data, error } = await supabase.from('trainings').select('*')

if (error) console.error('Fehler:', error)
else console.log('Trainings:', data)


</script>