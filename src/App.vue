<script setup>
import { reactive, ref, onMounted, toRaw } from 'vue'

let crime_url = ref('http://localhost:8000');
let dialog_err = ref(false);
let crime_loading = ref(false);
let crimes_list = ref([]);
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
// key = map's name = database name
const neighborhood_mapping = {
    "St Anthony Park": "St. Anthony",
    "Como": "Como",
    "North End": "North End",
    "Hamline-Midway": "Hamline/Midway",
    "Macalester-Groveland": "Macalester-Groveland",
    "Highland": "Highland",
    "West Side Community Organization": "West Side",
    "Summit Hill Association": "Summit Hill",
    "Summit-University": "Summit/University",
    "Thomas-Dale/Frogtown": "Thomas/Dale(Frogtown)",
    "West 7th Federation/Fort Road": "West Seventh",
    "CapitolRiver Council": "Capitol River",
    "Payne-Phalen": "Payne/Phalen",
    "The Greater East Side": "Greater East Side",
    "Dayton's Bluff": "Dayton's Bluff",
    "Eastview-Conway-Battle Creek-Highwood Hills": "Conway/Battlecreek/Highwood",
    "Union Park": "Union Park"
}
const locationText = reactive({
    value: "",
    isSearching: false
});
let neighborhoodBB = {};
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
            result.features.forEach((value, index) => {
                district_boundary.addData(value);
                neighborhoodBB[neighborhood_mapping[value.properties.name2]] = {
                    name: value.properties.name2,
                    bounds: L.latLngBounds(value.geometry.coordinates[0][0].map((coord) => [coord[1], coord[0]]))
                };
                let marker = L.marker([map.neighborhood_markers[index].location[0], map.neighborhood_markers[index].location[1]])
                    .bindPopup((layer) => `${map.neighborhood_markers[index].marker} crimes`)
                    .addTo(toRaw(map.leaflet));
                map.neighborhood_markers[index].marker = 0;
            });
        })
        .catch((error) => {
            console.log('Error:', error);
        });

    // Configure location text box
    map.leaflet.addEventListener("moveend", updateLocTextWithMap);
    //map.leaflet.addEventListener("zoomend", updateLocTextWithMap);

    map.leaflet.addEventListener("moveend", updateCrimes);
    //map.leaflet.addEventListener("zoomend", updateCrimes);
});


// FUNCTIONS
// Function called once user has entered REST API URL
let lastUpdate = 0;
async function updateCrimes() {
    crime_loading.value = true;
    let active_neighborhoods = [];
    let map_bounds = map.leaflet.getBounds();
    for(let neighborhood of neighborhoods.value) {
        let bounds = neighborhoodBB[neighborhood.name].bounds;
        //console.log(map_bounds, bounds);
        if(map_bounds.intersects(bounds)) {
            active_neighborhoods.push(neighborhood.id);
        }
    }

    let filters = new URLSearchParams();
    if (active_neighborhoods.length > 0) {
        filters.append("neighborhood", active_neighborhoods.join(","));
    }
    let url = crime_url.value + "/incidents?" + filters.toString();
    let thisUpdate = ++lastUpdate;
    let data = await fetch(url);
    let crimes = await data.json();
    
    if(thisUpdate !== lastUpdate) {
        // A newer update has been requested, discard this one
        return;
    }
    crimes_list.value = crimes;
    for (let marker of map.neighborhood_markers) {
        marker.marker = 0;
    }
    for(let crime of crimes) {
        map.neighborhood_markers[crime.neighborhood_number - 1].marker += 1;
    }
    crime_loading.value = false;
}
let specificMarker = null;
async function showMarker(crime) {
    if(specificMarker !== null) {
        map.leaflet.removeLayer(specificMarker);
        specificMarker = null;
    }
    let parts = crime.block.split(" ");
    parts[0] = parts[0].replaceAll("X", "0");
    let address = parts.join(" ") + ", St Paul, Minnesota";
    console.log(address);
    crime_loading.value = true;
    const res = await fetch(`https://nominatim.openstreetmap.org/search?q=${address}&format=json`);
    const json = await res.json();
    if(json.length === 0) {
        return;
    }
    specificMarker = L.marker([json[0].lat, json[0].lon])
        .bindPopup(
            `
            <b>Date:</b> ${crime.date}<br />
            <b>Time:</b> ${crime.time}<br />
            <b>Incident:</b> ${crime.incident}<br />
            `
        )
        .addTo(toRaw(map.leaflet))
        .openPopup();
    map.leaflet.setView([json[0].lat, json[0].lon], 16);
    window.scrollTo(0, 0);
    crime_loading.value = false;
}

let neighborhoods = ref([]);

function initNeighborhoods() {
    fetch(crime_url.value + "/neighborhoods").then((response) => {
        return response.json();
    }).then((result) => {
        result.forEach((neighborhood) => {
            neighborhoods.value.push(neighborhood);
        });
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
        updateCrimes();
        initNeighborhoods();
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

function deleteCrime(crime) {
    fetch(crime_url.value + "/remove-incident", {
        method: "DELETE",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({ case_number: crime.case_number })
    }).then((response) => {
        if (response.ok) {
            updateCrimes();
        } else {
            console.error("Failed to delete crime");
        }
    }).catch((error) => {
        console.error("Error:", error);
    });
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
        <div id="crimes-list">
            <h2>Crimes List</h2>
            <table>
                <thead>
                    <tr>
                        <th>Case Number</th>
                        <th>Date</th>
                        <th>Time</th>
                        <th>Incident</th>
                        <th>Police Grid</th>
                        <th>Neighborhood</th>
                        <th>Incident Type</th>
                        <th>Block</th>
                    </tr>
                </thead>
                <tbody :class="{ 'crime-loading': crime_loading }">
                    <tr class="crime-row" v-for="crime in crimes_list" :key="crime.case_number" @click="showMarker(crime)">
                        <td>{{ crime.case_number }}</td>
                        <td>{{ crime.date }}</td>
                        <td>{{ crime.time}}</td>
                        <td>{{ crime.incident }}</td>
                        <td>{{ crime.police_grid }}</td>
                        <td>{{ crime.neighborhood_name }}</td>
                        <td>{{ crime.incident_type }}</td>
                        <td>{{ crime.block }}</td>
                        <td><button @click="deleteCrime(crime)">Delete</button></td>
                    </tr>
                </tbody>
            </table>
        </div>
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

.crime-row {
    cursor: pointer;
}
.crime-row button {
    background-color: orange;
    border-radius: 5px;
    padding: 5px;
    cursor: pointer;
}
.crime-loading {
    filter: blur(2px);
}
</style>
