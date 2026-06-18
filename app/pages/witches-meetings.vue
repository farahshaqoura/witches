<template>
    <loading-message v-if="loading" />
    <map-component
        v-else
        :pageInfo="pageInfo"
        :originalMarkers="originalMarkers"
        :filtersGeneralInfo="filtersGeneralInfo"
        :filterProperties="filterProperties"
        :includeTimeline="false"
        iconBehaviour="constant"
    >
    </map-component>
</template>

<script setup>
import { SPARQLQueryDispatcher } from '~/assets/js/SPARQLQueryDispatcher'
import APIDataHandler from '~/assets/js/APIDataHandler'
import json from '../big-query-output.json'
import MapComponent from '../components/MapComponent.vue'
import LoadingMessage from '../components/LoadingMessage.vue'
import Swal from 'sweetalert2'
import filterDescriptions from '../public/filterDescriptions.json'

definePageMeta({
    layout: 'default',
})
const pageInfo = ref({
    title: 'Witches Meetings',
    html: '<div>This map shows the geographical residence location for each accused witch in Scotland taken from the Survey of Scottish Witchcraft Database. Out of the <b class="font-bold">3212</b> accused witches whose names are known, the residence for <b class="font-bold">3142</b> witches has been located. The majority of the residences are accurately located down to the precise settlement, while others range from parish to county depending on the records surviving for each accused witch. There is a total of 821 different locations recorded in the database; all but 25 of these have been identified. The remaining unidentified place-names are currently recorded as \‘County of’\ on the map.</div>',
    footer: 'witches.is.ed.ac.uk',
    confirmButtonText: 'Close',
    type: 'info',
    showCloseButton: true,
})
const queryOutput = json

const sparqlUrl = 'https://query.wikidata.org/sparql'
const loading = ref(true)
const originalMarkers = ref([])
const filtersToFind = [
    ['demonicPact', 'constant'],
    ['propertyDamage', 'constant'],
    ['meetingsPlaces', 'constant'],
    ['meetingsInfo', 'constant'],
]
const filtersGeneralInfo = {
    title: 'Case information filters',
    filtersShowing: true,
}
const filterProperties = ref({
    demonicPact: {
        label: 'Pact with the devil',
        description: '',
        descriptionShowing: false,
        filters: {},
        showing: true,
    },
    propertyDamage: {
        label: 'Property damage',
        description: '',
        descriptionShowing: false,
        filters: {},
        showing: false,
    },
    meetingsPlaces: {
        label: 'Meetings places',
        description: '',
        descriptionShowing: false,
        filters: {},
        showing: false,
    },
    meetingsInfo: {
        label: 'Meetings information',
        description: '',
        descriptionShowing: false,
        filters: {},
        showing: false,
    },
})
async function loadWikiEntries() {
    try {
        const sparqlQuery = `SELECT DISTINCT ?item ?LabelEN ?page_title
            WHERE {
              ?item wdt:P4478 ?witch .
              ?article schema:about ?item ; schema:isPartOf <https://en.wikipedia.org/> ;  schema:name ?page_title .
              ?item rdfs:label ?LabelEN filter (lang(?LabelEN) = "en") .
            }`

        const queryDispatcher = new SPARQLQueryDispatcher(sparqlUrl)
        const result = await queryDispatcher.query(sparqlQuery)
        let wikiPages = []
        for (let i = 0; i < result.results.bindings.length; i++) {
            let item = result.results.bindings[i]

            let wikiPage = {
                id: item.item.value,
                pageTitle: item.page_title.value,
            }

            wikiPages.push(wikiPage)
        }
        return wikiPages
    } catch (e) {
        console.error(e)
    }
}

function setFilters(filtersFound) {
    // Using a function in this page instead of just
    // assigning manually like in the other pages because
    // there are 6 of them.

    filtersToFind.map((filterProperty) => {
        filterProperties.value[filterProperty[0]].filters =
            filtersFound[filterProperty[0]]
    })
}
function hasLocalStorageExpired() {
    let hours = 24 // Reset when storage is more than 24hours
    let now = new Date().getTime()
    let setupTime = localStorage.getItem('setupTime')

    return setupTime === null || now - setupTime > hours * 60 * 60 * 1000
}

function setMarkersIcons() {
    for (let i = 0; i < originalMarkers.length; i++) {
        let marker = originalMarkers[i]
        marker.markerIcon = '/images/witch-single-orange.png'
    }
}
function loadDataFromLocalStorage() {
    originalMarkers = JSON.parse(localStorage.getItem('residenceMarkers'))
    let allFilters = JSON.parse(localStorage.getItem('allFilters'))
    setFilters(allFilters)
    setMarkersIcons()
}

async function loadData() {
    let icon = '/images/witch-single-orange.png'
    const config = useRuntimeConfig()

    try {
        let wikiPages = await loadWikiEntries()
        let queryOutput = await myFetch('/main.php?type=meetings')

        let getData = new APIDataHandler(queryOutput, wikiPages, null, icon)

        let filtersFound = null
        let markers = null
        ;[markers, filtersFound] = getData.loadAccussed(
            'residence',
            filtersToFind
        )

        originalMarkers.value = markers

        setFilters(filtersFound)
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
onMounted(() => {
    Object.keys(filterDescriptions).forEach((key) => {
        if (filterProperties[key]) {
            filterProperties[key].description = filterDescriptions[key]
        }
    })
    loadData()
})
</script>

<style></style>
