# Scrivania PDF

**Scrivania PDF** è un'applicazione web gratuita per modificare documenti PDF direttamente nel browser: firmare, aggiungere testo, evidenziare, coprire parti del documento, gestire le pagine, unire e dividere PDF. È un unico file HTML che funziona su computer e smartphone, anche senza connessione a Internet.

> **Privacy:** tutto avviene sul dispositivo di chi la usa. I PDF non vengono mai inviati a un server: chi pubblica l'app fornisce solo il file dell'applicazione, non riceve i documenti.

Autore: **Francesco Impallari** – Consiglio Nazionale delle Ricerche (CNR) – francesco.impallari@cnr.it

---

## 1. Come si usa

All'apertura una schermata iniziale propone quattro scelte:

| Scelta | Cosa fa |
|---|---|
| **Apri PDF** | Apre un PDF (o un'immagine JPG/PNG) per modificarlo |
| **Unisci PDF** | Riunisce due o più PDF in un unico documento |
| **Dividi PDF** | Separa un PDF in più file o ne estrae alcune pagine |
| **Contatti** | Indirizzo per segnalazioni e richieste di nuove funzioni |

I file si possono anche **trascinare** nella finestra: se non c'è un documento aperto vengono aperti, altrimenti vengono **aggiunti** al documento in lavorazione nella posizione scelta.

### Strumenti

| Strumento | Tasto | Descrizione |
|---|---|---|
| Seleziona | V | Seleziona, sposta (anche con la «manina»), ridimensiona, ruota ed elimina gli elementi aggiunti |
| Testo | T | Testo con 16 caratteri, dimensione, colore, grassetto, corsivo, sottolineato, barrato, rotazione |
| Firma | F | Firma disegnata, scritta in stile calligrafico o caricata da immagine (sfondo bianco rimosso); le firme si possono memorizzare |
| Penna | P | Tratto a mano libera |
| Evidenzia | E | Evidenziatore semitrasparente |
| Pennello | B | Pennello a colore pieno, spessore 4–60 pt |
| Riquadro | R | Rettangolo con bordo colorato |
| Copri | C | Riquadro pieno che nasconde il contenuto; prende automaticamente il colore dello sfondo |

**Colori:** colori pronti, tavolozza con OK/Annulla, codice esadecimale, **contagocce con lente d'ingrandimento** (preleva il colore esatto dal documento) e accesso alla tavolozza di sistema.

**Spostamento preciso:** guide di allineamento automatiche durante il trascinamento (⌥ per disattivarle), frecce della tastiera per spostare di 1 pt (Maiusc: 10 pt).

**Zoom:** pulsanti Riduci/Ingrandisci, pizzico sul trackpad, ⌘+ / ⌘− / ⌘0.

**Annulla / Ripeti** illimitati (⌘Z, ⇧⌘Z).

### Pagine
Dal pannello **Pagine**: ruotare (singola pagina o tutte), riordinare, eliminare, **aggiungere pagine** da altri PDF o da immagini scegliendo quali pagine e in quale posizione (inizio, dopo la pagina N, fine).

### Salvataggio
- **Salva PDF** – crea `nome-modificato.pdf`; l'originale non viene mai toccato.
- **Salva con nome** – scelta del nome (e della cartella su Chrome/Edge).
- **Salva PDF appiattito** – ogni pagina diventa un'immagine: ciò che è stato coperto viene eliminato davvero (utile per dati personali).
- **Dividi** – a blocchi di N pagine, per intervalli (`1-3, 4-7, 8-`) o estrazione (`2, 5, 8-10`); salvataggio in ZIP, file separati o in una cartella.

---

## 2. Architettura tecnica

### Un solo file
L'applicazione è un unico documento **HTML5** con CSS e JavaScript incorporati (circa 4,8 MB), senza installazione, senza server applicativo e senza dipendenze esterne a runtime. Il codice dell'applicazione è minificato con *terser*.

### Componenti

| Componente | Ruolo |
|---|---|
| **pdf.js 3.11** (Mozilla) | Lettura e rendering delle pagine PDF su `<canvas>`; usato anche per l'appiattimento |
| **pdf-lib 1.17** | Creazione del PDF di uscita: copia delle pagine originali, disegno di testo, immagini e tracciati |
| **@pdf-lib/fontkit 1.1** | Incorporamento dei caratteri (sottoinsieme dei glifi usati) nel PDF |
| **16 caratteri WOFF** | Incorporati in base64; usati sia a schermo (FontFace API) sia nel PDF, così ciò che si vede coincide con il risultato |
| **API del browser** | File API, Canvas 2D, Pointer Events, File System Access (`showSaveFilePicker`, `showDirectoryPicker`), EyeDropper, Clipboard, localStorage (solo per le firme salvate) |

I caratteri «come in Word» sono equivalenti liberi con le stesse metriche: Arimo (Arial), Tinos (Times New Roman), Cousine (Courier New), Carlito (Calibri), Caladea (Cambria), Gelasio (Georgia), EB Garamond (Garamond), Comic Neue (Comic Sans).

### Modello dei dati
- Il documento è un elenco di **pagine**; ogni pagina indica il PDF di origine, l'indice nel file originale, la rotazione e un elenco di **annotazioni**.
- Le annotazioni (testo, immagine/firma, tratto, rettangolo) sono memorizzate in **punti tipografici** (1 pt = 1/72 di pollice) nel sistema di coordinate della pagina visualizzata, indipendente dallo zoom.
- Ogni modifica salva un'istantanea dello stato: questo rende possibili Annulla/Ripeti.

### Visualizzazione
Ogni pagina è composta da due livelli sovrapposti:
1. un **canvas** con la pagina originale disegnata da pdf.js, alla risoluzione dello schermo (anche Retina); le pagine vengono disegnate solo quando entrano nell'area visibile (*IntersectionObserver*);
2. un **livello HTML/SVG** con le annotazioni, modificabili e selezionabili.

### Esportazione del PDF
1. Le pagine originali vengono **copiate** con pdf-lib (testo e grafica originali restano vettoriali e selezionabili).
2. Le coordinate dello schermo sono convertite in coordinate PDF tramite la `viewport` di pdf.js (`convertToPdfPoint`), tenendo conto di rotazione della pagina e *crop box*.
3. Il testo è scritto con il carattere scelto, incorporato; sottolineato e barrato sono tracciati come linee. La rotazione di testi e firme è applicata attorno al loro centro.
4. Tratti e rettangoli sono scritti come tracciati vettoriali, con opacità per l'evidenziatore.
5. Le firme sono incorporate come immagini PNG con trasparenza.
6. PDF protetti che pdf-lib non può copiare vengono rasterizzati automaticamente.

**Appiattimento:** il PDF prodotto viene ridisegnato da pdf.js a circa 160 dpi e ogni pagina viene salvata come immagine JPEG in un nuovo PDF.

### Altre funzioni tecniche
- **Immagini → PDF:** le foto vengono adattate su una pagina A4 (verticale o orizzontale secondo la forma).
- **ZIP:** archivio creato dall'app stessa (formato *store*, CRC-32), senza librerie esterne.
- **Copri automatico:** il colore viene calcolato campionando i pixel lungo il bordo esterno del riquadro e prendendo il colore più frequente.
- **Contagocce:** lente 11×11 pixel ingrandita, con lettura del pixel centrale dal canvas della pagina; su Chrome/Edge si usa l'EyeDropper nativo del browser.
- **Inserimento in un'altra pagina:** se l'app è aperta dentro un *iframe* dello stesso sito, si ridimensiona da sola e mostra un pulsante «Tutto schermo».

---

## 3. Compatibilità
- Browser: Safari, Chrome, Edge, Firefox (versioni recenti); computer, tablet e smartphone.
- «Salva con nome» con scelta della cartella e contagocce su tutto lo schermo: Chrome ed Edge; su Safari il file va in *Download* (o dove indicato nelle impostazioni di Safari).
- Per pubblicarla online serve un indirizzo **https**.

## 4. Limiti noti
- Il testo originale del PDF non si modifica: si può coprire e riscrivere.
- La firma è un'immagine, non una firma digitale qualificata.
- I PDF protetti da password di apertura non possono essere aperti.
- I caratteri coprono l'alfabeto latino dell'Europa occidentale (italiano, €).

## 5. Contenuto del repository
| File | Contenuto |
|---|---|
| `index.html` | L'applicazione completa |
| `icon.png` | Icona (1024×1024) |
| `NOTICE.md` | Componenti di terze parti e licenze |

## 6. Licenza
© 2026 Francesco Impallari. Tutti i diritti riservati.
I componenti di terze parti inclusi restano soggetti alle rispettive licenze (Apache 2.0, MIT, SIL Open Font License 1.1): vedi [NOTICE.md](NOTICE.md).

Applicazione sviluppata con il supporto dell'assistente di intelligenza artificiale Claude (Anthropic).
