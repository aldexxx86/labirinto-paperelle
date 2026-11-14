# Piano di progetto — "Paperelle nel Labirinto" (FPS 3D da browser)

*Versione 1 · 14 novembre 2026 · per matteo*

## 1. Sintesi

Uno sparatutto in prima persona 3D che gira in **una sola pagina HTML** nel browser. Il giocatore è un umano dentro un labirinto ispirato (solo come atmosfera) al vecchio salvaschermo "3D Maze" di Windows: corridoi a griglia, muri con texture a mattoni, soffitto e pavimento uniformi. Nel labirinto girano **~1000 paperelle colorate**; dei predatori (zombie, preti, suore "spippate", leoni) le mangiano. Il giocatore deve eliminare tutti i predatori prima che le paperelle finiscano. Il punteggio è proporzionale alle paperelle sopravvissute, con quelle arcobaleno che valgono di più.

Tutti gli asset (texture, modelli, suoni) saranno **originali**, generati via codice: nessun elemento grafico o sonoro preso dal salvaschermo Microsoft.

## 2. Tecnologia consigliata

**Scelta: Three.js (sopra WebGL 2), caricato da CDN, in un unico file HTML.**

| Opzione | Pro | Contro | Verdetto |
|---|---|---|---|
| WebGL "puro" | Controllo totale, zero dipendenze | Bisogna scrivere a mano shader, matrici, caricamento texture, instancing, picking: settimane di lavoro prima di vedere il gioco | Sconsigliato |
| **Three.js** | Instancing pronto (`InstancedMesh`), `PointerLockControls` per l'FPS, luci, materiali, raycasting, particelle; enorme documentazione | ~600 KB di libreria da CDN | **Consigliato** |
| Babylon.js | Motore più "completo" (fisica, GUI) | Più pesante, più opinioni da seguire | Valida alternativa |
| Motori (Unity/Godot export) | Editor visuale | Export pesante, non è "una pagina HTML" semplice | No |

Perché Three.js: il gioco ha geometria semplice (muri a griglia) ma **tante entità** (1000+ paperelle). Three.js dà l'instancing e il raycasting già fatti, così il tempo va sul gameplay. Non serve un motore fisico: le collisioni in un labirinto a griglia si risolvono con la griglia stessa.

Altre scelte tecniche:
- **Un solo `index.html`** con `<script type="module">` e Three.js importato da CDN con versione fissata (es. `three@0.160.0`). Si apre con doppio clic o da un hosting statico (GitHub Pages).
- **Texture procedurali** disegnate su `<canvas>` (mattoni, pavimento, legno delle casse) → nessun file esterno.
- **Audio** con Web Audio API, suoni sintetizzati (spari, esplosioni, "quack").
- **Input**: Pointer Lock API (mouse per guardare) + tastiera WASD.
- Nessun backend. Il record si salva in `localStorage`.

## 3. Architettura del codice

Anche in un solo file, il codice è diviso in moduli logici (sezioni/classi):

```
index.html
 ├─ Config          costanti di bilanciamento (velocità, danni, quantità)
 ├─ MazeGenerator   crea la griglia del labirinto (seed riproducibile)
 ├─ MazeRenderer    costruisce muri/pavimento/soffitto come mesh unite
 ├─ Grid/NavMap     griglia di celle: collisioni, line-of-sight, pathfinding
 ├─ EntityManager   array di entità con aggiornamento a "pool" (niente new nel loop)
 │   ├─ Ducks       dati in array tipizzati + 1 InstancedMesh
 │   ├─ Enemies     Zombie, Prete, Suora, Leone (macchina a stati)
 │   ├─ Pickups     casse, medikit, munizioni, armi
 │   └─ Projectiles granate, razzi, particelle
 ├─ Player         movimento, vita, inventario armi
 ├─ Weapons        definizioni armi + logica di fuoco (hitscan / proiettile / area)
 ├─ SpatialHash    lookup rapido "chi è vicino a me" sulla griglia
 ├─ HUD            overlay HTML/CSS sopra il canvas + minimappa
 ├─ Audio          sintesi suoni
 └─ Game           loop principale, stati (menu, gioco, pausa, vittoria, game over)
```

**Loop di gioco**: `requestAnimationFrame` con logica a passo fisso (es. 60 Hz) e rendering a frame variabile. L'IA delle paperelle può aggiornarsi a frequenza ridotta (es. 1/3 dei frame a rotazione) per risparmiare CPU.

## 4. Il labirinto

**Generazione**: algoritmo *recursive backtracker* (DFS) su una griglia, es. **41×41 celle**. Produce un labirinto "perfetto" (un solo percorso tra due punti). Poi si **rimuove il 10–15% di muri a caso** per creare anelli e piccole stanze: serve per il gameplay (i predatori non devono essere tutti in vicoli ciechi, le paperelle devono poter fuggire). Si aggiungono **4–8 stanze più larghe** (3×3 celle) per i combattimenti e le casse.

**Stile visivo** (ispirato, non copiato):
- Muri alti con texture a mattoni procedurale; varianti di colore per zona così ci si orienta.
- Pavimento e soffitto in tinta unita o leggermente quadrettati.
- Illuminazione semplice: luce ambiente + una luce sul giocatore (torcia leggera) + nebbia (`scene.fog`) che nasconde il fondo dei corridoi e aiuta le prestazioni.
- Elementi originali: lampade a muro, cartelli con frecce, macchie di "piume" dove una paperella è stata mangiata.

**Rendering**: tutti i muri uniti in **una sola geometria** (merge), così il labirinto intero è 1–3 draw call.

**Dimensione**: 41×41 celle da 4 m ≈ 160×160 m. Con 1000 paperelle sono circa 1,2 paperelle per cella percorribile: corridoi vivaci ma non intasati. Da tarare in prova.

## 5. Le paperelle

- **Numero iniziale**: 1000 (configurabile).
- **Colori**: 8–10 tinte vivaci assegnate a caso (colore per istanza su `InstancedMesh`).
- **Paperelle arcobaleno**: ~5% (≈50). Shader/colore che cicla nel tempo, leggera scia luminosa. Valgono **5 punti** contro **1** delle normali.
- **Modello**: low-poly originale costruito in codice (corpo + testa + becco), ~200 triangoli.
- **Comportamento** (molto leggero, perché sono tante):
  - *Girovagare*: scelgono una cella vicina e ci camminano, con un leggero dondolio.
  - *Fuga*: se un predatore è entro 2–3 celle, scappano nella direzione opposta (più veloci per pochi secondi).
  - *Seguire il giocatore* (opzionale, carino): se il giocatore è vicino e non spara, alcune lo seguono come anatroccoli. Utile per "scortarle" lontano dai pericoli.
- **Non fanno male al giocatore**.
- **Fuoco amico**: si possono colpire. Ogni paperella uccisa dal giocatore toglie i suoi punti **più una penalità** (es. −2 extra, −10 per un'arcobaleno) e mostra un messaggio "Hai colpito una paperella!". Granate e bazooka diventano quindi armi da usare con cervello.

## 6. I nemici (predatori)

Tutti mangiano paperelle e possono attaccare il giocatore. Ogni nemico ha una macchina a stati: **Vaga → Caccia paperella → Mangia → Attacca giocatore → Muore**.

| Nemico | Velocità | Vita | Danno al giocatore | Fame (paperelle) | Comportamento |
|---|---|---|---|---|---|
| **Zombie** | Lenta (1,5 m/s) | Alta (150) | Medio, a contatto | Bassa, mangia piano (3 s) | Barcolla dritto verso la paperella più vicina; in gruppo. Tanti, ~25 |
| **Prete** | Veloce (6 m/s) | Bassa (60) | Basso, a contatto | Alta, mangia in fretta (1 s) | Corre per i corridoi a caccia; scappa se ferito. ~12 |
| **Suora spippata** | Media ma erratica | Media (90) | Medio, colpi a raffica | Media | Cambia direzione a caso, scatti improvvisi, a volte gira su sé stessa: difficile da colpire. ~12 |
| **Leone** | Veloce negli scatti | Alta (200) | Alto, salto/morso | Media | Aggressivo: se vede il giocatore lo carica con preferenza sulle paperelle; ruggito d'avviso. ~6 |

Numeri iniziali indicativi, da bilanciare: con queste quantità e velocità le paperelle dovrebbero finire in **10–15 minuti** se il giocatore non fa nulla.

**IA e navigazione**:
- **Pathfinding A\*** sulla griglia del labirinto (piccola, quindi veloce). I percorsi si ricalcolano solo ogni 0,5–1 s o quando il bersaglio cambia cella.
- **"Mappa del flusso" verso il giocatore** (BFS dalla cella del giocatore, ricalcolata quando cambia cella): tutti i nemici che inseguono il giocatore la leggono gratis, senza A\* individuale.
- **Linea di vista** con la griglia (algoritmo DDA/Bresenham sulle celle), non con raycast 3D.
- **Ricerca paperelle** tramite *spatial hash* per cella: "le paperelle nella mia cella e nelle 8 vicine".
- **Mangiare**: il nemico si ferma qualche secondo (finestra in cui è vulnerabile, va comunicata visivamente).

**Grafica nemici**: modelli low-poly originali fatti in codice (forme semplici e riconoscibili: zombie verdastro con braccia avanti, prete in tonaca nera, suora con velo e movimenti a scatti, leone con criniera). Tono volutamente caricaturale/cartoon. Animazioni procedurali (oscillazione arti, saltelli) senza scheletri.

## 7. Il giocatore e le armi

**Giocatore**: 100 punti vita, movimento WASD + corsa (Shift), mouse per mirare, clic per sparare, rotellina/tasti 1–6 per cambiare arma, R per ricaricare, E per aprire casse. Collisione con i muri tramite la griglia.

**Armi** (da raccogliere, munizioni con limite per arma; il requisito "armi e munizioni infinite" è interpretato come *si possono raccogliere senza limite di numero di raccolte*, vedi sezione 12):

| Arma | Tipo di colpo | Danno | Cadenza | Munizioni | Note |
|---|---|---|---|---|---|
| **Pistola Colt** | Hitscan | 35 | Media | Infinite | Arma di partenza, precisa |
| **Fucile a pompa** | 8 pallini hitscan a cono | 8×15 | Lenta | 8 / 48 | Devastante da vicino, rischio paperelle |
| **Uzi** | Hitscan, dispersione | 12 | Molto alta | 32 / 192 | Leggera, poco precisa |
| **Mitragliatrice** | Hitscan | 20 | Alta | 100 / 300 | Pesante: rallenta il giocatore del 20% |
| **Bomba a mano** | Proiettile con rimbalzo, esplode dopo 2,5 s | 150 in area (raggio 5 m) | — | 5 | Rimbalza sui muri; uccide paperelle nell'area |
| **Bazooka** | Razzo visibile | 250 in area (raggio 6 m) | Molto lenta | 4 / 12 | Il più potente; danneggia anche il giocatore se troppo vicino |

- **Hitscan** con raycast contro: muri (griglia), nemici (sfere di collisione) e paperelle (sfere piccole). Il raycast si ferma al primo bersaglio, quindi una paperella davanti a un nemico "prende il colpo".
- Effetti: lampo alla canna, decal sui muri, particelle di piume/fumo, rinculo della telecamera.

## 8. Casse e raccolta

- **~40 casse** di legno sparse, con più probabilità nelle stanze larghe e nei vicoli ciechi.
- Contenuto casuale pesato: **medikit** (+25 o +50 vita), **munizioni** (per un'arma a caso o quella in uso), **arma nuova** (più rara per bazooka e mitragliatrice).
- Si aprono con E oppure sparandoci.
- **Ricomparsa**: le casse vuote ricompaiono dopo 60–90 s in un punto lontano dal giocatore, così non si resta mai a secco (coerente con "munizioni infinite").
- Le armi lasciate cadere dai nemici: opzionale, fase successiva.

## 9. Punteggio, vittoria e sconfitta

**Punteggio live** = (paperelle normali vive × 1) + (paperelle arcobaleno vive × 5) − penalità fuoco amico.
Il valore cala in tempo reale ogni volta che una paperella muore: è esattamente il "proporzionale alle sopravvissute" richiesto.

**Punteggio finale** (solo in caso di vittoria) = punteggio live × moltiplicatore (bonus per tempo impiegato e vita rimasta, opzionale) + piccolo bonus per nemico ucciso. Record salvato in locale.

**Condizioni di fine**:
- **Game over** se: la vita del giocatore arriva a 0, **oppure** tutte le paperelle sono morte.
- **Vittoria** se: tutti i nemici sono morti **e** almeno una paperella è viva.
- Caso limite: l'ultimo nemico e l'ultima paperella muoiono insieme (es. stessa esplosione) → vale la regola "nessuna paperella viva" = game over. Il controllo si fa a fine frame.

## 10. HUD e interfaccia

Overlay HTML/CSS sopra il canvas 3D (più facile da fare bello di un HUD in WebGL):
- **In basso a sinistra**: barra vita.
- **In basso a destra**: arma attuale, munizioni nel caricatore / di riserva, icone delle armi possedute.
- **In alto al centro**: punteggio grande, paperelle vive (normali + arcobaleno con icona), nemici rimasti per tipo.
- **Mirino** al centro, che si allarga con la dispersione.
- **Minimappa** (canvas 2D, in alto a destra): mostra la zona esplorata, punti colorati per paperelle vicine, segnale rosso quando un nemico sta mangiando.
- **Avvisi**: "Una paperella arcobaleno è stata mangiata!", "Hai colpito una paperella!", bordo rosso quando si subisce danno con indicatore di direzione.
- **Schermate**: menu iniziale (comandi, difficoltà), pausa (Esc), vittoria e game over con riepilogo e "Rigioca".

## 11. Prestazioni

Obiettivo: **60 fps** su un portatile medio con Chrome/Firefox/Edge.

- **Paperelle con un solo `InstancedMesh`**: 1000 paperelle = 1 draw call. Posizione, rotazione e colore aggiornati scrivendo direttamente nelle matrici di istanza. Lo stato (posizione, cella, direzione, stato IA) sta in array tipizzati (`Float32Array`), non in oggetti JS, per evitare il garbage collector.
- **Nemici** con un `InstancedMesh` per tipo (al massimo poche decine ciascuno).
- **Labirinto** unito in una geometria → poche draw call.
- **Nebbia + distanza massima di visualizzazione**: le paperelle oltre ~40 m non si animano, quelle in celle lontane aggiornano l'IA 2–5 volte al secondo invece che 60.
- **Spatial hash su griglia** per tutte le domande "vicino a me" (O(1) invece di O(n²)).
- **Pool di oggetti** per proiettili, particelle e decal (nessuna allocazione nel loop).
- **Ombre disattivate** o solo finte (blob sotto le entità).
- **Contatore fps di debug** e parametro di qualità (numero di paperelle, risoluzione di rendering) nel menu, per macchine lente.

Stima budget: < 50 draw call, < 300.000 triangoli, IA < 4 ms per frame.

## 12. Punti da confermare

1. **"Armi e munizioni infinite"**: ho interpretato che si possono raccogliere armi e munizioni senza limite, ma ogni arma ha un caricatore e una riserva massima, e le casse ricompaiono. In alternativa: munizioni davvero infinite (più facile, meno strategico). Consiglio la prima.
2. **Paperelle che seguono il giocatore**: idea in più non richiesta, da tenere o togliere.
3. **Tono dei nemici religiosi** (preti, suore): li tratterei in stile cartoon/caricaturale, senza simboli religiosi espliciti, così il gioco resta condivisibile.
4. **Difficoltà**: tre livelli (facile/normale/difficile) che cambiano numero e velocità dei nemici.

## 13. Milestone

Stime per sviluppo assistito da Claude, in sessioni di lavoro; ogni fase produce un `index.html` giocabile.

| Fase | Contenuto | Risultato verificabile |
|---|---|---|
| **M0 – Prototipo** | Pagina HTML + Three.js, generazione labirinto, rendering muri con texture procedurale, movimento FPS con collisioni | Si cammina nel labirinto |
| **M1 – Paperelle** | InstancedMesh, 1000 paperelle colorate + arcobaleno, girovagare, spatial hash, contatore fps | 1000 paperelle a 60 fps |
| **M2 – Nemici** | 4 tipi di nemico, macchina a stati, A\* e flow field, mangiare paperelle, attacco al giocatore, vita | Le paperelle calano, il giocatore può morire |
| **M3 – Armi** | 6 armi, hitscan, granate e razzi con esplosione ad area, fuoco amico con penalità, effetti | Si possono uccidere nemici e (per errore) paperelle |
| **M4 – Casse e loop di gioco** | Casse con medikit/munizioni/armi e ricomparsa, punteggio, vittoria e game over, menu | Partita completa dall'inizio alla fine |
| **M5 – HUD e rifinitura** | HUD completo, minimappa, avvisi, audio sintetizzato, animazioni procedurali, particelle | Gioco "presentabile" |
| **M6 – Bilanciamento** | Taratura numeri, livelli di difficoltà, test su più browser, impostazioni qualità, pubblicazione (GitHub Pages) | Versione 1.0 |

## 14. Rischi e mitigazioni

| Rischio | Impatto | Mitigazione |
|---|---|---|
| Prestazioni con 1000 paperelle + IA | Alto | Instancing, array tipizzati, IA a frequenza ridotta per le lontane, spatial hash; misurare già in M1 |
| Bilanciamento: paperelle finiscono troppo in fretta o troppo piano | Alto | Tutti i numeri in `Config`; modalità debug con velocità accelerata; prove ripetute in M6 |
| Esplosivi che "sterminano" paperelle e rendono frustrante il gioco | Medio | Penalità chiara, raggio d'area visibile, avviso quando ci sono tante paperelle vicine al punto di mira |
| Nemici bloccati nei muri o in loop | Medio | Navigazione solo su griglia, controllo "bloccato da > 2 s → ricalcola percorso" |
| Labirinto disorientante, giocatore perso | Medio | Minimappa, colori diversi per zona, segnale verso il nemico più vicino che sta mangiando |
| Un solo file HTML che cresce troppo | Basso | Codice organizzato in sezioni/classi; eventualmente separare in moduli in fase di sviluppo e unire in un file alla fine |
| Somiglianza eccessiva con asset Microsoft | Basso | Solo ispirazione stilistica; texture, modelli e suoni creati da zero in codice |
| Compatibilità browser (Pointer Lock, WebGL 2) | Basso | Test su Chrome, Firefox, Edge, Safari; messaggio chiaro se WebGL non è disponibile |

## 15. Prossimo passo

Confermare i punti della sezione 12 e partire con **M0 + M1** (labirinto percorribile con 1000 paperelle), che è anche il test decisivo per le prestazioni.
