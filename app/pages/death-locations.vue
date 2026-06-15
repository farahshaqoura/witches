<template>
    <loading-message v-if="loading" />
    <map-component
        v-else
        :pageInfo="pageInfo"
        :originalMarkers="originalMarkers"
        :filtersGeneralInfo="filtersGeneralInfo"
        :filterProperties="filterProperties"
        iconBehaviour="changing"
        :includeTimeline="false"
        :clustersOnInitial="false"
    >
    </map-component>
</template>

<script setup>
import Swal from 'sweetalert2'
import { SPARQLQueryDispatcher } from '@/assets/js/SPARQLQueryDispatcher'
import APIDataHandler from '@/assets/js/APIDataHandler'
import queryOutput from '../../big-query-output.json'
import FilteringMethods from '@/assets/js/FilteringMethods'
import filterDescriptions from '@public/filterDescriptions.json'

definePageMeta({
    layout: 'default',
})
const pageInfo = ref({
    title: 'Places of Death Map for Accused Witches',
    html: '',
    footer: 'witches.is.ed.ac.uk',
    confirmButtonText: 'Close',
    type: 'info',
    showCloseButton: true,
})
const sparqlUrl = 'https://query.wikidata.org/sparql'

let loading = ref(true)
let originalMarkers = ref([])
let filtersToFind = [
    ['socialClass', 'changing'],
    ['occupation', 'changing'],
]

let filtersGeneralInfo = ref({
    title: 'Accused witch filters',
    filtersShowing: true,
})
let filterProperties = ref({
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
const totalWitches = computed(() => {
    return originalMarkers.value.reduce((total, marker) => {
        return total + (marker.witches.length || 0)
    }, 0)
})
const icons = computed(() => {
    const { icons } = useIcons()
    return icons.value
})
onMounted(() => {
    Object.keys(filterDescriptions).forEach((key) => {
        if (filterProperties[key]) {
            filterProperties[key].description = filterDescriptions[key]
        }
    })

    loadData()
})

async function loadWikiEntries() {
    const sparqlQuery = `SELECT DISTINCT ?item ?LabelEN ?page_title
            WHERE {
              ?item wdt:P4478 ?witch .
              ?article schema:about ?item ; schema:isPartOf <https://en.wikipedia.org/> ;  schema:name ?page_title .
              ?item rdfs:label ?LabelEN filter (lang(?LabelEN) = "en") .
            }`
    const wikiPages = []
    const queryDispatcher = new SPARQLQueryDispatcher(sparqlUrl)

    const result = await queryDispatcher.query(sparqlQuery)
    for (let i = 0; i < result.results.bindings.length; i++) {
        let item = result.results.bindings[i]

        let wikiPage = {
            id: item.item.value,
            pageTitle: item.page_title.value,
        }

        wikiPages.push(wikiPage)
    }
    return wikiPages
}
function setMarkersIcons() {
    let Filtering = new FilteringMethods(filterProperties.value, 'sex')
    for (let i = 0; i < originalMarkers.length; i++) {
        let marker = originalMarkers[i]
        ;[marker.markerIcon, marker.active] =
            Filtering.getMarkerStateIconDependant(marker)
    }
}
function updatePageInfo() {
    pageInfo.html = `<div>This map communicates the recorded locations for accused witches’ places of death. These deaths are the result of <strong">execution under the charge of witchcraft</strong>. Although there were <strong>3212</strong> accused witches named, there are recorded places of death for only <stong>${totalWitches}<strong> of them. Many of these locations are recorded as precise sites of execution. There are another <strong>119</strong> accused witches who were executed without a recorded geographical location. Many of the accused witches were executed by being strangled and then burned. For most of the accused witches, the surviving documentation does not show their fate. Most of them were probably executed, but the records that would have shown this no longer survive.</div>`
}
function hasLocalStorageExpired() {
    let hours = 24 // Reset when storage is more than 24hours
    let now = new Date().getTime()
    let setupTime = localStorage.getItem('setupTime')

    return setupTime === null || now - setupTime > hours * 60 * 60 * 1000
}
function loadDataFromLocalStorage() {
    let allFilters = JSON.parse(localStorage.getItem('allFilters'))
    filterProperties.value.socialClass.filters = allFilters.socialClass
    filterProperties.value.occupation.filters = allFilters.occupation
}
async function loadData() {
    const config = useRuntimeConfig()
    try {
        const wikiPages = await loadWikiEntries()

        let queryOutputFromApi = await myFetch('/main.php?type=death')
        let getData = new APIDataHandler(
            queryOutputFromApi || queryOutput,
            wikiPages,
            icons,
            null
        )
        let filtersFound = null
        ;[originalMarkers, filtersFound] = getData.loadAccussed(
            'placeOfDeath',
            filtersToFind
        )

        filterProperties.value.socialClass.filters = filtersFound.socialClass
        filterProperties.value.occupation.filters = filtersFound.occupation

        setMarkersIcons()
        loading.value = false

        // Update the pageInfo with the calculated number of witches
        updatePageInfo()
    } catch (e) {
        Swal.fire({
            title: 'Server Error',
            html: `<div>We are unable to connect to the server to pull in map info. Please refresh the page and try again. If this error persists, please contact <a href="mailto:${config.public.supportEmail}">${config.public.supportEmail}</a></div>`,
            footer: 'witches.is.ed.ac.uk',
            confirmButtonText: 'Close',
            type: 'error',
            showCloseButton: true,
        })
    }
}
</script>

<style></style>
