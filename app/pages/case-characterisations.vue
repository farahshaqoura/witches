<template>
    <loading-message v-if="loading" />
    <map-component
        v-else
        :pageInfo="pageInfo"
        :originalMarkers="originalMarkers"
        :filtersGeneralInfo="filtersGeneralInfo"
        :filterProperties="filterProperties"
        iconBehaviour="constant"
    >
    </map-component>
</template>

<script setup>
import { SPARQLQueryDispatcher } from '~/assets/js/SPARQLQueryDispatcher'
import APIDataHandler from '~/assets/js/APIDataHandler'
import queryOutput from '../../big-query-output.json'
import Swal from 'sweetalert2'
import filterDescriptions from '../public/filterDescriptions.json'

definePageMeta({
    layout: 'default',
})
const pageInfo = ref({
    title: 'Primary and Secondary Characterisations',
    html: '<div>This map shows the geographical residence location for each accused witch in Scotland taken from the Survey of Scottish Witchcraft Database. Out of the <strong>3212</strong> accused witches whose names are known, the residence for <strong>3142</strong> witches has been located. The majority of the residences are accurately located down to the precise settlement, while others range from parish to county depending on the records surviving for each accused witch. There is a total of 821 different locations recorded in the database; all but 25 of these have been identified. The remaining unidentified place-names are currently recorded as \‘County of’\ on the map.</div>',
    footer: 'witches.is.ed.ac.uk',
    confirmButtonText: 'Close',
    type: 'info',
    showCloseButton: true,
})
const sparqlUrl = 'https://query.wikidata.org/sparql'

const loading = ref(true)
const originalMarkers = ref([])
const filtersToFind = [
    ['primary', 'constant'],
    ['secondary', 'constant'],
]
const filtersGeneralInfo = ref({
    title: 'Case characterisations',
    filtersShowing: true,
})
const filterProperties = ref({
    primary: {
        label: 'Primary',
        description: '',
        descriptionShowing: false,
        filters: {},
        showing: true,
    },
    secondary: {
        label: 'Secondary',
        description: '',
        descriptionShowing: false,
        filters: {},
        showing: false,
    },
})
async function loadWikiEntries() {
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
    filterProperties.primary.filters = allFilters.primary
    filterProperties.secondary.filters = allFilters.secondary
    setMarkersIcons()
    console.log(filterProperties)
}

async function loadData() {
    let icon = '/images/witch-single-orange.png'

    try {
        let wikiPages = await loadWikiEntries()

        let queryOutputFromApi = await myFetch(
            '/main.php?type=primarysecondary'
        )

        let getData = new APIDataHandler(
            queryOutputFromApi,
            wikiPages,
            null,
            icon
        )

        let filtersFound = null
        let markers = null

        ;[markers, filtersFound] = getData.loadAccussed(
            'residence',
            filtersToFind
        )
        originalMarkers.value = markers
        filterProperties.value.primary.filters = filtersFound.primary
        filterProperties.value.secondary.filters = filtersFound.secondary
        setMarkersIcons()
        loading.value = false
    } catch (e) {
        console.error(e, 'eeee')
        Swal.fire({
            title: 'Server Error',
            html: '<div>We are unable to connect to the server to pull in map info. Please refresh the page and try again. If this error persists, please contact <a href="mailto:ltw-apps-dev.ed.ac.uk">ltw-apps-dev.ed.ac.uk</a></div>',
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
