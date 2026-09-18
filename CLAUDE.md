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

## Mobile (< 760px)

Il layout desktop a 3 colonne (sidebar 210 + board + pool 230) non entra in 375px:
`#main` collassava a larghezza 0 e l'app era inutilizzabile. Sotto i 760px:

- `#app` passa a **una colonna**; sidebar e pool diventano pannelli `position:fixed`
  fuori schermo, richiamati da una **tab bar in basso** (Board / Da ordinare / Menu).
  Chi tocca lo schermo ha il pollice in basso, non in alto.
- **`#movebar`**, la barra "sposta in": su touch il drag fra board e pool è impossibile
  (sono due schede distinte), quindi selezionando delle tessere compare una barra con
  le righe di destinazione. È l'unico modo di ordinare da telefono.
- Delle 5 icone per riga ne restano 2 (colore, elimina): impilate imponevano ~140px
  di altezza a **ogni** riga, anche vuota, e in 844px se ne vedevano 3.
- Il pool diventa una **griglia** `auto-fill` invece del wrap flex.

Trappole verificate sul campo, non teoriche:

- **`grid-auto-rows:min-content` è obbligatorio** sul pool. Con `auto` le tracce si
  risolvono a 4px (residuo del `gap` del layout flex desktop) e le tessere alte 89px
  si sovrappongono: il tap colpisce la tessera sbagliata.
- **`renderAll()` non deve scrivere `--tile` inline su mobile**: uno stile inline batte
  la media query, quindi le tessere restavano a 96px ignorando i 72/64px previsti.
- **`100dvh`, non `100vh`**: con `100vh` la barra indirizzi di Safari iOS fa sforare
  il layout oltre lo schermo.
- **Input a font ≥ 16px**: sotto quella soglia Safari iOS zooma sulla pagina al focus.
- `viewport-fit=cover` + `env(safe-area-inset-*)` per notch e home indicator.

**Le immagini sono ridimensionate a 512px** sul lato lungo e ricompresse in WebP
all'import (`downscale`): tiene il DB leggero e il render fluido.

## Undo/redo

Stack di snapshot JSON (`rows`+`pool`+`items`+`name`), max 60 livelli.
`pushUndo()` va chiamato **prima** di mutare lo stato. Per le modifiche continue
(digitazione nel titolo o in un'etichetta riga) si usa `lastSnap`: si cattura al `focus`
e si spinge sullo stack al `blur`, così l'intera digitazione è un solo undo.

## Preset

`ROW_PRESETS` — 6 strutture di righe (S–F, giudizio, voti…).
`DATA_PRESETS` — 29 set di tessere pronti (703 voci, 629 uniche): SI base/derivate/composte,
prefissi decimali e binari, lunghezza, massa, tempo, superficie, volume, velocità, energia,
pressione, temperatura, angoli, elettromagnetismo, luce, chimica, nautica, cucina, storiche,
curiose, informatica, sport, economia, bar di Ascoli, specializzazioni mediche (le 51 scuole
di specializzazione italiane, area medica/chirurgica/servizi clinici), partiti politici
italiani (i 13 dei sondaggi nazionali più Ora!, DSP e Italia del Domani, con i simboli
opzionali).
Aggiungerne uno = una voce nell'oggetto, la UI si costruisce da sola in `buildPresets()`.

Ogni card di `#dataPresets` ha due azioni: il corpo accoda le voci alla textarea
(si combinano più preset prima di confermare), il link in fondo chiama
`newProjectFromPreset()` e crea invece una tier list a sé, intitolata come il preset
e col pool già pieno — la lista aperta non viene toccata.

Un preset può dichiarare **`emblems`**: una mappa `voce → [sfondo, testo, sigla]` che
aggiunge alla card una terza azione, "coi simboli". `projectFromPreset(key, rows, true)`
passa allora da tessere testo a tessere immagine e intitola la lista `<nome> (simboli)`.
Ce l'ha finora solo `partiti`. Le voci senza emblema in un preset misto restano testo,
e se il canvas o l'archivio falliscono la singola tessera ricade su testo invece di
far saltare l'intera creazione.

**`emblemBlob(bg, fg, sigla)`** disegna il simbolo su canvas — disco, anello bianco,
sigla rimpicciolita finché entra — e lo rasterizza in WebP. Da lì in poi è una tessera
immagine come quelle importate: stesso store `blobs`, stesso `imageURL()`, stesso export
PNG, stessa clonazione in `duplicateProject()`. Niente SVG (il rendering dei font dentro
`<img>` non è affidabile) e niente file esterni, così l'app resta offline e senza
dipendenze. Il disco è alzato al 45% dell'altezza: gli ultimi ~14% della tessera li
copre la didascalia `.cap` col nome del partito, che altrimenti taglierebbe la sigla.
**Non sono i contrassegni ufficiali depositati**, sono dischi in stile scheda elettorale
coi colori del partito.

Il blob va scritto **prima** del progetto che lo referenzia: `gcBlobs()` spazza i blob
che nessun progetto cita, quindi `projectFromPreset()` è `async` e i suoi due chiamanti
(`newProjectFromPreset`, `seedPreset`) salvano il progetto subito dopo.

`allUnits()` unisce tutti i set tranne quelli in `NOT_UNITS` (`sport`, `economia`,
`curiose`, `barAscoli`, `medicina`, `partiti`) e deduplica: 433 voci. Aggiungendo un preset che non
contiene unità di misura vere, va inserito in `NOT_UNITS`; tutti i conteggi si ricalcolano
da soli.

È usata in due punti: il pulsante **Carica TUTTE le unità di misura** (riempie la
textarea) e il **seeding al primo avvio** — a IndexedDB vuoto il progetto iniziale
"Unità di misura" nasce con tutte le 433 tessere già nel pool. Il seeding scatta solo
se non esiste alcun progetto: un reload non risemina e "Nuova tier list" resta vuota.

## Seeding

Due meccanismi distinti, da non confondere.

**Primo avvio** — a IndexedDB vuoto nasce "Unità di misura" col pool pieno. Scatta solo
se non esiste alcun progetto: un reload non risemina e "Nuova tier list" resta vuota.

**`seedPreset(key, flag, emblems)`** — crea un preset come tier list anche negli archivi
già esistenti, dove il primo avvio non scatta più (è così che "Bar di Ascoli",
"Specializzazioni mediche" e le due liste dei partiti sono comparse a chi usava già
l'app). L'elenco sta nella tabella `SEEDS`, che `seedAll()` percorre in ordine:
aggiungerne uno = una riga. Il flag in `meta` la
rende irripetibile: chi elimina la lista non se la ritrova al reload. Con l'archivio
vuoto la lista si aggiunge accanto a "Unità di misura", che resta quella aperta; con un
archivio già popolato la lista appena creata viene aperta al posto di `last`, una volta
sola — `seedAll()` chiama `seedPreset` in sequenza per ogni riga e restituisce l'ultimo
che ha effettivamente seminato, che il boot apre (se più preset scattano nello stesso boot, gli altri restano
comunque creati, solo non aperti).
Un archivio non accessibile non è un errore: il seed torna `null` e il boot prosegue.

## Test

Non c'è un runner installato nel progetto. La suite end-to-end usata in sviluppo
(playwright-core + il Chromium già presente in `%LOCALAPPDATA%/ms-playwright`)
copre boot, import testo/immagini, drag&drop, undo/redo, persistenza dopo reload,
export PNG/JSON e multi-progetto: 34 asserzioni.
Se si modifica il layout, ricontrollare che nessun elemento della sidebar/pool
intercetti i click della toolbar — è stato un bug reale.

Suite mobile separata (viewport 390x844, `isMobile`+`hasTouch`): 22 asserzioni su
layout, schede, barra "sposta in", selezione multipla, bottom sheet ed export.
Dopo ogni modifica al CSS mobile va ricontrollato che le tessere del pool **non si
sovrappongano** (confronto dei bounding box a coppie): è un errore che non si vede
a occhio in uno screenshot ma rende i tap inaffidabili.
