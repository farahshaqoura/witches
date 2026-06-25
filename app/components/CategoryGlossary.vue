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
            <div v-for="categoryObj in categories" :key="categoryObj.category">
                <!-- Assign ref dynamically based on category name -->
                <h2
                    :ref="`category-${categoryObj.category}`"
                    @click="toggleShowingCategory(categoryObj.category)"
                    class="flex justify-between items-center cursor-pointer px-4"
                >
                    <span>{{ categoryObj.category }}</span>
                    <span v-if="categoryObj.expanded">-</span
                    ><span v-else>+</span>
                </h2>
                <hr class="mb-3" />
                <ul v-show="categoryObj.expanded" class="mb-3 ml-4">
                    <li
                        v-for="item in sortedGlossary[categoryObj.category]"
                        :key="item.word"
                        class="mb-3"
                    >
                        <h3 class="font-semibold">{{ item.word }}</h3>
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
const props = defineProps({
    glossary: {
        type: Array,
        required: true,
    },
    initialCategory: {
        type: String,
        default: null,
    },
})
const categories = ref([]);

const  sortedGlossary= computed(() => {
            let grouped = {}

            props.glossary.forEach((item) => {
                let category = item.category || 'Uncategorized'
                if (!grouped[category]) {
                    grouped[category] = []
                }
                grouped[category].push(item)
            })

            const sortedGroups = {}
            Object.keys(grouped)
                .sort()
                .forEach((key) => {
                    sortedGroups[key] = grouped[key].sort((a, b) =>
                        a.word.localeCompare(b.word)
                    )
                })
                
            return sortedGroups
        })
// function handleCategoryFromQuery() {
//     console.log( route.query.category,' route.query.category')
//             const queryCategory =
//                 initialCategory || route.query.category
//             if (queryCategory) {
//                 const categoryObj = categories.value.find(
//                     (c) => c.category === queryCategory
//                 )
//                 if (categoryObj) {
//                     categoryObj.expanded = true

//                     // Scroll after the DOM has updated
//                     nextTick(() => {
//                         setTimeout(() => {
//                             const categoryHeader =
//                                 $refs[`category-${queryCategory}`]
//                             if (categoryHeader && categoryHeader[0]) {
//                                 categoryHeader[0].scrollIntoView({
//                                     behavior: 'smooth',
//                                     block: 'start',
//                                 })
//                             }
//                         }, 100) // Small delay before scrolling
//                     })
//                 }
//             }
//         }

function  toggleShowingCategory(category) {
            const categoryObj = categories.value.find(
                (c) => c.category === category
            )
            if (categoryObj) {
                categoryObj.expanded = !categoryObj.expanded
            }
        }

function expandAll() {
        categories.value.forEach((categoryObj) => {
                categoryObj.expanded = true
            })
        }

function collapseAll() {
        categories.value.forEach((categoryObj) => {
                categoryObj.expanded = false
            })
        }
onMounted(()=>{
        
       categories.value = Object.keys(sortedGlossary.value)
            .sort()
            .map((category) => {
                return { category, expanded: false }
            })
     

        // handleCategoryFromQuery()
        })


// watch(()=>route.query.category,
//         ()=>{
//    setTimeout(() => {
//                     handleCategoryFromQuery()
//                 }, 300) //
//         },
//         {   immediate: true}
//     )
    

   
</script>
