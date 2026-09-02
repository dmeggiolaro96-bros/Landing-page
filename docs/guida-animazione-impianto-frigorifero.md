# Guida: animare lo schema dell'impianto di refrigerazione (eliche + flusso gas)

Sorgente: `Disegno impianto di refrigerazione per cella frigorifera alimentare -25°C in gas R452A` (Fluicom).
Demo funzionante: `assets/animazioni/demo-impianto-frigorifero.html`

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

## 1. Preparare il file sorgente (da PDF a SVG)

Il PDF è vettoriale ma "piatto": tutte le forme sono sciolte, senza gruppi logici. Va ristrutturato prima di poter essere animato.

1. **Apri il PDF in Illustrator / Inkscape** (Inkscape è gratuito e sufficiente).
2. **Isola le due eliche** in due gruppi separati con id chiari:
   - `id="fan-evaporatore"` → le 5 pale + eventuale mozzo, RAGGRUPPATE insieme (non il cerchio/guscio esterno, quello resta fermo).
   - `id="fan-condensatore"` → stesso principio.
3. **Imposta il centro di rotazione**: seleziona il gruppo pale, verifica che il **bounding box sia centrato sul mozzo** della ventola (il puntino centrale nel disegno). Se le pale non sono simmetriche rispetto al centro, la rotazione "balla" — allinea manualmente i nodi o ricentra l'artboard del gruppo.
4. **Separa le tubazioni per colore/funzione** in path indipendenti (non un unico tracciato con tutto l'impianto):
   - `id="tubo-mandata"` per il rosso Ø28
   - `id="tubo-aspirazione"` per l'arancione Ø54
   - lascia le linee ciano come sono (statiche)
   - Ogni tubo deve essere un **singolo `<path>` continuo** nella direzione del flusso reale (dall'inizio alla fine del percorso), perché la direzione del tratteggio animato dipende dal verso in cui il path è disegnato.
5. **Esporta come SVG** (File → Esporta → SVG ottimizzato in Illustrator, oppure Salva come SVG plain in Inkscape). Pulisci l'SVG risultante con [SVGOMG](https://jakearchibald.github.io/svgomg/) per togliere metadati e ridurre il peso, **facendo attenzione a non far "flattenare" i gruppi** che hai appena creato (disattiva l'opzione "Collapse groups").

Risultato atteso: un unico file SVG con struttura tipo:

```html
<svg viewBox="0 0 1200 900">
  <g id="fan-evaporatore" style="transform-origin: 270px 240px;"> ... 5 pale ... </g>
  <g id="fan-condensatore" style="transform-origin: 950px 235px;"> ... 5 pale ... </g>
  <path id="tubo-mandata" d="M ..." />
  <path id="tubo-aspirazione" d="M ..." />
  <path class="segnale" d="M ..." />
  <!-- resto del disegno: box compressore, serbatoi, valvole, testi -->
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

## 7. Integrazione in Elementor Pro (tema HELLO)

1. Esporta l'SVG pulito (punto 1) e incollalo inline dentro un widget **HTML** di Elementor, nella sezione dove deve comparire lo schema impianto.
2. Nello stesso widget HTML, subito dopo l'SVG, incolla in un `<script>`:
   - il tag CDN GSAP (`https://cdnjs.cloudflare.com/ajax/libs/gsap/3.13.0/gsap.min.js`) — se GSAP è già caricato globalmente da Custom Code, salta questo step per evitare doppio caricamento;
   - il boilerplate anti-flicker (punto 2);
   - le funzioni di animazione (punti 3 e 4) avvolte nel controllo `prefers-reduced-motion` (punto 5).
3. Non far girare l'inizializzazione nell'editor di Elementor: aggiungi in cima allo script `if (window.elementorFrontend && elementorFrontend.isEditMode()) return;` per evitare che l'animazione parta anche in modalità modifica.
4. Verifica che il widget HTML non sia dentro un container con `overflow: hidden` più piccolo del viewBox dell'SVG, altrimenti l'impianto viene tagliato su schermi stretti — imposta `width: 100%; height: auto;` sull'SVG.

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

`assets/animazioni/demo-impianto-frigorifero.html` è una ricostruzione semplificata ma funzionante dello schema (stessa struttura: 2 eliche + tubazioni rosse/arancioni/ciano), con il codice dei punti 2–5 già applicato e commentato. Usala come base da adattare quando l'SVG definitivo (preparato secondo il punto 1) sarà pronto: gli id dei gruppi/path andranno allineati a quelli del file reale.
