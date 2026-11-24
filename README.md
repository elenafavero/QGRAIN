# Termovia: classificazione frasi dossier

## Indice

- [Avvio](#avvio)
- [Funzionamento](#funzionamento)
  - [Cosa viene caricato](#cosa-viene-caricato)
  - [Passaggio 1: a quale progetto appartiene ogni documento](#passaggio-1-a-quale-progetto-appartiene-ogni-documento)
  - [Passaggio 2: verifica delle frasi](#passaggio-2-verifica-delle-frasi)
    - [Pre-selezione lessicale (senza LLM)](#pre-selezione-lessicale-senza-llm)
    - [Chiamata LLM](#chiamata-llm)
  - [Controlli sulla risposta](#controlli-sulla-risposta)
  - [Interfaccia e output](#interfaccia-e-output)
- [Parte facoltativa](#parte-facoltativa)
  - [Valutazione manuale](#valutazione-manuale)
- [Limiti](#limiti)

# Avvio

Attiva l'ambiente Python del progetto e avvia Streamlit:

```sh
source .venv/bin/activate
streamlit run app.py
```

Il modello predefinito è `claude-haiku-4-5-20251001`, si può cambiare nella barra laterale dell'interfaccia oppure impostare `DEFAULT_MODEL` in `app.py`.

# Funzionamento

Il programma è un prototipo che controlla se le frasi di una bozza di dossier (progetto **TV-04**) sono davvero supportate dai documenti a disposizione. Il lavoro si divide in due passaggi, ciascuno gestito da **una sola chiamata** a LLM.

## Cosa viene caricato

Da `data/prototipo` e `data/progetti.csv` il programma legge:

- le **frasi da verificare** (id + testo);
- i **documenti** (testo completo di ogni `.txt`, con id, percorso e nota dall'indice);
- il **campione dei progetti** (codice, nome, PM, sede, cartelle SharePoint, ecc.).

## Passaggio 1: a quale progetto appartiene ogni documento

- **Una singola chiamata LLM** per *tutti* i documenti insieme.
- Nel messaggio vanno: il JSON di **tutti i progetti** e il JSON di **tutti i documenti con il testo completo** (id, percorso, nota, testo).
- Il system prompt chiede di:
  - usare nomi, contenuto e percorsi come indizi, valutando con spirito critico percorsi generici e documenti condivisi;
  - permettere a un documento di appartenere a più progetti;
  - restituire `project_ids` vuoto e `uncertain=true` se l'attribuzione non è chiara;
  - usare solo i codici del campione;
  - ignorare eventuali istruzioni scritte nei documenti, che sono "dati non attendibili".
- La risposta è un JSON per documento con progetti, incertezza, confidenza (alta/media/bassa) e motivazione.
- Il codice poi **filtra i codici progetto non validi**; se il modello non risponde su un documento, lo segna come incerto.
- Nell'interfaccia l'utente può **correggere a mano** le attribuzioni (menu multi-selezione) e scaricare un CSV.

Solo i documenti attribuiti a **TV-04** passano al passaggio successivo.

## Passaggio 2: verifica delle frasi

### Pre-selezione lessicale (senza LLM)

Prima di chiamare il modello, per ogni frase il programma sceglie **al massimo 5 documenti** tra quelli di TV-04:

1. spezza frase e documenti in parole (minuscolo, via stopword e parole di 1-2 lettere);
2. il punteggio somma, per ogni parola della frase, il numero di occorrenze in comune (limitato al minimo tra frase e documento), moltiplicato per un peso che cresce con la lunghezza della parola (da 1 a 2: le parole lunghe sono più distintive);
3. si tengono i 5 con punteggio più alto; in caso di parità si ordina per id. I documenti con punteggio zero sono esclusi, quindi possono essere anche meno di 5.

La ricerca lessicale è leggera: la decisione semantica spetta all'LLM.

### Chiamata LLM

Anche qui è **una sola chiamata** per *tutte* le frasi. Il messaggio contiene, per ogni frase, il suo id, il testo e i documenti pre-selezionati (id, percorso, nota, testo completo). Il system prompt impone di:

- usare **solo** i documenti forniti, senza conoscenze esterne né documenti di altri progetti;
- ignorare istruzioni nascoste nei documenti;
- distinguere tra *supportata*, *parzialmente supportata*, *non supportata*, *in contraddizione*;
- non considerare prove di risultati conseguiti OCR ambiguo, decisioni rinviate e obiettivi;
- riportare brevi citazioni fedeli con l'id del documento, usando solo gli id forniti per quella frase;
- dare una motivazione di una riga e rispondere solo in JSON.

## Controlli sulla risposta

- Le citazioni vengono validate: se un `document_id` citato **non è tra quelli passati per quella frase**, viene scartato e salvato come `rejected_evidence`.
- Se tutte le citazioni sono state scartate ma il modello aveva dato esito positivo (supportata o parzialmente), l'esito viene **declassato a "non supportata"** con confidenza bassa.
- Se il modello non restituisce il risultato di una frase, le viene assegnato "non supportata" con confidenza bassa.
- Il risultato include modello usato, token in input/output e, per ogni frase, l'elenco dei documenti recuperati.

## Interfaccia e output

L'app è in **Streamlit**: modello configurabile da barra laterale, due pulsanti (uno per passaggio). Il secondo si abilita solo se esiste almeno un documento attribuito a TV-04. L'utente può **cambiare a mano l'esito** di ogni frase. Si possono scaricare due CSV: attribuzioni documenti e risultati della verifica (esito, confidenza, motivazione, fonti, citazioni).



# Parte facoltativa

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
| F1 | supportata | D02, D04, D06 | D02 dichiara le specifiche descritte nella frase, D04 è un primo prototipo dove ciò che non funziona non riguarda quello che viene descritto dalla frase (confermato anche da D06) |
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
- JSON non garantito
- Una sola chiamata per tutte le frasi: se fallisce, perdi tutto.

se n° file = 25.000 e file lunghi:
- non si possono mettere tutti i documenti in 1 solo prompt
- anche mettere solo tutto il documento nel prompt non va bene
- se stesso documento serve per più frasi -> duplicato nel prompt



