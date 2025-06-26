<template>
    <nav class="bg-white shadow">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <div class="flex space-x-4 h-16 items-center">
                <NuxtLink to="/" class="px-3 py-2 rounded-md text-sm font-medium"
                    :class="$route.path === '/' ? 'bg-blue-500 text-white' : 'text-gray-700 hover:bg-gray-100'">Home
                </NuxtLink>
                <NuxtLink to="/groups" class="px-3 py-2 rounded-md text-sm font-medium"
                    :class="$route.path.startsWith('/groups') ? 'bg-blue-500 text-white' : 'text-gray-700 hover:bg-gray-100'">
                    Groups</NuxtLink>
                <NuxtLink to="/rounds" class="px-3 py-2 rounded-md text-sm font-medium"
                    :class="$route.path.startsWith('/rounds') ? 'bg-blue-500 text-white' : 'text-gray-700 hover:bg-gray-100'">
                    Rounds</NuxtLink>
                <NuxtLink to="/tracking" class="px-3 py-2 rounded-md text-sm font-medium"
                    :class="$route.path.startsWith('/tracking') ? 'bg-blue-500 text-white' : 'text-gray-700 hover:bg-gray-100'">
                    Tracking</NuxtLink>
            </div>
        </div>
    </nav>
</template>

<script setup>
import { useRoute } from 'vue-router'
</script>
