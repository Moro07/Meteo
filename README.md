# MeteoMap

## Stakeholder

Applicazione web per visualizzare in tempo reale le temperature su una mappa geografica interattiva.

---

## Analisi dei requisiti

### Funzionali

L’applicazione deve:

* Mostrare una mappa interattiva con posizionamento geografico preciso
* Recuperare e visualizzare la temperatura in tempo reale da Open-Meteo
* Consentire l’aggiunta di nuove località cliccando sulla mappa
* Memorizzare le località e le temperature in un database locale (PocketBase)
* Visualizzare popup informativi con latitudine, longitudine e temperatura

### Non funzionali

* Interfaccia chiara e semplice
* Caricamento rapido della mappa con tile OpenStreetMap
* Aggiornamento dati meteorologici in tempo reale
* Buone performance anche con più marker sulla mappa

### Vincoli

* HTML, CSS e JavaScript
* Servizi open-source o gratuiti
* Architettura client-server locale (PocketBase)

### Analisi di fattibilità

Il progetto è realizzabile con API gratuite (Open-Meteo, Nominatim, OpenStreetMap) e PocketBase per la persistenza locale.

---

## Progettazione

### Componenti principali

1. **Interfaccia Utente (UI)**

   * Mappa interattiva con Leaflet.js
   * Popup informativi con latitudine, longitudine e temperatura
   * Click sulla mappa per inserire nuove località

2. **Logica applicativa**

   * Recupero temperatura tramite Open-Meteo
   * Geocoding inverso tramite Nominatim
   * Aggiornamento marker e popup sulla mappa
   * Cache locale dei dati per ridurre richieste duplicate

3. **Persistenza Dati**

   * PocketBase per memorizzare località e temperature
   * Salvataggio di nuove posizioni al click sulla mappa

---

## Flusso dati

```
Utente clicca sulla mappa
          ↓
Nuova posizione inviata a PocketBase
          ↓
Recupero temperatura da Open-Meteo
          ↓
Recupero nome area tramite Nominatim (reverse geocoding)
          ↓
Dati salvati in cache e su PocketBase
          ↓
Marker aggiornato sulla mappa con popup
```

---

## Implementazione

**Inizializzazione mappa e PocketBase:**

```javascript
import PocketBase from 'pocketbase';

const mappa = L.map("map").setView([45.25, 10.04], 3);
const datiCache = {};

L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '&copy; OpenStreetMap'
}).addTo(mappa);

const db = new PocketBase('http://127.0.0.1:8090');
```

**Recupero temperatura da Open-Meteo:**

```javascript
const getTemperatura = async (lat, lon) => {
    const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m`;
    const risposta = await fetch(url);
    if (!risposta.ok) throw new Error(`Errore Open-Meteo: ${risposta.status}`);
    const meteo = await risposta.json();
    return meteo.current.temperature_2m;
};
```

**Aggiornamento marker e statistiche:**

```javascript
const aggiornaStats = (listaTemp) => {
    const valide = listaTemp.filter(v => !isNaN(v));
    const totale = valide.length;
    let media = "N/D", massima = "N/D", minima = "N/D";

    if (totale > 0) {
        const somma = valide.reduce((a, b) => a + b, 0);
        media = (somma / totale).toFixed(2);
        massima = Math.max(...valide).toFixed(2);
        minima = Math.min(...valide).toFixed(2);
    }

    document.getElementById("count").textContent = totale;
    document.getElementById("avg-temp").textContent = media;
    document.getElementById("max-temp").textContent = massima;
    document.getElementById("min-temp").textContent = minima;
};
```

**Caricamento dati e marker sulla mappa:**

```javascript
const caricaTutto = async () => {
    const tempCorrenti = [];

    mappa.eachLayer(layer => {
        if (layer instanceof L.Marker)
            mappa.removeLayer(layer);
    });

    const res = await db.collection("prova").getList();
    const punti = res.items;

    await Promise.all(
        punti.map(async (p) => {
            const lat = p.geopoint.lat;
            const lon = p.geopoint.lon;
            const chiave = `${lat},${lon}`;

            let nomeArea, temperatura;

            if (datiCache[chiave]) {
                ({ nomeArea, temperatura } = datiCache[chiave]);
            } else {
                try {
                    const nomiRes = await fetch(
                        `https://nominatim.openstreetmap.org/reverse?format=geojson&lat=${lat}&lon=${lon}`
                    );
                    const geo = await nomiRes.json();
                    nomeArea = geo.features[0].properties.address.city ||
                               geo.features[0].properties.address.county ||
                               "N/D";
                } catch {
                    nomeArea = "Errore città";
                }

                try {
                    temperatura = await getTemperatura(lat, lon);
                } catch {
                    temperatura = NaN;
                }

                datiCache[chiave] = { nomeArea, temperatura };
            }

            tempCorrenti.push(parseFloat(temperatura));

            L.marker([lat, lon]).addTo(mappa)
                .bindPopup(`Località: ${nomeArea} <br>Lat: ${lat} <br>Lon: ${lon} <br>Temperatura: ${isNaN(temperatura) ? "N/D" : temperatura + " °C"}`);
        })
    );

    aggiornaStats(tempCorrenti);
};

caricaTutto();
```

**Aggiunta di nuove località cliccando sulla mappa:**

```javascript
mappa.on("click", async (evento) => {
    const nuovo = {
        geopoint: { lat: evento.latlng.lat, lon: evento.latlng.lng }
    };
    await db.collection("prova").create(nuovo);
    caricaTutto();
});
```

---

## Verifica e test

* Caricamento mappa e marker
* Recupero dati meteo da Open-Meteo
* Geocoding inverso con Nominatim
* Inserimento di nuove località tramite click sulla mappa
* Persistenza dei dati su PocketBase

---

## Distribuzione

* **Repository:** GitHub
* **Hosting:** localhost (PocketBase)

---

## Evoluzione futura

* Filtri per temperature o località
* Aggiornamento automatico periodico dei dati
* Visualizzazione media, minima e massima delle temperature in ogni luogo

---

## Stack Tecnologico

* **Frontend:** Leaflet.js, HTML, JavaScript 
* **Backend/Dati:** PocketBase, Open-Meteo, Nominatim
* **Mappe:** OpenStreetMap

---
