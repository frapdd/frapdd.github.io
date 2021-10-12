
## Synchronization

## 2 Logical time
Ci interessa la relazione di causalità, ma è complicata quindi alleggeriamo definendo una relazione di "potenziale casualità". 
Definiamo la relazione transitiva "happens before": 

- A hb B <--> A,B sono nello stesso processo e A precede B
- A hb B <--> A,B sono in diversi processi e A = sendmsg(x) e B = recvmsg(x)

Se due eventi non sono in hb allora sono concorrenti.

### 2.1 Scalar clocks (Lamport clocks)
Con questo protocollo troviamo una condizione necessaria per la hb: A hb B --> L(A) < L(B).

Ogni processo ha un clock (variabile scalare). 

- manda un messaggio: allega come timestamp clock++
- ricevi un messaggio: aggiorna clock = MAX(timestamp messaggio, clock)

Col protocollo descritto si ottiene partial ordering, se aggiungiamo un prefisso al clock otteniamo ordine globale.

### 2.2 Vector clocks

Con questo protocollo troviamo una condizione necessaria e sufficiente per la hb: A hb B <--> V(A) < V(B). Implementiamo esattamente questa relazione.

Ogni processo mantiene un vettore di clock, uno per ogni processo attivo. In ogni posizione del vettore viene memorizzato il numero di eventi accaduti in quel processo. 

- evento locale: vett[io]++
- manda un messaggio: allega il vettore aggiornato (incrementato di uno per l'evento locale di mandare un mess)
- ricevi un messaggio: per ogni posizione (diversa dalla mia) vett[i] = max(vett[i], timestamp[i]) e poi vett[io]++

---

## 3 Mutual exclusion
> Canali affidabili

> I processi non crashano

Si tratta di un problema tipico anche nei sistemi centralizzati, qua è più complicato perché la memoria non è condivisa. Bisogna garantire: 

- Safety: al più un processo alla volta nelle sequenze critiche
- Liveness: no deadlock, no starvation
- Priorità secondo la relazione hb
    
### 3.1 Coordinator
Soluzione analoga all'implementazione per sistemi centralizzati. 
Il server centrale fornisce il token di accesso solo a un processo alla volta, è semplice da implementare ma bottleneck + single point of failure.

_Messages per entry: 2_

_Delay before entry: 2_

### 3.2 Scalar clocks
Ad ogni richiesta viene allegato un timestamp, e poi viene mandata in broadcast. Ogni processo che la riceve può fare una di queste 3 cose: 

1) non possiede la risorsa richiesta + non è interessato ad averla --> manda un ack al mittente

2) non possiede la risorsa richiesta + ha già mandato richiesta anche lui --> mette la richiesta in una coda locale ordinata per timestamp

3) possiede la richiesta --> mette la richiesta in una coda di richieste ordinata per il timestamp
    
Se un processo riceve gli ack da tutti, allora gli viene concesso l'accesso alla risorsa. 
Appena dopo aver rilasciato la risorsa, il processo manda l'ack a tutti i processi nella sua coda locale. 

_Messages per entry: 2(n-1)_

_Delay before entry: 2(n-1)_

### 3.3 Token ring
I processi sono ordinati su un anello (grafo orientato in un verso). Un token per l'accesso alla risorsa viene passato circolarmente. 

_Messages per entry: 1 to infinity_

_Delay before entry: 0 to n-1_

---

## 4 Leader election
> Nodi distinguibili tra di loro (dotati di id)

> I nodi conoscono il proprio id e quello di tutti gli altri nodi

> Canali affidabili

> Si sa chi crasha

Per eleggere i coordinatori necessari in alcuni protocolli. L'idea è che il processo attivo con l'id più alto deve essere eletto come leader alla fine del processo. 

### 4.1 Bully election 
Il protocollo consiste nel loop di questo algoritmo: 

- Quando il coordinatore non risponde più a un processo, questo inizia una nuova elezione
- All'inizio dell'elezione, il processo manda un messaggio ELECT ai processi con id maggiore del suo (candidatura)
- Se un processo riceve un messaggio ELECT, esso risponde con un ack e inizia una nuova elezione
- Se un processo non riceve nessuna risposta ai propri messaggi ELECT allora vince e manda un messaggio COORD a tutti i processi con id inferiore
    
### 4.2 Ring-based

I nodi sono organizzati con una topologia ad anello. 
Il protocollo consiste nel loop di questo algoritmo: 

- Quando il coordinatore non risponde più a un processo, questo inizia una nuova elezione
- All'inizio dell'elezione, il processo manda un messaggio ELECT con il proprio id al suo vicino nell'anello
- Se un processo riceve un messaggio ELECT, controlla se il proprio id è nella lista di id allegata al messaggio, e se manca lo aggiunge prima di inoltrare di nuovo il messaggio
- Se il processo ritrova il proprio id nella lista, allora il messaggio diventa di tipo COORD e viene ri-inoltrato attraverso l'anello
- Ogni volta che si riceve un messaggio COORD, si considera il processo con più alto id come leader

---

## 5 Global state
> Canali affidabili

> Canali FIFO

> I processi non crashano

> Grafo fortemente connesso 

Lo stato è l'unione degli stati dei singoli processi al tempo t + i messaggi in viaggio al tempo t. Nella pratica approssimiamo lo stato con un "distributed snapshot", che formalmente rappresenta un possibile stato e si implementa con il concetto di "taglio consistente". Le tecniche per catturare il global state possono essere utilizzate anche per decretare la terminazione del sistema. 

### 5.1 Distributed snapshot di Lamport
Questo algoritmo ci restituisce un taglio consistente (dimostrazione sulle slide). 
Può eseguire senza bloccare il funzionamento dei nodi. 

- Quando un processo vuole iniziare uno snapshot, manda in broadcast (ai suoi vicini nella rete) un messaggio MARKER. Dopo di che si mette in ascolto e registra i messaggi in entrata
- Alla ricezione di un messaggio MARKER, il processo chiude il canale dal quale proveniva quel messaggio (dopo aver mandato un ack), inoltra un messaggio MARKER in broadcast ai suoi vicini e si mette anch'esso a registrare i messaggi  
- Quando un processo riceve un ACK da tutti i suoi vicini, allora l'algoritmo termina e si ha un taglio consistente dello stato
    
Se si vuole usarlo per termination detection, va runnato finché dallo stato non appare che tutti hanno terminato. 
    
### 5.2 Dijkstra-Scholten's termination detection
Funziona per sistemi di "diffusing computation", cioè sistemi in cui tutti i nodi (a parte un coordinatore) sono risvegliati dallo stato idle solo per l'elaborazione di messaggi. 
La condizione di terminazione del sistema è: tutti i processi sono in idle + nessun messaggio è in viaggio. 

- Viene creato un albero aciclico dei processi attivi (ogni processo ha come "figli" i processi a cui manderà messaggi) Questo viene costruito mandando in broadcast messaggi e aggiungendo i processi che si risvegliano (se sono già accesi allora fanno già parte dell'albero)
- Quando un nodo non ha più figli e termina la computazione richiede la propria rimozione dall'albero al genitore
- Quando rimane solo la root, allora il sistema ha terminato

---
## 6 Distributed transactions
Si definisce transazione una sequenza di operazioni protetta da proprietà ACID (Atomic, Consistent, Isolated, Durable). Queste sono le classiche transazioni flat. 
Le transazioni "Nested" sono gruppi di mini-transazioni in cui la durabilità viene garantita solo a livello di gruppo (posso rollbackare le mini transazioni). 
Le transazioni si definiscono "distribuite" quando tengono in considerazione la distribuzione dei dati. 
In un sistema distribuito, l'entità che si occupa di garantire Atomicity e Concurrency si chiama "Transaction manager", mentre ad occuparsi di Consistency e Isolation è lo "Scheduler". 

### 6.1 Methods for distributed atomicity
Ci sono due approcci duali. Il primo -"private workspace"- prevede che la transazione operi in uno spazio ad hoc che viene cancellato in caso di abort (efficiente se tanti abort = pessimistico), il secondo -"writeahead log"- fa operare la transazione sui dati veri ma salva una copia dello stato precedente da restorare in caso di rollback (efficiente se pochi abort = ottimistico). 

### 6.2 Methods for distributed concurrency
Nei sistemi distribuiti, lo scheduler e il transaction manager collaborano con i data manager (uno per ogni macchina fisica) per garantire la serializzabilità delle transazioni eseguite: la sequenza deve corrispondere a qualche linearizzazione delle transazioni coinvolte. Per ottenerla, si può ricorrere a locking o timestamping, inoltre gli approcci possibili si dividono anche tra pessimistici e ottimistici.  

#### 6.2.1 2PL
Idea: tutte le transazioni acquisiscono i lock fino (growing phase) finché una transazione non rilascia uno dei suoi, da quel punto in avanti le altre transazioni possono solo rilasciare lock (shrinking phase) quando vogliono. Nella variante strict, tutti i lock vengono rilasciati nello stesso momento. Garantisce serializzabilità ma ha problemi di deadlock. Può essere implementata in maniera distribuita. 

#### 6.2.2 Timestamp
Nella variante pessimistica, ogni risorsa ha associata una coppia di variabili rT ed wT (read timestamp e write timestamp), e ad ogni transazione viene allegato un timestamp ts. 
    
- write: ts > rT && ts > wT -->  write + wT = ts
- read: ts > wT --> read + rT = max(ts, rT)
- se le condizioni per leggere/scrivere non sono rispettate, la transazione viene abortita e ricreata con un nuovo timestamp

Nella versione ottimistica,gli accessi alle risorse sono liberi, si usano i timestamp per tenere traccia degli accessi. Se al momento di commit si evidenziano dei conflitti allora abort. 
Questo metodo exploita al massimo il parallelismo ma potrebbe non essere efficiente in ambienti grandi. 

Entrambi i metodi non soffrono di deadlock. 

### 6.3 Distributed deadlocks
In un sistema distribuito non c'è nessun coordinatore che possa costruire il grafo globale delle attese. Ad ogni processo viene associato un id. 

- Un processo viene messo in attesa --> manda un messaggio PROBE a chi sta aspettando, allegando il proprio id
- Un processo riceve un messaggio PROBE --> se non è lui stesso il mittente originale, si limita ad inoltrarlo in avanti aggiungendo il proprio id alla lista, altrimenti viene rilevato un deadlock 
- Il processo con il più alto id all'interno del ciclo viene killato 
