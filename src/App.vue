<script setup>
import { reactive, ref, onMounted, toRaw } from 'vue'

let crime_url = ref('http://localhost:8000');
let dialog_err = ref(false);
let crime_loading = ref(false);
let crimes_list = ref([]);
let incident_checkboxes = ref([]);
let neighborhood_checkboxes = ref([]);
let start_date = ref('');
let end_date = ref('');
let max_incidents = ref(1000);
let possible_incidents = ref([
    {name: "Murder/Homicide", codes: [100, 110, 120]},
    {name: "Rape", codes: [210, 220]},
    {name: "Robbery", codes: [300, 311, 312, 313, 314, 321, 322, 323, 324, 331, 332, 333, 334, 341, 342, 343, 344, 351, 352, 353, 354, 361, 362, 363, 364, 371, 372, 373, 374]},
    {name: "Aggravated Assault", codes: [400, 410, 411, 412, 420, 421, 422, 430, 431, 432, 440, 441, 442, 450, 451, 452, 453]},
    {name: "Burglary", codes: [500, 510, 511, 513, 515, 516, 520, 521, 523, 525, 526, 530, 531, 533, 535, 536, 540, 541, 543, 545, 546, 550, 551, 553, 555, 556, 560, 561, 563, 565, 566]},
    {name: "Theft", codes: [600, 601, 603, 611, 612, 613, 621, 622, 623, 630, 631, 632, 633, 640, 641, 642, 643, 651, 652, 653, 661, 662, 663, 671, 672, 673, 681, 682, 683, 691, 692, 693]},
    {name: "Motor Vehicle Theft", codes: [700, 710, 711, 712, 720, 721, 722, 730, 731, 732]},
    {name: "Assault, Domestic", codes: [810, 861, 862, 863]},
    {name: "Arson", codes: [900, 901, 903, 905, 911, 913, 915, 921, 922, 923, 925, 931, 933, 941, 942, 951, 961, 971, 972, 975, 981, 982]},
    {name: "Criminal Damage to Property", codes: [1400, 1401, 1410, 1415, 1416, 1420, 1425, 1426, 1430, 1435, 1436]},
    {name: "Narcotics", codes: [1800, 1810, 1811, 1812, 1813, 1814, 1815, 1820, 1822, 1823, 1824, 1825, 1830, 1835, 1840, 1841, 1842, 1843, 1844, 1845, 1850, 1855, 1860, 1865, 1870, 1880, 1885]},
    {name: "Other", codes: [614, 2619, 3100, 9954, 9959, 9986]}
]);
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
    if (neighborhood_checkboxes.value.length > 0) {
        console.log(neighborhood_checkboxes.value);
        filters.append("neighborhood", neighborhood_checkboxes.value.join(","));
    } else {
        if(active_neighborhoods.length != 0) {
            filters.append("neighborhood", active_neighborhoods.join(","));
        }
    }
    if(incident_checkboxes.value.length > 0) {
        console.log(incident_checkboxes.value);
        let codes = [];
        for(let incident_name of incident_checkboxes.value) {
            let incident = possible_incidents.value.find(i => i.name === incident_name);
            if(incident) {
                codes = codes.concat(incident.codes);
            }
        }
        filters.append("code", codes.join(","));
    }
    if(start_date.value !== '') {
        filters.append("start_date", start_date.value);
    }
    if(end_date.value !== '') {
        filters.append("end_date", end_date.value);
    }
    filters.append("limit", max_incidents.value);
    let url = crime_url.value + "/incidents?" + filters.toString();
    let thisUpdate = ++lastUpdate;
    let data = await fetch(url);
    let crimes = await data.json();
    
    if(thisUpdate !== lastUpdate) {
        // A newer update has been requested, discard this one
        return;
    }
    for (let marker of map.neighborhood_markers) {
        marker.marker = 0;
    }
    for(let crime of crimes) {
        map.neighborhood_markers[crime.neighborhood_number - 1].marker += 1;
        // violent, property, other.
        const type = crime.code;
        const types = {
            violent: [[100, 453], [810, 863], [3100, 3100]],
            property: [[500, 613], [621, 732], [900, 1436]]
        }
        crime.is_violent = false;
        crime.is_property = false;
        crime.is_other = true;
        for(let range of types.violent) {
            if(type >= range[0] && type <= range[1]) {
                crime.is_violent = true;
                crime.is_property = false;
                crime.is_other = false;
                break;
            }
        }
        for(let range of types.property) {
            if(type >= range[0] && type <= range[1]) {
                crime.is_property = true;
                crime.is_violent = false;
                crime.is_other = false;
                break;
            }
        }
    }
    crime_loading.value = false;
    crimes_list.value = crimes;

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
    try {
        const res = await fetch(`https://nominatim.openstreetmap.org/search?q=${address}&format=json`);
        const json = await res.json();
        if(json.length === 0) {
            crime_loading.value = false;
            alert("Could not find location for this crime.");
            return;
        }
        specificMarker = L.marker([json[0].lat, json[0].lon])
            .bindPopup(
                `
                <b>Date:</b> ${crime.date}<br />
                <b>Time:</b> ${crime.time}<br />
                <b>Incident:</b> ${crime.incident}<br />
                <button class="pop-delete" onclick="globalDeleteCrime('${crime.case_number}')">Delete</button>
                `
            )
            .addTo(toRaw(map.leaflet))
            .openPopup();
        map.leaflet.setView([json[0].lat, json[0].lon], 16);
        window.scrollTo(0, 0);
    } finally {
        crime_loading.value = false;
    }
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

function deleteCrime(case_number) {
    if(specificMarker !== null) {
        map.leaflet.removeLayer(specificMarker);
        specificMarker = null;
    }
    fetch(crime_url.value + "/remove-incident", {
        method: "DELETE",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({ case_number: case_number })
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
window.globalDeleteCrime = function(case_number) {
    deleteCrime(case_number);
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
        <div id="crimes-list">
            <h2>Crimes List</h2>
            <span>Click on a row to jump to its location on the map. It may take a while to go there due to the location API call.</span>
            <br>
            <span class="violent legend">Violent</span>
            <span class="property legend">Property</span>
            <span class="other legend">Other</span>
            <div>
                <h3>Filters</h3>
                <div class="grid-x">
                    <div class="cell small-3">
                        <h5>Incident Types</h5>
                        <div v-for="possible_incident in possible_incidents">
                            <input type="checkbox" :id="possible_incident.name" :value="possible_incident.name" v-model="incident_checkboxes">
                            <label :for="possible_incident.name">{{ possible_incident.name }}</label>
                        </div>
                    </div>
                    <div class="cell small-3">
                        <h5>Neighborhoods </h5>
                        (if none selected, will use what is visible on map)
                        <div v-for="neighborhood in neighborhoods">
                            <input type="checkbox" :id="neighborhood.name" :value="neighborhood.id" v-model="neighborhood_checkboxes">
                            <label :for="neighborhood.name">{{ neighborhood.name }}</label>
                        </div>
                    </div>
                    <div class="cell small-3">
                        <h5>Date Range</h5>
                        <label for="start-date">Start Date:</label>
                        <input type="date" id="start-date" v-model="start_date" />
                        <br />
                        <label for="end-date">End Date:</label>
                        <input type="date" id="end-date" v-model="end_date" />
                    </div>
                    <div class="cell small-3">
                        <h5>Max incidents</h5>
                        <input type="number" v-model="max_incidents" min="1" />
                    </div>
                </div>
                <button @click="updateCrimes" class="search-button">Search Crimes</button>
            </div>
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
                    <tr class="crime-row" :class="{violent: crime.is_violent, property:crime.is_property, other: crime.is_other}" v-for="crime in crimes_list" :key="crime.case_number" @click="showMarker(crime)">
                        <td>{{ crime.case_number }}</td>
                        <td>{{ crime.date }}</td>
                        <td>{{ crime.time}}</td>
                        <td>{{ crime.incident }}</td>
                        <td>{{ crime.police_grid }}</td>
                        <td>{{ crime.neighborhood_name }}</td>
                        <td>{{ crime.incident_type }}</td>
                        <td>{{ crime.block }}</td>
                        <td><button @click="deleteCrime(crime.case_number)">Delete</button></td>
                    </tr>
                </tbody>
            </table>
        </div>
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

.crime-row {
    cursor: pointer;
}

.crime-loading {
    filter: blur(2px);
}
.violent {
    background-color: #ff9999;
}
.property {
    background-color: #ffff99;
}
.other {
    background-color: #99ccff;
}
.legend {
    margin: 5px;
    padding: 5px;
}
.search-button {
    background-color: #7694f8;
    border-radius: 5px;
    padding: 5px;
    cursor: pointer;
}
</style>
<style>
    .crime-row button, .pop-delete {
    background-color: orange;
    border-radius: 5px;
    padding: 5px;
    cursor: pointer;
}
</style>