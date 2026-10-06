<script>
const initialBauwerke = [
    { nr: "210", name: "Hochstr. über die Bonner Straße", bereich: "KB Süd", gruppe: "Gruppe 1", lat: 51.16974, lng: 6.840276 },
    { nr: "213", name: "Hochstr. Bahnhof Benrath", bereich: "KB Süd", gruppe: "Gruppe 4", lat: 51.16133, lng: 6.878507 },
    { nr: "401", name: "Str.-Br. Aderkirchweg", bereich: "KB Süd", gruppe: "Gruppe 4", lat: 51.19845, lng: 6.742439 },
    { nr: "407", name: "Str.-Br. Frankfurter Str.über Südallee", bereich: "KB Süd", gruppe: "Gruppe 5", lat: 51.15575, lng: 6.884619 },
    { nr: "414", name: "Str.-Br. Paul-Thomas-Straße", bereich: "KB Süd", gruppe: "Gruppe 5", lat: 51.16863, lng: 6.851337 }
];

let appData =
    JSON.parse(localStorage.getItem("bauunterhaltung_desktop_db_v7")) || {};

let activeBauwerkNr = null;
let activeMapMode = "offline";
let leafletMap = null;
let mapMarkersLayer = null;
let activeBegehungType = null;

initialBauwerke.forEach(b => {
    if (!appData[b.nr]) {
        appData[b.nr] = {
            laufend: {},
            beobachtung: {},
            history: []
        };
    }
});

function saveDB() {
    localStorage.setItem(
        "bauunterhaltung_desktop_db_v7",
        JSON.stringify(appData)
    );
}

function getCurrentPeriodInfo() {

    const now = new Date();
    const year = now.getFullYear();
    const month = now.getMonth();

    let periodIndex = 1;
    let periodName = "1. Abschnitt (Jan – Apr)";

    if (month >= 4 && month <= 7) {
        periodIndex = 2;
        periodName = "2. Abschnitt (Mai – Aug)";
    }
    else if (month >= 8) {
        periodIndex = 3;
        periodName = "3. Abschnitt (Sep – Dez)";
    }

    return {
        year,
        periodIndex,
        periodName
    };
}

function getBauwerkStatus(b) {

    const { year, periodIndex } =
        getCurrentPeriodInfo();

    const data =
        appData[b.nr] || {
            laufend: {},
            beobachtung: {},
            history: []
        };

    const laufendDone =
        !!data.laufend?.[`${year}-${periodIndex}`];

    const beobachtungDone =
        !!data.beobachtung?.[`${year}`];

    return {
        overallColor:
            laufendDone ? "gruen" : "gelb",
        laufendDone,
        beobachtungDone,
        data
    };
}

function updateStatistics() {

    let gruen = 0;
    let gelb = 0;
    let rot = 0;
    let offen = 0;

    initialBauwerke.forEach(b => {

        const status =
            getBauwerkStatus(b);

        if (status.overallColor === "gruen")
            gruen++;
        else
            gelb++;

        if (!status.beobachtungDone)
            offen++;
    });

    document.getElementById("statTotal").innerText =
        initialBauwerke.length;

    document.getElementById("statGruen").innerText =
        gruen;

    document.getElementById("statGelb").innerText =
        gelb;

    document.getElementById("statRot").innerText =
        rot;

    document.getElementById("statOffeneBeobachtung").innerText =
        offen;

    document.getElementById("mapCountBadge").innerText =
        `${initialBauwerke.length} Bauwerke`;
}

function getTourBauwerke() {

    return initialBauwerke
        .filter(b => b.lat && b.lng)
        .sort((a, b) => a.lat - b.lat);
}

function renderTourPlan() {

    const list =
        getTourBauwerke()
        .filter(b =>
            !getBauwerkStatus(b).laufendDone)
        .slice(0, 5);

    document.getElementById(
        "tourCountSummary"
    ).innerText =
        `Nächste ${list.length} Stationen`;

    document.getElementById(
        "tourNext5Grid"
    ).innerHTML = list.map((b, i) => `
        <div class="bg-slate-800 p-2 rounded-lg">
            <div class="text-amber-400 font-black">
                ${i + 1}
            </div>
            <div class="font-bold">
                ${b.nr}
            </div>
            <div class="text-[10px]">
                ${b.name}
            </div>
        </div>
    `).join("");
}

function renderHistory(nr) {

    const history =
        appData[nr]?.history || [];

    const box =
        document.getElementById("historyLog");

    if (!history.length) {

        box.innerHTML =
            "<div class='text-slate-500'>Keine Dokumentationen vorhanden.</div>";

        return;
    }

    box.innerHTML =
        history.map(entry => `
            <div class="border rounded-lg p-2 bg-white">
                <div class="font-bold">
                    ${entry.typ}
                </div>
                <div class="text-slate-500">
                    ${entry.datum}
                </div>
                <div>
                    ${entry.note || "-"}
                </div>
            </div>
        `).join("");
}

function renderApp() {

    const { year, periodName } =
        getCurrentPeriodInfo();

    document.getElementById(
        "currentPeriodDisplay"
    ).innerText =
        `${periodName} ${year}`;

    const search =
        document
            .getElementById("searchInput")
            ?.value
            ?.toLowerCase() || "";

    const list =
        document.getElementById(
            "bauwerkeList"
        );

    list.innerHTML = "";

    initialBauwerke
        .filter(b =>
            !search ||
            b.nr.toLowerCase().includes(search) ||
            b.name.toLowerCase().includes(search)
        )
        .forEach(b => {

            const status =
                getBauwerkStatus(b);

            const color =
                status.overallColor === "gruen"
                    ? "bg-emerald-500"
                    : "bg-amber-500";

            const div =
                document.createElement("div");

            div.className =
                "p-3 rounded-xl border bg-white shadow-sm cursor-pointer flex justify-between items-center";

            div.onclick =
                () => openModal(b.nr);

            div.innerHTML = `
                <div>
                    <span class="${color} text-white px-2 py-0.5 rounded text-xs font-black">
                        Nr. ${b.nr}
                    </span>
                    <span class="ml-2 font-semibold">
                        ${b.name}
                    </span>
                </div>
                <span class="text-xs text-slate-500">
                    Öffnen →
                </span>
            `;

            list.appendChild(div);
        });

    updateStatistics();
    renderOfflineSvgMap();
    renderTourPlan();

    if (mapMarkersLayer) {
        renderLeafletMarkers();
    }
}

function renderOfflineSvgMap() {

    const svg =
        document.getElementById(
            "offlineMapSvg"
        );

    svg.innerHTML = `
        <rect width="950" height="850" fill="#f8fafc"/>
        <text x="330" y="420" fill="#334155" font-size="18" font-weight="bold">
            Offline Karte aktiv
        </text>
    `;
}

function renderLeafletMarkers() {

    if (!mapMarkersLayer) return;

    mapMarkersLayer.clearLayers();

    initialBauwerke.forEach(b => {

        const status =
            getBauwerkStatus(b);

        const color =
            status.overallColor === "gruen"
                ? "#10b981"
                : "#f59e0b";

        L.circleMarker(
            [b.lat, b.lng],
            {
                radius: 8,
                color: color,
                fillColor: color,
                fillOpacity: 1
            }
        )
        .bindPopup(
            `<b>${b.nr}</b><br>${b.name}`
        )
        .addTo(mapMarkersLayer);
    });
}

function initLeafletMap() {

    if (leafletMap) return;

    leafletMap =
        L.map("leafletMapDiv")
        .setView([51.17, 6.84], 12);

    L.tileLayer(
        "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
        {
            attribution: "&copy; OpenStreetMap"
        }
    ).addTo(leafletMap);

    mapMarkersLayer =
        L.layerGroup().addTo(leafletMap);

    renderLeafletMarkers();
}

function setMapMode(mode) {

    activeMapMode = mode;

    document
        .getElementById("mapContainerOffline")
        .classList.toggle(
            "hidden",
            mode !== "offline"
        );

    document
        .getElementById("mapContainerOnline")
        .classList.toggle(
            "hidden",
            mode !== "online"
        );

    if (mode === "online") {

        initLeafletMap();

        setTimeout(() => {
            leafletMap.invalidateSize();
        }, 100);
    }
}

function openModal(nr) {

    activeBauwerkNr = nr;

    const b =
        initialBauwerke.find(
            x => x.nr === nr
        );

    const status =
        getBauwerkStatus(b);

    document.getElementById(
        "modalNummer"
    ).innerText =
        "Nr. " + b.nr;

    document.getElementById(
        "modalGruppe"
    ).innerText =
        b.gruppe;

    document.getElementById(
        "modalName"
    ).innerText =
        b.name;

    document.getElementById(
        "laufendStatusDetails"
    ).innerHTML =
        status.laufendDone
            ? "<span class='text-emerald-600 font-bold'>✓ erledigt</span>"
            : "<span class='text-amber-600 font-bold'>offen</span>";

    document.getElementById(
        "beobachtungStatusDetails"
    ).innerHTML =
        status.beobachtungDone
            ? "<span class='text-emerald-600 font-bold'>✓ erledigt</span>"
            : "<span class='text-amber-600 font-bold'>offen</span>";

    renderHistory(nr);

    document
        .getElementById("modal")
        .classList.remove("hidden");
}

function closeModal() {

    document
        .getElementById("modal")
        .classList.add("hidden");
}

function openBegehungForm(type) {

    activeBegehungType = type;

    document
        .getElementById("begehungFormTitle")
        .innerText =
        type === "laufend"
            ? "Laufende Beobachtung"
            : "Jahres-Beobachtung";

    document
        .getElementById("begehungFormBox")
        .classList.remove("hidden");
}

function closeBegehungForm() {

    document
        .getElementById("begehungFormBox")
        .classList.add("hidden");
}

function saveBegehung() {

    const note =
        document
            .getElementById("begehungNote")
            .value
            .trim();

    if (!activeBauwerkNr) return;

    const now = new Date();

    const year = now.getFullYear();

    const period =
        getCurrentPeriodInfo()
        .periodIndex;

    const entry = {
        datum:
            now.toLocaleDateString("de-DE"),
        typ:
            activeBegehungType,
        note
    };

    if (activeBegehungType === "laufend") {

        appData[activeBauwerkNr]
            .laufend[`${year}-${period}`] =
            entry;
    }
    else {

        appData[activeBauwerkNr]
            .beobachtung[`${year}`] =
            entry;
    }

    appData[activeBauwerkNr]
        .history
        .unshift(entry);

    saveDB();

    document.getElementById(
        "begehungNote"
    ).value = "";

    closeBegehungForm();

    openModal(activeBauwerkNr);

    renderApp();
}

function startGoogleMapsTourNext5() {

    const next =
        getTourBauwerke()
        .filter(b =>
            !getBauwerkStatus(b).laufendDone)
        .slice(0, 5);

    if (next.length < 2) return;

    const origin =
        `${next[0].lat},${next[0].lng}`;

    const destination =
        `${next[next.length - 1].lat},${next[next.length - 1].lng}`;

    const waypoints =
        next
        .slice(1, -1)
        .map(x =>
            `${x.lat},${x.lng}`)
        .join("|");

    const url =
        `https://www.google.com/maps/dir/?api=1&origin=${origin}&destination=${destination}&travelmode=driving&waypoints=${waypoints}`;

    window.open(url, "_blank");
}

function exportData() {

    const blob =
        new Blob(
            [JSON.stringify(appData, null, 2)],
            {
                type: "application/json"
            }
        );

    const a =
        document.createElement("a");

    a.href =
        URL.createObjectURL(blob);

    a.download =
        "backup.json";

    a.click();
}

function importData(e) {

    const file =
        e.target.files[0];

    if (!file) return;

    const reader =
        new FileReader();

    reader.onload = ev => {

        appData =
            JSON.parse(ev.target.result);

        saveDB();

        renderApp();
    };

    reader.readAsText(file);
}

renderApp();
</script>
