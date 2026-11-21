# Spiegazione di `app.py`

## Cosa mostra l'applicazione

`app.py` realizza una piccola applicazione Streamlit per controllare le affermazioni contenute nella bozza del dossier TV-04 confrontandole con un insieme di documenti. Il lavoro è diviso in due passaggi, perché prima occorre stabilire quali documenti appartengono al progetto TV-04 e poi verificare le frasi usando soltanto quelle fonti.

All'apertura, la pagina mostra il titolo **“Verifica dossier TV-04”** e una nota che avverte che i risultati sono una revisione assistita e che le citazioni vanno controllate prima di riutilizzarle. Nella barra laterale si può impostare il modello Anthropic da usare. Il valore iniziale è quello della variabile `ANTHROPIC_MODEL`, se presente; altrimenti viene usato `claude-haiku-4-5-20251001`.

La schermata principale procede così:

1. **Riconoscimento dei documenti.** Il pulsante “Riconosci i progetti” invia al modello i documenti e il campione dei progetti. Per ogni documento il modello propone zero o più codici progetto, una confidenza, una motivazione e un'indicazione di incertezza. L'utente può correggere l'attribuzione con un selettore multiplo per ciascun documento. L'app mostra poi quali documenti risultano assegnati a TV-04 e permette di scaricare le attribuzioni in CSV.
2. **Verifica delle frasi.** Dopo il riconoscimento, si abilita “Verifica le frasi” se almeno un documento è assegnato a TV-04. L'app confronta ogni frase della bozza soltanto con le fonti di TV-04 selezionate. Per ogni frase mostra l'esito, la confidenza, una motivazione breve, le citazioni e gli ID dei documenti recuperati. L'esito può essere modificato manualmente. Anche i risultati si possono scaricare in CSV.

Se i dati necessari non si caricano, l'app mostra un errore e interrompe il rendering. Se manca la chiave API, il pulsante di riconoscimento mostra un messaggio esplicativo senza invocare Anthropic. Durante le chiamate al modello viene mostrato uno spinner.

## Preparazione e costanti

All'inizio vengono importati i moduli standard per CSV, JSON, testo, percorsi e conteggi, oltre alle librerie `anthropic`, `streamlit` e `python-dotenv`.

`ROOT` è la cartella che contiene `app.py`. Il programma chiama `load_dotenv(ROOT / ".env", override=False)` per rendere disponibili nell'ambiente le impostazioni eventualmente definite nel file `.env`, senza sovrascrivere variabili già impostate. Il codice usa la chiave `ANTHROPIC_API_KEY` per autenticare le richieste e può leggere `ANTHROPIC_MODEL` come modello predefinito. Questa spiegazione descrive il comportamento del codice, senza leggere il contenuto di alcun file `.env`.

Le directory e i file dati sono definiti da:

- `DATA_DIR`: `data/prototipo`;
- `DOCS_DIR`: `data/prototipo/documenti`;
- `PROJECTS_FILE`: `data/progetti.csv`;
- `DEFAULT_MODEL`: modello Anthropic di riserva.

Il set `STOPWORDS` elenca parole italiane molto comuni da ignorare nel confronto lessicale. Parole come “di”, “il” o “con” compaiono in moltissimi testi e normalmente aiutano poco a distinguere una fonte.

## Funzioni nell'ordine in cui compaiono

### `tokens(text)`

Normalizza il testo in minuscolo e, con un'espressione regolare, estrae sequenze di lettere (comprese quelle accentate) e cifre. Elimina poi i termini lunghi due caratteri o meno e le parole presenti in `STOPWORDS`.

Il risultato è una lista di parole da usare per un confronto semplice tra una frase e i documenti. Non è un'analisi linguistica: per esempio non riconduce automaticamente plurali, forme coniugate o sinonimi alla stessa radice.

### `load_inputs()`

Carica i dati locali necessari e restituisce una tupla con tre liste: `claims`, `documents` e `projects`.

1. Legge `bozza_TV-04_frasi.csv` e crea una lista di dizionari, uno per frase, con campi come `id` e `frase`.
2. Legge `indice.csv`, che associa gli ID dei documenti ai percorsi e alle note.
3. Legge `data/progetti.csv`, contenente il campione dei progetti e i relativi metadati.
4. Per ogni riga dell'indice determina l'ID del documento. Se `id` è vuoto o manca, ricava l'ID dal nome del file indicato in `percorso`.
5. Cerca il testo corrispondente in `documenti/<ID>.txt`. Se il file esiste, costruisce un dizionario con `id`, `path`, `note` e `text` e lo aggiunge a `documents`. Le voci dell'indice senza il relativo file di testo vengono saltate.

I CSV vengono aperti con codifica `utf-8-sig`, utile per gestire anche file CSV con BOM, e con `newline=""`, modalità raccomandata per il modulo `csv`.

La funzione genera `ValueError` se non trova frasi o documenti, oppure se il campione dei progetti è vuoto. Errori di file o di struttura, come `OSError` e `KeyError`, sono gestiti dal blocco `try` dell'interfaccia.

### `recognize_projects(documents, projects, model)`

Chiede al modello Anthropic di associare ogni documento a zero, uno o più progetti del campione. Crea un client con `ANTHROPIC_API_KEY` e manda una richiesta con:

- un messaggio di sistema che specifica il compito, il formato JSON richiesto e come trattare contenuti ambigui o istruzioni eventualmente presenti nei documenti;
- un messaggio utente che contiene in JSON l'elenco dei progetti e quello dei documenti, incluso il loro testo.

La richiesta ammette attribuzioni a più progetti quando il contenuto le supporta. Se l'attribuzione non è chiara, chiede `project_ids` vuoto e `uncertain=true`. Il modello deve restituire solo un oggetto JSON con una lista `results`.

La risposta testuale viene passata a `extract_json()`. La funzione costruisce quindi l'insieme dei codici progetto validi e indicizza i risultati del modello per `document_id`. Per ciascun documento di input:

- conserva soltanto codici progetto presenti nel campione;
- imposta l'incertezza secondo il risultato del modello (se il campo manca, considera incerta un'attribuzione senza progetti);
- se l'esito è incerto, svuota comunque la lista dei progetti;
- aggiunge valori predefiniti di confidenza e motivazione se il modello non ha restituito quei campi.

Restituisce una voce normalizzata per ogni documento, anche se il modello ne ha omesso qualcuno. Questa normalizzazione limita gli ID progetto accettati, ma l'utente può comunque rivedere le assegnazioni nell'interfaccia.

### `retrieve(claim, documents, limit=5)`

Seleziona i documenti da sottoporre al controllo di una singola affermazione. Per la frase e per ogni documento (testo più percorso) calcola i token con `tokens()` e ne conta le occorrenze con `Counter`.

Il punteggio di un documento somma, per ogni parola della frase, il minore tra la frequenza della parola nella frase e quella nel documento, moltiplicato per un peso che cresce con la lunghezza della parola. In questo modo una parola distintiva più lunga pesa più di una parola breve, mentre ripetizioni eccessive nel documento non aumentano senza limite il contributo rispetto alla frase.

I documenti con punteggio zero non vengono inclusi. Gli altri sono ordinati per punteggio decrescente e, a parità, per ID crescente. La funzione restituisce al massimo `limit` documenti (cinque per impostazione predefinita), senza includere i punteggi. Questo è un filtro lessicale preliminare; la decisione sul significato viene demandata al modello.

### `extract_json(text)`

Ripulisce la risposta testuale del modello rimuovendo eventuali delimitatori Markdown come ```` ```json ```` e ```` ``` ````. Cerca poi il primo `{` e l'ultimo `}` e prova a interpretare il contenuto fra questi estremi con `json.loads()`.

Se non trova un oggetto o se il JSON non contiene `results` come lista, solleva `ValueError`. Gli errori di sintassi JSON possono invece arrivare come `json.JSONDecodeError`; entrambi i casi sono mostrati nell'interfaccia come risposta non leggibile. Il controllo non verifica in modo completo la correttezza semantica di tutti i campi: alcune verifiche e normalizzazioni sono svolte dalle funzioni chiamanti.

### `review_claims(claims, documents, model)`

Verifica tutte le affermazioni usando il gruppo di documenti ricevuto. Nell'applicazione, tale gruppo è stato filtrato in precedenza per includere soltanto i documenti assegnati a TV-04.

Per prima cosa chiama `retrieve()` per ogni frase e salva i documenti selezionati in `evidence`, associandoli all'ID della frase. Prepara poi un payload con l'ID e il testo di ogni affermazione e, per ognuna, i soli documenti recuperati (ID, percorso, nota e contenuto).

Invia il payload ad Anthropic con istruzioni per classificare ogni frase come supportata, parzialmente supportata, non supportata o in contraddizione; usare solo le fonti fornite; ignorare istruzioni nei documenti; riportare citazioni brevi con ID; e restituire una motivazione di una riga. La richiesta specifica anche la struttura JSON attesa.

Dopo aver decodificato la risposta:

1. indicizza i risultati per ID della frase;
2. aggiunge un esito prudenziale “non supportata”, con confidenza bassa, per le frasi prive di risultato;
3. per ciascuna frase costruisce l'insieme degli ID effettivamente forniti al modello e separa le citazioni con ID ammessi da quelle con ID non ammessi;
4. salva le citazioni non ammesse in `rejected_evidence` e conserva in `evidence` solo quelle valide;
5. se tutte le citazioni di un esito positivo vengono scartate, cambia l'esito in “non supportata”, abbassa la confidenza e annota il problema nella motivazione.

Infine aggiunge al risultato il nome del modello, i conteggi di token comunicati dall'API, la mappa `retrieval` (ID frase → ID dei documenti selezionati) e gli ID ordinati delle fonti passate alla funzione. Questi dati servono anche all'interfaccia per mostrare i dettagli del controllo.

### `export_csv(claims, results)`

Prepara un CSV scaricabile con una riga per affermazione. Le colonne sono `id`, `frase`, `esito`, `confidenza`, `motivazione`, `fonti` e `citazioni`.

Indicizza prima i risultati per ID, poi percorre le affermazioni originali per mantenere l'ordine e il testo della bozza. Gli ID fonte sono uniti con `; ` e le citazioni con ` | `. `dict.fromkeys()` evita di ripetere lo stesso ID nella colonna delle fonti preservandone l'ordine. Il contenuto viene restituito come byte UTF-8 con BOM (`utf-8-sig`), che facilita l'apertura in alcuni fogli di calcolo.

### `export_recognition_csv(documents, results)`

Genera il CSV delle attribuzioni, con le colonne `documento`, `percorso`, `progetti`, `confidenza` e `motivazione`. Per ogni documento associa il risultato tramite ID; se non ha progetti assegnati scrive “incerto”. Anche questo file è codificato in UTF-8 con BOM.

## Flusso dell'interfaccia Streamlit

### Inizializzazione e caricamento

`st.set_page_config()` imposta titolo della scheda, icona e layout largo. `st.title()` e `st.caption()` presentano il nome dell'app e la nota sulla natura assistita della revisione.

Il blocco `try` chiama `load_inputs()`. Se si verifica un errore di lettura, un dato obbligatorio è mancante o una chiave attesa non esiste, `st.error()` mostra il messaggio e `st.stop()` interrompe la pagina: senza i dati non è possibile proseguire.

Nella sidebar `st.text_input()` consente di modificare il modello per la sessione corrente. Due chiavi di `st.session_state`, `recognition` e `review`, memorizzano i risultati fra i rerun di Streamlit. Se non sono ancora presenti, vengono inizializzate a `None`.

### Passaggio 1: attribuzione ai progetti

Il pulsante “Riconosci i progetti” controlla prima che `ANTHROPIC_API_KEY` sia disponibile. Se manca mostra un errore. Altrimenti esegue `recognize_projects()` sotto uno spinner, salva il risultato nello stato della sessione e cancella un'eventuale verifica precedente, perché cambiano le fonti disponibili. Gli errori API e gli errori di decodifica vengono visualizzati con messaggi distinti.

Quando esiste un risultato, il codice crea una mappa progetto → etichetta leggibile e una mappa documento → risultato. Per ogni documento mostra un `st.multiselect()` con l'opzione “incerto” e i codici progetto. Le selezioni iniziali sono quelle proposte dal modello; l'utente può modificarle. La scelta viene riscritta nel risultato conservato in sessione: “incerto” selezionato, o nessuna selezione, produce `uncertain=True` e una lista di progetti vuota. Sotto ogni selettore viene mostrata la confidenza e la motivazione del modello.

L'app filtra quindi `all_documents` per tenere soltanto quelli il cui elenco `project_ids` contiene `TV-04`. Se l'insieme delle fonti cambia rispetto a quello registrato nell'ultima verifica, azzera `review`: una verifica svolta su fonti diverse non deve restare visibile come se fosse ancora attuale. La pagina mostra gli ID delle fonti correnti e un pulsante per scaricare il CSV di attribuzione.

### Passaggio 2: verifica delle frasi

Il pulsante “Verifica le frasi” è disabilitato quando `tv04_documents` è vuoto. Quando viene premuto, `review_claims()` riceve le frasi, solo i documenti TV-04 e il modello selezionato. La chiamata avviene sotto uno spinner; gli errori API e quelli di risposta non leggibile vengono mostrati all'utente.

Se il riconoscimento non è ancora stato eseguito, compare un messaggio che invita ad avviare il passaggio 1. Se è stato eseguito ma non c'è ancora una verifica, compare un messaggio per completare il passaggio 2.

### Presentazione e correzione dei risultati

Quando esiste `review`, la pagina mostra il modello e i token di input/output riportati dall'API. Crea una mappa dei risultati per ID frase e, per ogni affermazione, apre un `st.expander()` inizialmente espanso.

Dentro ogni pannello:

- un `st.selectbox()` mostra l'esito del modello e permette di cambiarlo; la modifica aggiorna l'oggetto in sessione;
- vengono mostrati confidenza e motivazione;
- le citazioni ammesse sono presentate con l'ID del documento;
- se non ci sono citazioni valide, un messaggio chiarisce che l'assenza di evidenza non dimostra che l'affermazione sia falsa;
- le citazioni respinte per ID non ammesso vengono indicate con un avviso;
- viene mostrato l'elenco dei documenti recuperati per quella frase.

Il pulsante finale usa `export_csv()` per scaricare `verifica_TV-04.csv`. Il CSV esporta i risultati e le citazioni; l'esito modificato nel selettore è aggiornato nell'oggetto condiviso con `review["results"]`.

## Aspetti da tenere presenti

- La prima selezione dei documenti per frase è lessicale e limitata ai primi cinque risultati; un documento rilevante senza parole in comune può non arrivare al modello.
- Le attribuzioni ai progetti e la revisione delle frasi sono proposte da un modello. L'interfaccia consente di correggere attribuzioni ed esiti, e invita a verificare le citazioni.
- Il modello riceve il testo integrale dei documenti nel passaggio di riconoscimento; nel passaggio di verifica riceve invece solo i documenti recuperati per ciascuna frase.
- La validazione delle citazioni controlla che l'ID appartenga alle fonti fornite per quella specifica frase. Non controlla automaticamente che il testo della citazione sia letteralmente presente nel documento.
- L'applicazione non salva i risultati su disco: li conserva nello stato della sessione Streamlit e consente di esportarli in CSV.
