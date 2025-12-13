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
        ],
        isUserGesturing: false
    }
);
const locationText = reactive({
    value: "",
    isSearching: false,
    cancelSearch: null
});
const newIncForm = reactive({
    case_number: "",
    date: "",
    time: "",
    code: undefined,
    incident: "",
    police_grid: undefined,
    neighborhood_number: undefined,
    block: "",
    isSending: false,
    errorMsg: ""
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
    // Weird undocumented behavior: programmatic movestart events have their target set to the Leaflet map,
    // while user-initiated pans/zooms do not
    map.leaflet.addEventListener("movestart", (ev) => {
        if (ev.target !== map.leaflet) map.isUserGesturing = true;
    });
    map.leaflet.addEventListener("moveend", (ev) => {
        if (map.isUserGesturing) {
            if (locationText.cancelSearch) {
                locationText.cancelSearch(true);
                locationText.cancelSearch = null;
            }
            updateLocTextWithMap();
        }
        map.isUserGesturing = false;
    })
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
            locationText.isSearching = false;
        }
    } catch (err) {
        console.error(err);
        await updateLocTextWithMap();
    }
}

// Location text box update on map pan/zoom
async function updateLocTextWithMap() {
    locationText.isSearching = true;
    const lat = map.leaflet.getCenter().lat;
    const lng = map.leaflet.getCenter().lng;
    await new Promise(async (resolve, reject) => {
        locationText.cancelSearch = reject;
        resolve(await fetch(`https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`));
    }).then(async (res) => {
        const json = await res.json();
        locationText.value = json.display_name;
        locationText.isSearching = false;
    }, (reason) => {
        if (!reason) throw reason;
    }).catch(console.error);
}

// New incident form submit
function submitNewIncForm(ev) {
    ev.preventDefault();
    newIncForm.isSending = true;
    fetch(`${crime_url.value}/new-incident`, {
        method: "PUT",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify(newIncForm)
    }).then((res) => {
        if (!res.ok) {
            newIncForm.errorMsg = "Invalid input. Please try again.";
        } else {
            // Reset fields
            newIncForm.errorMsg = "";
            newIncForm.block = "";
            newIncForm.case_number = "";
            newIncForm.code = undefined;
            newIncForm.date = "";
            newIncForm.incident = "";
            newIncForm.neighborhood_number = undefined;
            newIncForm.police_grid = undefined;
            newIncForm.time = "";
        }
    }).catch(err => {
        newIncForm.errorMsg = "Something went wrong. Please try again.";
        console.error(err);
    }).finally(() => newIncForm.isSending = false)
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
        <h2>Create New Incident</h2>
        <form @submit="submitNewIncForm">
            <div class="new-incident-form">
                <div class="form-element">
                    <label for="new-inc-case-num">Case #</label>
                    <input type="text" id="case-num" required :disabled="newIncForm.isSending" v-model="newIncForm.case_number" />
                </div>
                <div class="form-element short">
                    <label for="new-inc-date">Date</label>
                    <input type="text" id="new-inc-date" required :disabled="newIncForm.isSending" v-model="newIncForm.date" />
                </div>
                <div class="form-element short">
                    <label for="new-inc-time">Time</label>
                    <input type="text" id="new-inc-time" required :disabled="newIncForm.isSending" v-model="newIncForm.time" />
                </div>
                <div class="form-element short">
                    <label for="new-inc-code">Code</label>
                    <input type="text" id="new-inc-code" required :disabled="newIncForm.isSending" v-model="newIncForm.code" />
                </div>
                <div class="form-element short">
                    <label for="new-inc-grid">Police Grid #</label>
                    <input type="text" id="new-inc-grid" required :disabled="newIncForm.isSending" v-model="newIncForm.police_grid" />
                </div>
                <div class="form-element short">
                    <label for="new-inc-neighborhood">Neighborhood #</label>
                    <input type="text" id="new-inc-neighborhood" required :disabled="newIncForm.isSending" v-model="newIncForm.neighborhood_number" />
                </div>
                <div class="form-element">
                    <label for="new-inc-block">Block</label>
                    <input type="text" id="new-inc-block" required :disabled="newIncForm.isSending" v-model="newIncForm.block" />
                </div>
                <div class="form-element">
                    <label for="new-inc-desc">Incident</label>
                    <textarea id="new-inc-desc" required :disabled="newIncForm.isSending" v-model="newIncForm.incident"></textarea>
                </div>
            </div>
            <div class="form-element">
                <button class="button" type="submit" :disabled="newIncForm.isSending">Submit</button>
                <span class="dialog-error" v-text="newIncForm.errorMsg"></span>
            </div>
        </form>
    </div>
    <footer>
        <a href="/about.html">About the project</a>
    </footer>
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

.new-incident-form {
    display: flex;
    flex-flow: row wrap;
    gap: 1rem;
    margin-bottom: 1rem;
}

.form-element {
    display: flex;
    align-items: center;
    gap: 1rem;
}

.form-element > input {
    width: 10rem;
}

.form-element.short > input {
    width: 5rem;
}

.form-element > label {
    flex: 1 0 auto;
}

footer {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 1rem;
}

label {
    line-height: 1;
}

input,
textarea {
    margin: 0;
}

span {
    margin: 0;
}

button {
    margin: 0;
}
</style>
