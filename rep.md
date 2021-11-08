# Replication

L'obiettivo principale di tutta questa parte è garantire consistenza tra le varie repliche dei dati limitando l'overhead di comunicazione necessaria per ottenerla e la degradazione della performance. 
Il tutto si può riassumere con: idealmente ogni read dovrebbe ritornare il valore dell'ultima write. 

Il "consistency model" (implementato da un "consistency protocol) è un l'insieme delle garanzie offerte dal distributed file system (contratto):
- Garanzie sul contenuto: massima differenza del contenuto di diverse repliche
- Garanzie sull'ordine degli update: routine adottate in caso di conflitti
- Garanzie sulla latenza: bound sui tempi prima della propagazione di un evento su tutte le repliche 

Il CAP theorem applicato ai sistemi distribuiti ci dice che, in presenza di fail della rete, puoi avere availability o consistency ma non entrambe. Un sistema si definisce altamente available se non richiede comunicazione sincrona/bloccante tra le repliche per offrire il servizio: anche se c'è qualche problema di rete, un utente può comunque continuare a interagire con almeno una delle repliche. 

## 1 Single leader protocols

Si tratta di sistemi in cui esiste un'entità di riferimento incaricata di garantire la consistenza di più repliche. 

La versione più basic è quella di semplice backup: il leader propaga i cambiamenti alle repliche. Se le scritture avvengono in modo sincrono (cioè si aspetta un ack dalle repliche per ritornare la write) allora garantisce fault-tolerance. Le repliche sono totalmente passive, non possono gestire richieste dell'utente (no sharing dello workload).

I protocolli single leader veri e propri fanno passare tutte le write attraverso il leader (come con il backup semplice) ma permettono di leggere da qualsiasi replica oltre che dal leader. A seconda dell'implementazione della propagazione della scrittura, il livello di consistenza garantito è più o meno alto. Si può variare tra:

1) Sincrono: grosso overhead di comunicazione ma massima consistenza

2) Asincrono: opposto del primo 

Il trade off è il modello semi-sincrono: si decide di aspettare l'ack di K repliche  si decide di aspettare l'ack di K repliche.

I protocolli a singolo leader sono spesso adottati per database distribuiti in contesti ristretti (singolo data center) che permettono l'implementazione sincrona. Adatti a database relazionali che sono molto read-intensive. 

## 2 Multi leader protocols

L'idea di base è che le scritture possono essere fatte su diverse repliche in maniera concorrente, e poi ci si occupa dei conflitti. L'operazione in se è complessa ma ci sono un sacco di casi in cui i conflitti potenziali sono pochi e facili da risolvere.

Sono utili nel caso di sistemi molto distribuiti geograficamente in cui contattare un leader centrale potrebbe essere molto costoso per gli utenti. Vengono affiancati ai single leader per la comunicazione tra diversi data centers. 

Anche in questo caso l'utente che vuole scrivere manda una sola richiesta, scegliendo la replica a lui più conveniente. 

## 3 Leaderless protocols

Variante in cui l'utente manda la richiesta di read o write a più repliche alla volta. 

Per evitare i conflitti si implementa una sorta di sistema di votazione: 

- Write: l'utente chiede l'operazione alla maggioranza di repliche, così che poi queste possano concordare sul valore da scrivere (non è stato spiegato chiaramente a lezione)
- Read: l'utente confronta le varie risposte che ottiene dalle repliche e decide. Se si accorge che alcune repliche hanno ancora il valore vecchio, può propagare l'update (propagazione di update "read repair"). Questa strategia, se adottata, rinforza la consistenza maggiormente quando il load è alto. 

Possono essere abbinari a un "anti-entropy process" per la propagazione degli update: si tratta di un processo che runna sempre in background e controlla se ci sono differenze tra le copie. Nel caso, fa in modo di correggerle mettendo in comunicazione le repliche tra loro. Nella pratica, questa tecnica è implementata con check periodici e abbinata a read repair su ogni operazione. 

Al contrario degli altri protocolli, qua la replicazione non è trasparente all'utente, che ora ha un ruolo attivo nel protocollo. 

## 4 Data-centric consistency models

> I client sono sticky: interagiscono sempre con la stessa replica

Idealmente tutte le operazioni vengono completate in un singolo istante in un ordine globale corrispondente all'ordine di creazione di ognuna. Considerata l'assenza di un clock condiviso, vengono implementate misure di coordinazione per garantire che le operazioni concorrenti vengano eseguite in un qualche ordine globale. 

### 4.1 Sequential consistency

Tutte le operazioni appaiono nello stesso ordine globale a tutti i processi. 

A livello di programmazione, deve essere implementata esplicitamente con blocchi sincronizzati.

A livello di database viene implementata spesso con protocolli single leader. La necessità di sincronizzazione che segue (il leader che propaga il cambiamento) rende impossibile avere sequential consistency insieme ad high availability (implementazioni bloccanti). 

Esistono anche implementazioni leaderless. In queste definiamo due quorum NR (per le read) e NW (per le write) con i constraint: 

- Anti read-write: NW + NR > N (questo implica che dividendo gli N processi in due set, la loro intersezione è non vuota)
- Anti write-write: NW > N/2

In questo modo le operazioni sono sicure: 

- Read: contatti NR nodi e di sicuro almeno uno di essi avrà la versione più aggiornata (primo constraint)
- Write: ogni processo deve contattare più della metà dei nodi, quindi due processi concorrenti scriveranno di sicuro almeno su un elemento in comune che sa chi è arrivato prima ed è determinante nella decisione sul version number

Questo modello non prende in considerazione il tempo fisico ma solo quello logico: se una write si sta ancora propagando potrei leggere un valore vecchio. 

### 4.2 Linearizability / atomic consistency 

Ogni operazione deve apparire come istantanea in qualche momento (valore del suo timestamp) tra il suo inizio (quando viene richiesta dal client) e la sua fine (quando è memorizzata in ogni replica). A differenza di sequential, viene preso in considerazione il tempo fisico. 

Può essere implementata con protocolli single leader:

- Write: vengono ordinate dal leader
- Read: le repliche vengono aggiornate in modo sincrono quindi sono consistenti se si aggiunge un locking protocol per evitare di leggere mentre una write si sta propagando. 

### 4.3 Causal consistency

Le write che sono potenzialmente "causally related" devono essere viste da tutti processi nello stesso ordine. Tutte le operazioni "concorrenti" possono essere viste in qualsiasi ordine (non è una relazione di ordine totale). 

La relazione causale è definita così:

- Write: legata a ogni operazione precedente nello stesso processo, anche se su variabili differenti
- Read: legata alla write precedente nello stesso processo, sulla stessa variabile 

Nella pratica viene implementato con protocolli multi-leader in cui ad ogni write viene allegato un vector clock come timestamp: definisce quale era la situazione quando è stata creata la write. 

Quando un processo riceve un write, non la applica finché non è arrivato alla situazione descritta dal vector clock. Questo rende l'implementazione highly-available: le read ritorneranno valori vecchi, le write verranno salvate e poi propagate facilmente in quanto concorrenti a quelle avvenute nel frattempo nel resto del mondo. Tutto ciò fa forte affidamento sull'assumption che i client sono sticky. 

### 4.4 FIFO consistency

Le write di un determinato processo vengono viste da tutti nell'ordine in cui sono state create dal processo. Le write di processi diversi sono concorrenti. 

Nella pratica si allega a ogni write un sequence number e ogni replica non applica una write finché non ha ricevuto tutte quelle con il numero inferiore. Questo tuttavia rende necessario rendere visibili tutte le write a tutti i processi, mentre ci sono situazioni in cui le vorremmo nascondere (quelle interne a una transazione). Bisogna sempre considerare quanta libertà si lascia al developer rispetto al momento in cui sincronizzare. 

### 4.5 Eventual consistency

> Nessun update simultaneo
>
> Prevalenza di read 

Si garantisce solo che le write verranno propagate prima o poi a tutte le repliche. Nelle implementazioni si usano tipi di dato conflict-free che permettono solo modifiche commutative (idempotenti) o con sistemi some "last write wins": tutte le operazioni hanno un id random che viene usato in caso di conflitti, facendo vincere l'operazione con l'id maggiore (garantisce convergenza). 

Può essere rinforzata ("strong eventual consistency") imponendo che, a prescindere dall'ordine in cui vengono ricevuti i messaggi, tutti i processi arriveranno allo stesso stato. 

All'interno dei data centers viene usata linearizability, ma tra di essi si impone solo eventual consistency. 

## 5 Client-centric consistency models

Trattano i contesti in cui non si può assumere che i client siano sticky. Offrono garanzie dal punto di vista del singolo client. L'idea di base è simile a quella di git: ogni client lavora con la propria copia dei dati, e al momento della sincronizzazione contratta con le altre repliche. 

Questi modelli vengono implementati mantenendo per ogni client, per ogni variabile un read set e uno write set, che loggano le operazioni rilevanti (a ognuna viene associato un id incrementale). Queste informazioni vengono poi utilizzate in maniera diversa a seconda del modello di consistenza che si vuole imporre. 

Attenzione: le seguenti speifiche devono essere applicate a livello di processo, non di replica (client centric). 

### 5.1 Monotonic reads

Una volta che un processo legge il valore di una variabile, le read successive non potranno restituire un valore precedente a quello già osservato. 

Prima di leggere un valore controllo la sua versione confrontandola con il mio read set. 

Attenzione: in questo modello non viene considerata la monotonicità delle write, devo solo stare attento di non leggere nuovi valori prima di aver aggiornato la copia con i valori vecchi. 

### 5.2 Monotonic writes

Essenzialmente questo è un constraint sul propagare ogni write su tutte le repliche prima di farne un'altra. 

Prima di scrivere un valore controllo la sua versione confrontandola con il mio write set. 

### 5.3 Read your writes

Una volta che modifico una variabile, l'effetto di questa modifica sarà visibile in tutte le letture successive. 

Anche questo modello forza la propagazione delle write ma attraverso una relazione con le read. 

### 5.4 Writes follow reads

Una write che avviene dopo una read deve modificare a partire dal valore di quella read (o da uno più recente). 

Prima di scrivere un valore controllo la sua versione confrontandola con il mio read set, aspetto che sia corretta e poi aggiungo la write al write set. 

## 6 Design strategies

Questa sezione si occupa di alcune scelte di design pratiche necessarie nella progettazione di un database distribuito. 

### 6.1 Replica placement

Ci sono 3 possibilità:

- Fisse
- Server-initiated: tipico dei servisti di hosting, in cui il sistema crea le repliche minime necessarie a mantenere le performance accordate, adattandosi dinamicamente al load dell'applicazione. Quando è necessario sposta le repliche sempre più vicino al client. 
- Client-initiated: le cache locali dei client permettono di non dover continuamente richiedere le stesse informazioni al server nei contesti in cui gli update non sono frequenti. 

### 6.2 Update propagation

Si può scegliere tra inviare alle repliche i dati modificati da ogni update oppure le operazioni da svolgere per ottenere il risultato (active replication). 

Trade-off tra inviare dati (comunicazione pesante ma i riceventi non devono fare nulla) e inviare operazioni (comunicazione leggera ma i riceventi devono essere attivi). 

Si può anche considerare la possibiltà di propagare solo notifiche per condividere le informazioni sulle operazioni effettuate ma senza propagare le stesse in alcun modo finché non è necessario. Funziona bene quando hai molte write e poche read. 

### 6.3 Propagation techniques 

Ci sono 2 approcci: 

- Push-based: si propaga tutto a tutte le repliche, anche se non gli servono. Si tratta di una misura conservativa. 
- Pull-based: gli update vengono richiesti dal client quando necessario, tipico delle cache. 


