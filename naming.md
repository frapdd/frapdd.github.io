# Naming

L'idea di base è che per vari motivi non conviene usare l'indirizzo di una risorsa per accederci, ma piuttosto un nome "location-independent" che sia quindi "opaco", cioè indipendente dall'access point. Le tecniche di naming infatti decouplano il problema del mapping nomi/entità (posso avere più access points) da quello di localizzare un entità (può cambiare posizione).

Le tecniche di "name resolution" si classificano in base al "naming schema":

## 1 Flat naming

I nomi sono semplicemente stringhe senza struttura. 

Soluzione semplice: per trovare un'indirizzo si manda in broadcast (o multicast) il nome corrispondente e le entità che possono soddisfare la richiesta rispondono. Si tratta di un sistema con evidenti problemi di scalabilità. 

### 1.1 Forwarding pointers

Quando un entità si sposta rimane rintracciabile lasciando una reference alla nuova posizione nella vecchia. Quando si vuole creare un proxy per l'entità si deve ripercorrere la catena, ma una volta recuperata la vera posizione si instaura un canale diretto. I nodi intermedi della catena ("skeleton") vengono eliminati quando nessuno fa più riferimento a loro come proxy dell'entità. 

### 1.2 Home-based

Si tratta di un approccio a 2 tier. Ogni nodo mobile si mantiene sempre reperibile direttamente da un nodo fisso di riferimento per gli altri ("home"). Quando qualcuno non riesce a risolvere il nome del nodo mobile può contattare la home e poi salvarsi il nuovo indirizzo. 

Questo sistema ha problemi di latenza (peggio se il nodo mobile migra definitivamente) e scalabilità geografica, inoltre va supportato il nodo home per tutto il tempo di vita del nodo mobile. 

### 1.3 Distributed hash table

Sfruttiamo il fatto che le hash table sono implicitamente dei naming system: definiamo una relazione biunivoca tra chiave dell'hash e stringa del nome da risolvere. Esistono diverse implementazioni note basate su "overlay networks" distribuite, noi trattiamo ad esempio quella ad anello ("chord"):

Tutti i nomi sono suddivisi tra nodi, ognuno con un id a m bit, logicamente organizzati in un anello. Per assegnare ogni item (tupla nome:indirizzo) a un nodo si calcola l'hash a m bit dell'item e si inserisce quest'ultimo nel nodo con il più piccolo id >= hash. 

Ogni query può essere risolta navigando l'anello da un nodo casuale (complessità lineare) o utilizzanod la tecnica della "finger table" per ottimizzare e ottenere una complessità logaritmica: 

Una tabella di m record viene memorizzata in ogni nodo. La riga i della tabella del nodo n contiene il nodo a cui fare riferimento per l'item con hash n+2^i. In questo modo ogni hop dovrebbe circa dimezzare la distanza dal nodo cercato. 

### 1.4 Hierarchical approach

La rete viene divisa in una gerarchia di sottodomini, in cui le foglie sono LAN.  Ogni nodo contiene una directory con tutti i nodi nel proprio dominio. Nella directory, ogni nome è associato al nodo successivo nella discesa della gerarchia, o all'indirizzo (nelle foglie). In caso di replication uno stesso nome può essere associato a due indirizzi diversi (in due domini differenti). 

Quando un nuovo nome deve essere aggiunto al sistema, l'informazione si propaga dalle foglie fino alla radice. 

La resolution di un nome parte dalle foglie (la LAN del richiedente) e risale l'albero fino a che non trova tracce del nome cercato, a quel punto segue il path. L'approccio gerarchico funziona bene se le ricerche hanno un carattere locale, ma anche nel peggiore dei casi la root deve essere attraversata solo da metà delle richieste in media. La root può essere distribuita per migliorare la performance. 

La struttura si può ottimizzare implementando un sistema di caching, ripercorrendo i nodi della risoluzione al contrario e memorizzando il  collegamento a un nodo adeguato: uno abbastanza più in alto della foglia per essere resistente a un minimo di mobilità dell'entità.

Può essere considerata una generalizzazione degli home based. 

## 2 Structured naming 

I nomi sono organizzati in un albero ("name space") di directory nodes e leaf nodes (le entità con i rispettivi indirizzi). La risoluzione di un nome consiste nella costruzione del suo path. Una entità può avere potenzialmente più path diretti (hard link) o indiretti (symbolic link: memorizzano il path assoluto per la vera risorsa). 

I name server adibiti a gestire un name space sono organizzati in una struttira gerarchica (grafo aciclico) solitamente a 3 livelli: global, administrational, managerial. I livelli più alti sono quelli più stabili, per cui si può sfruttare di più il caching. I livelli più bassi sono quelli per cui i tempi di risposta e di update sono minori (a livello globale si possono ammettere incosistenze anche di giorni).  Le assunzioni alla base di questo sistema (per renderlo efficiente, non sono strettamente necessarie): 

> I layer alti della gerarchia sono stabili
>
> Le richieste sono per lo più locali

Ci sono due approcci alla resolution di un nome: 

- Iterativo: il client contatta in ordine tutti i livelli dal più alto a scendere per risolvere i vari step del path. I server di solito supportano solo questa modalità in quanto è più semplice. 
- Ricorsivo: il client contatta solo il livello più alto, e poi ogni livello interroga autonomamemte quello sottostante (senza passare per il client). Questo approccio riduce l'overhead di comunicazione e rende più efficace il caching, perché può essere eseguito sui name server stessi, ma ha il contro di lasciare tutti i thread in attesa sui name server. 

In caso di spostamento di un host in un dominio diverso devo mantenere anche il nome vecchio per portabilità:

- Mettere il nuovo ip nel nodo vecchio: i lookup saranno efficienti ma gli update futuri no, in quanto non saranno più locali. 
- Mettere il nuovo nome nel nodo vecchio: i lookup saranno rallentati (doppio lookup) ma gli update futuri saranno efficienti. 

## 3 Attribute-based naming

A ogni nome viene associato un set di attributi, a questo punto si denotano due possibili interazioni con il naming system: 

- Lookup: classica risoluzione nome --> indirizzo, ritorna un solo risultato
- Search: ricerca a partire dal valore di alcuni degli attributi, potenzialmente ritorna più risultati

Questo sistema prende anche il nome di "directory service". 

Viene combinato spesso con structured naming, ad esempio nel caso di LDAP: 

- Ad ogni record viene associata una mappa attributo (tipizzato) --> valori. 
- Il nome di ogni record è composto dalla concatenazione di "attributo=valore" per i vari attributi. 
- I dati possono essere organizzati in una struttura ad albero (potenzialmente distribuito) per ottimizzare il lookup (specificando tutti i valori per identificare univocamente l'entità), ma per fare search devo avere un database centralizzato. 

## 4 Removing unreferenced entities 

Le soluzioni classiche sono centralizzate e prevedono di fermare tutto il sistema per individuare i nodi da eliminare, ma questo non è applicabile in un contesto distribuito, in cui non ho una visione globale. 

### 4.1 Reference counting

> Il sistema di comunicazione garantisce "esattamente un messaggio"

Ogni oggetto mantiene un contatore di quanti processi stanno attualmente utilizzando una reference a lui, e si cancella quando arriva a 0. 

Quando un processo A manda ad un processo B una reference a un oggetto X, contestualmemte A deve anche segnalare ad X questo evento, in modo che X possa incrementare il proprio contatore. Quando B non ha più bisogno della reference, allora manda un messaggio a X in modo che possa decrementare il proprio contatore. 

Questo sistema è soggetto a race condition, ma può essere fixato con un ack aggiuntivo (overhead di comunicazione). Ci sono diverse strategie per implementare l'ack risolutivo, evito i dettagli su questo e la race condition perché non credo sia molto importante. 

### 4.2 Weighted reference counting

> Il sistema di comunicazione garantisce "esattamente un messaggio"

Implementazione alternativa per evitare la race condition. Ogni oggetto mantiene 2 variabili intere inizializzate allo stesso valore (potenza di 2): la prima rimane fissa e si chiama "total weight", la seconda decrementa man mano che vengono creati proxy (condivise le reference) e si chiama "partial weight". 

Ogni volta che viene creato un proxy P per l'oggetto X (proxy di primo livello: collegamento diretto tra P ed X), il partial weight di X si dimezza: la metà del valore precedente va infatti a inizializzare il partial weight di P (i proxy hanno solo partial weight). Ogni volta che P genera un ulteriore proxy, la procedura si ripete analogamente. Quando un proxy viene rimosso, il suo peso viene ri-sommato al partial weight di chi l'ha creato così che, quando si vede che partial weight e total weight sono uguali, si può essere sicuri che l'oggetto può essere rimosso. 

Questo approccio riduce di molto la comunicazione: non bisogna più informare l'oggetto originale nel momento in cui vengono passate le reference. Tuttavia, in teoria limita il numero di referenze al valore iniziale del total weight. Quest'ultima cosa si può fixare facilmente con una tecnica chiamata "indirection": quando un proxy arriva ad avere partial weight 1, crea un nuovo oggetto vuoto (essendo un oggetto e non un proxy avrà un total weight) da condividere. 

### 4.3 Reference listing

L'idea è che ogni oggetto mantiene un set (nel senso di python) degli id dei propri proxy. Questo approccio rende i messaggi di update all'oggetto originale idempotenti (invece che "incrementa il contatore delle reference" ora è "aggiungi il proxy x al set"), e questo rende non più necessaria l'assumption sulla garanzia di "esattamente un messaggio", anche solo "almeno un messaggio" va bene. 

Per essere resistente ad eventuali crash dei processi, l'oggetto originale pinga periodicamente i propri proxy. 

Questo sistema non si accorge di un set circolare irraggiungibile di reference, in quanto gli oggetti al suo interno avranno tutti un set non vuoto. 

### 4.4 Distributed mark&sweep

Dobbiamo considerare un sistema distribuito in cui ogni "location" gestisce un sottoinsieme dei nodi e runna un garbage collector locale che segue questo algoritmo (qua in seguito riassunto e semplificato):

1) Tutte le entità locali (proxy e oggetti) vengono inizialmente settate come bianche
2) Un oggetto direttamente raggiungibile dalla root viene settato grigio, e insieme ad esso tutti i proxy che contiene
3) Quando un proxy viene settato come grigio, esso manda un messaggio al proprio oggetto settandolo come grigio (e si ripete come al punto 2)
4) Quando un oggetto grigio manda un ack, allora viene settato come nero. 

Non appena tutte le entità locali sono bianche o nere, allora parte l'eliminazione delle bianche. 

Per implementare questo algoritmo, il grafo deve rimanere stabile durante il processo, e per questo l'intero sistema deve essere temporaneamente bloccato (problema di scalability) embeddando tutto l'algoritmo in una distributed transaction. 



