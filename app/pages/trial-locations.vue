<template>
    <div id="outer">
        <div id="inner">
            <div id="page-intro" class="pl-5 pr-5 pt-3 pb-3">
                <br />
                <div class="flex content-start items-center">
                    <h2>
                        Locations of Trials for Accused Witches
                        <div
                            class="inline-flex items-center justify-center align-middle w-6 h-6 hover:w-7 hover:h-7 mb-0.5 ml-1 cursor-pointer"
                        >
                            <img
                                src="/images/infoIcon.svg"
                                class="w-full h-full pt-0.5"
                                aria-label="Page Information Popup"
                                @click="showPageInfo()"
                            />
                            <span class="visually-hidden"
                                >Page Information Pop Up</span
                            >
                        </div>
                    </h2>
                </div>
                <div>
                    <br />
                    <span v-for="(tile, index) in tiles" :key="index">
                        <input
                            type="radio"
                            name="tile"
                            :checked="tile.name === currentTileName"
                            @change="filterTiles(tile)"
                        />&nbsp;{{ tile.name }}&nbsp;
                    </span>
                </div>
                <br />
                <h2>Year: {{ sliderYear[0] }} - {{ sliderYear[1] }}</h2>
                <div class="p-2">
                    <Slider
                        name="slider"
                        v-model="numberRangeValue"
                        :min="0"
                        :max="8"
                        :format="getYearLabel"
                        showTooltip="drag"
                        :merge="1"
                        @change="filterDates()"
                        :lazy="false"
                    />
                </div>
                <!-- Display slider years below the slider -->
                <div class="slider-years">
                    <span
                        v-for="(year, index) in sliderYears"
                        :key="index"
                        class="slider-year"
                        >{{ year }}</span
                    >
                </div>
                <br /><br />
            </div>
            <div id="map-wrapper">
                <client-only>
                    <LMap
                        style="height: 100%; width: 100%"
                        :zoom="zoom"
                        :center="center"
                    >
                        <LTileLayer
                            :url="url"
                            :attribution="attribution"
                        ></LTileLayer>

                        <LMarker
                            v-for="(marker, index) in activeMarkers"
                            :key="index"
                            :lat-lng="marker.longLat"
                        >
                            <LPopup class="adapted-popup">
                                <h2>{{ marker.location }}</h2>
                                <br />
                                <div
                                    :class="
                                        marker.trials.length > 1
                                            ? 'witch-scroller'
                                            : 'no-witch-scroller'
                                    "
                                >
                                    <div
                                        v-for="(trial, index) in marker.trials"
                                        :key="index"
                                    >
                                        <strong>{{ trial.witchName }}</strong
                                        ><br />
                                        Trial Date: {{ trial.date }}<br />
                                        Trial Year: {{ trial.year }}<br />
                                        <a :href="trial.link" target="_blank"
                                            >More Info</a
                                        ><br /><br />
                                    </div>
                                </div>
                            </LPopup>
                            <LIcon
                                :icon-size="[25, 38]"
                                :icon-anchor="iconAnchor"
                                :iconUrl="getIcon(marker)"
                                :shadowUrl="shadowUrl"
                                :shadowSize="[32, 22]"
                                :shadowAnchor="shadowAnchor"
                            >
                            </LIcon>
                        </LMarker>
                    </LMap>
                </client-only>
            </div>
        </div>
    </div>
</template>

<script setup>
import { SPARQLQueryDispatcher } from '~/assets/js/SPARQLQueryDispatcher'
import Swal from 'sweetalert2'
import Slider from '@vueform/slider'

definePageMeta({
    layout: 'default',
})

const sparqlUrl = 'https://query.wikidata.org/sparql'
const url = 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png'
const attribution =
    'Map data &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap contributors</a>. Historical Maps Layer, 1919-1947 from the <a href="http://maps.nls.uk/projects/api/">NLS Maps API</a>'
const zoom = 7
const center = [55.95, -3.198888888]
 const markers = ref([])
const originalMarkers = ref([])
const iconAnchor = [11, 41]
const shadowUrl = '/images/North-Berwick-witch-shadow.png'
const shadowAnchor = [11, 26]
let noItems = 0

const currentTileName = 'Modern Map'
const tiles = [
    {
        name: 'Modern Map',
        url: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
        active: true,
    },
    {
        name: 'Historic Map',
        url: 'https://nls.tileserver.com/nls/{z}/{x}/{y}.jpg',
        active: false,
    },
]
const sliderYear = [1550, 1750]
const sliderYears = [1550, 1575, 1600, 1625, 1650, 1675, 1700, 1725, 1750]
const numberRangeValue = ref([0, 8])

function convertPointToLongLatArray(pointString) {
    pointString = pointString.substr(6)
    pointString = pointString.slice(0, -1)
    let pointArray = pointString.split(' ')
    let longLatArray = [pointArray[1], pointArray[0]]
    return longLatArray
}
function getYearLabel(value) {
    return sliderYears[value]
}
async function loadTrials() {
    try {
        const sparqlQuery = `SELECT ?item ?residenceLabel ?coords ?personLabel ?date ?link
            WHERE
            {
              ?item wdt:P4532 ?witch .
              ?item wdt:P276 ?residence .
              ?residence wdt:P625 ?coords .
              ?item wdt:P1591 ?person .
              ?item wdt:P585 ?date .
              ?item wdt:P4532 ?link .
              SERVICE wikibase:label { bd:serviceParam wikibase:language "[AUTO_LANGUAGE],en". }
            }`

        const queryDispatcher = new SPARQLQueryDispatcher(sparqlUrl)
        const result = await queryDispatcher.query(sparqlQuery)
        noItems = result.results.bindings.length

        for (let i = 0; i < result.results.bindings.length; i++) {
            let item = result.results.bindings[i]
            let trialDate = item.date.value
            trialDate = trialDate.substr(0, 10)

            let trialYear = item.date.value
            trialYear = trialDate.substr(0, 4)
            let trialMonth = trialDate.substr(5, 2)
            let trialDay = trialDate.substr(8, 2)

            trialDate = trialDay + '/' + trialMonth + '/' + trialYear

            let trial = {
                id: item.item.value,
                location: item.residenceLabel.value,
                witchName: item.personLabel.value,
                link:
                    'http://witches.shca.ed.ac.uk/index.cfm?fuseaction=home.trialrecord&search_string&trialref=' +
                    item.link.value,
                longLat: convertPointToLongLatArray(item.coords.value),
                date: trialDate,
                year: trialYear,
            }
            let markersLocal= markers.value;

            let marker = markers.value.find((marker) => {
                return marker.location === trial.location
            })

            if (marker) {
                marker.trials.push(trial)
            } else {
                let marker = {
                    location: item.residenceLabel.value,
                    longLat: convertPointToLongLatArray(item.coords.value),
                    trials: [trial],
                }

               markersLocal.push(marker)
            }
        }
        markers.value=markersLocal;

        originalMarkers.value = markers;
 filterDates()
    } catch (e) {
        console.error(e, 'error')
    }
}

function getIcon(marker) {
    return '/images/North-Berwick-witch.png'
}

function filterTiles(tile) {
    currentTileName = tile.name
    url = tile.url
}
function filterDates() {
    let markers = originalMarkers.value

    markers.forEach((marker) => {
        marker.trials = marker.trials.filter(
            (trial) =>
                trial.year >= this.sliderYears[this.numberRangeValue[0]] &&
                trial.year <= this.sliderYears[this.numberRangeValue[1]]
        )
    })

    markers = markers
}
function showPageInfo() {
    Swal.fire({
        title: 'Trial Location Map',
        html: '<div>This map indicates the location of trial for the accused witches. There is an option to change the year, to show how the numbers of trials changed with time. A few people had <strong>multiple trials</strong>, which could have been held in <strong>different locations</strong>. There are <b class="font-bold">3211</b> recorded trials which have been related to witchcraft within the database. However, there are geographical locations noted for only <strong>432</strong> trials, meaning that many trial locations were not recorded in the surviving documents.</div>',
        footer: 'witches.is.ed.ac.uk',
        confirmButtonText: 'Close',
        type: 'info',
        showCloseButton: true,
    })
}

onMounted(async () => {
    await loadTrials()
})

const activeMarkers = computed(() => {
    return markers.value.filter((marker) => marker.trials.length > 0)
})
const max = computed(() => sliderYears.value.length - 1)
</script>

<style>
.slider-years {
    display: flex;
    justify-content: space-between;
    padding: 0 5px;
}

.slider-year {
    font-size: 14px;
    color: #606f7b;
}
.zoomed-in-img {
    float: left;
    width: 25px;
    height: 38px;
}

.icon-shadow {
    position: absolute;
    top: 15px !important;
    left: 0;
    z-index: -1;
    width: 32px;
    height: 22px !important;
}
</style>
