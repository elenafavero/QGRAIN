# Prototipo di verifica dossier TV-04

App Streamlit per i due passaggi obbligatori della parte 2.


## Avvio

Attiva l'ambiente Python del progetto e avvia Streamlit:

```sh
source .venv/bin/activate
streamlit run app.py
```

Il modello predefinito è `claude-haiku-4-5-20251001`, si può cambiare nella barra laterale dell'interfaccia oppure impostare `DEFAULT_MODEL` in `app.py`.

## Flusso

1. Legge tutti i documenti di `data/prototipo/documenti`, il campione in `data/progetti.csv` e le frasi in `data/prototipo/bozza_TV-04_frasi.csv`.
2. Chiede ad Anthropic di attribuire ogni documento a uno o più progetti campione oppure di segnarlo come **incerto**. Le attribuzioni sono modificabili nell'interfaccia ed esportabili in CSV.
3. Seleziona esclusivamente i documenti attribuiti a TV-04 nel passaggio 1 e per ogni frase recupera fino a cinque fonti candidate, selezionate attraverso uno score di somiglianza lessicale.
4. Invia tutte le frasi e le fonti a una seconda richiesta API, che associa ad ogni frase le specifiche fonti che la supportano o meno. Mostra una delle quattro etichette richieste (supportata, non supportata, ecc...), motivazione, confidenza e citazioni; consente di correggere l'esito.
5. Le associazioni frase -> documenti sono modificabili nell'interfaccia ed esportabili in CSV.


## Parte facoltativa

### Valutazione manuale 

Le seguenti sono le associazioni svolte manualmente:

| documento | progetti |
| :--- | :--- |
| D01 | TV06 |
| D02 | TV04 |
| D03 | TV07 |
| D04 | TV04 |
| D05 | TV02 |
| D06 | TV04, TV01 |
| D07 | TV01 |
| D08 | TV04 |
| D09 | incerto |
| D10 | TV03 |
| D11 | TV04 |
| D12 | TV09 |


| frase | etichetta | documenti | motivazione |
| :--- | :--- | :--- | :--- |
| F1 | suportata | D02, D04, D06 | D02 dichiara le specifiche descritte nella frase, D04 è un primo prototipo dove ciò che non funziona non riguarda quello che viene descritto dalla frase (confermato anche da D06) |
| F2 | supportata | D02 | è scritto uguale |
| F3 | supportata | D04 | da 1.4 kg a 0.9kg si è perso 0.5kg == 0.357 = 36% e la capacità è quasi invariata |
| F4 | in contraddizione | D08 | da 6.9kg a 4.6kg si è perso 2.3kg = 33%. Assunzione fatta: "brazed P2 core" == scambiatore |
| F5 | supportata | D02, D06, D04 | è scritto uguale |
| F6 | parzialmente supportata | D06 | che ci sia la maldistribuzione si, ma non c'è scritto da nessuna parte che è stata "risolta" |
| F7 | non supportata | D06 | "Valutata l'ipotesi di una domanda di brevetto sul collettore: decisione rinviata a giugno." |
| F8 | non supportata | | mai citata |


Il programma restituisce le stesse associazioni per quanto riguarda: documento -> progetti

Per la associazione dei documenti alle singole frasi:
- il programma seleziona sempre 1 o 2 documenti in più rispetto a quelli del processo manuale, tranne nei casi di: F5, F7, F8 (no supportata) 
- nel caso di F4, manualmente è stata etichettata "in contraddizione" utilizzando D08, mentre il programma non ha utilizzato D08 e l'ha classificata come "non supportata". NB: nell'associazione manuale si è assunto che "brazed P2 core" == scambiatore (no sicurezza riguardo questa assunzione)


# Limiti
1. fase 2: quando si selezionano 5 documenti per frase, si fa una ricerca lessicale -> Limite: non c’è stemming (riduzione alla radice) né sinonimi. es: “riduzione” e “ridotto” per sono considerate parole diverse. La decisione semantica vera la prende l'LLM, ma solo sui documenti che la ricerca gli passa. Un documento rilevante può non arrivare mai al modello.
2.  In review_claims il payload è così:

    ```json
        payload = [
            {
                "id": claim["id"],
                "claim": claim["frase"],
                "documents": [
                    {"id": doc["id"], "path": ..., "note": ..., "text": doc["text"]}
                    for doc in evidence[claim["id"]]
                ],
            }
            for claim in claims
        ]
    ```

    Ogni frase porta con sé la propria lista di documenti, con il testo completo. Non esiste una tabella condivisa di documenti a cui le frasi fanno riferimento: il testo viene copiato dentro ogni frase che lo recupera. Se lo stesso documento compare in frasi diverse, il testo del documento viene inviato n volte uguale nella richiesta all'LLM -> spreco di token e di contesto (magari non sufficientemente ampio se documenti lunghi)


3. Il prompt dice al modello di riportare “brevi citazioni fedeli” con l’ID del documento, ma non viene verificato che queste citazioni esistano davvero nel documento
   
NB:
- tokens <= 2 non considerati: check se ha senso
- Retrieval lessicale. Niente sinonimi, stemming o lingue diverse: D08 (inglese) non arriva a F4. Numeri e OCR rovinato (l,4, M0DENA) peggiorano la situazione.
- operazioni aritmetiche lasciate al modello
- controllare che la quote sia davvero nel documento
- JSON non garantito.
- Una sola chiamata per tutte le frasi: se fallisce, perdi tutto.

se n° file = 25.000 e file lunghi:
- non si possono mettere tutti i documenti in 1 solo prompt
- anche mettere solo tutto il documento nel prompt non va bene
- se stesso documento serve per più frasi -> duplicato nel prompt




# Soluzione
# Riassunto completo della soluzione

## 0. Il problema in una frase

Hai un archivio enorme e disordinato (25.000 file, anche scansioni e duplicati) e delle frasi di un dossier da verificare. Per ogni frase devi dire se è supportata, e da quale **passaggio esatto** di quale documento. I documenti non dicono a che progetto appartengono, quindi prima li assegni ai progetti, poi verifichi le frasi solo con i documenti del progetto giusto.

## 1. Visione d'insieme

```
         INGESTIONE (una volta per file)                 VERIFICA (una volta per frase)
┌───────────────────────────────────────────┐     ┌──────────────────────────────────────┐
│ nuovo file → hash → testo → chunk          │     │ nuova frase → (scomposizione)        │
│   → associazione chunk-progetto            │     │   → ricerca nell'indice              │
│   → indicizzazione                         │     │      (solo chunk del progetto)       │
└───────────────────────────────────────────┘     │   → AI giudica → controlli codice    │
                                                    │   → salvataggio evidenze + verdetto  │
                                                    └──────────────────────────────────────┘
```

Due principi guida:
- **Le regole prima, l'AI dopo.** L'AI costa, è lenta e può sbagliare. Si usa solo dove le regole non bastano.
- **L'AI propone, il codice verifica.** Ogni citazione viene controllata da un programma normale.

## 2. Le tabelle

Database: **SQLite** per il prototipo (un solo file, nessun server). In produzione si passerebbe a PostgreSQL, ma la struttura è la stessa.

### 2.1 `progetti`
Il campione dei progetti (è il tuo `progetti.csv`).

| Campo | Significato |
|---|---|
| `codice` (chiave) | TV-01, TV-04… |
| `nome` | "Scambiatore microcanale in alluminio" |
| `pm` | A. Ghidoni |
| `sede` | Modena |
| `cartelle` | cartelle SharePoint del progetto (`/R&D/Scambiatori/Microchannel`, `/Lab/Test_termici_2026`) |
| `epic_jira` | RD-102 |
| `n_file`, `formati`, `lingue`, `ore_sap` | metadati utili come indizi |

**Come si popola:** una volta sola, importando il CSV. Si aggiorna solo se cambia l'elenco progetti.

### 2.2 `documenti`
Una riga per file (e per percorso, vedi sotto).

| Campo | Significato |
|---|---|
| `id` (chiave) | D01, D02… generato dal sistema |
| `percorso` | percorso originale su SharePoint |
| `nome_file` | `Specifica_Microcanale_v2.docx` |
| `autore` | A. Ghidoni |
| `modificato_il` | data di ultima modifica |
| `formato` | pdf, docx, eml… |
| `hash` | impronta del contenuto (es. SHA-256 del file) |
| `stato` | `da_elaborare`, `testo_estratto`, `indicizzato`, `errore` |
| `duplicato_di` | id del documento originale, se è una copia (altrimenti vuoto) |
| `origine_testo` | `nativo` oppure `ocr` (serve per sapere quanto fidarsi) |

**Come si popola:** a ogni nuovo file. L'`hash` è la chiave per riconoscere i doppioni: due file con lo stesso hash hanno lo stesso contenuto (ad esempio `Specifica_HP_ibrida_v3_FINALE.docx` in `HP_Hybrid` e in `HP_Hybrid_OLD`, stesse dimensioni e stessa data). In quel caso non si rielabora: si registra il nuovo percorso con `duplicato_di` valorizzato.

Nota: un file con nome simile ma hash diverso (`FINALE` contro `FINALE_rev2`) **non** è un duplicato, è una **versione**. Si può aggiungere un campo `versione_di` e una regola che dice quale versione prevale (di solito la più recente).

### 2.3 `chunk`
Una riga per pezzo di documento (pagina, paragrafo o sezione).

| Campo | Significato |
|---|---|
| `id` (chiave) | `D04-p1-c2` (documento, pagina, numero del pezzo) |
| `documento_id` | a quale documento appartiene |
| `pagina` | numero di pagina (per i PDF) |
| `posizione` | ordine del pezzo nel documento |
| `testo` | il testo del pezzo |
| `testo_normalizzato` | testo ripulito (es. correzioni tipiche dell'OCR: `l,4` → `1,4`, `M0DENA` → `MODENA`), usato per cercare e per controllare le citazioni |
| `riassunto` | (opzionale) una riga che dice cosa contiene |

**Come si popola:** dopo l'estrazione del testo. Il documento viene spezzato in pezzi di dimensione ragionevole (alcune centinaia di parole), cercando di non tagliare a metà una tabella o un elenco. L'ID è **stabile**: serve alla certificatrice per vedere dov'è il passaggio.

### 2.4 `associazioni` (chunk ↔ progetto)
È la tabella che dice "questo pezzo riguarda questo progetto".

| Campo | Significato |
|---|---|
| `chunk_id` | il pezzo |
| `progetto` | il codice progetto |
| `punteggio` | 0-1: quanto è sicura l'attribuzione |
| `metodo` | `regola_cartella`, `regola_autore`, `regola_codice`, `ai` o `manuale` |
| `motivo` | spiegazione breve ("cartella coincide con TV-04") |
| `confermata` | sì/no: se una persona l'ha verificata |

Chiave: `(chunk_id, progetto)`. Un chunk può avere più righe, cioè stare in più progetti. Un chunk senza righe (o con un marcatore `incerto`) è un chunk non attribuito.

**Come si popola:** vedi la sezione 4.

### 2.5 `frasi`
Le affermazioni del dossier da verificare.

| Campo | Significato |
|---|---|
| `id` (chiave) | F1, F2… (unico, ad esempio `TV-04-F1`) |
| `progetto` | TV-04: il progetto a cui la frase appartiene |
| `testo` | la frase |
| `frase_madre` | se è una parte di una frase composta, l'id della frase originale |
| `origine` | da dove arriva (agente QGRAIN, inserimento manuale) |

**Come si popola:** quando l'agente genera un paragrafo del dossier (o si importa il CSV). Una riga per frase. Il progetto è noto fin dall'inizio, perché il dossier è di un progetto preciso.

### 2.6 `evidenze` (frase ↔ chunk)
È **il risultato** del sistema: collega ogni frase ai pezzi di documento che la supportano o la contraddicono.

| Campo | Significato |
|---|---|
| `frase_id` | la frase |
| `chunk_id` | il pezzo citato |
| `citazione` | il brano esatto, copiato dal chunk |
| `ruolo` | `supporta`, `contraddice`, `parziale` |
| `citazione_verificata` | sì/no: il codice ha controllato che la citazione sia davvero nel chunk |

### 2.7 `verdetti`
Uno per frase (e per esecuzione).

| Campo | Significato |
|---|---|
| `frase_id` | la frase |
| `esito` | supportata / parzialmente supportata / non supportata / in contraddizione |
| `confidenza` | alta / media / bassa |
| `motivazione` | una riga |
| `modello`, `versione_prompt` | per sapere con cosa è stato prodotto |
| `output_originale` | la risposta grezza dell'AI |
| `esito_umano` | la correzione di una persona, se c'è |
| `data` | quando è stato generato |

Tenere `esito` (AI) e `esito_umano` separati permette di misurare quanto sbaglia il sistema.

### 2.8 Le tabelle di ricerca (indici)
Non sono tabelle "di dati", ma strutture costruite per ritrovare i chunk in fretta.
- **`chunk_fts`**: tabella speciale **FTS5** di SQLite. Contiene il testo dei chunk e permette ricerche per parole (`MATCH 'peso scambiatore'`).
- **`chunk_embedding`** (opzionale): per ogni chunk, un vettore numerico che ne rappresenta il significato. Serve a trovare chunk vicini per senso anche senza parole in comune (es. "peso" e "weight").

## 3. Cosa succede quando arriva un nuovo documento

1. **Registrazione.** Si crea la riga in `documenti` con percorso, autore, data, formato, stato `da_elaborare`.
2. **Hash.** Si calcola l'impronta del file.
   - Se l'hash esiste già: si imposta `duplicato_di`, si registra il percorso e ci si ferma. Nessuna elaborazione, nessun costo.
   - Altrimenti si continua.
3. **Estrazione del testo.** Lettore normale per DOCX, XLSX, PPTX, EML. Per i PDF con testo, estrazione diretta. Per i PDF scansionati, **OCR** (`origine_testo = ocr`). Qui non serve l'AI generativa: è un programma di OCR.
4. **Divisione in chunk.** Si crea una riga in `chunk` per ogni pezzo, con ID stabile, e si salva anche il testo normalizzato.
5. **Attribuzione ai progetti** (sezione 4).
6. **Indicizzazione.** Ogni chunk viene scritto in `chunk_fts` e, se usato, si calcola l'embedding.
7. **Stato.** `documenti.stato = indicizzato`.

Tutto avviene **una volta per file**. Se il file viene modificato (hash diverso), si ripete solo per quel file.

## 4. Come si associano i chunk ai progetti

È un **imbuto a tre livelli**: dal più economico e sicuro al più costoso.

### Livello 1: regole sugli indizi lampanti (senza AI)
Si confrontano i metadati del documento con quelli della tabella `progetti`:

| Indizio | Esempio | Risultato |
|---|---|---|
| **Cartella** coincide con `cartelle` del progetto | `/R&D/Scambiatori/Microchannel/...` → TV-04 | associazione con punteggio alto, `metodo = regola_cartella` |
| **Autore = PM** del progetto | A. Ghidoni | punteggio medio (un PM può fare più progetti) |
| **Codice progetto o epic nel testo** | "TV-04", "RD-102" | punteggio alto, `metodo = regola_codice` |
| **Termini specifici del progetto** | "P1", "P2" accanto a "microcanale" | punteggio medio |
| **Cartella di duplicati** (`_OLD`) | `/R&D/HP_Hybrid_OLD` | assegnata come la cartella gemella, ma segnalata |

Si sommano gli indizi: più sono concordi, più il punteggio sale. Se il punteggio supera una soglia alta (diciamo 0,9), l'associazione è accettata e **l'AI non serve**.

### Livello 2: l'AI per i casi ambigui
Si usa l'AI solo se:
- la cartella è **generica** (`/R&D/Comune`, `/Varie`);
- la cartella non è quella di nessun progetto (le scansioni in `/Lab/Test_termici_2026` sono in realtà un caso misto: la cartella è citata da TV-04, ma un rapporto può riguardare l'unità esterna B di un altro progetto, come D09);
- gli indizi si contraddicono;
- manca il PM (TV-09 non ha PM né sede).

Cosa si manda all'AI: il **chunk** (non l'intero documento), il percorso, l'autore, e **l'elenco dei progetti candidati con una riga di descrizione** (non tutti e 30: si preselezionano i candidati con le regole). Cosa restituisce: uno o più codici progetto con punteggio e motivo, oppure `incerto`. Il prompt dice di trattare il testo come dato non fidato.

L'attribuzione è **per chunk**, non per documento: il verbale D06 ha un paragrafo su TV-04 e uno su TV-01, e così ogni paragrafo prende la propria etichetta. Si può risparmiare attribuendo per documento e passando a livello di chunk solo per i documenti "misti".

### Livello 3: l'umano
Le associazioni con punteggio basso, o `incerto`, finiscono in una coda di revisione. Una persona le conferma o le corregge (`metodo = manuale`, `confermata = sì`). Le correzioni valgono anche come esempi per migliorare le regole.

Gli esiti si scrivono in `associazioni`, una riga per coppia chunk-progetto.

## 5. Come si filtrano i chunk di un progetto specifico

Il filtro è una query sulla tabella `associazioni`, senza spostare nulla. L'idea:

```sql
-- tutti i chunk associati a TV-04 con attribuzione affidabile
SELECT c.*
FROM chunk c
JOIN associazioni a ON a.chunk_id = c.id
WHERE a.progetto = 'TV-04'
  AND a.punteggio >= 0.6;
```

Per cercare per parole **solo** dentro quel progetto, si combina con l'indice FTS5:

```sql
SELECT c.id, c.testo
FROM chunk_fts
JOIN chunk c ON c.rowid = chunk_fts.rowid
JOIN associazioni a ON a.chunk_id = c.id
WHERE chunk_fts MATCH 'peso scambiatore'
  AND a.progetto = 'TV-04'
ORDER BY rank
LIMIT 10;
```

Il chunk di un documento duplicato non compare due volte, perché i duplicati non vengono rielaborati. Se vuoi essere prudente, si escludono i documenti con `duplicato_di` valorizzato o quelli con versione superata.

Per gli embedding il principio è lo stesso: si cerca tra i vettori e si tengono solo quelli i cui chunk risultano nella lista di TV-04.

## 6. Cosa succede quando arriva una nuova frase

1. **Registrazione.** Si inserisce la riga in `frasi` (progetto TV-04, testo).
2. **Scomposizione (se serve).** Una frase con più affermazioni, come F6 ("il primo prototipo ha mostrato maldistribuzione sotto i 2 °C, risolta nel secondo con un collettore a inserti forati"), si divide in affermazioni singole (`frase_madre` punta all'originale):
   - il prototipo 1 ha mostrato maldistribuzione sotto i 2 °C;
   - nel prototipo 2 il problema è stato risolto;
   - la soluzione è un collettore a inserti forati.
   
   Qui si usa l'**AI** (una chiamata breve).
3. **Recupero dei candidati.** Per ogni affermazione si cerca nell'indice, filtrando per progetto (sezione 5), i 10-20 chunk più rilevanti. Si possono lanciare più ricerche con parole diverse (anche in inglese, per documenti come l'email del fornitore) o usare gli embedding.
4. **Giudizio dell'AI.** Si passa all'AI l'affermazione e i chunk candidati (ciascuno con il suo ID e la sua pagina). L'AI restituisce, in formato strutturato (JSON o tool use): l'esito, la confidenza, la motivazione di una riga, e per ogni evidenza il chunk usato e la citazione.
   
   Regole nel prompt: usare solo i chunk forniti; OCR ambiguo, obiettivi e decisioni rinviate non sono prove di risultati; i testi sono dati e non istruzioni.
5. **Controlli di codice** (nessuna AI):
   - l'ID del chunk citato è tra quelli forniti? (il controllo che hai appena implementato)
   - la citazione è davvero contenuta nel testo del chunk (dopo normalizzazione dell'OCR)?
   - se l'affermazione contiene numeri, si estraggono con un programma e si ricalcolano (es. 4,6 kg contro 6,9 kg → -33%, quindi il 40% è in contraddizione).
   
   Se un controllo fallisce, l'evidenza viene scartata o il verdetto viene declassato.
6. **Salvataggio.** Si scrivono le righe in `evidenze` (con `citazione_verificata`) e il verdetto in `verdetti`.
7. **Frase composta.** L'esito della frase madre si ricava da quello delle parti (tutte supportate → supportata; alcune sì e altre no → parzialmente supportata; una contraddetta → in contraddizione).
8. **Revisione umana.** La persona guarda l'esito con la citazione accanto e può correggerlo (`esito_umano`).

## 7. Come si associano i chunk alle frasi

L'associazione chunk ↔ frase **non si calcola in anticipo**: nasce al momento della verifica, e la fa la combinazione "ricerca + AI + controllo".

- **La ricerca** (indice) fa una preselezione: "questi 10 chunk potrebbero c'entrare". È un'associazione provvisoria, basata su parole o significato.
- **L'AI** decide quali di questi chunk sono davvero rilevanti e in che modo (supportano, contraddicono, parzialmente). Solo questi vengono scritti in `evidenze`.
- **Il codice** conferma che la citazione sia davvero nel chunk.

Il risultato in `evidenze` è l'associazione definitiva: ogni riga dice "la frase F3 si basa sul chunk D04-p1-c2, con questa citazione, e lo supporta". Il chunk ha un ID stabile con pagina, quindi la certificatrice può aprire il documento e andare al punto preciso.

Un chunk può servire più frasi, e una frase può avere più chunk. Se per una frase non si trova nessuna evidenza, il verdetto è "non supportata" e si registra **cosa è stato cercato** (le query usate), così si distingue "la fonte non esiste" da "non l'ho trovata". Da lì si può generare una **domanda mirata** al project manager.

## 8. Dove si usa l'AI e dove no

| Passaggio | AI? | Perché |
|---|---|---|
| Hash e duplicati | No | Calcolo deterministico |
| Estrazione del testo | No | Lettori di formato |
| OCR delle scansioni | No (programma OCR) | Specializzato |
| Divisione in chunk | No | Regole su pagine e paragrafi |
| Attribuzione: regole | No | Cartelle, autori, codici |
| Attribuzione: casi ambigui | **Sì** | Serve capire il contenuto |
| Riassunto del chunk | Sì (opzionale) | Aiuta la ricerca |
| Embedding | Sì (modello dedicato) | Trasforma il testo in vettore |
| Ricerca per parole | No | Indice FTS5 |
| Scomposizione delle frasi | **Sì** | Serve capire la struttura |
| Giudizio sulla frase | **Sì** | Cuore del lavoro |
| Controllo ID e citazione | No | Verifica deterministica |
| Controllo dei numeri | No | Calcolo |
| Revisione finale | Persona | Responsabilità e audit |

## 9. Come si lega al tuo prototipo

Il prototipo non implementa tutto questo, ed è giusto così:

| Prototipo | Sistema a scala |
|---|---|
| Documenti in file `.txt` | Tabelle `documenti` e `chunk` |
| Passo 1: attribuzione solo con l'AI, per documento | Regole prima, AI sui casi ambigui, per chunk |
| `retrieve` con parole | Indice FTS5 + embedding, filtrato per progetto |
| Tutto in una chiamata | Una chiamata per affermazione, in parallelo |
| Controllo dell'ID citato | + controllo della citazione nel testo + controllo dei numeri |
| Risultati in `session_state` | Tabelle `evidenze` e `verdetti` |
| Nessun duplicato gestito | Hash + `duplicato_di` + versioni |

Al colloquio il messaggio giusto è: il prototipo mostra il **comportamento** (i due passi, le citazioni, l'interfaccia); questa struttura mostra come lo faresti **crescere**, e sai quali sono i limiti di ciò che hai consegnato.

## 10. Passaggi minimi se vuoi costruirne una versione ridotta

Se hai tempo e vuoi mostrare qualcosa di concreto senza stravolgere il prototipo:

1. SQLite con `documenti`, `chunk`, `associazioni` e `evidenze`.
2. Hash in ingestione, per mostrare la gestione dei duplicati (D01 e la sua copia `_OLD`).
3. Regole di cartella e autore **prima** dell'AI nel passo 1.
4. Verifica della `quote` nel testo del chunk.
5. Tutto il resto (FTS5, embedding, agente con tool) lo descrivi nel README come passo successivo.

Se vuoi, ti scrivo lo schema SQL completo di queste tabelle, oppure un esempio di codice per l'ingestione di un file (hash, chunk, regole di attribuzione).

