# TierForge

App single-file per creare tier list con tessere immagine e/o testo.
Nessuna build, nessuna dipendenza, funziona offline: si apre `index.html` col browser.

## Struttura

Tutto in `index.html` (~1780 righe): CSS nel `<head>`, JS in un unico `<script>` in fondo.
L'ordine delle sezioni JS è: helper → IndexedDB → preset → stato/undo → immagini/testo →
render → drag&drop → export → progetti → wiring UI → boot.

## Modello dati

```js
project = {
  id, name, updated,
  rows: [{ id, label, color, items: [itemId] }],
  pool: [itemId],                    // tessere non ancora ordinate
  items: { [id]: { id, type, ... } },// type: 'text' | 'image'
  settings: { tile }                 // lato tessera in px
}
```

`items` è il dizionario di tutte le tessere; `rows[].items` e `pool` contengono solo id.
Una tessera sta **sempre** in esattamente una lista: gli spostamenti la rimuovono da
tutte le liste prima di reinserirla.

- tessera testo: `{ id, type:'text', text }`
- tessera immagine: `{ id, type:'image', blobKey, label }` — il pixel data sta in
  IndexedDB store `blobs` sotto `blobKey`, mai nel JSON del progetto.

## Persistenza

IndexedDB `tierforge`, tre store:
- `projects` — un record per tier list (keyPath `id`)
- `blobs` — immagini come Blob, chiave `b_<itemId>`
- `meta` — `last` (ultimo progetto aperto), `theme`

**localStorage non è utilizzabile**: la quota di ~5MB salta al primo import di una cartella
di immagini. `save()` è debounced a 260ms.

`gcBlobs()` gira al boot e dopo ogni eliminazione: rimuove i blob non più referenziati
da nessun progetto. Per questo `duplicateProject()` **clona** i blob invece di
condividere il `blobKey` — altrimenti eliminare una copia cancellerebbe le immagini dell'altra.

## Decisioni tecniche non ovvie

**Pointer Events, non HTML5 drag&drop.** L'API nativa non permette il placeholder di
inserimento preciso e non funziona su touch. Il drag parte solo dopo 5px di movimento,
così un clic semplice resta una selezione. Se si trascina una tessera già selezionata
e la selezione ha più elementi, si muovono tutte insieme.

**Export PNG disegnato a mano su canvas** (`exportPNG`), non html2canvas: sarebbe una
dipendenza esterna che romperebbe l'uso offline e sbaglia il rendering dei font.
Le immagini vanno precaricate in `bitmaps` prima di disegnare, perché il draw è sincrono.

**Container query, non media query**, per il collasso delle etichette in toolbar
(`@container main`). La toolbar va misurata rispetto a `#main`, che le due sidebar
restringono — il viewport può essere largo 1280px mentre la toolbar ne ha 780.
Con una media query i pulsanti finivano sotto il pannello laterale.

**`min-height:0` sui figli di `#app`.** Senza, i figli in overflow gonfiano la griglia
oltre il viewport invece di scrollare internamente (la board diventava alta 876px in
una finestra da 720px).

**`paintSel()` invece di `renderAll()` per la sola selezione.** Cambiare selezione non
cambia la struttura: si alterna la classe `.sel` sui nodi esistenti. Ricostruire il DOM
rilancerebbe una `imageURL()` per ogni tessera immagine. Con 433 tessere: 9ms.

**Le immagini sono ridimensionate a 512px** sul lato lungo e ricompresse in WebP
all'import (`downscale`): tiene il DB leggero e il render fluido.

## Undo/redo

Stack di snapshot JSON (`rows`+`pool`+`items`+`name`), max 60 livelli.
`pushUndo()` va chiamato **prima** di mutare lo stato. Per le modifiche continue
(digitazione nel titolo o in un'etichetta riga) si usa `lastSnap`: si cattura al `focus`
e si spinge sullo stack al `blur`, così l'intera digitazione è un solo undo.

## Preset

`ROW_PRESETS` — 6 strutture di righe (S–F, giudizio, voti…).
`DATA_PRESETS` — 27 set di tessere pronti (636 voci, 562 uniche): SI base/derivate/composte,
prefissi decimali e binari, lunghezza, massa, tempo, superficie, volume, velocità, energia,
pressione, temperatura, angoli, elettromagnetismo, luce, chimica, nautica, cucina, storiche,
curiose, informatica, sport, economia, bar di Ascoli.
Aggiungerne uno = una voce nell'oggetto, la UI si costruisce da sola in `buildPresets()`.

Ogni card di `#dataPresets` ha due azioni: il corpo accoda le voci alla textarea
(si combinano più preset prima di confermare), il link in fondo chiama
`newProjectFromPreset()` e crea invece una tier list a sé, intitolata come il preset
e col pool già pieno — la lista aperta non viene toccata.

`allUnits()` unisce tutti i set tranne quelli in `NOT_UNITS` (`sport`, `economia`,
`curiose`, `barAscoli`) e deduplica: 433 voci. Aggiungendo un preset che non contiene unità di
misura vere, va inserito in `NOT_UNITS`; tutti i conteggi si ricalcolano da soli.

È usata in due punti: il pulsante **Carica TUTTE le unità di misura** (riempie la
textarea) e il **seeding al primo avvio** — a IndexedDB vuoto il progetto iniziale
"Unità di misura" nasce con tutte le 433 tessere già nel pool. Il seeding scatta solo
se non esiste alcun progetto: un reload non risemina e "Nuova tier list" resta vuota.

## Seeding

Due meccanismi distinti, da non confondere.

**Primo avvio** — a IndexedDB vuoto nasce "Unità di misura" col pool pieno. Scatta solo
se non esiste alcun progetto: un reload non risemina e "Nuova tier list" resta vuota.

**`seedPreset(key, flag)`** — crea un preset come tier list anche negli archivi già
esistenti, dove il primo avvio non scatta più (è così che "Bar di Ascoli" è comparsa
a chi usava già l'app). Il flag in `meta` la rende irripetibile: chi elimina la lista
non se la ritrova al reload. Con l'archivio vuoto la lista si aggiunge accanto a
"Unità di misura", che resta quella aperta; con un archivio già popolato la lista
appena creata viene aperta al posto di `last`, una volta sola.
Un archivio non accessibile non è un errore: il seed torna `null` e il boot prosegue.

## Test

Non c'è un runner installato nel progetto. La suite end-to-end usata in sviluppo
(playwright-core + il Chromium già presente in `%LOCALAPPDATA%/ms-playwright`)
copre boot, import testo/immagini, drag&drop, undo/redo, persistenza dopo reload,
export PNG/JSON e multi-progetto: 34 asserzioni.
Se si modifica il layout, ricontrollare che nessun elemento della sidebar/pool
intercetti i click della toolbar — è stato un bug reale.
