# PDF2MD — applicazione desktop per convertire PDF in Markdown

Data: 2026-11-20 · Stato: bozza da approvare

## 1. Intento e perché

I docenti dell'istituto hanno materiale didattico in PDF che vogliono dare in pasto a un'IA (ChatGPT, Claude, Gemini…).
Molti colleghi hanno solo il piano gratuito: caricare un PDF consuma molta quota. Fornire invece il **testo già
estratto in Markdown** fa risparmiare token. PDF2MD è un'app desktop, con interfaccia grafica in italiano, che
converte i PDF in `.md` usando la libreria **markitdown** (versione 0.1.9 allegata, codice Microsoft, licenza MIT)
e permette di aggiornare markitdown quando escono nuove release.

Utenti: il docente proponente e i colleghi, in gran parte non tecnici. Sistemi: **Windows e macOS**.

## 2. Obiettivi, non-obiettivi, anti-obiettivi

**Obiettivi**
- Convertire uno o più PDF "digitali" (con testo selezionabile) in `.md`, salvati in una cartella locale.
- Copiare il testo convertito negli appunti con un clic, pronto da incollare nella chat dell'IA.
- Inserire segnaposto di pagina (`--- pagina N ---`) per poter citare le pagine all'IA.
- Ridurre i token: togliere intestazioni/piè di pagina ripetuti e righe vuote in eccesso.
- Mostrare una stima dei token per ogni file.
- Riconoscere i PDF (o le pagine) scansionati e avvisare, invece di produrre un file vuoto.
- Aggiornare markitdown dall'interno dell'app, con ritorno automatico alla versione precedente se qualcosa non va.
- Nessuna installazione di Python per i colleghi: si scarica l'app e la si apre.

**Non-obiettivi (per ora)**
- OCR / riconoscimento di scansioni e scrittura a mano: **congelato** finché la scuola non decide sull'uso delle API
  a pagamento. Il codice deve però lasciare un punto di estensione chiaro per aggiungerlo (vedi §6.3).
- Estrazione di immagini, formule, ricostruzione di titoli/grassetti (markitdown non li fornisce; per l'IA non servono).
- Conversione di formati diversi dal PDF.
- Firma digitale/notarizzazione dell'app (costo annuo; gli avvisi del sistema si superano una volta).

**Anti-obiettivi** (fallimento anche se "funziona")
- Un aggiornamento di markitdown che lascia l'app inutilizzabile.
- Un `.md` vuoto prodotto in silenzio da un PDF scansionato.
- Un'app che un collega non tecnico non riesce ad avviare o capire.
- Invio di dati a servizi esterni (in questa versione l'unica connessione di rete è il controllo aggiornamenti, su richiesta).

## 3. Vincoli

- Partire dal codice markitdown **0.1.9 allegato** (`markitdown-0.1.9.zip`, pacchetto `packages/markitdown`).
- Dipendenze per il PDF: extra `[pdf]` = `pdfminer.six>=20251230`, `pdfplumber>=0.11.9`.
  `pdfplumber` fissa una versione **esatta** di `pdfminer.six`; `pdfplumber` dipende da `Pillow` e `pypdfium2` (binari);
  `pdfminer.six` da `cryptography` (binario); markitdown da `magika` → `onnxruntime` (binario, pesante).
- Le librerie binarie non possono essere aggiornate a caldo: restano quelle incluse nell'eseguibile.
- markitdown restituisce un unico testo per tutto il documento, **senza confini di pagina** (le pagine vuote vengono saltate).
- Interfaccia in italiano.
- Repository: `mermau/pdf_2_md` su GitHub (vuoto al momento).

## 4. Approcci considerati

| | Descrizione | Pro | Contro |
|---|---|---|---|
| 1 | Python installato + script `Avvia.bat`/`Avvia.command` che crea un venv | Semplice, aggiornamenti via pip robusti | Ogni collega deve installare Python; script bloccati su Mac |
| **2 ✔** | **App autonoma (PyInstaller) con markitdown in una cartella utente sostituibile** | Zero installazioni; aggiornamento di markitdown dall'app | Più lavoro; avvisi "sviluppatore non identificato"; un markitdown futuro che richiede librerie binarie più nuove richiede una nuova versione dell'app |
| 3 | Lanciatore basato su `uv` che scarica Python e dipendenze | Nessun Python; aggiornamenti automatici | Download via script al primo avvio, facilmente bloccato su PC scolastici |

**Scelto: approccio 2** (scelta dell'utente, su mia raccomandazione), perché per colleghi non tecnici conta
"scarico, apro, funziona".

Toolkit grafico: **PySide6 (Qt)** — mia scelta: aspetto nativo su macOS, drag & drop integrato, buon supporto PyInstaller.
tkinter scartato (aspetto datato su Mac, drag & drop richiede librerie esterne).

## 5. Esperienza utente (decisa con l'utente)

```
┌─ PDF → Markdown ───────────────────────────────┐
│  [ Trascina qui i PDF o clicca per sceglierli ] │
│                                                 │
│  ✔ dispensa_cap3.pdf     12 pag.  ~4.800 token  │
│  ⚠ verifica_scan.pdf     sembra una scansione:  │
│                          nessun testo trovato   │
│                                                 │
│  Salva in: [stessa cartella del PDF ▾]          │
│  [ Converti ]   [ Copia testo ]   [ Apri .md ]  │
│                                                 │
│  markitdown 0.1.9 · [Controlla aggiornamenti]   │
└─────────────────────────────────────────────────┘
```

- Selezione multipla di PDF (trascinamento o finestra di scelta). Un `.md` per ogni PDF.
- Destinazione: "stessa cartella del PDF" (predefinito) oppure una cartella scelta dall'utente. Il file `.md`
  viene **sempre salvato** su disco (richiesta esplicita).
- "Copia testo" copia negli appunti il Markdown del file selezionato nell'elenco.
- "Apri .md" apre il file con l'applicazione predefinita del sistema; aggiungere anche "Apri cartella" (mia scelta).
- Segnaposto di pagina e pulizia intestazioni attivi di default, disattivabili da Impostazioni (preferenze
  salvate con `QSettings`).
- Conversione in un thread separato con barra di avanzamento; l'interfaccia non si blocca.
- Stato per file: in attesa / in corso / ✔ fatto (pagine, token stimati) / ⚠ scansione / ✖ errore (messaggio comprensibile).

## 6. Architettura

### 6.1 Moduli (proposta; nomi indicativi)

```
src/pdf2md/
  __main__.py          avvio; prepara sys.path per markitdown esterno (vedi 6.4) e lancia la GUI
  gui/                 finestra principale, dialogo impostazioni, dialogo aggiornamenti (PySide6)
  core/
    convert.py         conversione pagina per pagina via markitdown
    cleanup.py         rimozione intestazioni/piè di pagina ripetuti, compressione righe vuote
    tokens.py          stima token
    scan_detect.py     riconoscimento pagine senza testo
  updater/
    runtime.py         gestione cartella markitdown esterna, versione attiva, rollback
    pypi.py            interrogazione PyPI, download wheel, verifica dipendenze
  paths.py             cartelle dati utente per piattaforma
tests/
vendor/markitdown-0.1.9/   sorgente del pacchetto markitdown dallo zip (con LICENSE)
```

La logica (core, updater) non importa nulla di Qt, così è testabile con pytest senza interfaccia.

### 6.2 Conversione

1. Aprire il PDF con `pypdfium2` (già presente come dipendenza di pdfplumber) e ricavare il numero di pagine.
2. Per ogni pagina, creare in memoria un PDF di una sola pagina e passarlo a
   `MarkItDown().convert_stream(stream, stream_info=StreamInfo(extension=".pdf"))` — API pubblica di markitdown.
   Così si ottengono i confini di pagina e il rilevamento delle scansioni per pagina.
   (Conseguenza accettata: la scelta interna di markitdown tra pdfplumber e pdfminer avviene per pagina, non per documento.)
3. Pagina con testo estratto vuoto o quasi (soglia indicativa: < 20 caratteri non spaziali) → contata come "senza testo".
   Se **tutte** le pagine sono senza testo → stato ⚠ "sembra una scansione", nessun `.md` scritto (o `.md` con solo una
   nota: scelta del builder, preferenza per non scriverlo). Se solo alcune → `.md` scritto, con nel punto della pagina
   `--- pagina N --- [pagina senza testo: probabilmente un'immagine]` e avviso nell'elenco.
4. Pulizia (se attiva): righe che compaiono, dopo normalizzazione dei numeri, tra le prime/ultime 3 righe di almeno
   il 50% delle pagine (min. 3 pagine) vengono rimosse; più di una riga vuota consecutiva → una sola.
5. Composizione: `--- pagina N ---` prima del contenuto di ogni pagina (se attivo), poi scrittura in UTF-8 con
   nome `<nome pdf>.md`. Se il file esiste già: sovrascrivere (mia scelta, il `.md` è un derivato).
6. Stima token: `ceil(caratteri / 4)`, mostrata come "~N token" (approssimata, nessun tokenizer reale).

Riferimento: il convertitore PDF di markitdown 0.1.9 è in
`packages/markitdown/src/markitdown/converters/_pdf_converter.py`.

### 6.3 Punto di estensione per OCR futuro

Le pagine "senza testo" passano da un'interfaccia `PageTextFallback` (nessuna implementazione in questa versione).
In futuro un'implementazione basata su API (es. Claude) potrà essere attivata da Impostazioni con una chiave API.

### 6.4 markitdown sostituibile e aggiornamenti

- **Cosa è aggiornabile:** il pacchetto `markitdown` e le sue dipendenze **in puro Python**, compresi `pdfminer.six` e
  `pdfplumber` (puro Python; *mia scelta*: vivono anch'essi nella cartella runtime e si aggiornano **in coppia**, perché
  pdfplumber fissa la versione esatta di pdfminer.six). Le dipendenze binarie (`onnxruntime`, `Pillow`, `pypdfium2`,
  `cryptography`, …) restano quelle incluse nell'app.
- **Dove sta markitdown:** non dentro l'archivio PyInstaller (il loader congelato avrebbe precedenza), ma in
  `<cartella dati utente>/PDF2MD/runtime/<versione>/`. Cartella dati: `%LOCALAPPDATA%\PDF2MD` su Windows,
  `~/Library/Application Support/PDF2MD` su macOS.
  L'app include la wheel di markitdown 0.1.9 costruita dal sorgente in `vendor/` e le wheel di pdfminer.six/pdfplumber
  compatibili; al primo avvio le estrae lì.
  Un file `active.json` indica la versione attiva e quella precedente.
  All'avvio `__main__` inserisce la cartella della versione attiva in testa a `sys.path` prima di importare markitdown.
  Se l'import fallisce → ripristino automatico della versione precedente (o della 0.1.9 inclusa) e avviso.
- **Controllo aggiornamenti** (solo su richiesta, pulsante): legge `https://pypi.org/pypi/markitdown/json`;
  considera solo versioni **< 0.2** (un salto di versione minore può cambiare l'API); mostra versione disponibile.
- **Installazione aggiornamento:**
  1. scarica la wheel `py3-none-any` di markitdown (verifica sha256 fornito da PyPI);
  2. legge `Requires-Dist` (dipendenze base + extra `pdf`) e confronta con le versioni incluse nell'app (`packaging`);
  3. dipendenze in puro Python mancanti o troppo vecchie → scaricate anch'esse come wheel `py3-none-any`;
     la risoluzione è ricorsiva e tratta pdfplumber+pdfminer.six come una coppia;
     dipendenze binarie non soddisfatte → **aggiornamento rifiutato** con messaggio "serve una nuova versione di PDF2MD";
  4. estrae in una nuova cartella `runtime/<versione>/`;
  5. **prova di funzionamento**: in un processo separato (l'eseguibile stesso con un'opzione `--selftest <cartella>`),
     converte un PDF di prova incluso nell'app e verifica che il testo atteso sia presente;
  6. successo → `active.json` aggiornato, richiesto il riavvio dell'app; fallimento → cartella eliminata, versione invariata.
- Pulsante "Torna alla versione precedente" nel dialogo aggiornamenti.

## 7. Distribuzione

- Build con **PyInstaller** (modalità cartella; su macOS bundle `.app`), Python 3.12.
- **GitHub Actions** sul repository `mermau/pdf_2_md`: job su `windows-latest` e `macos-latest` che eseguono i test,
  costruiscono l'app e la allegano come artefatto/Release (`PDF2MD-windows.zip`, `PDF2MD-macos.zip` o `.dmg`).
- App non firmata: nel README istruzioni in italiano, con figure, per superare SmartScreen ("Ulteriori informazioni →
  Esegui comunque") e Gatekeeper (tasto destro → Apri).
- README per l'utente finale in italiano: scaricare, aprire, usare, aggiornare.

## 8. Test

- pytest sulla logica: conversione di PDF di prova generati nei test (testo, tabella, pagina vuota/immagine),
  pulizia intestazioni, segnaposto, stima token, rilevamento scansioni, verifica dipendenze dell'updater
  (casi: compatibile, puro Python mancante, binario incompatibile), rollback.
- Test di avvio della GUI in modalità offscreen (`QT_QPA_PLATFORM=offscreen`) per verificare che la finestra si costruisca.
- In CI: `--selftest` eseguito sull'app costruita, su entrambi i sistemi.

## 9. Lasciato al builder

- Dettagli grafici (icone, colori, layout esatto), purché semplice e in italiano.
- Nomi e struttura interna dei moduli.
- Soglie esatte del rilevamento scansioni e della pulizia intestazioni (valori indicativi sopra).
- Formato del pacchetto macOS (`.zip` o `.dmg`).
