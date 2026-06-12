<template>
    <loading-message v-if="loading" />
    
    <map-component
        v-else
        :pageInfo="pageInfo"
        :originalMarkers="originalMarkers"       
        :filtersGeneralInfo="filtersGeneralInfo"
        :filterProperties="filterProperties"
        iconBehaviour="changing"
       
    >
    </map-component>
</template>

<script setup>
import Swal from 'sweetalert2'
import { SPARQLQueryDispatcher } from '~/assets/js/SPARQLQueryDispatcher'
import APIDataHandler from '~/assets/js/APIDataHandler'
import FilteringMethods from '~/assets/js/FilteringMethods'
import json from '../big-query-output.json'
import filterDescriptions from '../public/filterDescriptions.json'

definePageMeta({
    layout: 'default',
})


const loading = ref(true)
// const wikiPages = ref([])
const originalMarkers = ref([])

const sparqlUrl = 'https://query.wikidata.org/sparql' // plain const — never mutated
 
const pageInfo = ref({
    title: 'Witchcraft Prosecutions In Time and Place',
    html: '<div>This map shows the geographical residence location for each accused witch in Scotland taken from the Survey of Scottish Witchcraft Database. Out of the <b class="font-bold">3212</b> accused witches whose names are known, the residence for <b class="font-bold">3142</b> witches has been located. The majority of the residences are accurately located down to the precise settlement, while others range from parish to county depending on the records surviving for each accused witch. There is a total of 821 different locations recorded in the database; all but 25 of these have been identified. The remaining unidentified place-names are currently recorded as \'County of\' on the map.</div>',
    footer: 'witches.is.ed.ac.uk',
    confirmButtonText: 'Close',
    type: 'info',
    showCloseButton: true,
})
 
const filtersToFind = [
    ['socialClass', 'changing'],
    ['occupation', 'changing'],
]
 
const filtersGeneralInfo = ref({
    title: 'Accused witch filters',
    filtersShowing: true,
})
 
const filterProperties = ref({
    sex: {
        label: 'Gender',
        description: '',
        descriptionShowing: false,
        filters: {
            male: {
                label: 'Male',
                active: true,
                iconUrl: '/images/witch-single-blue.png',
            },
            female: {
                label: 'Female',
                active: true,
                iconUrl: '/images/witch-single-orange.png',
            },
            unknown: {
                label: 'Unknown',
                active: true,
                iconUrl: '/images/witch-single-BW.png',
            },
        },
        showing: true,
    },
    socialClass: {
        label: 'Social Classification',
        description: '',
        descriptionShowing: false,
        filters: {},
        showing: false,
    },
    occupation: {
        label: 'Occupations',
        description: '',
        descriptionShowing: false,
        filters: {},
        showing: false,
    },
    hasWikiPage: {
        label: 'Wikipedia Page',
        description: '',
        descriptionShowing: false,
        filters: {
            hasWiki: {
                label: 'Has wiki',
                active: true,
                iconUrl: '/images/witch-single-blue.png',
            },
            noWiki: {
                label: 'No wiki',
                active: true,
                iconUrl: '/images/witch-single-orange.png',
            },
        },
        showing: false,
    },
})
 const icons = computed(() => {
    const { icons } = useIcons()
    return icons.value
})

async function loadWikiEntries() {
    const sparqlQuery = `SELECT DISTINCT ?item ?LabelEN ?page_title
    WHERE {
      ?item wdt:P4478 ?witch .
      ?article schema:about ?item ; schema:isPartOf <https://en.wikipedia.org/> ;  schema:name ?page_title .
      ?item rdfs:label ?LabelEN filter (lang(?LabelEN) = "en") .
    }`
    const wikiPages =[]
 
    const queryDispatcher = new SPARQLQueryDispatcher(sparqlUrl)
    const result = await queryDispatcher.query(sparqlQuery);
     for (let i = 0; i < result.results.bindings.length; i++) {
            let item = result.results.bindings[i]
            wikiPages.push({
                id: item.item.value,
                pageTitle: item.page_title.value,
            })
           
        }
    
    return wikiPages;
}

function hasLocalStorageExpired() {
    const hours = 24
    const now = new Date().getTime()
    const setupTime = localStorage.getItem('setupTime')
    return setupTime === null || now - setupTime > hours * 60 * 60 * 1000
}
 
function loadDataFromLocalStorage() {
    originalMarkers.value = JSON.parse(localStorage.getItem('residenceMarkers'))
    const allFilters = JSON.parse(localStorage.getItem('allFilters'))
    filterProperties.value.socialClass.filters = allFilters.socialClass
    filterProperties.value.occupation.filters = allFilters.occupation
}
 
function saveDataToLocalStorage(foundFilters) {
    localStorage.setItem('setupTime', new Date().getTime())
    localStorage.setItem('residenceMarkers', JSON.stringify(originalMarkers.value))
    localStorage.setItem('allFilters', JSON.stringify(foundFilters))
}
 
function setMarkersIcons() {
    const Filtering = new FilteringMethods(filterProperties.value, 'sex')
    for (let i = 0; i < originalMarkers.value.length; i++) {
        let marker = originalMarkers.value[i]
        ;[marker.markerIcon, marker.active] =
            Filtering.getMarkerStateIconDependant(marker)
    }
}
async function loadData() {
  const config = useRuntimeConfig()
 
    try {
        const wikiPages = await loadWikiEntries()
      
        let queryOutput = await $fetch('http://localhost:8181/main.php?type=accused') 
       
    const getData = new APIDataHandler(
        queryOutput || json,
        wikiPages,
        icons,
        null
    )
     let filtersFound = null;
     let markers = null;

[markers, filtersFound] =  getData.loadAccussed(
        'residence',
        filtersToFind
    )
    originalMarkers.value = markers;
   filterProperties.value.socialClass.filters = filtersFound.socialClass
    filterProperties.value.occupation.filters = filtersFound.occupation
    setMarkersIcons()
    loading.value = false

    } catch (e) {

   
        Swal.fire({
            title: 'Server Error',
            html: `<div>We are unable to connect to the server to pull in map info. Please refresh the page and try again. If this error persists, please contact <a href="mailto:${config.public.supportEmail}">${config.public.supportEmail}</a></div>`,
            footer: 'witches.is.ed.ac.uk',
            confirmButtonText: 'Close',
            type: 'error',
            showCloseButton: true,
        })
        return
    }
 

 
   
  
}
 
function numberOfWitches() {
    return originalMarkers.value.reduce(
        (total, marker) => total + marker.witches.length, 0
    )
}
 watch(
    originalMarkers,
    () => {
        pageInfo.value.html = `<div>This map shows the geographical residence location for each accused witch in Scotland taken from the Survey of Scottish Witchcraft Database. Out of the <b class="font-bold">3212</b> accused witches whose names are known, the residence for <b class="font-bold">${numberOfWitches()}</b> witches has been located. The majority of the residences are accurately located down to the precise settlement, while others range from parish to county depending on the records surviving for each accused witch. There is a total of 821 different locations recorded in the database; all but 25 of these have been identified. The remaining unidentified place-names are currently recorded as 'County of' on the map.</div>`
    },
    { immediate: true, deep: true }
)

onMounted(() => {

    Object.keys(filterDescriptions).forEach((key) => {
        if (filterProperties.value[key]) {
            filterProperties.value[key].description = filterDescriptions[key]
        }
    })
    loadData()
})
// export default {
//     components: { MapComponent, LoadingMessage },
//     data: () => ({
//         pageInfo: {
//             title: 'Witchcraft Prosecutions In Time and Place',
//             html: '<div>This map shows the geographical residence location for each accused witch in Scotland taken from the Survey of Scottish Witchcraft Database. Out of the <b class="font-bold">3212</b> accused witches whose names are known, the residence for <b class="font-bold">3142</b> witches has been located. The majority of the residences are accurately located down to the precise settlement, while others range from parish to county depending on the records surviving for each accused witch. There is a total of 821 different locations recorded in the database; all but 25 of these have been identified. The remaining unidentified place-names are currently recorded as \‘County of’\ on the map.</div>',
//             footer: 'witches.is.ed.ac.uk',
//             confirmButtonText: 'Close',
//             type: 'info',
//             showCloseButton: true,
//         },
//         queryOutput: json,
//         sparqlUrl: 'https://query.wikidata.org/sparql',
//         wikiPages: [],
//         loading: true,
//         originalMarkers: [],
//         filtersToFind: [
//             ['socialClass', 'changing'],
//             ['occupation', 'changing'],
//         ],
//         filtersGeneralInfo: {
//             title: 'Accused witch filters',
//             filtersShowing: true,
//         },
//         filterProperties: {
//             sex: {
//                 label: 'Gender',
//                 description: '',
//                 descriptionShowing: false,
//                 filters: {
//                     male: {
//                         label: 'Male',
//                         active: true,
//                         iconUrl: '/images/witch-single-blue.png',
//                     },
//                     female: {
//                         label: 'Female',
//                         active: true,
//                         iconUrl: '/images/witch-single-orange.png',
//                     },
//                     unknown: {
//                         label: 'Unknown',
//                         active: true,
//                         iconUrl: '/images/witch-single-BW.png',
//                     },
//                 },
//                 showing: true,
//             },
//             socialClass: {
//                 label: 'Social Classification',
//                 description: '',
//                 descriptionShowing: false,
//                 filters: {},
//                 showing: false,
//             },
//             occupation: {
//                 label: 'Occupations',
//                 description: '',
//                 descriptionShowing: false,
//                 filters: {},
//                 showing: false,
//             },
//             hasWikiPage: {
//                 label: 'Wikipedia Page',
//                 description: '',
//                 descriptionShowing: false,
//                 filters: {
//                     hasWiki: {
//                         label: 'Has wiki',
//                         active: true,
//                         iconUrl: '/images/witch-single-blue.png',
//                     },
//                     noWiki: {
//                         label: 'No wiki',
//                         active: true,
//                         iconUrl: '/images/witch-single-orange.png',
//                     },
//                 },
//                 showing: false,
//             },
//         },
//     }),
//     computed: {
//         icons() {
//             const { icons } = useIcons()
//             return icons.value
//         },
//     },
//     methods: {
//         loadWikiEntries: function () {
//             const sparqlQuery = `SELECT DISTINCT ?item ?LabelEN ?page_title
//             WHERE {
//               ?item wdt:P4478 ?witch .
//               ?article schema:about ?item ; schema:isPartOf <https://en.wikipedia.org/> ;  schema:name ?page_title .
//               ?item rdfs:label ?LabelEN filter (lang(?LabelEN) = "en") .
//             }`

//             const queryDispatcher = new SPARQLQueryDispatcher(this.sparqlUrl)
//             queryDispatcher.query(sparqlQuery).then((result) => {
//                 for (let i = 0; i < result.results.bindings.length; i++) {
//                     let item = result.results.bindings[i]

//                     let wikiPage = {
//                         id: item.item.value,
//                         pageTitle: item.page_title.value,
//                     }

//                     this.wikiPages.push(wikiPage)
//                 }
//             })
//         },

//         // Local storage functions:
//         hasLocalStorageExpired: function () {
//             let hours = 24 // Reset when storage is more than 24hours
//             let now = new Date().getTime()
//             let setupTime = localStorage.getItem('setupTime')

//             return (
//                 setupTime === null || now - setupTime > hours * 60 * 60 * 1000
//             )
//         },
//         loadDataFromLocalStorage: function () {
//             this.originalMarkers = JSON.parse(
//                 localStorage.getItem('residenceMarkers')
//             )
//             let allFilters = JSON.parse(localStorage.getItem('allFilters'))
//             this.filterProperties.socialClass.filters = allFilters.socialClass
//             this.filterProperties.occupation.filters = allFilters.occupation
//         },
//         saveDataToLocalStorage: function (foundFilters) {
//             let now = new Date().getTime()
//             localStorage.setItem('setupTime', now)
//             localStorage.setItem(
//                 'residenceMarkers',
//                 JSON.stringify(this.originalMarkers)
//             )
//             localStorage.setItem('allFilters', JSON.stringify(foundFilters))
//         },
//         setMarkersIcons: function () {
//             let Filtering = new FilteringMethods(this.filterProperties, 'sex')

//             for (let i = 0; i < this.originalMarkers.length; i++) {
//                 let marker = this.originalMarkers[i]
//                 ;[marker.markerIcon, marker.active] =
//                     Filtering.getMarkerStateIconDependant(marker)
//             }
//         },
//         async loadData() {
//             await this.loadWikiEntries()
//             const icons = this.icons
//             const config = useRuntimeConfig()

//             try {
//                 let response = await useMyFetch('/main.php?type=accused')
//                 this.queryOutput = response
//             } catch (e) {
//                 Swal.fire({
//                     title: 'Server Error',
//                     html: `<div>We are unable to connect to the server to pull in map info. Please refresh the page and try again. If this error persists, please contact <a href="mailto:${config.public.supportEmail}">${config.public.supportEmail}</a></div>`,
//                     footer: 'witches.is.ed.ac.uk',
//                     confirmButtonText: 'Close',
//                     type: 'error',
//                     showCloseButton: true,
//                 })

//                 return
//             }

//             let getData = new APIDataHandler(
//                 this.queryOutput,
//                 this.wikiPages,
//                 icons,
//                 null
//             )
//             let filtersFound = null

//             ;[this.originalMarkers, filtersFound] = getData.loadAccussed(
//                 'residence',
//                 this.filtersToFind
//             )

//             this.filterProperties.socialClass.filters = filtersFound.socialClass
//             this.filterProperties.occupation.filters = filtersFound.occupation
//             this.setMarkersIcons()
//             this.loading = false
//         },
//         numberOfWitches: function () {
//             let noWitches = 0

//             for (let i = 0; i < this.originalMarkers.length; i++) {
//                 noWitches += this.originalMarkers[i].witches.length
//             }

//             return noWitches
//         },
//     },
//     watch: {
//         originalMarkers: {
//             handler(newVal) {
//                 const numberOfWitches = this.numberOfWitches()
//                 this.pageInfo.html = `<div>This map shows the geographical residence location for each accused witch in Scotland taken from the Survey of Scottish Witchcraft Database. Out of the <b class="font-bold">3212</b> accused witches whose names are known, the residence for <b class="font-bold">${numberOfWitches}</b> witches has been located. The majority of the residences are accurately located down to the precise settlement, while others range from parish to county depending on the records surviving for each accused witch. There is a total of 821 different locations recorded in the database; all but 25 of these have been identified. The remaining unidentified place-names are currently recorded as \‘County of’\ on the map.</div>`
//             },
//             immediate: true, // Computes numberOfWitches immediately when originalMarkers gets filled
//         },
//     },
//     mounted: function () {
//         // Load descriptions from the JSON file
//         Object.keys(filterDescriptions).forEach((key) => {
//             if (this.filterProperties[key]) {
//                 this.filterProperties[key].description = filterDescriptions[key]
//             }
//         })
//         this.loadData()
//     },
// }
</script>

<style></style>
