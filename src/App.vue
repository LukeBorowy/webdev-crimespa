<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('http://localhost:8000');
let dialog_err = ref(false);
let map = reactive(
    {
        leaflet: null,
        center: {
            lat: 44.955139,
            lng: -93.102222,
            address: ''
        },
        zoom: 12,
        bounds: {
            nw: { lat: 45.008206, lng: -93.217977 },
            se: { lat: 44.883658, lng: -92.993787 }
        },
        neighborhood_markers: [
            { location: [44.942068, -93.020521], marker: null },
            { location: [44.977413, -93.025156], marker: null },
            { location: [44.931244, -93.079578], marker: null },
            { location: [44.956192, -93.060189], marker: null },
            { location: [44.978883, -93.068163], marker: null },
            { location: [44.975766, -93.113887], marker: null },
            { location: [44.959639, -93.121271], marker: null },
            { location: [44.947700, -93.128505], marker: null },
            { location: [44.930276, -93.119911], marker: null },
            { location: [44.982752, -93.147910], marker: null },
            { location: [44.963631, -93.167548], marker: null },
            { location: [44.973971, -93.197965], marker: null },
            { location: [44.949043, -93.178261], marker: null },
            { location: [44.934848, -93.176736], marker: null },
            { location: [44.913106, -93.170779], marker: null },
            { location: [44.937705, -93.136997], marker: null },
            { location: [44.949203, -93.093739], marker: null }
        ]
    }
);
const locationText = reactive({
    value: "",
    isSearching: false
});

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);

    // Get boundaries for St. Paul neighborhoods
    let district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
        .then((response) => {
            return response.json();
        })
        .then((result) => {
            result.features.forEach((value) => {
                district_boundary.addData(value);
            });
        })
        .catch((error) => {
            console.log('Error:', error);
        });

    // Configure location text box
    map.leaflet.addEventListener("moveend", updateLocTextWithMap);
    map.leaflet.addEventListener("zoomend", updateLocTextWithMap);
});


// FUNCTIONS
// Function called once user has entered REST API URL
function initializeCrimes() {
    fetch(crime_url.value + "/incidents").then((response) => {
        return response.json();
    }).then((result) => {
        console.log(result);
    }).catch((error) => {
        console.log('Error:', error);
    });
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}

// Location text box keybinds
/** @param {KeyboardEvent} ev */
function doLocTextKeybinds(ev) {
    if (ev.key === "Enter") {
        ev.preventDefault();
        updateMapWithLocText();
    }
}

// Location text box submission
async function updateMapWithLocText() {
    try {
        locationText.isSearching = true;
        const res = await fetch(`https://nominatim.openstreetmap.org/search?q=${locationText.value}&format=json`);
        const json = await res.json();
        const lat = clamp(json[0].lat, map.bounds.se.lat, map.bounds.nw.lat);
        const lng = clamp(json[0].lon, map.bounds.nw.lng, map.bounds.se.lng);
        map.leaflet.setView([lat, lng], 16);
        if (!almostEqual(lat, json[0].lat) || !almostEqual(lng, json[0].lon)) {
            await updateLocTextWithMap();
        } else {
            locationText.value = json[0].display_name;
        }
    } catch (err) {
        console.error(err);
        await updateLocTextWithMap();
    }
    locationText.isSearching = false;
}

// Location text box update on map pan/zoom
async function updateLocTextWithMap() {
    try {
        locationText.isSearching = true;
        const lat = map.leaflet.getCenter().lat;
        const lng = map.leaflet.getCenter().lng;
        const res = await fetch(`https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`);
        const json = await res.json();
        locationText.value = json.display_name;
    } catch (err) {
        console.error(err);
    }
    locationText.isSearching = false;
}

/**
 * @param {number} value
 * @param {number} range1
 * @param {number} range2
 */
function clamp(value, range1, range2) {
    let min, max;
    if (range1 < range2) {
        min = range1;
        max = range2;
    } else {
        min = range2;
        max = range1;
    }
    return Math.min(Math.max(min, value), max);
}

/**
 * @param {number} value1
 * @param {number} value2
 */
function almostEqual(value1, value2) {
    return Math.abs(value1 - value2) < 0.01;
}
</script>

<template>
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br />
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>
    <div class="main-content">
        <div class="location-ctn">
            <label for="location-text">Location</label>
            <input type="text" id="location-text" :disabled="locationText.isSearching" v-model="locationText.value" @keydown="doLocTextKeybinds" />
            <button class="button" type="button" :disabled="locationText.isSearching" @click="updateMapWithLocText">Go</button>
        </div>
        <div id="leafletmap"></div>
    </div>
</template>

<style scoped>
.main-content {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin: 1rem;
}

#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
}

#leafletmap {
    height: 500px;
}

.dialog-header {
    font-size: 1.2rem;
    font-weight: bold;
}

.dialog-label {
    font-size: 1rem;
}

.dialog-input {
    font-size: 1rem;
    width: 100%;
}

.dialog-error {
    font-size: 1rem;
    color: #D32323;
}

.location-ctn {
    display: flex;
    gap: 1rem;
    align-items: center;
}

label {
    line-height: 1;
}

input {
    margin: 0;
}

button {
    margin: 0;
}
</style>
