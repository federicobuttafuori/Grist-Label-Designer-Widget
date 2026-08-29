# Debug learnings: contorno in stampa (bianco → nero) e regressione outline

## Problema originale

- **Sintomo**: contorno impostato **bianco** (o chiaro) su testo/immagini; **in stampa** risultava **nero** o comunque errato, mentre a schermo poteva sembrare accettabile.
- **Contesto**: app etichette in `index.html`, contorno gestito da proprietà elemento (`textOutlineEnabled`, `textOutlineWidth`, `textOutlineColor`, contorno secondario opzionale).

## Cosa causava il bug (causa vera)

1. **Pipeline di stampa del browser / driver**: `text-shadow` e `filter: drop-shadow(...)` su contenuti stampati vengono spesso **rasterizzati in modo diverso** rispetto allo schermo. I colori **molto chiari / bianchi** in quelle ombre possono essere **interpretati male** (es. finire come inchiostro nero o “alone” sporco), soprattutto su output **monocromatico** (termica / PDF B/N).
2. **Non era un semplice “colore sbagliato nel CSS”**: cambiare leggermente l’hex del bianco non risolveva in modo affidabile perché il problema era nel **modello di compositing in stampa**, non nel valore `#ffffff` in sé.

## Tentativi che **non** hanno risolto (o hanno peggiorato)

| Approccio | Perché non è bastato / effetto collaterale |
|-----------|---------------------------------------------|
| Sostituire `#fff` / `#ffffff` con un bianco “quasi bianco” (es. `#fefefe`) **solo in stampa** nelle stringhe di shadow/drop-shadow | Non affronta la rasterizzazione; spesso **nessun miglioramento** visibile in stampa. |
| In stampa, **omettere** i contorni “quasi bianchi” (non applicare shadow se colore chiaro) | Evita il nero “fantasma” ma **cancella** il contorno bianco desiderato → non accettabile se l’utente vuole quell’effetto. |
| Passare a un **unico** filtro SVG fisso (`id="outline"`) + classe `.outlined` con `radius` e colore **fissi** | Stampa più stabile ma **ignora** spessore e colore dalle proprietà → regressioni (contorno sempre uguale); su icone può apparire un **alone nero grosso / a gradini** rispetto all’intento. |

## Fix finale (soluzione attuale)

1. **Host SVG** in pagina: `<defs id="outline-filters-defs">` (inizio `body`).
2. Per ogni combinazione **spessore + colore** (e doppio contorno se attivo), **`ensureOutlineSvgFilter(el)`** crea un `<filter>` con:
   - `feMorphology` (dilate) su `SourceAlpha` → contorno geometrico pulito attorno alla forma (testo o alpha immagine),
   - `feFlood` + `feComposite` per il **colore scelto**,
   - con **due anelli** se contorno secondario abilitato.
3. Applicazione **`filter: url(#id)` inline** su contenitore testo + span interni (regex) e su `<img>`, così **rispettano** `textOutlineWidth` / `textOutlineColor` (e secondario).

In sintesi: **non** affidarsi a shadow/drop-shadow per il contorno in stampa; usare **SVG filter** parametrico legato alle props.

## Cosa imparare per agenti futuri

- **Stampa ≠ schermo**: tecniche che “funzionano sempre” in CSS screen (`text-shadow`, catene di `drop-shadow`) possono **fallire in print** o con driver termici; verificare sempre su **Stampa → Anteprima / PDF**.
- **Workaround colore** (#fefefe, ecc.) su effetti non solidi spesso **maschera** il problema invece di risolverlo.
- **Filtro SVG “unico”** con parametri fissi è facile da introdurre ma **rompe** il binding con le proprietà UI → ogni fix deve **preservare** width/color (o documentare esplicitamente la semplificazione).
- **Pulizia**: dopo la soluzione definitiva, **rimuovere** helper e path morti (vecchie funzioni shadow/drop-shadow non più chiamate) per evitare confusione.

## File coinvolti

- Implementazione: [`index.html`](../index.html) (`outline-filters-defs`, `ensureOutlineSvgFilter`, `applySvgOutlineToTextContainer`, `applySvgOutlineToImage`).

---

# Debug learnings: impossibile deselezionare in multiselect / gruppo

## Problema originale

- **Sintomo**: con più elementi selezionati (multiselect o gruppo) non si riusciva più a **togliere la selezione**. Click sul vuoto, sul riquadro di gruppo o sull’area intorno all’etichetta non azzeravano `selectedIds`.
- **Contesto**: `index.html`, mousedown su `#label-canvas` (`lc`) e `#canvas-container` (`cc`). Con un gruppo selezionato esiste un overlay `.group-bounds-frame` (AABB + handle).

## Cosa causava il bug (causa vera)

Il lasso / clear partiva **solo** se `event.target === lc` oppure `event.target === cc`.

Qualsiasi altro target — `#canvas-wrapper`, `.group-bounds-frame`, overlay, handle saltato dal hit-test — faceva `return` **senza** avviare il marquee e **senza** `setSelection([])`. In multiselect/gruppo gran parte dei click “sul vuoto” non colpisce `lc`/`cc` nudi, quindi la selezione restava.

La deselezione era demandata al **mouseup del marquee a dimensione zero**. Se il marquee non partiva, non c’era nessun altro percorso che azzerasse la selezione.

## Tentativi che **non** hanno risolto

| Approccio | Perché non è bastato |
|-----------|----------------------|
| Ipotesi: Ctrl+click su un già selezionato non fa più toggle (per non rompere il Ctrl-drag) | Non era il gesto che l’utente usava per uscire dal multiselect; il click era sul **vuoto**. |
| `pendingSelectClick` + toggle-off / collapse su mouseup se il puntatore non si è mosso (`dragMoved`, soglia 4px) | Click su elemento già selezionato ≠ click sul background; il gruppo in più **escludeva** il collapse (`getGroupForElement`). |
| Esc per `setSelection([])` | Workaround di tastiera; non ripristina il click sul canvas. Inoltre Esc è ignorato se il focus è su input (es. Scale %). |

## Fix finale (soluzione attuale)

1. **`isCanvasObjectTarget`**: è un oggetto del canvas solo se il target è dentro `.label-element` o `.resize-handle`.
2. **`beginBackgroundPointer`**: se non è Ctrl/Cmd, **`setSelection([])` subito** al mousedown; poi avvia il marquee (così un drag successivo può riselezionare).
3. **`cc` mousedown**: se il target **non** è un oggetto, chiama `beginBackgroundPointer` (wrapper, padding grigio, overlay).
4. **`lc` mousedown** con `!id`: stessa funzione, **senza** il filtro `t === lc \|\| t === cc`.

## Cosa imparare per agenti futuri

- Se l’utente “non riesce a deselezionare”, **non** partire da Ctrl+click o da toggle sugli elementi: verificare **cosa è `event.target` sul click a vuoto** (wrapper, overlay di gruppo, handle). Un guard `target === canvas` è troppo stretto.
- Overlay con `pointer-events: none` **non** garantisce `target === canvas`: altri nodi (wrapper, frame se diventa target) restano fuori dal guard.
- Deselezionare **al mousedown** sul background è più affidabile che aspettare un marquee 0×0 al mouseup.
- Non accumulare workaround (Esc, click-vs-drag sugli item) prima di aver confermato il gesto reale; dopo il fix, **rimuoverli**.

## File coinvolti

- [`index.html`](../index.html): `beginBackgroundPointer`, `isCanvasObjectTarget`, listener mousedown di `cc` / `lc`.

---

# Debug learnings: resize al bordo canvas, secondo drag esce

## Problema originale

- **Sintomo**: espandi un elemento fino al bordo del canvas, rilascia, riprendi l’handle e tira verso l’esterno → **a volte** la dimensione supera il canvas. Non è sistematico: succede dopo un primo resize arrivato al confine.
- **Contesto**: `index.html`, `setElementSizeStoppedAtWall` (resize singolo, anche con rotazione). Ctrl/Cmd resta il gesto esplicito per uscire.

## Cosa causava il bug (causa vera)

Due pezzi insieme:

1. **Snap al pixel dopo il clamp**: la size massima ancora *dentro* (con `eps` ~1e-4 cm) veniva passata a `snapToPixel` (arrotondamento al pixel **più vicino**). L’arrotondamento per eccesso poteva spingere l’AABB **un filo oltre** il bordo. Il primo drag sembrava corretto; lo stato salvato era già fuori.
2. **Bypass “già fuori → applica want”**: se al mousemove successivo `startW/startH` non passava `elementAabbInsideCanvas`, il codice applicava la size richiesta **senza clamp**. Un overflow da snap di ~0.5 px bastava a togliere il limite per **tutto** il secondo drag.

Intermittente perché dipende dal verso dell’arrotondamento (nearest pixel in su o in giù).

## Tentativi che **non** hanno risolto

| Approccio | Perché non è bastato |
|-----------|----------------------|
| Solo fermare la crescita verso `want` se `want` è fuori, tenendo `start` | Se `start` è già fuori (snap precedente), il binary search da `start` verso `want` ha t=0 ancora fuori e non recupera. |
| Bypass “se start è fuori, lascia want” (per non imprigionare chi è uscito con Ctrl) | Confonde **fuori voluto (Ctrl)** con **fuori accidentale (snap)**. Al secondo drag senza Ctrl sblocca l’uscita. |

## Fix (parziale — poi sostituito)

1. **Mai** applicare `want` solo perché `start` è fuori.
2. Snap: nearest, poi floor, poi size pre-snap.
3. Ctrl/Cmd continua a bypassare `setElementSizeStoppedAtWall`.

`pullElementSizeInside` (binary search **da minSize 0.2 cm** verso `start` se l’AABB non era dentro) **non** è la soluzione: è la causa del bug successivo (collasso a ~0×0). Vedi la sezione sotto. Il vincolo attuale è **non aumentare l’overflow AABB rispetto a `start`**, non “AABB deve stare tutto dentro”.

## Cosa imparare per agenti futuri

- Un clamp “fino al bordo” **seguito da round-to-nearest** può invalidare il clamp. Dopo lo snap, **ri-verificare** l’overflow; se peggiora, arrotondare **verso il basso**, non applicare la size richiesta.
- Non usare “elemento già fuori → disable clamp” come scorciatoia: lo stato fuori può essere **errore numerico**, non intento utente.
- Bug **intermittenti** al confine: sospettare snap/float/`eps`, non il gesto del secondo drag in sé.
- Verificare il **secondo** drag dopo uno stop al muro, non solo il primo.
- **Non** recuperare un AABB “fuori” rimpicciolendo verso minSize.

## File coinvolti

- [`index.html`](../index.html): `setElementSizeStoppedAtWall`, `applySnappedSizeInside`, `overflowNoWorseThan`, `snapToPixelDown`.

---

# Debug learnings: resize ruotato, angolo originale sul bordo → size ~0×0

## Problema originale

- **Sintomo**: elemento **ruotato**, punto di ancoraggio (angolo in alto a sinistra *locale*, quello che resta fisso nel resize) **sul bordo del canvas**. Al ridimensionamento, larghezza e altezza vanno a ~0.
- **Contesto**: terza volta sul confine canvas. Stesso pezzo: `setElementSizeStoppedAtWall` + pin di `getLocalTopLeftWorld`.

## Cosa causava il bug (causa vera — errore di base)

Due vincoli insieme sono **incompatibili** dopo una rotazione:

1. Il resize **fissa l’angolo originale** nello spazio canvas (non l’angolo AABB).
2. Il clamp chiedeva **AABB interamente dentro** il canvas.

Con rotazione intorno al centro, l’AABB sporge **oltre** quell’angolo. Se l’angolo è già sul muro, **nessuna size positiva** può avere AABB dentro: lo sbalzo cresce con w/h.

Il recovery precedente (`pullElementSizeInside`) faceva binary search **da 0.2 cm**. Ogni candidato era “fuori”, `lo` restava 0, applicava minSize → collasso visivo a 0×0.

Non era un altro bug di snap: era il test “dentro sì/no” applicato a una geometria che non può mai essere “dentro”.

## Tentativi che **non** risolvono

| Approccio | Perché non basta |
|-----------|------------------|
| Allargare `eps` a 1 px sul test “dentro” | Maschera overflow da snap, non il caso ruotato (sbalzo di centimetri). Può far accumulare overflow tra un drag e l’altro. |
| Se `start` è fuori, congelare sempre la size | Evita il collasso ma impedisce anche lo **shrink** (che riduce lo sbalzo) e tratta “fuori per rotazione” come “bloccato”. |
| Riportare dentro cercando da minSize | È il bug: più piccolo ≠ più dentro se l’ancora è sul muro. |
| Lasciare `want` se `start` è fuori | Bug 2: sblocca l’uscita dal canvas. |

## Fix finale (soluzione attuale)

Vincolo: **non aumentare** l’overflow AABB rispetto alla size di partenza (`overflowNoWorseThan` + cap misurato su `start`).

- Crescere fino a un altro muro: l’overflow su quel lato aumenta → binary search `start → want` si ferma.
- Angolo originale sul bordo + rotazione: lo sbalzo su quel lato è già nel cap; una grow che lo **peggiora** resta a `start`; uno shrink che lo **riduce** è accettato.
- Snap: nearest se non peggiora il cap, altrimenti floor, altrimenti unsapped, altrimenti `start`. Mai cercare da minSize.
- Ctrl/Cmd resta libero.

## Cosa imparare per agenti futuri

- Pin angolo originale + “AABB ⊆ canvas” è un insieme **vuoto** per box ruotati con l’angolo sul muro. Il test giusto è **overflow non peggiore di start**.
- Binary search da un `from` che **non** soddisfa il vincolo applica `from` (`lo=0`). Se `from` è minSize, collassi.
- Terzo bug sullo stesso bordo: prima di un altro workaround, chiedere se il **predicato** del clamp è sbagliato, non solo lo snap/`eps`.
