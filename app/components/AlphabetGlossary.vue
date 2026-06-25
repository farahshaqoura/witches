<template>
    <div>
        <div class="text-right mb-5 mr-[10%]">
            <button
                @click="expandAll"
                class="hover:text-gray-400 text-gray-800 font-bold py-2 px-1 underline text-xs"
            >
                Expand All
            </button>
            <button
                @click="collapseAll"
                class="hover:text-gray-400 text-gray-800 font-bold py-2 px-1 underline text-xs"
            >
                Collapse All
            </button>
        </div>
        <div class="mr-[10%] ml-[10%]">
            <div v-for="letterObj in letters" :key="letterObj.letter">
                <h2
                    @click="toggleShowingLetter(letterObj.letter)"
                    class="flex justify-between items-center cursor-pointer px-4"
                >
                    <span>{{ letterObj.letter }}</span>
                    <span v-if="letterObj.expanded">-</span
                    ><span v-else>+</span>
                </h2>
                <hr class="mb-3" />
                <ul v-show="letterObj.expanded" class="mb-3 ml-4">
                   
                    <li
                        v-for="item in letterObj.category"
                        :key="item.word"
                        class="mb-3"
                    >
                        <h3 class="font-semibold">{{ item.word }}</h3>
                        <p class="mb-0">
                            <span class="font-medium">Category:</span>
                            {{ item.category }}
                        </p>
                        <p class="mb-0">
                            <span class="font-medium">Definition:</span>
                            {{ item.definition }}
                        </p>
                    </li>
                </ul>
            </div>
        </div>
    </div>
</template>

<script setup>
const letters = ref([])

const props = defineProps({
    glossary: {
        required: true,
    },
})

function toggleShowingLetter(letter) {
    const letterObj = letters.value.find((l) => l.letter === letter)
    if (letterObj) {
        letterObj.expanded = !letterObj.expanded
    }
}
function expandAll() {
    
    letters.value.forEach((letterObj) => {
        letterObj.expanded = true
    })
}
function collapseAll() {
    letters.value.forEach((letterObj) => {
        letterObj.expanded = false
    })
}

const sortedGlossary = computed(() => {
    
    // object to store groups of glossary items by letter
    let grouped = {}

    // group by the first letter of each word
  
    props.glossary.forEach((item) => {
        let letter = item.word.charAt(0).toUpperCase()
        if (!grouped[letter]) {
            grouped[letter] = []
        }
        grouped[letter].push(item)
    })
 
    const sortedGroups = {}
    Object.keys(grouped)
        .sort()
        .forEach((key) => {
            sortedGroups[key] = grouped[key].sort((a, b) => {
                return a.word.localeCompare(b.word)
            })
        })
     
    return sortedGroups
})

onMounted(() => {
  letters.value= Object.keys(sortedGlossary.value)
        .sort()
        .map((letter) => {

            return { letter, category:sortedGlossary.value[letter], expanded: false }
        })
       
})
</script>
