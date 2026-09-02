# Guida: animare lo schema dell'impianto di refrigerazione (eliche + flusso gas)

Sorgente: `Disegno impianto di refrigerazione per cella frigorifera alimentare -25°C in gas R452A` (Fluicom).
Demo funzionante: `assets/animazioni/demo-impianto-frigorifero.html`

**Importante — fedeltà al disegno originale.** La demo NON è una reinterpretazione grafica: lo sfondo è il PDF originale renderizzato 1:1 (stesso layout, stessi colori, stesso testo, stesse proporzioni — nulla è stato ridisegnato in uno stile diverso). Sopra a quello sfondo, immutato, vengono sovrapposti **solo** gli elementi che devono muoversi: le 5 pale di ciascuna elica (in un gruppo ruotabile) e i tratti di tubazione Ø28/Ø54 (ridisegnati esattamente sopra il tracciato reale, con un mascheramento bianco che copre la linea statica originale). Questo è il metodo corretto quando l'obiettivo è animare un disegno tecnico esistente senza alterarne l'aspetto: non si ricostruisce lo schema da zero con uno stile proprio, si anima *sopra* l'originale.

## 0. Cosa c'è nel disegno (analisi preliminare)

Prima di animare qualunque cosa, ho letto lo schema per capire cosa deve muoversi e cosa no:

| Elemento | Cosa animare | Come |
|---|---|---|
| Ventola **EVAPORATORE** (5 pale, guscio circolare) | Rotazione continua | `rotation: 360`, loop infinito |
| Ventola **CONDENSATORE** (5 pale, guscio circolare) | Rotazione continua | `rotation: -360` (senso opposto, varietà visiva) |
| Tubazione **rossa Ø28** (mandata/liquido alta pressione: compressore → olio → condensatore → ricevitori → valvola termostatica) | Flusso animato | tratteggio in movimento (`stroke-dashoffset`) |
| Tubazione **arancione Ø54** (aspirazione gas bassa pressione: evaporatore → compressore) | Flusso animato | tratteggio in movimento, direzione opposta/indipendente |
| Linee **ciano sottili** (prese di pressione/temperatura) | **Non animare** | sono segnali statici, non flusso di fluido — tratteggio fisso o nessuna animazione |
| Triangoli neri lungo i tubi | Già indicano la direzione nel disegno originale | usali come riferimento per il **verso** dell'animazione dashoffset, non serve ridisegnarli |

Questa distinzione (fluido in movimento vs. segnale statico) è il primo errore da evitare: animare anche le linee ciano renderebbe lo schema illeggibile e tecnicamente scorretto.

## 1. Preparare il "palco": sfondo originale + overlay animato

Ci sono due strade per animare un PDF tecnico esistente. La demo usa la **strada B**, che è quella corretta quando non si può/deve alterare il disegno:

- **Strada A — vettorializzazione completa**: si apre il PDF in Illustrator/Inkscape, si isolano TUTTI gli elementi in gruppi puliti, si esporta un SVG interamente ridisegnato. Rischio: è facilissimo, ricostruendo a mano, introdurre differenze (proporzioni, spessori, posizioni) rispetto all'originale — è quello che è successo nel primo tentativo di questa guida, ed è stato corretto.
- **Strada B — overlay su sfondo originale** *(usata qui)*: il PDF originale viene renderizzato com'è (nessuna ricostruzione) e usato come immagine di sfondo alla risoluzione/scala esatta del documento. Sopra, si disegnano **solo** i pochi elementi che devono muoversi — pale e tubi — posizionati con le coordinate reali del PDF, mascherando con un patch bianco la linea/pala statica sottostante. Il resto del disegno (testi, valvole, serbatoi, quotatura) non viene mai toccato: è l'immagine originale, quindi è impossibile che "sia diverso".

Passaggi concreti (Strada B):

1. **Estrai lo sfondo** renderizzando la pagina PDF a una risoluzione fissa e usando le **stesse unità del PDF come `viewBox`** dell'SVG (es. `viewBox="0 0 792 612"` se il PDF è 792×612pt) — così ogni coordinata che leggi dal PDF si traduce 1:1 in coordinate SVG, senza conversioni di scala da tenere a mente.
   ```python
   import fitz  # PyMuPDF
   doc = fitz.open("impianto.pdf")
   page = doc[0]
   pix = page.get_pixmap(matrix=fitz.Matrix(2.5, 2.5))  # 2.5x per nitidezza
   pix.save("sfondo.png")
   print(page.rect)  # dimensioni in unità PDF = dimensioni del viewBox
   ```
2. **Trova le coordinate reali** degli elementi da animare, invece di ridisegnarli a occhio. Per le tubazioni, il modo più affidabile è cercare i pixel del colore esatto della linea (rosso/arancio) nell'immagine e ricostruire i segmenti orizzontali/verticali (l'impianto qui è tutto ortogonale, quindi bastano run-length scan riga per riga e colonna per colonna):
   ```python
   # scansiona l'immagine, classifica ogni pixel come 'red'/'orange'/None,
   # poi raggruppa i pixel colorati contigui in segmenti H (stessa riga) e V (stessa colonna)
   # → ogni segmento trovato è un tratto di tubo con coordinate reali in unità PDF
   ```
   Per i centri delle eliche, ritaglia (`page.get_pixmap(clip=...)`) l'area della ventola e leggi a occhio il centro del mozzo e il raggio del cerchio/guscio esterno sull'immagine ritagliata ingrandita.
3. **Disegna solo l'overlay animato** con quelle coordinate:
   - un `<circle>` bianco (raggio = raggio pala + qualche unità) per mascherare le pale originali, poi ring + mozzo + 5 pale ridisegnate dentro `<g id="fan-evaporatore" style="transform-origin: CXpx CYpx;">` (stesso principio per il condensatore);
   - per ogni tratto di tubo: un `<path class="mask-line">` bianco leggermente più spesso della linea originale (copre anche le frecce di direzione già presenti), seguito da un `<path id="tubo-...">` colorato con le stesse coordinate, pronto per l'animazione del tratteggio.
4. **Verifica per sovrapposizione**: fai uno screenshot statico (o apri il file) e confronta a occhio l'overlay con lo sfondo — le linee mascherate/ridisegnate devono coincidere esattamente con quelle originali, senza doppie linee visibili né aree bianche "sbagliate".

Risultato atteso — un unico SVG dove lo sfondo è l'immagine originale e sopra ci sono solo gli elementi animati:

```html
<svg viewBox="0 0 792 612">
  <image href="data:image/png;base64,..." x="0" y="0" width="792" height="612" />

  <path class="mask-line" d="M 74 126 L 74 474" />
  <!-- ... una mask-line per ogni tratto di tubo ... -->

  <circle cx="399" cy="154" r="42" fill="#fff" /> <!-- maschera pale evaporatore -->
  <circle class="fan-ring" cx="399" cy="154" r="38" />
  <g id="fan-evap" style="transform-origin: 399px 154px;"> <!-- 5 pale --> </g>

  <path id="p-red-r1" class="pipe pipe-alta" d="M 74 126 L 74 474" />
  <!-- ... resto dei tratti tubo, rossi e arancioni ... -->
</svg>
```

## 2. Boilerplate anti-flicker (base di ogni animazione GSAP)

Da incollare per primo, sempre — evita che l'impianto appaia "scattato" o lampeggi al caricamento della pagina:

```html
<script>document.documentElement.classList.add('gsap-anim');</script>
<style>
  html.gsap-anim [data-gsap] { opacity: 0; }
</style>
```

Nell'HTML, avvolgi l'SVG dell'impianto in un contenitore con `data-gsap`, e alla fine dell'inizializzazione GSAP fai `gsap.set('[data-gsap]', {autoAlpha: 1})`. Così, se JS fallisce, il disegno resta comunque visibile (niente contenuto invisibile per sempre).

## 3. Rotazione delle eliche

```js
gsap.to('#fan-evaporatore', {
  rotation: 360,
  transformOrigin: '50% 50%',
  duration: 3,        // 3s per giro: velocità "industriale", non frenetica
  ease: 'linear',      // OBBLIGATORIO: una ventola non accelera/decelera ad ogni giro
  repeat: -1
});

gsap.to('#fan-condensatore', {
  rotation: -360,       // senso opposto: le due ventole non sembrano sincronizzate a comando
  transformOrigin: '50% 50%',
  duration: 2.4,
  ease: 'linear',
  repeat: -1
});
```

Punti critici:
- **`ease: 'linear'` sempre**, mai `power`/`elastic`/ecc. su una rotazione ciclica meccanica: qualsiasi altro ease crea un "singhiozzo" visibile ad ogni giro perché velocità in entrata e uscita del ciclo non combaciano.
- `transformOrigin: '50% 50%'` funziona perché il gruppo `#fan-evaporatore` contiene **solo le pale**, il cui bounding box è centrato sul mozzo (vedi punto 1.3). Se ruota "storto", il problema è quasi sempre qui, non nel codice GSAP.
- Durate diverse (3s vs 2.4s) e versi opposti (360 vs -360) evitano che le due ventole sembrino un'unica animazione duplicata.

## 4. Flusso del gas nei tubi (tratteggio animato)

Tecnica: il tubo ha un contorno tratteggiato (`stroke-dasharray`) che viene fatto scorrere lungo il proprio percorso animando `stroke-dashoffset` all'infinito. L'occhio percepisce delle "particelle" di gas che si muovono nella tubazione.

```css
.pipe { stroke-linecap: round; stroke-linejoin: round; fill: none; }
#tubo-mandata     { stroke: #e02e22; stroke-width: 7; }  /* rosso, Ø28 */
#tubo-aspirazione { stroke: #e08a1e; stroke-width: 9; }  /* arancio, Ø54 */
```

```js
function animaFlusso(selector, lunghezzaTratto, spazio, velocita) {
  const el = document.querySelector(selector);
  el.style.strokeDasharray = `${lunghezzaTratto} ${spazio}`;
  gsap.to(el, {
    strokeDashoffset: -(lunghezzaTratto + spazio), // negativo = scorre nel verso di disegno del path
    duration: velocita,
    ease: 'none',   // no easing: il gas non "rallenta" periodicamente
    repeat: -1
  });
}

animaFlusso('#tubo-mandata', 16, 12, 1.1);       // alta pressione: tratto più corto, più veloce
animaFlusso('#tubo-aspirazione', 20, 14, 1.3);   // bassa pressione: tratto più lungo, leggermente più lento
```

Punti critici:
- Il **verso del path SVG** (come è stato disegnato in Illustrator/Inkscape, dal primo al ultimo nodo) determina la direzione del flusso: `strokeDashoffset` negativo scorre "in avanti" lungo quel verso. Se il gas nel disegno originale scorre nella direzione opposta a come appare in anteprima, **non cambiare segno nel JS**: ridisegna il path nel verso corretto in fase di preparazione SVG (punto 1), altrimenti dovrai ricordarti manualmente quali tubi sono invertiti.
- Usa le **frecce già presenti nel disegno tecnico originale** come riferimento di verità per la direzione — sono già corrette dal progettista dell'impianto.
- Differenzia leggermente velocità e passo del tratteggio tra linea di mandata (alta pressione, più "urgente") e linea di aspirazione (bassa pressione): rende l'animazione più leggibile e meno meccanica.
- **Non animare** le linee ciano di segnale pressione/temperatura: sono statiche nella realtà, animarle comunicherebbe un'informazione falsa.

## 5. Accessibilità: `prefers-reduced-motion`

Obbligatorio, non opzionale — un impianto con due eliche che girano all'infinito e tubi che scorrono è esattamente il tipo di movimento continuo che infastidisce chi ha attivato la riduzione del movimento:

```js
const mm = gsap.matchMedia();
mm.add(
  { reduced: '(prefers-reduced-motion: reduce)', full: '(prefers-reduced-motion: no-preference)' },
  (ctx) => {
    if (ctx.conditions.full) {
      // qui dentro: rotazione eliche + animazione flusso (sezioni 3 e 4)
    } else {
      // versione ridotta: tratteggio statico visibile, nessuna rotazione/movimento
      document.querySelectorAll('.pipe').forEach(el => el.style.strokeDasharray = '16 12');
    }
  }
);
```

Con `prefers-reduced-motion: reduce`, l'utente vede comunque lo schema tecnico corretto (tubi tratteggiati, colori distinti) ma senza nulla che si muove.

## 6. Performance

- Anima solo `transform` (rotazione) e `stroke-dashoffset`: **non toccare mai `width`/`height`/posizione degli elementi del disegno tecnico**, altrimenti il layout dell'SVG si ridisegna ad ogni frame (costoso, e a rischio di deformare lo schema).
- Un solo file SVG inline (non `<img src="...svg">`): serve poter targettizzare gruppi/path per id da GSAP, cosa impossibile se l'SVG è caricato come immagine esterna.
- Su mobile, se l'impianto ha molte tubazioni animate, valuta di **ridurre il numero di tubi animati** a quelli principali (mandata + aspirazione) e lasciare statiche le diramazioni secondarie: il beneficio visivo aggiuntivo è marginale rispetto al costo.

## 7. Mettere la demo sul sito web

Ci sono due file pronti in `assets/animazioni/`:

- **`demo-impianto-frigorifero.html`** — pagina HTML completa e autonoma (doctype, head, body). Usala se vuoi ospitarla come pagina a sé stante o incorporarla via `<iframe>`.
- **`demo-impianto-frigorifero-embed.html`** — lo stesso contenuto ma **senza** `<!doctype>`/`<html>`/`<head>`/`<body>`: solo `<link>` font + `<style>` + markup + `<script>`. È il formato giusto da incollare **dentro** una pagina esistente (widget HTML di Elementor, un blocco "Custom HTML", ecc.), perché quei tag non sono ammessi dentro un widget.

**Opzione A — pagina/iframe** (più semplice, isolamento totale dagli stili del sito):
1. Carica `demo-impianto-frigorifero.html` sul tuo hosting (o come pagina statica).
2. Incorporalo dove serve con:
   ```html
   <iframe src="/percorso/demo-impianto-frigorifero.html" style="width:100%; border:0; aspect-ratio:1040/760;" loading="lazy"></iframe>
   ```
   L'`aspect-ratio` evita salti di layout mentre l'iframe carica.

**Opzione B — Elementor Pro (widget HTML), incorporato nella pagina**:
1. Trascina un widget **HTML** nel punto della pagina dove deve comparire lo schema.
2. Apri `demo-impianto-frigorifero-embed.html`, copia **tutto** il contenuto e incollalo nel widget.
3. Se GSAP è già caricato globalmente da Custom Code del tema, rimuovi il tag `<script src=".../gsap.min.js">` duplicato dal frammento incollato (altrimenti nessun danno, viene solo caricato due volte).
4. Aggiungi in cima allo script `if (window.elementorFrontend && elementorFrontend.isEditMode()) return;` per non far partire l'animazione dentro l'editor di Elementor.
5. Il markup usa `width:100%; height:auto` sull'SVG e non ha overflow nascosti: si adatta alla larghezza del contenitore automaticamente, niente da configurare oltre alla larghezza della sezione/colonna che lo ospita.

In entrambi i casi la pagina è già autosufficiente (font da Google Fonts, GSAP da cdnjs, immagine di sfondo incorporata in base64): non servono altri asset da caricare.

## 8. Checklist di collaudo

- [ ] Le eliche ruotano in loop fluido, nessuno scatto al passaggio da un giro al successivo (verifica `ease: linear`)
- [ ] Le pale ruotano centrate sul mozzo, non "a manovella" fuori asse
- [ ] Il tratteggio nei tubi rossi e arancioni scorre nella direzione indicata dalle frecce del disegno originale
- [ ] Le linee ciano di segnale restano ferme
- [ ] Con "Riduci movimento" attivo nel sistema operativo, eliche e flusso si fermano ma lo schema resta leggibile
- [ ] Nessun elemento anima `width`/`height`/posizione (solo `transform` e `stroke-dashoffset`)
- [ ] Il file SVG resta leggibile e proporzionato da mobile a desktop (`viewBox` + `width:100%`)
- [ ] In Elementor, l'animazione non parte in modalità editor

## Demo di riferimento

`assets/animazioni/demo-impianto-frigorifero.html` usa **esattamente il disegno originale** come sfondo (renderizzato dal PDF, non ridisegnato) con overlay animato posizionato sulle coordinate reali estratte dal PDF (punto 1, Strada B). Il codice dei punti 2–5 è applicato e commentato all'interno del file. Tre correzioni fatte dopo il primo giro di feedback, utili come promemoria per casi simili:

- **Eliche**: la maschera bianca copre solo l'area *interna* all'anello (raggio = raggio dell'anello interno, non oltre), non un cerchio più grande. Coprire troppo cancellava anche l'anello/guscio esterno originale (obbligando a ridisegnarlo, con inevitabili differenze) e troncava le serpentine che passano proprio a ridosso della ventola. Mascherando solo l'area delle pale, gli anelli restano quelli originali al 100% e le serpentine restano intatte fin sotto la ventola.
- **Serpentine (coil) di evaporatore e condensatore**: inizialmente erano lasciate statiche (solo i tratti di collegamento esterni erano animati), col risultato di un flusso che si "blocca" a metà percorso. Sono state estratte anch'esse dal PDF (stessa tecnica di scansione colore, applicata alle singole righe/colonne della serpentina) e animate con lo stesso tratteggio in movimento, così il flusso è continuo dall'inizio alla fine del circuito, senza tratti congelati. Attenzione allo spessore dell'overlay: se più sottile dell'originale lascia un bordo "seghettato" visibile (l'originale spunta ai lati), se troppo spesso le righe ravvicinate della serpentina si fondono in un blocco unico perdendo l'effetto "a serpentina" — va tarato per coprire esattamente lo spessore originale.
- **Cartiglio**: rimosso interamente (nome progettista, data, scala, formato, logo aziendale e titolo dello schema) con un rettangolo bianco disegnato sull'immagine di sfondo, dimensionato per coprire l'intero blocco senza toccare la cornice del disegno tecnico che lo circonda.
- **Ordine di sovrapposizione (z-index)**: le eliche vanno disegnate **dopo** (quindi sopra, in primo piano) i tratti di tubazione/serpentina — non prima. Con l'ordine sbagliato il tratteggio animato passa sopra le pale creando un effetto confuso di linee che si incrociano. La maschera bianca della ventola deve inoltre coprire l'intero guscio esterno (non solo l'area delle pale): così la serpentina si "infila" pulita dietro un disco bianco pieno, esattamente come nel disegno originale, e anello + pale ridisegnati sopra restano nitidi senza interferenze dal tratteggio sottostante.
- **Cornice del foglio tecnico**: il rettangolo che racchiude tutto il disegno (bordo del foglio A3) è stato rimosso dallo sfondo — per uso su una pagina web non serve, e dopo aver ripulito il cartiglio nell'angolo restava comunque un pezzo di bordo "monco" a metà, visivamente rotto. Rimosso su tutti e 4 i lati (stessa tecnica: rettangoli bianchi sull'immagine di sfondo, coordinate lette dal PDF), il disegno ora è a bordo pieno, senza cornice.
