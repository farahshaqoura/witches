<template>
    <div>
        <h1 class="text-center mb-6">Glossary</h1>
        <p class="text-center mb-3">
            This glossary contains unusual or specialist words used across this
            website, to help you better understand the terms and their meanings.
        </p>
        <div class="flex justify-center mb-5">
            <button
                @click="setSortMode('letters')"
                :class="[
                    'px-4 py-2 mx-2',
                    {
                        'bg-gray-200 font-semibold outline':
                            sortMode === 'letters',
                    },
                ]"
                class="bg-white border rounded"
            >
                Sort Alphabetically
            </button>
            <button
                @click="setSortMode('category')"
                :class="[
                    'px-4 py-2 mx-2',
                    {
                        'bg-gray-200 font-semibold outline':
                            sortMode === 'category',
                    },
                ]"
                class="bg-white border rounded"
            >
                Sort by Category
            </button>
        </div>
        <div class="mr-[10%] ml-[10%]">
            <AlphabetGlossary
                v-if="sortMode === 'letters'"
                :glossary="glossary"
            />
            <CategoryGlossary
                v-if="sortMode === 'category'"
                :glossary="glossary"
                :initialCategory="initialCategory"
            />
        </div>
    </div>
</template>

<script setup>
definePageMeta({
    layout: 'basic',
})
import glossaryJSON from '@public/glossary.json'
const route = useRoute()

const sortMode = ref('letters')
const glossary = glossaryJSON
const initialCategory = ref(null)

function setSortMode(mode) {
    sortMode.value = mode
}
onMounted(() => {
    const queryCategory = route.query.category || null
    if (queryCategory) {
        sortMode.value = 'category'
        initialCategory.value = queryCategory
    }
})
</script>
